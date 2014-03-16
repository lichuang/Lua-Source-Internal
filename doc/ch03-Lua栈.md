#Lua栈
上一篇中,已经将Lua代码从分析到执行的大致流程分析了一遍,但仅有这些,还不足以了解Lua的运作机制,这里要分析一下Lua中的另外一个重要概念:栈.

Lua的栈在Lua中扮演着一个非常重要的中间层的角色.如果说Lua虚拟机模拟的是CPU的运作,那么Lua栈模拟的就是内存的角色.在Lua内部,参数的传递是通过Lua栈,同时Lua与C等外部进行交互的时候也是使用的栈.这些暂时不展开讨论,先关注的是Lua栈的分配,管理和相关的数据结构.

Lua虚拟机在初始化创建lua_State结构体时,会走到stack_init函数中,这个函数主要就是对Lua栈和CallInfo数组的初始化:

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
 	
可以看到的是,在这个函数中初始化了两个数组,分别保存Lua栈和CallInfo结构体数组.
其中,与Lua栈相关的lua_State结构体成员变量有base,stack,top,stack保存的是数组的初始位置,base会根据每次函数调用的情况发生变化,top指针指向的是当前第一个可用的栈位置,每次向栈中增加/删减元素都要对应的增减top指针.

CallInfo结构体,是每次有函数调用时都会去初始化的一个结构体,它的成员变量中,也有top,base指针,同样的是指向Lua栈的位置,所不同的是,它关注的仅是函数调用时的相关位置.从代码中可以看出,CallInfo数组是有限制的,换言之,在Lua中的嵌套函数调用层次也是有限制,不能超过一定数量.

可以看到,在stack_init函数调用完毕之后,将有如下的结果:

1. 创建了一个大小为ASIC_STACK_SIZE + EXTRA_STACK,数据类型为TValue的数组,这个数组就是Lua栈的所在.其中,lua_State中的成员stack指针指向该数组的首地址,另一个成员top指针指向下一个可用的栈位置



