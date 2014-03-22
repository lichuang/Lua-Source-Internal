#Lua虚拟机指令格式

Lua虚拟机的指令格式如下图所示:


![OpCode](https://raw.github.com/lichuang/Lua-Source-Internal/master/pic/opcode.png "OpCode")

我们来对上图做一些解释和分析.

首先能看到的时,Lua的指令是32位的,由低位到高位来进行解释,首先解析最低6位的OpCode,再根据OpCode获取该指令的具体格式,因此Lua最多支持2^6=64个指令.Lua代码中,将每个OpCode及其对应的指令格式都在lopcodes.h中的OpCode枚举类型中定义:

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
	
每个OpCode右边紧随的注释说明了该OpCode的具体格式.

从图中可以看出,Lua共有三种指令格式.
	

