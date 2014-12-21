#Lua虚拟机概述

##何为"虚拟机"?
在一门脚本语言中,总会有一个虚拟机,可是”虚拟机”是什么?简而言之,这里的”虚拟机”就是使用代码实现的用于模拟计算机运行的程序.
每一门脚本语言都会有自己定义的opcode(operation code,中文一般翻译为”操作码”),可以理解为这门程序自己定义的”汇编语言”.一般的编译型语言,比如C等,经过编译器编译之后生成的都是与当前硬件环境相匹配的汇编代码;而脚本型的语言,经过前端的处理之后,生成的就是opcode,再将该opcode放在这门语言的虚拟机中逐个执行.
可见,虚拟机是个中间层,它处于脚本语言前端和硬件之间的一个程序(有些虚拟机是作为单独的程序独立存在,而Lua由于是一门嵌入式的语言是附着在宿主环境中的).

##Lua虚拟机工作流程
有了以上的概念,下面来简单讲解在Lua中,一份Lua代码从词法分析到语法分析再到生成opcode,最后进入虚拟机执行的大体流程.

Lua的API中提供了luaL_dofile函数,它实际上是个宏,内部首先调用luaL_loadfile函数,加载Lua代码进行语法,词法分析,生成Lua虚拟机可执行的代码,再调用lua_pcall函数,执行其中的代码:

	(lauxlib.h)
	111 #define luaL_dofile(L, fn) \
	112   (luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))

前半部分调用luaL_loadfile函数对Lua代码进行词法和语法分析,后半部分调用lua_pcall将第一步中分析的结果(也就是opcode)到虚拟机中执行.

首先来看luaL_loadfile函数,暂时不深入其中研究它如何分析一个Lua代码文件,先看它最后输出了什么.它最终会调用f_parser函数,这是对一个Lua代码进行分析的入口函数:

	(ldo.c)
	490 static void f_parser (lua_State *L, void *ud) {
	491   int i;
	492   Proto *tf;
	493   Closure *cl;
	494   struct SParser *p = cast(struct SParser *, ud);
	495   int c = luaZ_lookahead(p->z);
	496   luaC_checkGC(L);
	497   tf = ((c == LUA_SIGNATURE[0]) ? luaU_undump : luaY_parser)(L, p->z,
	498                                                              &p->buff, p->name);
	499   cl = luaF_newLclosure(L, tf->nups, hvalue(gt(L)));
	500   cl->l.p = tf;
	501   for (i = 0; i < tf->nups; i++)  /* initialize eventual upvalues */
	502     cl->l.upvals[i] = luaF_newupval(L);
	503   setclvalue(L, L->top, cl);
	504   incr_top(L);
	505 } 	
	
如果暂时不深究词法分析的细节,仅看这个函数对外的输出,那么可以看到:在完成词法分析之后,返回了Proto类型的指针tf,然后将其绑定在新创建的Closure指针上,初始化UpValue,最后压入Lua栈中.

不难想像,Lua词法分析之后产生的opcode等相关数据都在这个Proto类型的结构体中.

回过头再来看lua_pcall函数是如何将产生的opcode放入虚拟机执行的.

	(lapi.c)
	805 LUA_API int lua_pcall (lua_State *L, int nargs, int nresults, int errfunc) {
 	806   struct CallS c;
 	807   int status;
 	808   ptrdiff_t func;
 	809   lua_lock(L);
 	810   api_checknelems(L, nargs+1);
 	811   checkresults(L, nargs, nresults);
 	812   if (errfunc == 0)
 	813     func = 0;
 	814   else {
 	815     StkId o = index2adr(L, errfunc);
 	816     api_checkvalidindex(L, o);
 	817     func = savestack(L, o);
 	818   }
 	819   c.func = L->top - (nargs+1);  /* function to be called */
 	820   c.nresults = nresults;
 	821   status = luaD_pcall(L, f_call, &c, savestack(L, c.func), func);
 	822   adjustresults(L, nresults);
 	823   lua_unlock(L);
 	824   return status;
 	825 }
 
 lua_pcall函数中,首先获取需要调用的函数指针:
 
  	819   c.func = L->top - (nargs+1);  /* function to be called */
  	
