#loadk指令
分析loadk指令时使用的Lua语句如下:

	a = 10
	
随着后面分析的深入,还会对这个语句做些变化进行其它方面的分析.

这条语句,在使用Lua的递归下降语法分析器时,调用的函数路径依次是:

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


