##Lua GC

###原理
一个GC算法,其原理大体就是,遍历系统中的所有对象,看哪些对象没有被引用,没有引用关系的就认为是可以回收的对象进行删除操作.

这里的关键在于,如何找出没有"引用"的对象.

使用引用计数的GC算法,会在一个对象被引用的情况下将该对象的引用计数加1;反之减1.如果引用计数为0,那么就是没有被引用的对象.引用计数算法的优点是不需要扫描每一个对象,对象本身的引用计数只需要减到0自然就会被回收.缺点是会有循环引用问题.

另一种算法是"Mark & Sweep",标记及回收算法.它的原理是每一次做GC的时候首先扫描并且标记系统中的所有对象,被扫描并且标记到的对象认为是可达的(reachable),这些对象在这些GC中并不会回收,反之如果在这次扫描中都没有被标记的对象认为是可以回收的.Lua采用的就是这种算法.

早期的Lua5.0使用的是"Two-Color Mark & Sweep",两色标记及回收算法,其算法的原理是,系统中的每个对象非黑即白,也就是要么被引用要么没有被引用.我们来简单看看这个算法的伪代码:

	每个新创建的对象颜色为白色

	// 初始化阶段
	遍历在root链表中的对象,加入到对象链表

	// 标记阶段
	当对象链表中还有未扫描的元素:
		从中取出一个对象,标记为黑色
		遍历这个对象关联的其他所有对象:
			标记为黑色

	// 回收阶段
	遍历所有对象:
		如果为白色:
			这些对象都是没有被引用的对象,逐个回收
		否则:
			重新加入对象链表中等待下一轮的GC检查

这个算法的缺陷在于,如前面所言,它是"二元"的,每个对象只可能有一种状态,不能有其他中间的状态,那么就要求这个算法每次做GC操作时不可被打断的一次性扫描并清除完毕所有的对象.

来看看这个过程不能被打断的原因.在遍历对象链表标记每个对象颜色的过程中被打断,新增了一个对象,那么应该将这个对象标记为白色还是黑色?如果标记为白色,假如GC已经到了回收阶段,那么这个对象就会在没有遍历其关联对象的情况下被回收;如果标记为黑色,假如GC已经到了回收阶段,那么这个对象在本轮GC中并没有被扫描过就认为是不必回收的.可以看到,在双色标记扫描算法中,标记阶段和回收阶段必须合在一起完成.不能被打断,也就意味着每次GC操作的代价极大.

Lua5.1采用了在该算法基础上改进的"Tri-Color Incremental Mark & Sweep",三色增量标记及回收算法,比之前面而言,每个对象的颜色多了一种(实际在Lua中是四种,后面展开讨论),多了一种颜色状态的好处在于,它不必再要求一次GC要一次性扫描完毕所有的对象,而是这个GC过程可以是增量,可以被中断再恢复继续进行的.

	每个新创建的对象颜色为白色

	// 初始化阶段
	遍历在root链表中的对象,加入到灰色链表

	// 标记阶段
	当灰色链表中还有未扫描的元素:
		从中取出一个对象,标记为黑色
		遍历这个对象关联的其他所有对象:
			如果是白色:
				标记为灰色,加入灰色链表

	// 回收阶段
	遍历所有对象:
		如果为白色:
			这些对象都是没有被引用的对象,逐个回收
		否则:
			重新加入对象链表中等待下一轮的GC检查

在这里,同前面一样,白色代表没有被引用的对象,黑色代表已经被扫描过并且其引用的对象也被扫描过的对象,而多出来的灰色表示的是已经被扫描但是其引用的对象还没有被扫描的对象.

我们来看看多了一种颜色具体带来了什么好处.


来看看三色标记扫描算法中的改进.同样是前面的

###数据结构

###具体流程

