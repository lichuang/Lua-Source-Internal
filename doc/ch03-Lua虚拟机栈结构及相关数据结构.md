## Lua虚拟机栈结构及相关数据结构
这节重点来介绍Lua虚拟机的结构,Lua栈的结构,以及相关的数据结构,理解本节的内容是理解后面内容的基础,但是又是与后面的内容相辅相成,所以在看到内容的时候可能需要时不时回顾本节中的内容.

### Lua的栈结构

![3.1 Lua stack](https://raw.github.com/lichuang/Lua-Source-Internal/master/pic/3.1-lua%20stack.png "3.1 Lua stack")

(3.1 Lua 栈)

图中,最左边的框中是lua\_State中三个与Lua栈相关的成员;中间是一个函数栈的结构,其最底部是函数相关的数据,紧挨着它的依次是该函数的参数;在它右边是函数寄存器数组与函数栈数据的对应关系.接下来我们将依次对它们做解释.

#### lua_State结构体
lua\_State结构体是使用Lua时接触最多的结构体,它的成员很多,但是就本节而言,我们仅关注到与Lua栈相关的几个成员,首先来看看这几个成员在哪里进行的初始化:

```
(lstate.c)
42 static void stack_init (lua_State *L1, lua_State *L) {
	// ....
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
	
可以看到,最开始创建了一个TValue的数组,由stack保存它的首地址;接下来将一个nil值压入栈中,最后base和top成员都指向stack的下一个位置(因为第一个位置已经保存了一个nil值).

这就是整个Lua虚拟机环境在初始化的时候创建的Lua栈结构,在后面这里将是实际执行Lua代码的舞台.

#### 函数栈的环境
接下来看看函数栈的环境在调用函数之前以及之后的变化情况.

前面介绍了lua\_State中的stack/top/base这三个成员,lua\_State中还有CallInfo类型的数组,这个成员中存放每次函数调用时的环境:

```
(lstate.h)
48 typedef struct CallInfo {
49   StkId base;  /* base for this function */
50   StkId func;  /* function index in the stack */
51   StkId top;  /* top for this function */
52   const Instruction *savedpc;
53   int nresults;  /* expected number of results from this function */
54   int tailcalls;  /* number of tail calls lost under this entry */
55 } CallInfo;
```

其中的nresults,tailcalls这里暂且不做解释.
 
而lua\_State中CallInfo类型数组的初始化,同样在前面stack\_init函数中:
 
```
(lstate.c)
42 static void stack_init (lua_State *L1, lua_State *L) {
43   /* initialize CallInfo array */
44   L1->base_ci = luaM_newvector(L, BASIC_CI_SIZE, CallInfo);
45   L1->ci = L1->base_ci;
46   L1->size_ci = BASIC_CI_SIZE;
47   L1->end_ci = L1->base_ci + L1->size_ci - 1;
	// ...
54   L1->ci->func = L1->top;
55   setnilvalue(L1->top++);  /* `function' entry for this `ci' */
56   L1->base = L1->ci->base = L1->top;
57   L1->ci->top = L1->top + LUA_MINSTACK;
58 }
```
 
可以看到,在初始化之后:
 
 * lua_State的ci成员指向CallInfo类型数组的第一个成员,成员ci同时也是当前在调用的函数相关的CallInfo指针,在发生函数调用的时候这个变量会随之改变.
 * ci的fun成员指向lua\_State的top指针,在赋值完毕之后向函数栈中压入了一个nil值,也就是说,第一个CallInfo数组的元素,其func位置存放的是一个nil值.
 * 最后,ci的top指针指向lua\_State的top指针之后的LUA\_MINSTACK位置.
 
有了这些准备,首先来看看在调用函数之前函数栈环境相关的操作,这里只列举出关键的代码:
 
```
(ldo.c)
264 int luaD_precall (lua_State *L, StkId func, int nresults) {
272   if (!cl->isC) {  /* Lua function? prepare its call */
288     ci = inc_ci(L);  /* now `enter' new function */
289     ci->func = func;
290     L->base = ci->base = base;
291     ci->top = L->base + p->maxstacksize;
292     lua_assert(ci->top <= L->stack_last);
296     for (st = L->top; st < ci->top; st++)
297       setnilvalue(st);
298     L->top = ci->top;
304     return PCRLUA;
305   }
```

把关键的代码挑拣出来之后,上面代码的含义就一目了然了:

* 从lua\_State的CallInfo数组中得到一个新的CallInfo结构体,设置它的func/base/top指针.
* 296-297行的作用,是把多余的函数参数赋值为nil,比如一个函数定义中需要的是两个参数,实际传入的只有一个,那么多出来的那个参数在这里会被赋值为nil.
* 将这里创建的CallInfo指针的top/base指针赋值给lua\_State结构体的top/base指针.

调用函数完毕之后,又需要根据前一个函数调用时的环境进行恢复,代码如下:

```C
(ldo.c)
342 int luaD_poscall (lua_State *L, StkId firstResult) {
343   StkId res;
344   int wanted, i;
345   CallInfo *ci;
348   ci = L->ci--;
349   res = ci->func;  /* res == final position of 1st result */
351   L->base = (ci - 1)->base;  /* restore base */
358   L->top = res;
360 }
```

代码同样很简单直白:
* 首先将当前ci-1,这样就得到调用者的CallInfo指针.
* 根据调用者指针恢复lua\_State结构体的top/base指针,注意这里省略了恢复lua\_State的top指针的具体操作,具体细节待后面讲解到函数指令时再详述.

有了这两部分的讲解,相信回头再看图3.1就会清楚不少了.




	

 
 





