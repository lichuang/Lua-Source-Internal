##Lua词法

[Lua词法](http://lua.lickert.net/syntax/Lua_syntax.pdf)

	chunk -> { stat [`;'] }
	block -> chunk
	
	stat:
	statement -> ifstat | whilestat | dostat | forstat | repeatstat | funcstat | localstat | retstat | breakstat | exprstat
	
	ifstat -> IF cond THEN block {ELSEIF cond THEN block} [ELSE block] END
	cond -> exp
	exp -> subexpr
	subexpr -> (simpleexp | unop subexpr) { binop subexpr }
	simpleexp -> NUMBER | STRING | NIL | true | false | ... | constructor | FUNCTION body | primaryexp
	constructor -> '{'(new table) '}'
	primaryexp -> prefixexp { `.' NAME | `[' exp `]' | `:' NAME funcargs | funcargs }
	prefixexp -> NAME | '(' expr ')'
	funcargs -> `(' [ explist1 ] `)' | constructor | STRING
	
	whilestat -> WHILE cond DO block END
	
	dostat -> DO block END
	
	forstat -> FOR (fornum | forlist) END
	fornum -> NAME = exp1,exp1[,exp1] forbody
	forlist -> NAME {,NAME} IN explist1 forbody
	forbody -> DO block
	
	repeatstat -> REPEAT block UNTIL cond
	
	funcstat -> FUNCTION funcname body
	funcname -> NAME {field} [`:' NAME]
	body ->  `(' parlist `)' chunk END
	field -> ['.' | ':'] NAME
	parlist -> [ param { `,' param } ]
	param -> NAME | ...
	
	localstat -> LOCAL localfunc | localstat
	localfunc -> funcstat
	localstat -> NAME {`,' NAME} [`=' explist1]
	
	retstat -> RETURN explist
	explist -> ??????
	
	breakstat -> ????????
	
	exprstat ->  func | assignment
	func -> ??????
	assignment -> ?????
	
	explist1 -> expr { `,' expr }
	exp1 -> ???????
	
	forbody -> DO block








 
	





