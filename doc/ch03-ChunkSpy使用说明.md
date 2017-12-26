## ChunkSpy使用说明

ChunkSpy是一个可以对Lua二进制文件进行反编译拿到对应Opcode的工具,其作者也是文档<<A No-Frills Introduction to Lua 5.1 VM Instructions>>的作者,这篇文档是我阅读Lua代码中参考最多的文档,结合着ChunkSpy工具,可以很好的分析Lua代码对应的Opcode指令,它的下载地址在[这里](http://chunkspy.luaforge.net/).

简单介绍一下ChunkSpy工具的使用,主要有两种方式,一种是交互式的使用,这种方式下可以进入一个类似shell的命令行中,只要在其中输入正确的Lua代码,马上就能得到相应的Opcode,比如:

	>local a=1
	; source chunk: (interactive mode)
	; x86 standard (32-bit, little endian, doubles)
	
	; function [0] definition (level 1)
	; 0 upvalues, 0 params, 2 stacks
	.function  0 0 2 2
	.local  "a"  ; 0
	.const  1  ; 0
	[1] loadk      0   0        ; 1
	[2] return     0   1      
	; end of function
	
简单介绍一下它的输出格式:

* 以";"开头的,可以认为是介绍性的信息,或者说是注释.
* 以"."开头的,比如上面的".function",".local",".const"是不同类型数据的定义,但是这里并不是真正的opcode,它只是写出来让读者明白这里有哪些类型的数据定义.
* 以"[数字]"开头的,就是真正的opcode,对应FuncState中code中的一条指令,其中的数字是code数组中的索引,具体的格式与每个opcode相关,在每条指令后面可能会带一些以";"开头的注释.

另一种方式是对已经通过luac编译生成的Lua二进制文件进行反编译,同样是可以得到对应的opcode,比如最简单的Lua代码:

	local i =1

通过luac编译之后生成luac.out文件,再通过ChunkSpy进行反编译:

	lua ChunkSpy.lua  luac.out  my.lua -o test

它将luac.out的十六进制代码进行了反编译,结果如下:

	Pos   Hex Data           Description or Code
	------------------------------------------------------------------------
	0000                     ** source chunk: luac.out
	                         ** global header start **
	0000  1B4C7561           header signature: "\27Lua"
	0004  51                 version (major:minor hex digits)
	0005  00                 format (0=official)
	0006  01                 endianness (1=little endian)
	0007  04                 size of int (bytes)
	0008  08                 size of size_t (bytes)
	0009  04                 size of Instruction (bytes)
	000A  08                 size of number (bytes)
	000B  00                 integral (1=integral)
	                         * number type: double
	                         * x86 standard (32-bit, little endian, doubles)
	                         ** global header end **
	                         
	000C                     ** function [0] definition (level 1)
	                         ** start of function **
	000C  0800000000000000   string size (8)
	0014  406D792E6C756100   "@my.lua\0"
	                         source name: @my.lua
	001C  00000000           line defined (0)
	0020  00000000           last line defined (0)
	0024  00                 nups (0)
	0025  00                 numparams (0)
	0026  02                 is_vararg (2)
	0027  02                 maxstacksize (2)
	                         * code:
	0028  02000000           sizecode (2)
	; (1)  local i =1
	002C  01000000           [1] loadk      0   0        ; 1
	0030  1E008000           [2] return     0   1      
	                         * constants:
	0034  01000000           sizek (1)
	0038  03                 const type 3
	0039  000000000000F03F   const [0]: (1)
	                         * functions:
	0041  00000000           sizep (0)
	                         * lines:
	0045  02000000           sizelineinfo (2)
	                         [pc] (line)
	0049  01000000           [1] (1)
	004D  01000000           [2] (1)
	                         * locals:
	0051  01000000           sizelocvars (1)
	0055  0200000000000000   string size (2)
	005D  6900               "i\0"
	                         local [0]: i
	005F  01000000             startpc (1)
	0063  01000000             endpc   (1)
	                         * upvalues:
	0067  00000000           sizeupvalues (0)
	                         ** end of function **
	
	006B                     ** end of chunk **

如果需要更简洁的输出结果,可以加入--brief参数.

需要说明的是,ChunkSpy在2006年之后就不怎么更新,对现在的64位系统支持可能不够好,如果在执行的时候发现提示如下的报错:

	ChunkSpy.lua:1120: mismatch in size_t size (needs 4 but read 8)

那么可以尝试着把代码中两个定义size_size_t的地方由4改为8.
