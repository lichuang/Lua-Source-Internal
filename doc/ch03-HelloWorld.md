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

##初始化Lua虚拟机指针
首先来看第一步,调用lua_open函数创建并且初始化一个Lua虚拟机指针,来看看这里具体做了什么事情.

##读取脚本文件

##词法分析

##语法分析

##语义分析

##虚拟机执行指令