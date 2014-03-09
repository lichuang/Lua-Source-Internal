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

紧跟着再来看lua_pcall函数是如何将产生的opcode放入虚拟机执行的.

lua_pcall函数中,首先获取需要调用的函数指针:



