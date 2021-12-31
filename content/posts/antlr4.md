---
title: "基于antlr4构建规则引擎简介"
date: 2021-12-31T17:47:55+08:00
draft: false
---

# 基于antlr4构建规则引擎简介

## 1.什么是antlr

ANTLR (ANother Tool for Language Recognition) 是一个强大的文本或者二进制文件解析工具，被广泛应用于构建语法以及各种工具和框架。如

- Twitter's search query language
- MySQL Workbench
- Hibernate
- Trino (SQL query engine)
- ...

antlr可以对某一种语言描述，根据对应的语法，生成一颗语法树，在这颗语法树中包含了语言描述与对应语法规则的关系。当你去遍历这颗语法树时，可以灵活处理遍历前和遍历后的规则，实现某种效果。（可以理解为根据一组确定的语法规则，处理一段数据，如实现某种声明、运算、调用等，从而得到某种结果）

antlr最新的大版本是v4，了解相关变更详情可参考 Why do we need ANTLR v4

## 2.可以用来做什么

antlr可以用来做各种各样的事情，比如本文要说的基于antlr4构建的规则引擎。 除此之外，还有海量的场景可以用到antlr，比如

- vscode中的插件，基于antlr做语法规则检查
- grafana中的查询语法解析
- 格式转换工具，如json、yaml、xml等互相转换
- ...

## 3.用antlr4实现类golang语法解析器的规则引擎

最近在研究规则引擎相关的实现，g社上发现一个bilibili的开源项目gengine，基于antlr4实现了类golang的语法解析器，下面我们来以gengine为例看一下如何基于antlr4实现这个事情。

下面文件中描述了一个规则的表达，其中具体的规则部分表达与golang语法基本类似。这个就是在gengine中规则描述的基本形式，在gengine中，用户可以根据具体的场景构造若干个规则，通过这个规则去做一些事情。下面的描述中， rule为固定的规则描述声明词，"rulename" "rule-description" 为规则的名称和描述，salience为固定的规则优先级描述声明词，后面的10代表这个规则的优先级为10。begin和end中包围的部分为具体的规则内容，在gengine中其表达方式类似于golang的语法。

```txt
rule "rulename" "rule-description" salience  10
begin

//规则体

end
```

对应的antlr4中的语法描述文件(gengine.g4，为了展示方便省略了部分内容，全部内容可通过链接查看)如下，具体的.g4文件结构可参考 Grammer Structure

```g4
grammar gengine;

primary: ruleEntity+;

ruleEntity:  RULE ruleName ruleDescription? salience? BEGIN ruleContent END;
ruleName : stringLiteral;
ruleDescription : stringLiteral;
salience : SALIENCE integer;
ruleContent : statements;
statements: statement* returnStmt? ;

statement : ifStmt | functionCall | methodCall  | threeLevelCall | assignment | concStatement  | forStmt | breakStmt |forRangeStmt | continueStmt;

concStatement : CONC LR_BRACE ( functionCall | methodCall | threeLevelCall | assignment )* RR_BRACE;

expression : mathExpression
            | expression comparisonOperator expression
            | expression logicalOperator expression
            | notOperator ? expressionAtom
            | notOperator ? LR_BRACKET expression  RR_BRACKET
            ;

mathExpression : mathExpression  mathMdOperator mathExpression
               | mathExpression  mathPmOperator mathExpression
               | expressionAtom
               | LR_BRACKET mathExpression RR_BRACKET
               ;

expressionAtom
    : functionCall
    | methodCall
    | threeLevelCall
    | constant
    | mapVar
    | variable
    ;

assignment : (mapVar | variable) assignOperator (mathExpression| expression);

returnStmt : RETURN expression?;

ifStmt : IF expression LR_BRACE statements RR_BRACE elseIfStmt*  elseStmt? ;

elseIfStmt : ELSE IF expression LR_BRACE statements RR_BRACE;

elseStmt : ELSE LR_BRACE statements RR_BRACE;

functionArgs
    : (constant | variable  | functionCall | methodCall | threeLevelCall | mapVar | expression)  (','(constant | variable | functionCall | methodCall | threeLevelCall | mapVar | expression))*
    ;

integer : MINUS? INT;

realLiteral : MINUS? REAL_LITERAL;

stringLiteral: DQUOTA_STRING ;

booleanLiteral : TRUE | FALSE;

functionCall : SIMPLENAME LR_BRACKET functionArgs? RR_BRACKET;

variable :  SIMPLENAME | DOTTEDNAME | DOUBLEDOTTEDNAME;

mathPmOperator : PLUS | MINUS ;

mathMdOperator : MUL | DIV ;

comparisonOperator : GT | LT | GTE | LTE | EQUALS | NOTEQUALS ;

assignOperator: ASSIGN | SET | PLUSEQUAL | MINUSEQUAL | MULTIEQUAL | DIVEQUAL ;

// ...

fragment DEC_DIGIT          : [0-9];
fragment A                  : [aA] ;
fragment B                  : [bB] ;
fragment C                  : [cC] ;
// ...
fragment Z                  : [zZ] ;
fragment EXPONENT_NUM_PART  : ('E'| 'e') '-'? DEC_DIGIT+;

NIL                         : N I L;
RULE                        : R U L E  ;
AND                         : '&&' ;
OR                          : '||' ;

CONC                        : C O N C;
IF                          : I F;
ELSE                        : E L S E;
RETURN                      : R E T U R N;

// ...

BEGIN                       : B E G I N;
END                         : E N D;

SIMPLENAME :  ('a'..'z' |'A'..'Z'| '_')+ ( ('0'..'9') | ('a'..'z' |'A'..'Z') | '_' )* ;

INT : '0'..'9' + ;
PLUS                        : '+' ;
MINUS                       : '-' ;
DIV                         : '/' ;
MUL                         : '*' ;

EQUALS                      : '==' ;
GT                          : '>' ;
LT                          : '<' ;
GTE                         : '>=' ;
// ...

SEMICOLON                   : ';' ;
LR_BRACE                    : '{';
RR_BRACE                    : '}';
LR_BRACKET                  : '(';
RR_BRACKET                  : ')';
DOT                         : '.' ;
DQUOTA_STRING               : '"' ( '\\'. | '""' | ~('"'| '\\') )* '"';
DOTTEDNAME                  : SIMPLENAME DOT SIMPLENAME  ;
DOUBLEDOTTEDNAME            : SIMPLENAME DOT SIMPLENAME DOT SIMPLENAME;
REAL_LITERAL                : (DEC_DIGIT+)? '.' DEC_DIGIT+
                            | DEC_DIGIT+ '.' EXPONENT_NUM_PART
                            | (DEC_DIGIT+)? '.' (DEC_DIGIT+ EXPONENT_NUM_PART)
                            | DEC_DIGIT+ EXPONENT_NUM_PART
                            ;

SL_COMMENT: '//' .*? '\n' -> skip ;
WS  :   [ \t\n\r]+ -> skip ;
```

上述.g4文件声明了gengine中依赖的语法规则。按照这种语法规则，假设当前有以下一段输入

```txt
rule "elseif_test" "test"
begin

a = 8
if a < 1 {
        println("a < 1")
} else if a >= 1 && a <6 {
        println("a >= 1 && a <6")
} else {
        println("a >= 6")
}

end
```

根据gengine.g4文件中提供的语法规则，antlr4将输入内容组织成一颗语法树，语法树的结构可见下图。遍历这颗语法树，可以获得对应的语法树结构，这是实现规则引擎的基础。

![image](/image/antlr/if_else.png)

下面谈一下antlr4是如何做到这个事情的。可以通过如下命令来解析.g4文件，执行之后生成了几种类型的文件

```bash
$ wget <http://www.antlr.org/download/antlr-4.7-complete.jar>
$ alias antlr4='java -jar $PWD/antlr-4.7-complete.jar'
$ antlr4 -Dlanguage=Go -o parser gengine.g4
```

A. lexer

lexer的作用是将任意的文本输入返回为一系列token，如上述例子中假设输入为 a < 1 , lexer会返回SIMPLENAME(a), LT(<), INT(1)

B. parser

parser的作用是接受lexer的输出并将其用于语法规则(rule)，构造更高级别的结构，比如将expression赋值给variable。

