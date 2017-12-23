# Lua中的数据类型

在Lua中,分为以下几种数据类型:
      
	(lua.h)  
 	72 #define LUA_TNONE   (-1)
 	73 
 	74 #define LUA_TNIL    0
 	75 #define LUA_TBOOLEAN    1
 	76 #define LUA_TLIGHTUSERDATA  2
 	77 #define LUA_TNUMBER   3
 	78 #define LUA_TSTRING   4
 	79 #define LUA_TTABLE    5
 	80 #define LUA_TFUNCTION   6
 	81 #define LUA_TUSERDATA   7
 	82 #define LUA_TTHREAD   8


其中的LUA_TLIGHTUSERDATA和LUA_TUSERDATA一样,对应的都是void*指针,区别在于,LUA_TLIGHTUSERDATA的分配释放是由Lua外部的使用者来完成,而LUA_TUSERDATA则是通过Lua内部来完成的,换言之,前者不需要Lua去关心它的生存期,由使用者自己去关注,后者则反之.

Lua内部用一个宏,表示哪些数据类型需要进行gc操作的:

	(lobject.h)
	189 #define iscollectable(o)  (ttype(o) >= LUA_TSTRING)


可以看到,LUA_TSTRING(包括LUA_TSTRING)之后的数据类型,都需要进行gc操作.

那么,对于这些需要进行gc操作的数据类型,在Lua中是如何表示的呢?

这些需要gc的数据类型,都会有一个CommonHeader的成员,并且这个成员在结构体定义的最开始部分,如:

	(lobject.h)
	338 typedef struct Table {
	339   CommonHeader;
	340   lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
	341   lu_byte lsizenode;  /* log2 of size of `node' array */
	342   struct Table *metatable;
	343   TValue *array;  /* array part */
	344   Node *node;
	345   Node *lastfree;  /* any free position is before this position */
	346   GCObject *gclist;
	347   int sizearray;  /* size of `array' array */
	348 } Table;

其中CommonHeader的定义如下:

	(lobject.h)
 	39 /*
 	40 ** Common Header for all collectable objects (in macro form, to be
 	41 ** included in other objects)
 	42 */
 	43 #define CommonHeader  GCObject *next; lu_byte tt; lu_byte marked

同时,还有一个名为GCheader的结构体,其中的成员只有CommonHeader:

	(lobject.h)
 	46 /* 
 	47 ** Common header in struct form
 	48 */             
 	49 typedef struct GCheader {
 	50   CommonHeader;
 	51 } GCheader;   

于是,在Lua中就使用了一个GCObject的union将所有可gc类型囊括了进来:

	(lstate.h)
	133 /*
	134 ** Union of all collectable objects
	135 */
	136 union GCObject {
	137   GCheader gch;
	138   union TString ts;
	139   union Udata u;
	140   union Closure cl;
	141   struct Table h;
	142   struct Proto p;
	143   struct UpVal uv;        
	144   struct lua_State th;  /* thread */
	145 };

我们整理一下前面提到的这么几个结构体,可以得到这样的结论:

>1) 任何需要gc的Lua数据类型,必然以CommonHeader做为该结构体定义的最开始部分.如果熟悉C++类实现原理的人,可以将CommonHeader这个成员理解为一个基类的所有成员,而其他需要gc的数据类型均从这个基类中继承下来,所以它们的结构体定义开始部分都是这个成员.


>2) GCObject这个union,将所有需要gc的数据类型全部囊括其中,这样在定位和查找不同类型的数据时就来的方便多了,而如果只想要它们的GC部分,可以通过GCheader gch,如:

	(lobject.h)
	91 #define gcvalue(o)  check_exp(iscollectable(o), (o)->value.gc)


仅表示了需要gc的数据类型还不够,还有几种数据类型是不需要gc的,Lua中将GCObject和它们一起放在了union Value中:

	(lobject.h)
 	56 /* 
 	57 ** Union of all Lua values
 	58 */ 
 	59 typedef union {           
 	60   GCObject *gc;           
 	61   void *p;
 	62   lua_Number n;           
 	63   int b;
 	64 } Value;

到了这一步,已经差不多可以表示所有在Lua中存在的数据类型了,但是还欠缺一点东西,就是这些数据到底是什么类型的.于是Lua代码中又有了一个TValuefields将Value和类型结合在一起:

	(lobject.h)
	71 #define TValuefields  Value value; int tt

这些合在一起,最后形成了Lua中的TValue结构体,在Lua中的任何数据都可以通过该结构体进行表示:

	(lobject.h)
 	73 typedef struct lua_TValue {
 	74   TValuefields;           
 	75 } TValue; 

![Tvalue](https://raw.github.com/lichuang/Lua-Source-Internal/master/pic/gcobject.png "TValue")
