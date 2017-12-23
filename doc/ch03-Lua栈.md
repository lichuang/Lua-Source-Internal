# Lua栈

上一篇中,已经将Lua代码从分析到执行的大致流程分析了一遍,但仅有这些,还不足以了解Lua的运作机制,这里要分析一下Lua中的另外一个重要概念:栈.

Lua的栈在Lua中扮演着一个非常重要的中间层的角色.如果说Lua虚拟机模拟的是CPU的运作,那么Lua栈模拟的就是内存的角色.在Lua内部,参数的传递是通过Lua栈,同时Lua与C等外部进行交互的时候也是使用的栈.这些暂时不展开讨论,先关注的是Lua栈的分配,管理和相关的数据结构.

Lua虚拟机在初始化创建lua\_State结构体时,会走到stack\_init函数中,这个函数主要就是对Lua栈和CallInfo数组的初始化:

```
(lstate.c)
42 static void stack_init (lua_State *L1, lua_State *L) {
43   /* initialize CallInfo array */
44   L1->base_ci = luaM_newvector(L, BASIC_CI_SIZE, CallInfo);
45   L1->ci = L1->base_ci;
46   L1->size_ci = BASIC_CI_SIZE;
47   L1->end_ci = L1->base_ci + L1->size_ci - 1;
48   /* initialize stack array */
49   L1->stack = luaM_newvector(L, BASIC_STACK_SIZE + EXTRA_STACK, TValue);
50   L1->stacksize = BASIC_STACK_SIZE + EXTRA_STACK;
51   L1->top = L1->stack;
52   L1->stack_last = L1->stack+(L1->stacksize - EXTRA_STACK)-1;
53   /* initialize first ci */
54   L1->ci->func = L1->top;
55   setnilvalue(L1->top++);  /* `function' entry for this `ci' */
56   L1->base = L1->ci->base = L1->top;
57   L1->ci->top = L1->top + LUA_MINSTACK;
58 }
```
	
可以看到的是,在这个函数中初始化了两个数组,分别保存Lua栈和CallInfo结构体数组.

其中,与Lua栈相关的lua\_State结构体成员变量有base,stack,top.stack保存的是数组的初始位置,base会根据每次函数调用的情况发生变化,top指针指向的是当前第一个可用的栈位置,每次向栈中增加/删减元素都要对应的增减top指针.注意到这个数组的成员类型是TValue类型的,从前面的讨论已经知道了TValue这个数据类型是Lua中表示所有数据的通用类型.

CallInfo结构体,是每次有函数调用时都会去初始化的一个结构体,它的成员变量中,也有top,base指针,同样的是指向Lua栈的位置,所不同的是,它关注的仅是函数调用时的相关位置.从代码中可以看出,CallInfo数组是有限制的,换言之,在Lua中的嵌套函数调用层次也是有限制,不能超过一定数量.

以上两种数据结构的base指针之间的关联在于,每次将调用一个函数时,会将当前lua\_State的base指针调整为该函数相关的CallInfo指针的base指针,而在调用完毕之后将lua_State中的base指针还原回来:

```
(ldo.c)
264 int luaD_precall (lua_State *L, StkId func, int nresults) {
	....
290     L->base = ci->base = base;


342 int luaD_poscall (lua_State *L, StkId firstResult) {
	....
351   L->base = (ci - 1)->base;  /* restore base */
```
	
到目前为止,我们暂时不用去管每个CallInfo结构体的base指针是如何计算,在调用一个函数的前后过程中是如何变化的,只需要知道的是,它会影响到lua\_State的base指针,简而言之一句话,lua\_State的base指针,永远跟着当前所在函数的CallInfo指针的base指针走.

base指针是相当重要的一个数据,原因在于它的指向所在是当前运行环境,因为无论是取得当前数据的函数index2adr:

```
(lapi.c)
49 static TValue *index2adr (lua_State *L, int idx) {
50   if (idx > 0) {
51     TValue *o = L->base + (idx - 1);
52     api_check(L, idx <= L->ci->top - L->base);
53     if (o >= L->top) return cast(TValue *, luaO_nilobject);
54     else return o;
55   }
  	...
76 }
```

还是执行Lua虚拟机OpCode的函数luaV_execute:
  	
```C
(lvm.c)
343 #define RA(i) (base+GETARG_A(i))
344 /* to be used after possible stack reallocation */
345 #define RB(i) check_exp(getBMode(GET_OPCODE(i)) == OpArgR, base+GETARG_B(i))
346 #define RC(i) check_exp(getCMode(GET_OPCODE(i)) == OpArgR, base+GETARG_C(i))
347 #define RKB(i)  check_exp(getBMode(GET_OPCODE(i)) == OpArgK, \
348   ISK(GETARG_B(i)) ? k+INDEXK(GETARG_B(i)) : base+GETARG_B(i))
349 #define RKC(i)  check_exp(getCMode(GET_OPCODE(i)) == OpArgK, \
350   ISK(GETARG_C(i)) ? k+INDEXK(GETARG_C(i)) : base+GETARG_C(i))
351 #define KBx(i)  check_exp(getBMode(GET_OPCODE(i)) == OpArgK, k+GETARG_Bx(i))

373 void luaV_execute (lua_State *L, int nexeccalls) {
		...
382   base = L->base;
		...
762 } 
```
	
都是基于lua\_State的base指针的位置来读取数据的.

top指针和base指针一样,lua\_State结构体中的top指针也是随着CallInfo中top指针一起变化,其相关代码不在这里一一列举出来.

可以总结一下Lua栈的一些特点:

*	Lua栈是一个数组来模拟的数据结构,在每个lua\_State创建时就进行了初始化,其中stack指针指向该数组的起始位置,top指针会随着从Lua栈中压入/退出数据而有所增/减,而base指针则随着每次Lua调用函数的CallInfo结构体的base指针做变化.
*	每个CallInfo结构体与函数调用相关,虽然它也有自己的base/top指针,但是这两个指针还是指向lua\_State的Lua栈.
*	在每次函数调用的前后,lua\_State的base/top指针会随着该次函数调用的CallInfo指针的base/top指针做变化.
*	lua\_State的base指针是一个很重要的数据,因为读取Lua栈的数据,以及执行Lua虚拟机的OpCode时拿到的数据,都是以这个指针为基准位置来获取的.
*	综合以上3,4两点可知,其实拿到的也就是当前函数调用环境中的数据.

Lua中提供了一系列的API用于Lua栈的操作,大体可以分为以下几类:

*	向Lua栈中压入数据,这类函数的命名规律是lua\_push*,压入相应的数据到Lua栈中之后,都会自动将所操作的lua\_State的top指针递增,因为原先的位置已经被新的数据占据了,递增top指针指向Lua栈的下一个可用位置.
*	获取Lua栈的元素,这类函数的命名规律是Lua\_to*,这类函数根据传入的Lua栈索引,取出相应的数组元素,返回所需要的数据.