####初始化阶段
初始化阶段,将从root节点出发,遍历所有root链表上的节点,将它们的颜色从白色变成灰色,加入到gray链表中.阶段的入口函数式markroot函数:

	(lgc.c)
	500 /* mark root set */
	501 static void markroot (lua_State *L) {
	502   global_State *g = G(L);
	503   g->gray = NULL;
	504   g->grayagain = NULL;
	505   g->weak = NULL;
	506   markobject(g, g->mainthread);
	507   /* make global table be traversed before main stack */
	508   markvalue(g, gt(g->mainthread));
	509   markvalue(g, registry(L));
	510   markmt(g);
	511   g->gcstate = GCSpropagate;
	512 }
	
其中的markobject和markvalue函数都是用于标记对象颜色为灰色,所不同的是前者是针对object而后者是针对TValue,它们最终都会调用reallymarkobject函数:

	(lgc.c)
 	69 static void reallymarkobject (global_State *g, GCObject *o) {
	70   lua_assert(iswhite(o) && !isdead(g, o));
	71   white2gray(o);
	72   switch (o->gch.tt) {
	73     case LUA_TSTRING: {
	74       return;
	75     }
	76     case LUA_TUSERDATA: {
	77       Table *mt = gco2u(o)->metatable;
	78       gray2black(o);  /* udata are never gray */
	79       if (mt) markobject(g, mt);
	80       markobject(g, gco2u(o)->env);
	81       return;
	82     }
	83     case LUA_TUPVAL: {
	84       UpVal *uv = gco2uv(o);
	85       markvalue(g, uv->v);
	86       if (uv->v == &uv->u.value)  /* closed? */
	87         gray2black(o);  /* open upvalues are never black */
	88       return;
	89     }
	90     case LUA_TFUNCTION: {
	91       gco2cl(o)->c.gclist = g->gray;
	92       g->gray = o;
	93       break;
	94     }
	95     case LUA_TTABLE: {
	96       gco2h(o)->gclist = g->gray;
	97       g->gray = o;
	98       break;
	99     }
	100     case LUA_TTHREAD: {
	101       gco2th(o)->gclist = g->gray;
	102       g->gray = o;
	103       break;
	104     }
	105     case LUA_TPROTO: {
	106       gco2p(o)->gclist = g->gray;
	107       g->gray = o;
	108       break;
	109     }
	110     default: lua_assert(0);
	111   }
	112 }

可以看到,绝大部分类型的对象,在这个函数中的操作只是简单的将其改变颜色为灰色加入到gray链表中.但是有几个类型是做区别处理的.

对于字符串类型数据,由于这种类型没有引用其他的数据,所以略过将其改变颜色灰色的流程,后面只要不是黑色的字符串对象直接进行回收.

对于udata类型数据,因为这种类型永远也不会引用其它的数据,所以这里也是一步到位直接标记为黑色,另外对于这种类型,还需要标记对应的metatable和env表.

对于Upvalue类型数据,

另外需要注意的是,这里并没有针对这个对象所引用的对象递归调用reallymarkobject函数进行递归调用,比如Table类型的对象就没有遍历它的Key和Value数据进行mark,而在udata中要标记metable和env表,是因为除了这里之外并没有直接能访问到它的其它地方了.并没有递归去标记引用对象的原因,是想让这个标记过程尽量的快.

####扫描标记阶段
在这一步,将扫描所有gray链表中的对象,将它们以及所引用到的对象标记成黑色.需要注意的是,前面的初始化阶段是一次性到位的,而这一步却是可以多次进行,每次扫描之后会返回本次扫描标记的对象大小之和,其入口函数是propagatemark,下一次再次扫描时,只要gray链表中还有待扫描的对象,就继续执行这个函数进行标记.

这个函数与前面的reallymarkobject函数,做的事情其实差不多,都是对一个对象进行标记颜色的动作,区别在于,这里将对象从灰色标记成黑色,表示这个对象及其所引用的对象都已经被标记过,另一个区别在于,前面的流程不会递归对一个对象所引用的对象进行标记,而这里会根据不同的类型调用对应的traverse*函数遍历引用到的对象进行标记.实际工作中,里面的每种类型的对象,处理还不太一样,下面来看看:

