Lua的表
=====================
表可以说是Lua中唯一的数据类型了,使用Lua表,可以模拟出其他各种数据结构--数组,链表,树等等.首先来看Lua表的数据类型定义:

        319 /*
        320 ** Tables
        321 */
        322 
        323 typedef union TKey {
        324   struct {
        325     TValuefields;
        326     struct Node *next;  /* for chaining */
        327   } nk;
        328   TValue tvk;
        329 } TKey;
        330 
        331 
        332 typedef struct Node {
        333   TValue i_val;
        334   TKey i_key;
        335 } Node;
        336 
        337 
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
        (lobject.h文件)

来逐个解释一下Table这个数据结构中的成员定义:

1) CommonHeader,在前面的通用数据类型中已经做过解释.

2) flags,这是一个byte类型的数据,用于表示在这个表中提供了哪些meta method.在最开始,这个flags是空的,也就是为0,当查找了一次之后,如果该表中存在某个meta method,那么将该meta method对应的flag bit置为1,这样下一次查找时只需要比较这个bit就可以知道了.每个meta method对应的bit定义在ltm.h文件中:
        
         14 /*
         15 * WARNING: if you change the order of this enumeration,
         16 * grep "ORDER TM"
         17 */
         18 typedef enum {
         19   TM_INDEX,
         20   TM_NEWINDEX,
         21   TM_GC,
         22   TM_MODE,
         23   TM_EQ,  /* last tag method with `fast' access */
         24   TM_ADD,
         25   TM_SUB,
         26   TM_MUL,
         27   TM_DIV,
         28   TM_MOD,
         29   TM_POW,
         30   TM_UNM, 
         31   TM_LEN,
         32   TM_LT,
         33   TM_LE,
         34   TM_CONCAT,
         35   TM_CALL,
         36   TM_N    /* number of elements in the enum */
         37 } TMS;

3) lsizenode,该Lua表Hash桶大小的log2值.

4) metatable,存放该Lua表的meta表.

5) array,存放该Lua表的数组部分.

6) node,指向该Lua表的Hash部分.

7) lastfree,指向该Lua表Hash部分的最后部分.

8) gclist,GC相关的链表,在后面GC部分再回过头来讲解相关的内存.

9) sizearray,Lua表数组部分的大小.

这里注意一个细节,lsizenode使用的时byte类型,而sizearray使用的是int类型.由于在Lua的Hash桶部分,每个Hash值相同的数据,会以链表的形式串起来在该Hash桶中,所以即使数量用完了也不要紧.因此这里使用byte类型,而且是原数据的log2值,因为要根据这个值还原回原来的真实数据,也只是需要移位操作罢了,速度很快.这个细节中可以看到,在Lua的内存使用方面,还是扣了不少细节的地方.

紧跟着来看看Lua表中结点的数据类型.首先从Node的类型定义可以看出,它包含两个成员,其中一个表示结点的key,另一个表示结点的value.值部分就不多解释了,还是Lua中的通用数据类型TValue,来看看key部分的含义:

        329 typedef union TKey {
        330   struct {
        331     TValuefields;
        332     struct Node *next;  /* for chaining */
        333   } nk;
        334   TValue tvk;
        335 } TKey;

可以看到这个数据类型是一个union类型,一般看到一个数据类型是union,就可以知道这个数据想以一种较为省内存的方式来表示多种用途含义,而这些用途之间是"互斥"的,也就是说,在某个时刻该数据类型只会有其中的一个含义.这种C编程技巧在Lua中非常常见,可以回头再看看前面的通用数据类型消化一下这样的技巧.

从以上定义可以看到,Lua的表有两个部分,一个是数组,一个是Hash.
