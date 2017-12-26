## Lua指令执行过程
有了上一节Lua栈的基础,这节讲解Lua指令执行过程中涉及到的关键数据结构,流程以及相应的函数.

存放分析Lua代码之后生成相关的opcode,是存放在Proto结构体中,一个Lua文件有一个总的Proto结构体,如果它内部还有定义函数,那么每个函数也有一个对应的Proto结构体.这里仅列列举出与指令执行相关的几个成员:

* TValue *k:存放常量的数组.
* Instruction *code:存放指令的数组.
* struct Proto **p:该Proto内定义的函数相关的Proto数据.

Proto结构体是分析完一个Lua文件之后的产物,所以需要在分析阶段需要有中间使用到的临时数据用来保存它,向它里面写入数据,待分析完毕之后,这个在分析过程中临时使用的数据就不再使用了,这个结构体就是FuncState,这里也仅是列举出它里面与指令相关的成员:

* Proto *f:绑定的Proto结构体.
* int pc:当前的pc索引,类似于CPU执行指令时的计数器,在这里这个变量用于保存当前已经向Proto的code数组写入了多少数据.
* int freereg:当前的可用寄存器索引.

有了以上的准备,来看看何时向Proto中写入指令.Lua代码中对不同的指令格式提供了几个函数,luaK_codeABx/luaK_codeABC等,这几个函数最终都会调用函数luaK_code:

```C
(lcode.c)
797 static int luaK_code (FuncState *fs, Instruction i, int line) {
798   Proto *f = fs->f;
803   f->code[fs->pc] = i;
808   return fs->pc++;
809 }
```

如前面分析的那样,luaK_code中将指令写入Proto结构体的code数组中,返回code数组中下一个可写的位置,这个用FuncState的pc成员保存.

接下来看看Lua虚拟机真正执行每一条指令的函数luaV_execute:

```C
(lvm.c)
343 #define RA(i)   (base+GETARG_A(i))
344 /* to be used after possible stack reallocation */
345 #define RB(i)   check_exp(getBMode(GET_OPCODE(i)) == OpArgR, base+GETARG_B(i))
346 #define RC(i)   check_exp(getCMode(GET_OPCODE(i)) == OpArgR, base+GETARG_C(i))
347 #define RKB(i)  check_exp(getBMode(GET_OPCODE(i)) == OpArgK, \
348     ISK(GETARG_B(i)) ? k+INDEXK(GETARG_B(i)) : base+GETARG_B(i))
349 #define RKC(i)  check_exp(getCMode(GET_OPCODE(i)) == OpArgK, \
350     ISK(GETARG_C(i)) ? k+INDEXK(GETARG_C(i)) : base+GETARG_C(i))
351 #define KBx(i)  check_exp(getBMode(GET_OPCODE(i)) == OpArgK, k+GETARG_Bx(i))
	
373 void luaV_execute (lua_State *L, int nexeccalls) {
374   LClosure *cl;
375   StkId base;
376   TValue *k;
377   const Instruction *pc;
378  reentry:  /* entry point */
380   pc = L->savedpc;
381   cl = &clvalue(L->ci->func)->l;
382   base = L->base;
383   k = cl->p->k;
384   /* main loop of interpreter */
385   for (;;) {
386     const Instruction i = *pc++;
387     StkId ra;
398     ra = RA(i);
402     switch (GET_OPCODE(i)) {
760     }
761   }
762 }
```

首先注意到,在函数luaV_execute中,变量pc来自于lua_State的savedpc,它是在前面出现过的luaD_precall函数中进行赋值的,只不过在那个时候仅关注函数栈的相关变量,这里把这行代码列举出来:

```C
(ldo.c)
264 int luaD_precall (lua_State *L, StkId func, int nresults) {
293     L->savedpc = p->code;  /* starting point */
```

其次看到,这个函数的主体是一个大循环,依次取出pc指针指向的指令拿出来做解析执行,而需要注意到,RA/RB/RC等几个宏,其取数据都是以base也就是函数栈的起始地址来获取的,于是如果把上一节函数栈的内容和这一节的内容结合起来,那么可以得到一个显然的结论:

> 函数栈空间,也就是函数的执行环境是在函数开始执行之前创建好的,而在解析指令时并不知道将来指令执行时所在的具体位置,当时只能知道的相对于函数栈的相对位置.因此Lua指令中存放的地址都是相对于函数栈的偏移量,而执行时的函数栈环境,存放在base/top成员中.

虽然指令中存放的是相对偏移量,但是也需要保存一个值,知道当前函数栈中有多少值被分配出去了,存放这个数据的变量,就是FuncState结构体的freereg.

于是图3.1中最右边表示R[0]...R[n]的意义也就明白了.









	

 
 