如果是表对象,并且是一个弱表,那么需要将颜色从黑色回退到灰色去,还需要再做扫描.

如果是线程对象,也是从黑色回退到灰色去,同时是加入到grayagain链表中,需要在后面原子性的做扫描标记,这个过程不可被打断.

其他的对象都是正常的扫描标记流程.

当gray链表中的所有对象都被处理完毕,此时还不能立即进入下一个流程进行回收操作,因为在前面的流程中,

到了这里可以谈谈barrier操作了.

从前面的描述可以知道,分步增量式的扫描标记算法,由于中间可以被打断执行其他操作,那么就会出现新增加的对象与已经被扫描过的对象之间会有引用关系的变化,而算法中需要保证不会出现有黑色的对象引用的对象中却有白色对象的情况,于是需要两种不同的处理:

	1.标记过程向前走一步,这种情况指的是,如果有一个新创建的对象,其颜色是白色,而它被一个黑色对象引用了,那么将这个对象的颜色从白色变成灰色,也就是在这个GC过程中的进度向前走了一步.
	2.标记过程向后走一步,与前面的情况一样,但是此时是将黑色的对象回退到灰色,也就是需要重新被扫描,这相当于在GC过程中向后走了一步.
	
在代码中,最终调用luaC_barrierf函数的,都是向前走的操作;反之,调用luaC_barrierback的操作则是向后走的操作:

	(lgc.h)
	 86 #define luaC_barrier(L,p,v) { if (valiswhite(v) && isblack(obj2gco(p)))  \
	 87     luaC_barrierf(L,obj2gco(p),gcvalue(v)); }
	 88
	 89 #define luaC_barriert(L,t,v) { if (valiswhite(v) && isblack(obj2gco(t)))  \
	 90     luaC_barrierback(L,t); }
	 91
	 92 #define luaC_objbarrier(L,p,o)  \
	 93     { if (iswhite(obj2gco(o)) && isblack(obj2gco(p))) \
	 94         luaC_barrierf(L,obj2gco(p),obj2gco(o)); }
	 95
	 96 #define luaC_objbarriert(L,t,o)  \
	 97    { if (iswhite(obj2gco(o)) && isblack(obj2gco(t))) luaC_barrierback(L,t); }
 
可以看到的是,回退操作仅针对的Table类型的对象,而对于其他类型的对象都是向前操作,我们来看看这么做的原因.

Table是Lua中最常见的数据结构,而且一个Table与其关联的Key,Value之间是1比N的对应关系.如果针对Table对象做的是向前的标记操作,那么就意味着,但凡一个Table,只要有新增的对象,都需要将这个对象标记为灰色加入gray链表中等待扫描,实际上这样会有不必要的开销.所以针对Table类型的对象,使用的是针对该Table对象本身要做的向后的操作,这样不论有多少个对象新增到这个Table中,只要改变了一次就将这个Table对象回退到灰色状态,等待重新的扫描.但是这里需要注意的是,对Table对象进行回退操作时,并不是将它放入gray链表中,因为这样子做实际上还是会出现前面提到的多次反复标记的问题,针对Table对象,对它执行回退操作时是将它加入到grayagain链表中,用于在扫描完毕gray链表之后再一次性的原子扫描:

	(lgc.c)
	675 void luaC_barrierback (lua_State *L, Table *t) {
	676   global_State *g = G(L);
	677   GCObject *o = obj2gco(t);
	678   lua_assert(isblack(o) && !isdead(g, o));
	679   lua_assert(g->gcstate != GCSfinalize && g->gcstate != GCSpause);
	680   black2gray(o);  /* make table gray (again) */
	681   t->gclist = g->grayagain;
	682   g->grayagain = o;
	683 }

####回收阶段










 
	





