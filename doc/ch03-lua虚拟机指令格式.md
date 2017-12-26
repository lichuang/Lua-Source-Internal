# Lua虚拟机指令格式

Lua虚拟机的指令格式如下图所示：

![OpCode](https://raw.github.com/lichuang/Lua-Source-Internal/master/pic/opcode.png "OpCode")

我们来对上图做一些解释和分析。

首先能看到的时，Lua的指令是32位的，由低位到高位来进行解释，首先解析最低6位的OpCode，再根据OpCode获取该指令的具体格式，因此Lua最多支持2^6=64个指令。Lua代码中，将每个OpCode及其对应的指令格式都在lopcodes.h中的OpCode枚举类型中定义：

```C
	(lopcodes.h)
	150 typedef enum {
	151 /*----------------------------------------------------------------------
	152 name    args  description
	153 ------------------------------------------------------------------------*/
	154 OP_MOVE,/*  A B R(A) := R(B)          */
	155 OP_LOADK,/* A Bx  R(A) := Kst(Bx)         */
	156 OP_LOADBOOL,/*  A B C R(A) := (Bool)B; if (C) pc++      */
	157 OP_LOADNIL,/* A B R(A) := ... := R(B) := nil      */
	158 OP_GETUPVAL,/*  A B R(A) := UpValue[B]        */
	159
	/* ... */
	208 } OpCode;
```

每个OpCode右边紧随的注释说明了该OpCode的具体格式。

可是仅有注释里面写明的每个OpCode的格式还不够，因为这起不到程序上的约束和说明作用，于是Lua代码中使用了一个数组定义所有OpCode的一些说明信息：

```C
	(lopcodes.c)
	59 #define opmode(t,a,b,c,m) (((t)<<7) | ((a)<<6) | ((b)<<4) | ((c)<<2) | (m))
	60
	61 const lu_byte luaP_opmodes[NUM_OPCODES] = {
	62 /*       T  A    B       C     mode        opcode   */
	63   opmode(0, 1, OpArgR, OpArgN, iABC)        /* OP_MOVE */
	64  ,opmode(0, 1, OpArgK, OpArgN, iABx)        /* OP_LOADK */
```

这里用一个宏opmode封装了每个OpCode的具体格式，其中：

* T:表示这是不是一条逻辑测试相关的指令，这种指令可能会将PC指针自增1。
* A:表示这个指令会不会赋值给R(A)。
* B/C:B/C参数的格式。
* mode:这个OpCode的格式。

B/C参数的格式如下：

```C
	(lopcodes.h)
	245 enum OpArgMask {
	246   OpArgN,  /* argument is not used */
	247   OpArgU,  /* argument is used */
	248   OpArgR,  /* argument is a register or a jump offset */
	249   OpArgK   /* argument is a constant or register/constant */
	250 };
```

OpArgN表示这个参数没有被使用，但是这里的意思并不是真的没有被使用，只是没有作为R()或者RK()宏的参数使用。

从图中可以看出,Lua共有三种指令格式:iABC,iABx,iAsBx。

Lua代码中提供了根据指令中的值得到相应数据的几个宏,

```
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
```

RA/RB/RC自不必多做解释，前面讲解Lua指令执行的时候已经说过，其含义就是以参数为偏移量在函数栈中取数据。

RKB/RKC的意思有两层，第一是这个指令格式只可能作用在opcode的B/C参数上，不会作用在参数A上；第二层意思是这个数据除了从函数栈中获取之外，还有可能从常量数组(也就是K数组)中获取,关键在于宏ISK的判断：

```
	(lopcodes.h)
 	38  #define SIZE_B      9
	118 /* this bit 1 means constant (0 means register) */
	119 #define BITRK       (1 << (SIZE_B - 1))
	120
	121 /* test whether value is a constant */
	122 #define ISK(x)      ((x) & BITRK)
```

结合起来看，这个宏的含义就很简单了：判断这个数据的第八位是不是1，如果是则认为应该从K数组中获取数据，否则就是从函数栈寄存器中获取数据。后面会结合具体的指令来解释这个格式。

宏KBx也是自解释的，它不会从函数栈中取数据了，直接从K数组中获取数据。
