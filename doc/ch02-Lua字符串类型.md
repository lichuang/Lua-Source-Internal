# Lua字符串

首先来看Lua中表示字符串的数据结构定义:

	(lobject.h)
	196 /*
	197 ** String headers for string table
	198 */
	199 typedef union TString {
	200   L_Umaxalign dummy;  /* ensures maximum alignment for strings */
	201   struct {
	202     CommonHeader;
	203     lu_byte reserved;
	204     unsigned int hash;
	205     size_t len; 
	206   } tsv;
	207 } TString;
	
可以看见是一个union,其目的是为了让TString数据类型按照L_Umaxalign类型来进行对齐.来看看这个类型的定义:

	(llimitis.h)
	46 /* type to ensure maximum alignment */
 	47 typedef LUAI_USER_ALIGNMENT_T L_Umaxalign;

	(luaconf.h)
	588 /*
	589 @@ LUAI_USER_ALIGNMENT_T is a type that requires maximum alignment.
	590 ** CHANGE it if your system requires alignments larger than double. (For
	591 ** instance, if your system supports long doubles and they must be
	592 ** aligned in 16-byte boundaries, then you should add long double in the
	593 ** union.) Probably you do not need to change this.
	594 */
	595 #define LUAI_USER_ALIGNMENT_T union { double u; void *s; long l; }
	
在double/void */long三种类型中,尺寸最大的时double.
C语言中,struct/union这样的复合数据类型,是按照这个类型中最大对齐量的数据来进行对齐的,所以这里就是按照double类型的对齐量来进行对齐,一般而言是8字节.	
而在结构体tsv中,其最大的对齐单位肯定不会比double大,所以整个TString union是按照double的对齐量来进行对齐的.

在Lua中,所有字符串是一个保存在一个全局的地方,在global_state的strt里面,这是一个hash数组,专门用于存放字符串:

	(lstate.h)
 	38 typedef struct stringtable {
 	39   GCObject **hash;
 	40   lu_int32 nuse;  /* number of elements */
 	41   int size;
 	42 } stringtable;
 
 一个字符串TString,首先根据hash算法算出hash值,这就是stringtable中hash的索引值,如果这里已经有元素,则使用链表串接起来.
 
 同时,TString中的字段reserved,表示这个字符串是不是保留字符串,比如Lua的关键字,在最开始赋值的时候是这么处理的:
 
 	(llex.c)
 	64 void luaX_init (lua_State *L) {
 	65   int i;
 	66   for (i=0; i<NUM_RESERVED; i++) {
 	67     TString *ts = luaS_new(L, luaX_tokens[i]);
 	68     luaS_fix(ts);  /* reserved words are never collected */
 	69     lua_assert(strlen(luaX_tokens[i])+1 <= TOKEN_LEN);
 	70     ts->tsv.reserved = cast_byte(i+1);  /* reserved word */
 	71   }
 	72 }
 	
 这里存放的值,是数组luaX_tokens中的索引.这样一方面可以迅速定位到是哪个关键字,另方面如果这个reserved字段不为0,则表示该字符串是不可自动回收的,在GC过程中会略过这个字符串的处理:
 
 	(llex.c)
  	36 /* ORDER RESERVED */
 	37 const char *const luaX_tokens [] = {
 	38     "and", "break", "do", "else", "elseif",
 	39     "end", "false", "for", "function", "if",
 	40     "in", "local", "nil", "not", "or", "repeat",
 	41     "return", "then", "true", "until", "while",
 	42     "..", "...", "==", ">=", "<=", "~=",
 	43     "<number>", "<name>", "<string>", "<eof>",
 	44     NULL
 	45 }; 
 	
这里的每个字符串都是与某个保留字Token类型一一对应的:

	20 /*
 	21 * WARNING: if you change the order of this enumeration,
 	22 * grep "ORDER RESERVED"
 	23 */
 	24 enum RESERVED {
 	25   /* terminal symbols denoted by reserved words */
 	26   TK_AND = FIRST_RESERVED, TK_BREAK,
 	27   TK_DO, TK_ELSE, TK_ELSEIF, TK_END, TK_FALSE, TK_FOR, TK_FUNCTION,
 	28   TK_IF, TK_IN, TK_LOCAL, TK_NIL, TK_NOT, TK_OR, TK_REPEAT,
 	29   TK_RETURN, TK_THEN, TK_TRUE, TK_UNTIL, TK_WHILE,
 	30   /* other terminal symbols */
 	31   TK_CONCAT, TK_DOTS, TK_EQ, TK_GE, TK_LE, TK_NE, TK_NUMBER,
 	32   TK_NAME, TK_STRING, TK_EOS
 	33 };
 
需要说明的是,上面luaX_tokens字符串数组中的"\<number>", "\<name>", "\<string>", "\<eof>"这几个字符串并不真实做为Lua语言中的保留关键字存在,但是因为有相应的保留字Token类型,所以也就干脆这么定义一个对应的字符串了.