这里的nargs是由函数参数传入的,luaL_dofile中调用lua_pcall时这里传入的参数是0,换句话说,这里得到的函数对象指针就是在f_parser函数中最后放入Lua栈的指针.

继续往下执行,走到luaD_call函数:

	(ldo.c)
	369 void luaD_call (lua_State *L, StkId func, int nResults) {
	370   if (++L->nCcalls >= LUAI_MAXCCALLS) {
	371     if (L->nCcalls == LUAI_MAXCCALLS)
	372       luaG_runerror(L, "C stack overflow");
	373     else if (L->nCcalls >= (LUAI_MAXCCALLS + (LUAI_MAXCCALLS>>3)))
	374       luaD_throw(L, LUA_ERRERR);  /* error while handing stack error */
	375   }  
	376   if (luaD_precall(L, func, nResults) == PCRLUA)  /* is a Lua function? */
	377     luaV_execute(L, 1);  /* call it */
	378   L->nCcalls--; 
	379   luaC_checkGC(L);
	380 } 
	
该函数中的这一段代码:

	376   if (luaD_precall(L, func, nResults) == PCRLUA)  /* is a Lua function? */
	377     luaV_execute(L, 1);  /* call it */
	
将会进入luaV_execute函数,这里是虚拟机执行代码的主函数:

	(lvm.c)
	373 void luaV_execute (lua_State *L, int nexeccalls) {
	374   LClosure *cl;           
	375   StkId base;             
	376   TValue *k;              
	377   const Instruction *pc;
	378  reentry:  /* entry point */
	379   lua_assert(isLua(L->ci));
	380   pc = L->savedpc;
	381   cl = &clvalue(L->ci->func)->l; 
	382   base = L->base;  
	383   k = cl->p->k;
	384   /* main loop of interpreter */ 
	385   for (;;) {    
	386     const Instruction i = *pc++;   
	387     StkId ra;   
	388     if ((L->hookmask & (LUA_MASKLINE | LUA_MASKCOUNT)) &&
	389         (--L->hookcount == 0 || L->hookmask & LUA_MASKLINE)) { 
	390       traceexec(L, pc);
	391       if (L->status == LUA_YIELD) {  /* did hook yield? */
	392         L->savedpc = pc - 1;
	393         return;           
	394       }
	395       base = L->base;
	396     }
	397     /* warning!! several calls may realloc the stack and invalidate `ra' */
	398     ra = RA(i); 
	/* 后面是各种Opcode的处理流程 */
	
可以看到,这里的pc指针里存放的是虚拟机opcode代码,它最开始从L->savepc初始化而来,而L->savepc在luaD_precall中赋值:

	(ldo.c)
	293     L->savedpc = p->code;  /* starting point */
	
现在,大致的流程已经清楚了,我们来回顾一下整个流程:

	1) 函数f_parser中,对Lua代码文件的分析返回了Proto指针
	2) 函数luaD_precall中,将Lua_state的savepc指针指向1)中的Proto结构体的code指针
	3) 函数luaV_execute中,pc指针指向2)中的savepc指针,紧跟着就是一个大的循环体,依次取出其中的opcode进行执行.
	
因此,Lua虚拟机指令执行的两大入口函数,分别是:
	
	1) 词法/语法分析阶段的LuaY_parser,Lua为了提高效率,一遍遍历脚本文件不仅完成了词法分析,还完成了语法分析,生成的opcode存放在Proto结构体的code数组中.
	2) LuaV_execute是虚拟机执行指令阶段的入口函数,它取出第一步生成的Proto结构体中的指令执行.
	
	
![parser2vm](https://raw.github.com/lichuang/Lua-Source-Internal/master/pic/parser2vm.png "parser2vm")
	
	
可见,Proto是分析阶段的产物,在执行阶段将使用分析阶段生成的Proto来执行虚拟机指令,在分析阶段会有许多的数据结构参与其中,可它们都是临时的用于分析阶段的,或者说最终是用来辅助生成Proto结构体的.
		


