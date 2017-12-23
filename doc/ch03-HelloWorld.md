# Hello World

有了前面的基础,现在我们来简单分析一下最简单的打印"Hellow World"语句,究竟需要哪些步骤,以此作为后面分析Lua解释器指令的热身.

简单来说,Lua虚拟机解释执行一个Lua脚本文件,需要经过以下几个步骤:

1. 初始化Lua虚拟机数据结构.
2. 读取Lua脚本文件内容.
3. 依次对Lua脚本文件进行词法分析、语法分析、语义分析,最后生成该文件的Lua虚拟机指令.注意以上的过程仅需要一次遍历,这是Lua解释器做的非常好的地方.
4. 进入Lua虚拟机主循环,将第3步生成的指令取出来逐个执行.

我们接下来就以一个最简单的Lua代码:
	
	print("Hello World")

来解释前面的步骤是如何进行的.

## 实验用C代码
我在自己研究分析Lua解释器原理的时候,由于需要分不同的Lua指令来进行分析,所以我写了一个简单的C代码,根据命令行参数来读取不同的Lua文件来执行,然后我可以gdb来调试这个程序,于是就能较为方便的研究Lua解释器的工作原理.该代码如下:

```C
#include <stdio.h>
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>

int main(int argc, char *argv[]) {
  	char *file = NULL;
  		
  	if (argc == 1) {
    	file = "my.lua";
  	} else {
    	file = argv[1];
  	}

  	lua_State *L = lua_open();
  	luaL_openlibs(L);
  	luaL_dofile(L, file);

  	return 0;
}
```
	
假设上面的C代码编译生成了test,而前面的那个打印"Hello World"的Lua脚本文件名为"hello.lua",那么执行:

	./test hello.lua
就可以解释执行出hello.lua中的代码,通过使用gdb等调试工具,就可以来进一步观察分析Lua解释器的行为.现在以及以后对Lua指令的分析,就是通过这个程序执行不同的Lua脚本文件来进行分析的.

## Lua虚拟机数据结构
在开始下面的分析之前,首先来介绍一下两个重要的数据结构:lua_State以及global\_State.

lua\_State可以认为是每一个Lua"线程"所独有的一份数据,后面介绍到Lua协程时会明白这里所谓的"线程"是什么意思.

将其中各个成员变量分类介绍以含义如下:

### Lua栈相关

* StkId top:当前Lua栈顶,见Lua栈部分讲解
* StkId base:当前Lua栈底,见Lua栈部分讲解
* StkId stack_last:指向Lua栈最后位置
* StkId stack:指向Lua栈起始位置
* int stacksize:Lua栈大小

### Lua CallInfo数组相关
前面提过,每次函数调用都对应一个CallInfo结构体,

* CallInfo *ci:指向当前函数的CallInfo数据指针
* CallInfo *end\_ci:指向CallInfo数组的结束位置
* CallInfo *base\_ci:指向CallInfo数组的起始位置
* int size\_ci:CallInfo数组的大小

### hook相关

* lu\_byte hookmask:hook mask位,分别有以下几种值:LUA_MASKCALL
* lu\_byte allowhook;
* int basehookcount;
* int hookcount;
* lua_Hook hook;
  
### GC相关
* TValue l_gt
* GCObject *openupval;  /* list of open upvalues in this stack */
* GCObject *gclist;
  
### 其他
* CommonHeader:Lua通用数据相关,见第二章
* lu_byte status:当前状态
* global_State l_G:指向全局状态指针,见下面关于global_State的讲解
* const Instruction *savedpc:当前函数的pc指针
* unsigned short nCcalls:记录C调用层数
* unsigned short baseCcalls:恢复一个协程时的C调用层数
* struct lua_longjmp *errorJmp;  /* current error recover point */
* ptrdiff_t errfunc;  /* current error handling function (stack index) */


global\_State是一个进程独有的数据结构,它其中的很多数据会被该进程中所有的lua\_State所共享.换言之,一个进程只会有一个global\_State,但是却可能有多份lua\_State,它们之间是一对多的关系.
 
## 初始化Lua虚拟机指针
首先来看第一步,调用lua\_open函数创建并且初始化一个Lua虚拟机指针,来看看这里具体做了什么事情.