C. listener

经过parser，我们的输入已经成为了一个符合.g4文件中语法的语法树，listener的作用就是提供了遍历节点前和节点后的hooks，这些hooks中具体的函数要使用者根据具体的场景去实现。

遍历语法树后，我们可以得到一个与语法树对应的规则结构体，而这个结构体可以是规则引擎中的某个规则，规则可以实现func (r *RuleEntity) Execute(dc*context.DataContext) (interface{}, error, bool) 方法来实现具体规则的效果。

```go
// 遍历语法树后得到的规则结构体，部分字段已忽略

type RuleEntity struct {
        RuleName        string
        Salience        int64
        RuleDescription string
        RuleContent     *RuleContent
}

type RuleContent struct {
        Statements *Statements
}

type Statements struct {
        StatementList   []*Statement
        // ...
}

type Statement struct {
        IfStmt         *IfStmt
FunctionCall*FunctionCall
        Assignment     *Assignment
       // ...
}

type IfStmt struct {
        Expression     *Expression
StatementList*Statements
        ElseIfStmtList []*ElseIfStmt
ElseStmt*ElseStmt
}

type Expression struct {
        SourceCode
        ExpressionLeft     *Expression
ExpressionRight*Expression
        ExpressionAtom     *ExpressionAtom
MathExpression*MathExpression
        LogicalOperator    string
        ComparisonOperator string
        NotOperator        string
}

// ...

type ElseIfStmt struct {
        Expression    *Expression
StatementList*Statements
}
```

以IfStmt为例，简单介绍下在Execute中所做的事情。其实所做的事情很简单，就是正常的if逻辑，只不过在golang中用反射和语法树结构的下层调用表达出来，其中
`dc *context.DataContext` 与 `Vars map[string]reflect.Value` 中存放了变量的值，如下

```go
func (i*IfStmt) Evaluate(dc *context.DataContext, Vars map[string]reflect.Value) (reflect.Value, error, bool) {

        it, err := i.Expression.Evaluate(dc, Vars)
        if err != nil {
                return reflect.ValueOf(nil), err, false
        }
        // if 条件命中
        if it.Bool() {
                if i.StatementList == nil {
                        return reflect.ValueOf(nil), nil, false
                } else {
                        return i.StatementList.Evaluate(dc, Vars)
                }

        } else {
                // else if 条件判断
                if i.ElseIfStmtList != nil {
                        for _, elseIfStmt := range i.ElseIfStmtList {
                                v, err := elseIfStmt.Expression.Evaluate(dc, Vars)
                                if err != nil {
                                        return reflect.ValueOf(nil), err, false
                                }
                                // else if 条件命中
                                if v.Bool() {
                                        return elseIfStmt.StatementList.Evaluate(dc, Vars)
                                }
                        }
                }
                // else 条件判断
                if i.ElseStmt != nil {
                        return i.ElseStmt.Evaluate(dc, Vars)
                } else {
                        return reflect.ValueOf(nil), nil, false
                }
        }
}
```

现在知道规则如何跑起来，思考的另一个事情是参数如何传递。因为以规则引擎为例，想做的事情可以概括为在某种条件某种规则约束下对应的输出的是什么。输入必须是动态的，这样某个规则才有存在的意义。gengine提供了注入外部变量和函数的方式来实现这个事情, 使用的方式如下

```go
// 注入外部变量或者函数
dataContext := context.NewDataContext()
dataContext.Add("println", fmt.Println)
dataContext.Add("a",20)

v, err, bx := rule.Execute(dataContext)
// ...
```

到这里似乎基于antlr4构建规则引擎的事情就做完了，用户可以随意声明一段规则，然后基于规则去做对应的业务判断。antlr4帮助我们做的其实是将输入文本转换为一个根据.g4文件的语法树，具体要怎么用这个语法树就是使用者应该考虑的事情，在规则引擎这个范畴下，可以参考gengine对应的实现。另外，可以参考他们的test，里面有很多不同的用法。

## 4.参考

- <https://www.antlr.org/>

- <https://blog.gopheracademy.com/advent-2017/parsing-with-antlr4-and-go/>

- <https://github.com/bilibili/gengine>
