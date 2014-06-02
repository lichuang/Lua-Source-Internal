#loadk指令
本次分析时使用的Lua语句如下:

	local a = 10
	
随着后面分析的深入,还会对这个语句做些变化进行其它方面的分析.

这条语句,在使用Lua的递归下降语法分析器时,调用的函数路径依次是:

	(lparser.c)
	chunk
		statement
			exprstat
				primaryexp
					prefixexp
						singlevar
							singlevaraux
								searchvar
								init_exp
								markupval
								indexupvalue
				assignment
				
其重点有两个地方,一是如何识别变量a,其次是如何给变量a赋值常量10.


##查找识别变量
chunk函数会首先进入statement函数中,statement函数会根据下一个将处理的token类型,来进行下一步的工作,比如如果是关键字"if",将进入ifstat函数中进行处理.但是在这个实例代码中,下一个待处理的token是id为"a"的变量,于是就进入了exprstat函数中,这个函数用于处理表达式赋值或者函数调用这样的操作.

在exprstat函数中,看到了LHS_assign结构体,其定义如下:

	(lparser.c)
	892 /*
 	893 ** structure to chain all variables in the left-hand side of an
 	894 ** assignment
 	895 */
 	896 struct LHS_assign {
 	897   struct LHS_assign *prev;
 	898   expdesc v;  /* variable (global, local, upvalue, or indexed) */
 	899 };

这个结构体是用来表示赋值时等号左边的表达式所用,成员prev指向在同一个表达式中的上一个LHS_assign结构体指针,而v则真正用来存储表达式的信息.

例如这样的赋值:

	local a,b = 1,2
	
那么这里假设有LHS_assign数据a_data,b_data分别存储了a,b的信息的话,则其中b_data->prev = a_data.

再来看expdesc结构体的定义:

	(lparser.h)
	37 typedef struct expdesc {
 	38   expkind k;
 	39   union {
 	40     struct { int info, aux; } s;
 	41     lua_Number nval;
 	42   } u;
 	43   int t;  /* patch list of `exit when true' */
 	44   int f;  /* patch list of `exit when false' */
 	45 } expdesc;

该结构体中,k表示该表达式的类型,u是一个union为了节省空间之用,nval用来存储数据为数字的情况,自不必多解释,而info和aux根据不同的数据类型各自表示不同的信息,这些信息都可以在expkind enum的注释中看到:

	(lparser.h)
	19 typedef enum {
 	20   VVOID,  /* no value */
 	21   VNIL,
 	22   VTRUE,
 	23   VFALSE,
 	24   VK,   /* info = index of constant in `k' */
 	25   VKNUM,  /* nval = numerical value */
 	26   VLOCAL, /* info = local register */
 	27   VUPVAL,       /* info = index of upvalue in `upvalues' */
 	28   VGLOBAL,  /* info = index of table; aux = index of global name in `k' */
 	29   VINDEXED, /* info = table register; aux = index register (or `k') */ 
 	30   VJMP,   /* info = instruction pc */
 	31   VRELOCABLE, /* info = instruction pc */
 	32   VNONRELOC,  /* info = result register */
 	33   VCALL,  /* info = instruction pc */
 	34   VVARARG /* info = instruction pc */
 	35 } expkind; 

以这里的例子为例,变量a在这里是全局变量,于是它对应的expkind是VGLOBAL.

expdesc结构体中的t和f这两个变量在这里暂时用不到,留待后面继续解释.

前面解释了LHS_assign和expdesc结构体的含义,现在继续回到exprstat函数中,它会调用primaryexp函数来获取该表达式的expdesc结构体信息,而这会最终走入prefixexp函数中,由于这里要处理的token是a,它是一个TK_NAME类型的token,所以会进入singlevar函数中查找该变量到底是GLOBAL/LOCAL/UPVAL.

查找一个变量,主要的逻辑在函数singlevaraux,来看看它的代码:

	(lparser.c)
	224 static int singlevaraux (FuncState *fs, TString *n, expdesc *var, int base) {
 	225   if (fs == NULL) {  /* no more levels? */
 	226     init_exp(var, VGLOBAL, NO_REG);  /* default is global variable */
 	227     return VGLOBAL;
 	228   }
 	229   else {
 	230     int v = searchvar(fs, n);  /* look up at current level */
 	231     if (v >= 0) {
 	232       init_exp(var, VLOCAL, v);
 	233       if (!base)
 	234         markupval(fs, v);  /* local will be used as an upval */
 	235       return VLOCAL;
 	236     }
 	237     else {  /* not found at current level; try upper one */
 	238       if (singlevaraux(fs->prev, n, var, 0) == VGLOBAL)
 	239         return VGLOBAL;
 	240       var->u.s.info = indexupvalue(fs, n, var);  /* else was LOCAL or UPVAL */
 	241       var->k = VUPVAL;  /* upvalue in this level */
 	242       return VUPVAL;
 	243     }
 	244   }
 	245 }
 	
 	
首先看这个函数调用的四个参数含义.fs自不必说,是这条语句所在的chunk对应的FuncState指针,如果在本层chunk中查找不到这个变量,将使用它的prev指针在它的父chunk中继续查找,如果prev指针为空,说明到了全局作用域;n表示的时这个变量的变量名对应的TString指针;var是查找到此变量时用于保存其信息的expdesc指针;base是一个整型值,只有两个可能数据,1表示是在本层进行的查找,0表示是它的子chunk在它这一层做的查找,如果查找到这个变量,那么就是做为UpValue来给子chunk使用的.

有了前面对函数参数的这些分析,再回头看这个函数做的事情就一目了然了:

	1. 当传入的fs为NULL的时候,说明此时已经在最外一层了,此时认为这个变量是全局变量.
	2. 调用searchvar函数查找该变量,如果其返回值>=0,说明查找到了这个变量,对于这个chunk而言这个变量是局部变量,当base为0的时候,还需要对当前的block做一下标记,这个标记的作用是标记该block中有变量被外部做为UpValue引用了.
	3. 当searchvar返回值<0时,说明在这一层没有查找到这个变量,于是使用FuncState结构体的prev指针进入它的父chunk来进行查找,如果查找结果是全局变量就直接返回,否则就是UpValue了,需要对查找一下这个UpValue.indexupvalue这个函数做的事情很简单,就是在该FuncState对应的Proto结构体的Upvalue数组中查找该变量,如果之前没有这个UpValue就新增一个,最后返回这个UpValue在数组中的索引值.
	
##赋值
exprstat函数在调用primaryexp函数得到变量a的expdesc结构体信息之后,会做一个判断,如果前面得到的expdesc类型是一个函数调用(类型为VCALL)时,将做一些处理,由于不属于这里要讨论的情况不在这次中展开讨论;另一种情况则是一个赋值操作了,将会进入assignment函数中进行处理,在此之前会把expdesc结构体的prev指针赋值为NULL,因为这是这个赋值表达式左边的第一个变量,在它之前没有别的变量.

来看assignment函数的实现.

首先会做一个判断,如果下一个token是",",说明赋值表达式的等号左边有多个变量,继续调用primaryexp函数拿到这个变量的expdesc结构体信息,然后再调用assignment函数.

否则,先读入"="这个token,这时就可以开始处理赋值表达式等号右边的表达式了,这里调用explist1函数同样也是拿到等号右边的expdesc结构体信息.