lua\_open实际上是一个宏,其最终会调用函数luaL\_newstate来创建一个Lua\_State指针,这里主要完成的是lua_State结构体的初始化及其成员变量以及global\_State结构体的初始化工作.


## 读取脚本文件
初始化完毕lua\_State结构体之后,下一步就是读取Lua脚本文件,进行词法分析->语法分析->语义分析,最后生成Lua虚拟机指令.这一步的入口是调用luaL\_dofile,这也是一个宏:

```
	(lauxlib.h)
	111 #define luaL_dofile(L, fn) \
	112   (luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))
```

可以看到,这个宏包括了两个函数调用,首先调用luaL\_loadfile函数,这个函数主要是对Lua脚本进行上述所说的分析,而只有在这个函数返回0也就是调用成功的时候,才会调用lua\_pcall函数,这个函数将会根据上一步成功分析完毕之后生成的Lua虚拟机指令来执行.下面依次来看看这几个过程中涉及到的重要函数和数据结构.


### 分析Lua脚本文件
这一步涵盖了词法分析,语法分析以及语义分析,Lua解释器的一个特点这几个过程是一遍遍历完成的,因此Lua在这方面的效率很高,但是这也给分析这部分代码带来一定难度.
这部分最终的入口函数是f_parser函数,这里首先不具体深入分析各个分析过程,而是先来具体看看这个函数的代码,明白这个函数执行完毕之后对外部的输出,以及相关的数据结构:

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
	
首先看到的是SParser结构体,这个结构体只会在分析过程中使用,在它内部的成员变量有已经读取进内存的Lua脚本文件内容,文件名称等数据.这里将调用luaZ_lookahead函数拿到一个int型数据,以此来判断即将进行分析的内容,是已经经过了luac预编译的二进制内容,还是Lua脚本,根据不同的情况调用不同的parser来进行分析.

第一步分析的结果,最终会返回一个Proto结构体指针,这是一个重要的数据结构,它是用来存放分析完一个Lua函数之后,包括指令,参数,Upvalue等信息,来看看其成员变量的具体含义:

* CommonHeader:Lua通用数据相关的成员,前面已经做过讲解.

* TValue *k: 存放常量的数组.

* Instruction *code:存放指令的数组.

* struct Proto **p:如果本函数中还有内部定义的函数,那么这些内部函数相关的Proto指针就放在这个数组里.

* int *lineinfo:存放对应指令的行号信息,与前面的code数组内的元素一一对应.

* struct LocVar *locvars:存放函数内局部变量的数组.

* TString **upvalues:存放UpValue的数组,至于UpValue的含义,后面会做分析.

* TString  *source:Lua脚本文件名.

* int sizeupvalues:前面upvalues数组的大小.

* int sizek:k数组的大小.

* int sizecode:code数组的大小.

* int sizelineinfo:lineinfo数组的大小.

* int sizep:p数组的大小.

* int sizelocvars:localvars数组的大小.

* int linedefined:函数body的第一行行号.

* int lastlinedefined:函数body的最后一行.

* GCObject *gclist:gc链表.

* lu_byte nups:upvalue的数量.

* lu_byte numparams:函数参数数量.

* lu_byte is_vararg:有以下几种取值

```	
	#define VARARG_HASARG 1
	#define VARARG_ISVARARG 2
	#define VARARG_NEEDSAR 4
```
	
* lu_byte maxstacksize:函数最大stack大小,由类型是lu_byte可知,最大是255.
  
由上面的分析可知,当执行Lua解释器对一个Lua脚本文件进行分析之后,最终返回的就是一个Proto结构体指针,但是需要注意的是,前面对Proto结构体的说明中提过,这个结构体与Lua函数一一对应,但是这个说法并不是特别精确,比如我们这里的测试代码打印一个"Hello World"字符串,这就不是一个函数,但是即便如此,分析完这个文件之后也会有一个对应的Proto结构体,因此需要把Lua函数的概念扩大到Lua模块,可以说一个Lua模块也是一个Lua函数,对一个Lua模块分析也是相当于对一个Lua函数进行分析,只不过这个函数没有参数,没有函数名.

