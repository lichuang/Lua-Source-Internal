#Hello World
有了前面的基础,现在我们来简单分析一下最简单的打印"Hellow World"语句,究竟需要哪些步骤,以此做为后面分析Lua解释器指令的热身.

简单来说,Lua虚拟机在解释执行一个Lua脚本文件,需要经过以下几个步骤:

1. 初始化Lua虚拟机指针
2. 读取Lua脚本文件内容
3. 依次对Lua脚本文件进行词法分析,语法分析,语义分析,最后生成该文件的Lua虚拟机指令.注意以上的过程仅需要一步遍历,这是Lua解释器做的非常好的地方.
4. 进入Lua虚拟机主循环,将第1步的指令取出来逐个执行.

我们接下来就以一个最简单的Lua代码:
	
	print("Hello World")

来解释前面的步骤是如何进行的.

##实验用C代码
我在自己研究分析Lua解释器原理的时候,由于需要分不同的Lua指令来进行分析,所以我写了一个简单的C代码,根据命令行参数来读取不同的Lua文件来执行,然后我可以gdb来调试这个程序,于是就能较为方便的研究Lua解释器的工作原理.该代码如下:

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
	
假设上面的C代码编译生成了test,而前面的那个打印"Hello World"的Lua脚本文件名为"hello.lua",那么执行:

	./test hello.lua
就可以解释执行出hello.lua中的代码,通过使用gdb等调试工具,就可以来进一步观察分析Lua解释器的行为.现在以及以后对Lua指令的分析,就是通过这个程序执行不同的Lua脚本文件来进行分析的.

##Lua虚拟机数据结构
在开始下面的分析之前,首先来介绍一下两个重要的数据结构:lua_State以及global_State.

lua_State可以认为是每一个Lua"线程"所独有的一份数据,后面介绍到Lua协程时会明白这里所谓的"线程"是什么意思.

	(lstate.h)
 	97 /*  
 	98 ** `per thread' state
 	99 */
	100 struct lua_State {
	101   CommonHeader;
	102   lu_byte status;
	103   StkId top;  /* first free slot in the stack */
	104   StkId base;  /* base of current function */
	105   global_State *l_G;
	106   CallInfo *ci;  /* call info for current function */
	107   const Instruction *savedpc;  /* `savedpc' of current function */
	108   StkId stack_last;  /* last free slot in the stack */
	109   StkId stack;  /* stack base */
	110   CallInfo *end_ci;  /* points after end of ci array*/
	111   CallInfo *base_ci;  /* array of CallInfo's */
	112   int stacksize;
	113   int size_ci;  /* size of array `base_ci' */
	114   unsigned short nCcalls;  /* number of nested C calls */
	115   unsigned short baseCcalls;  /* nested C calls when resuming coroutine */
	116   lu_byte hookmask;
	117   lu_byte allowhook; 
	118   int basehookcount; 
	119   int hookcount;
	120   lua_Hook hook;
	121   TValue l_gt;  /* table of globals */
	122   TValue env;  /* temporary place for environments */
	123   GCObject *openupval;  /* list of open upvalues in this stack */
	124   GCObject *gclist;
	125   struct lua_longjmp *errorJmp;  /* current error recover point */
	126   ptrdiff_t errfunc;  /* current error handling function (stack index) */
	127 };

global_State是一个进程独有的数据结构,它其中的很多数据会被该进程中所有的lua_State所共享.换言之,一个进程只会有一个global_State,但是却可能有多份lua_State,它们之间是一对多的关系.

 	65 /*
 	66 ** `global state', shared by all threads of this state
 	67 */
 	68 typedef struct global_State {
 	69   stringtable strt;  /* hash table for strings */
 	70   lua_Alloc frealloc;  /* function to reallocate memory */
 	71   void *ud;         /* auxiliary data to `frealloc' */
 	72   lu_byte currentwhite;
 	73   lu_byte gcstate;  /* state of garbage collector */
 	74   int sweepstrgc;  /* position of sweep in `strt' */
 	75   GCObject *rootgc;  /* list of all collectable objects */
 	76   GCObject **sweepgc;  /* position of sweep in `rootgc' */
 	77   GCObject *gray;  /* list of gray objects */
 	78   GCObject *grayagain;  /* list of objects to be traversed atomically */
 	79   GCObject *weak;  /* list of weak tables (to be cleared) */
 	80   GCObject *tmudata;  /* last element of list of userdata to be GC */
 	81   Mbuffer buff;  /* temporary buffer for string concatentation */
 	82   lu_mem GCthreshold; 
 	83   lu_mem totalbytes;  /* number of bytes currently allocated */
 	84   lu_mem estimate;  /* an estimate of number of bytes actually in use */
 	85   lu_mem gcdept;  /* how much GC is `behind schedule' */
 	86   int gcpause;  /* size of pause between successive GCs */
 	87   int gcstepmul;  /* GC `granularity' */
 	88   lua_CFunction panic;  /* to be called in unprotected errors */
 	89   TValue l_registry;
 	90   struct lua_State *mainthread;
 	91   UpVal uvhead;  /* head of double-linked list of all open upvalues */
 	92   struct Table *mt[NUM_TAGS];  /* metatables for basic types */
 	93   TString *tmname[TM_N];  /* array with tag-method names */
 	94 } global_State;
 
##初始化Lua虚拟机指针
首先来看第一步,调用lua_open函数创建并且初始化一个Lua虚拟机指针,来看看这里具体做了什么事情.

lua_open实际上是一个宏,其最终会调用函数luaL_newstate来创建一个Lua_State指针.

##读取脚本文件

##词法分析

##语法分析

##语义分析

##虚拟机执行指令