继续下面的分析,返回Proto指针之后,将会创建一个Closure结构体:

	(lobject.h)
	309 typedef union Closure {
	310   CClosure c;
	311   LClosure l;
	312 } Closure;

从它的定义可知,它既可以用来存放C函数相关信息,也可以用来存放Lua函数相关信息.在这里当然是保存的Lua函数信息,主要就是要将返回的Proto信息保存下来,它将在后面Lua虚拟机执行指令时用到.

最后需要注意的是f_parser函数的最后两句代码,它将Closure指针push到了Lua栈中,同时将栈指针加1,由此可知分析完Lua脚本文件产生的Closure指针一定是在Lua栈顶.

到了这里,已经将分析Lua脚本文件的流程和其中重要的数据结构做了概述,也就是luaL_loadfile函数调用的过程,接下来就是分析如何根据这一步的结果来执行Lua虚拟机指令了.

### 虚拟机执行指令

调用luaL_loadfile并且成功返回之后,产生的Closure指针被放在了Lua栈顶,这时需要调用lua_pcall(L, 0, LUA_MULTRET, 0)执行Lua指令.这里对lua_pcall函数参数做一些解释,第一个参数自不必解释;第二个参数表示调用这个函数时的参数数量,前面已经提到过,对一个Lua文件进行分析时也可以看做是分析一个Lua函数,只不过其函数参数数量为0,因此此处传递进来的第二个参数是0;第三个参数表示的该函数返回参数的数量,调用完毕之后将根据这个参数对Lua栈进行调整,同样的由于这里是一个Lua文件,所以需要传递的是LUA_MULTRET这个常量,表示多返回值;最后一个参数表示的是出错处理函数在Lua栈中的位置,只有非0值才有意义.

lua_pcall函数最终会进入luaD_call函数来,可是在此之前,首先需要拿到所调用函数相关的Closure指针,这是如何拿到的呢?前面提到过,f_parser返回的Closure指针会放在Lua栈顶,在lua_pcall中有这么一行代码:

	(lapi.c)
	819   c.func = L->top - (nargs+1);  /* function to be called */
	
上面分析lua_pcall函数参数的时候提到过,这里传入的nargs参数是0,因为这里是分析一个Lua文件所以没有函数参数,所以这个操作就拿到了上一步中Closure指针在Lua栈中的地址.同时,从这个操作可以知道,在调用一个Lua函数时,该函数的Lua栈从下往上依次是函数Closure,函数的各个参数,这个问题后面具体分析到Lua函数调用再进行展开分析.

继续看看luaD_call函数:

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

这里首先对Lua函数调用层次做判断,接下来调用luaD_precall进行函数调用之前的一些准备工作,这里不展开讨论,简单的说其做的工作有这些:准备好Lua函数栈,创建CallInfo指针并且根据相应信息进行初始化.

到这里,准备工作已经做完了,下面就是调用luaV_execute逐个取出Lua指令来执行:

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
			/* 执行Lua指令 */
	761   } 
	762 } 

可以看到,luaV_execute函数首先根据CallInfo指针拿到LClosure指针,然后pc指针指向savedpc指针,再逐个从这个pc指针中拿出指令来逐条执行.而这个savedpc指针式在哪里进行初始化的呢,在luaD_precall函数中:

	(ldo.c)
	293     L->savedpc = p->code;  /* starting point */

上面的p就是分析Lua脚本之后生成的Proto指针,而它的code成员变量前面提到过存放的就是分析后的指令.

以上可以看出,其实Proto结构体才是Lua解释器分析Lua脚本过程中最重要的数据结构,而Closure,CallInfo结构体不过是这个过程为了提供给Lua虚拟机执行指令装载Proto结构体用的中间载体,最终用到的还是Proto结构体中得数据.

以上,将如何加载,分析,以及到最终执行Lua指令中间涉及到的重要函数,数据结构做了一些分析,碍于目前的进度,还没有对一些过程进行深入讨论,但是已经对Lua虚拟机的整体流程有了概观的了解,有了骨架,后面就是根据具体的知识点来进行深入分析了.
