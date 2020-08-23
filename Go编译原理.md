# Go 编译原理

[TOC]

## 概述

Go的编译过程会经过以下几个步骤

前端：
1. 词法分析
2. 语法分析
3. 类型检查
4. 中间代码生成

后端：
5. 目标代码生成
6. 机器码优化

## 词法和语法分析

### 词法分析

词法分析的过程就是把人容易看懂的代码转换成机器容易看懂的token序列的过程

#### lex
整个过程一般通过lex工具实现

首先在 `.l` 后缀定义要分析的规则， 这个规则是以类C语言的语法来写的

```c
%{
#include <stdio.h>
%}

%%
package      printf("PACKAGE ");
import       printf("IMPORT ");
\.           printf("DOT ");
\{           printf("LBRACE ");
\}           printf("RBRACE ");
\(           printf("LPAREN ");
\)           printf("RPAREN ");
\"           printf("QUOTE ");
\n           printf("\n");
[0-9]+       printf("NUMBER ");
[a-zA-Z_]+   printf("IDENT ");
%%
```

但这并不是一个真正的C语言语法， 所以我们要通过 lex 程序把它转换成 `.c` 后缀的真正的c语言文件, 使用命令如下

```
$ lex simplego.l
$ ls
lex.yy.c simplego.l
$ cat lex.yy.c
int yylex (void) {
	...
	while ( 1 ) {
		...
yy_match:
		do {
			register YY_CHAR yy_c = yy_ec[YY_SC_TO_UI(*yy_cp)];
			if ( yy_accept[yy_current_state] ) {
				(yy_last_accepting_state) = yy_current_state;
				(yy_last_accepting_cpos) = yy_cp;
			}
			while ( yy_chk[yy_base[yy_current_state] + yy_c] != yy_current_state ) {
				yy_current_state = (int) yy_def[yy_current_state];
				if ( yy_current_state >= 30 )
					yy_c = yy_meta[(unsigned int) yy_c];
				}
			yy_current_state = yy_nxt[yy_base[yy_current_state] + (unsigned int) yy_c];
			++yy_cp;
		} while ( yy_base[yy_current_state] != 37 );
		...
		
do_action:
		switch ( yy_act )
			case 0:
    			...

			case 1:
    			YY_RULE_SETUP
    			printf("PACKAGE ");
    			YY_BREAK
			...
}
```

会在本目录中生成 `lex.yy.c` 文件， `lex.yy.c` 文件的大部分是宏和函数声明和定义，后面生成的代码大都是为 `yylex` 这个函数服务的，这个函数使用有限自动机(DFA)的程序结构来分析输入的字符流，上述代码中 while 循环就是这个有限自动机的主体, 但是本文件并不包含main函数，main函数其实是在liblex库中定义的，所以在编译本c文件的时候要加上liblex库

```
$ cc lex.yy.c -o simplego -ll
```

这里会生成一个 simplego 二进制文件， 该二进制文件可以处理原始的go文件，把他们转换成机器容易看懂的 token 序列, 要注意的是需要给这个二进制传输输入流而不是以参数形式指定文件名

```
$ cat main.go | ./simplego

PACKAGE  IDENT

IMPORT  QUOTE IDENT QUOTE

IDENT  LPAREN
    IDENT  IDENT  = IDENT
    IDENT
    IDENT
RPAREN

IDENT  IDENT LPAREN RPAREN  LBRACE
    IDENT DOT IDENT LPAREN IDENT RPAREN
    IDENT DOT IDENT LPAREN IDENT RPAREN
RBRACE
```

这里可以稍微看得到原始go文件的缩影

#### Go语言的词法分析

Go语言的词法解析是通过 `src/cmd/compile/internal/syntax/scanner.go` 文件中的 `scanner` 结构体实现，这个结构体会持有当前扫描的数据源文件、启用的模式和当前被扫描到的token

```go
type scanner struct {
	source
	mode   uint
	nlsemi bool

	// current token, valid after calling next()
	line, col uint
	tok       token
	lit       string
	kind      LitKind
	op        Operator
	prec      int
}
```

`src/cmd/compile/internal/syntax/tokens.go` 文件中定义了Go语言中支持的全部Token类型，所有的token类型都是正整数
**提示**： `const _ type = iota` 中iota的意思是当前的常量初始化为当前常量在常量集合中的位置， 只要定义一次后续的常量也会按照这个逻辑处理

```
const (
	_    token = iota
	_EOF       // EOF

	// operators and operations
	_Operator // op
	// ...

	// delimiters
	_Lparen    // (
	_Lbrack    // [
	// ...

	// keywords
	_Break       // break
	// ...
	_Type        // type
	_Var         // var

	tokenCount //
)
```

词法分析主要是由 `scanner` 结构体的 next 方法来驱动

```go
func (s *scanner) next() {
	c := s.getr()
	for c == ' ' || c == '\t' || c == '\n' || c == '\r' {
		c = s.getr()
	}

	s.line, s.col = s.source.line0, s.source.col0

	if isLetter(c) || c >= utf8.RuneSelf && s.isIdentRune(c, true) {
		s.ident()
		return
	}

	switch c {
	case -1:
		s.tok = _EOF

	case '0', '1', '2', '3', '4', '5', '6', '7', '8', '9':
		s.number(c)

	// ...
	}
}
```

但是这个 next 方法是惰性的， 只有上层解析器需要时才会调用next获取最新的Token，与他的类型一起返回给上层

早期的 Go 语言虽然使用 lex 这种工具来生成词法解析器，但是最后还是使用 Go 来实现词法分析器，用自己写的词法分析器来解析自己

### 语法分析

语法分析是根据某种特定形式文法对Token序列构成的输入文本进行分析并确定其语法结构的一种过程。词法分析器输出的结果-Token序列会成为语法分析器的输入

语法分析的过程会使用**自顶向下**或者**自底向上**的方式进行推导

**提示**: 终结符是文法中无法再被展开的符号， 而非终结符与之相反，还可以通过生产规则进行展开

- 𝑁  有限个非终结符的集合
- Σ  有限个终结符的集合
- 𝑃 有限个生产规则12的集合
- 𝑆  非终结符集合中唯一的开始符号

文法被定义成上述的四元祖，其中最重要的是生产规则，每一个生产规则都会包含非终结符、终结符或者开始符号

以下是Go语言的部分文法生成规则

```go
SourceFile = PackageClause ";" { ImportDecl ";" } { TopLevelDecl ";" } .
PackageClause  = "package" PackageName .
PackageName    = identifier .

ImportDecl       = "import" ( ImportSpec | "(" { ImportSpec ";" } ")" ) .
ImportSpec       = [ "." | PackageName ] ImportPath .
ImportPath       = string_lit .

TopLevelDecl  = Declaration | FunctionDecl | MethodDecl .
Declaration   = ConstDecl | TypeDecl | VarDecl .
```

因为每个Go源代码文件最终都会被解析成一个独立的抽象语法树， 所以语法树最顶层的结构或者开始符号其实就是 `SourceFile`， 从 `SourceFile` 相关的生产规则我们可以看出，每一个文件都包含一个`package` 的定义以及可选的`import` 声明和其他的顶层声明(TopLevelDecl)， 每一个`SourceFile` 再编译器中都对应一个`File`结构体， 你能从它们的定义中轻松找到两者的联系：

```go
type File struct {
    PkgName   *Name
    DeclList  []Decl
    Lines     uint
    node
```

顶层声明有五大类型，分别是常量、类型、变量、函数和方法

```go
ConstDecl = "const" ( ConstSpec | "(" { ConstSpec ";" } ")" ) .
ConstSpec = IdentifierList [ [ Type ] "=" ExpressionList ] .

TypeDecl  = "type" ( TypeSpec | "(" { TypeSpec ";" } ")" ) .
TypeSpec  = AliasDecl | TypeDef .
AliasDecl = identifier "=" Type .
TypeDef   = identifier Type .

VarDecl = "var" ( VarSpec | "(" { VarSpec ";" } ")" ) .
VarSpec = IdentifierList ( Type [ "=" ExpressionList ] | "=" ExpressionList ) .
FunctionDecl = "func" FunctionName Signature [ FunctionBody ] .
FunctionName = identifier .
FunctionBody = Block .

MethodDecl = "func" Receiver MethodName Signature [ FunctionBody ] .
Receiver   = Parameters .

Block = "{" StatementList "}" .
StatementList = { Statement ";" } .

Statement =
	Declaration | LabeledStmt | SimpleStmt |
	GoStmt | ReturnStmt | BreakStmt | ContinueStmt | GotoStmt |
	FallthroughStmt | Block | IfStmt | SwitchStmt | SelectStmt | ForStmt |
	DeferStmt .

SimpleStmt = EmptyStmt | ExpressionStmt | SendStmt | IncDecStmt | Assignment | ShortVarDecl .
```

这些不同的语法结构共同定义了Go语言中能够使用的语法结构和表达式

#### 分析方法

- 自顶向下分析：可以被看作找到当前输入流最左推导的过程，对于任意一个输入流，根据当前的输入符号，确定一个生产规则，使用生产规则右侧的符号替代相应的非终结符向下推到
- 自底向上分析：语法分析器从输入流开始，每次都尝试重写最右侧的多个符号，这其实就是说解析器会从最简单的符号进行推导，在解析的最后合并成开始符号

Go 语言的解析器使用LALR文法来解析词法分析过程中输出的Token序列，最右推导加向前查看构成了Go语言解析器的最基本原理，也是大多数编程语言的选择
编译器有个主函数，该函数调用parseFiles就会对使用多个Goroutine来解析源文件，解析的过程就会使用`Parse`函数，该函数初始化了一个新的`parser`结构体并通过`fileOrNil`方法开启对当前文件的词法和语法解析

```go
func Parse(base *PosBase, src io.Reader, errh ErrorHandler, pragh PragmaHandler, mode Mode) (_ *File, first error) {
	var p parser
	p.init(base, src, errh, pragh, mode)
	p.next()
	return p.fileOrNil(), p.first
}
```

`fileOrNil`方法其实就是对上面介绍的Go语言文法的实现， 该方法首先会解析文件开头的package定义

```go
// SourceFile = PackageClause ";" { ImportDecl ";" } { TopLevelDecl ";" } .
func (p *parser) fileOrNil() *File {
	f := new(File)
	f.pos = p.pos()

	if !p.got(_Package) {
		p.syntaxError("package statement must be first")
		return nil
	}
	f.PkgName = p.name()
	p.want(_Semi)
```

从上面的这一段方法中我们可以看出，当前方法会通过`got`来判断下一个Token时不时`package`关键字， 如果是`package` 关键字， 就会执行 `name` 来匹配一个包名并将结果保存到返回的文件结构体中

```go
	for p.got(_Import) {
		f.DeclList = p.appendGroup(f.DeclList, p.importDecl)
		p.want(_Semi)
	}
```

确定了当前文件的报名之后，就开始解析可选的`import`声明， 每一个`import` 在解析器看来都是一个声明语句，这些声明语句都会被加入到文件的`DeclList`列表中

在这之后就会根据编译器获取的关键字进入switch的不同分支，这些分支调用 `appendGroup`方法并在方法中传入用于处理对应类型语句的`constDecl`、`typeDecl`函数

```go
	for p.tok != _EOF {
		switch p.tok {
		case _Const:
			p.next()
			f.DeclList = p.appendGroup(f.DeclList, p.constDecl)

		case _Type:
			p.next()
			f.DeclList = p.appendGroup(f.DeclList, p.typeDecl)

		case _Var:
			p.next()
			f.DeclList = p.appendGroup(f.DeclList, p.varDecl)

		case _Func:
			p.next()
			if d := p.funcDeclOrNil(); d != nil {
				f.DeclList = append(f.DeclList, d)
			}
		}
	}

	f.Lines = p.source.line

	return f
}
```

`fileOrNil` 方法使用了非常多的子方法对输入的文件进行语法分析，并在最后会返回文件最开始创建的`File`结构体

词法分析器`scanner`作为结构体被嵌入到了`parser`中，所以我们看不到词法分析的过程，`parser`的`p.next()`方法实际上是调用了 `scanner`的`next`方法，直接从输入流当中提取下一个Token， 所以词法分析和语法分析在Go语言中是一起进行的

`fileOrNil`、`constDecl`等方法对应了一个Go语言中的生产规则，例如`fileOrNil`实现的就是

```go
SourceFile = PackageClause ";" { ImportDecl ";" } { TopLevelDecl ";" } .
```

我们根据这个规则就能很好地理解语法分析器`parser`的实现原理 - 将编程语言的所有生产规则映射到对应的方法上，这些方法构成的树形结构最终会返回一个抽象语法树

##### 辅助方法

```go
func (p *parser) got(tok token) bool {
	if p.tok == tok {
		p.next()
		return true
	}
	return false
}

func (p *parser) want(tok token) {
	if !p.got(tok) {
		p.syntaxError("expecting " + tokstring(tok))
		p.advance()
	}
}
```

`got` 是用于快速判断一些语句中的关键字，如果当前解析器中的Token是传入的Token就会直接跳过该Token并返回true；而`want`就是对`got`的简单分装了，如果当前的Token不是我们期望的，就会立刻返回语法错误并结束这次编译

```go
func (p *parser) appendGroup(list []Decl, f func(*Group) Decl) []Decl {
	if p.tok == _Lparen {
		g := new(Group)
		p.list(_Lparen, _Semi, _Rparen, func() bool {
			list = append(list, f(g))
			return false
		})
	} else {
		list = append(list, f(nil))
	}

	return list
}
```

`appendGroup` 方法会调用传入的f方法对输入流进行匹配并将匹配的结果追加到另一个参数`File`结构体中的`DeclList`数组中， `import`、`const`、`var`、`type`和`func`声明语句都是调用`appendGroup`方法进行解析的

##### 节点

语法分析器最终会使用不同的结构体来构建抽象语法树中的节点,其中`File`结构体是根节点

```go
type File struct {
	PkgName  *Name
	DeclList []Decl
	Lines    uint
	node
}

type node struct {
	// commented out for now since not yet used
	// doc  *Comment // nil means no comment(s) attached
	pos Pos
}

type (
	Decl interface {
		Node
		aDecl()
	}

	FuncDecl struct {
		Attr   map[string]bool
		Recv   *Field
		Name   *Name
		Type   *FuncType
		Body   *BlockStmt
		Pragma Pragma
		decl
	}
}

type Node interface {
	// Pos() returns the position associated with the node as follows:
	// 1) The position of a node representing a terminal syntax production
	//    (Name, BasicLit, etc.) is the position of the respective production
	//    in the source.
	// 2) The position of a node representing a non-terminal production
	//    (IndexExpr, IfStmt, etc.) is the position of a token uniquely
	//    associated with that production; usually the left-most one
	//    ('[' for IndexExpr, 'if' for IfStmt, etc.)
	Pos() Pos
	aNode()
}

type Pos struct {
	base      *PosBase
	line, col uint32
}

type PosBase struct {
	pos       Pos
	filename  string
	line, col uint32
}
```

函数的主体其实就是一个`Stmt`数组，`Stmt`是一个接口，实现该接口的类型其实也非常多，总共有14种不同类型的`Stmt`实现：

- EmptyStmt
- LabeledStmt
- BlockStmt
- ExprStmt
- SendStmt
- DeclStmt
- AssignStmt
- BranchStmt
- CallStmt
- ReturnStmt
- IfStmt
- ForStmt
- SwitchStmt
- SelectStmt

这些不同类型的`Stmt`构成了全部命令式的Go语言代码，从中我们可以看到非常多熟悉的控制结构

## 类型检查

### 强弱类型

- 强类型：强类型的编程语言在编译期间会有着更严格的类型限制，也就是编译器会在编译期间发现变量赋值、返回值和函数调用时的类型错误
- 弱类型：弱类型的语言在出现类型错误时可能会在运行时进行隐式的类型转换，在类型转换时可能会造成运行错误
**结论**： Go 语言因为会在编译期间发现类型错误，所以也应该是强类型的编程语言

### 静态与动态类型

- 静态类型检查：静态类型检查是基于对源代码的分析来确定运行程序类型安全的过程3，如果我们的代码能够通过静态类型检查，那么当前程序在一定程度上就满足了类型安全的要求，它也可以被看作是一种代码优化的方式，能够减少程序在运行时的类型检查
- 动态类型检查：动态类型检查就是在运行时确定程序类型安全的过程，这个过程需要编程语言在编译时为所有的对象加入类型标签和信息，运行时就可以使用这些存储的类型信息来实现动态派发、向下转型、反射以及其他特性。只使用动态类型检查的编程语言就叫做动态类型编程语言，常见的动态类型编程语言就包括 JavaScript、Ruby 和 PHP，这些编程语言在使用上非常灵活也不需要经过编译器的编译，但是有问题的代码该出错还是出错并不会因为更加灵活就会减少错误

### 执行过程

Go 语言的编译器不仅使用静态类型检查来保证程序运行的类型安全，还会在编程期引入类型信息，让工程师能够使用反射来判断参数和变量的类型，当我们想要将 interface 转换成具体类型时就会进行动态类型检查，如果无法发生转换就可能会造成程序崩溃。

#### 切片 OTARRAY

如果当前节点的操作类型是 OTARRAY，那么这个分支首先会对右节点，也就是切片或者数组中元素的类型进行类型检查

```go
case OTARRAY:
		r := typecheck(n.Right, Etype)
		if r.Type == nil {
			n.Type = nil
			return n
		}
```

然后该分支会根据当前节点的左节点不同，分三种不同的情况更新当前 Node 的类型，也就是三种不同的声明方式 []int、[...]int 和 [3]int

- `[]int`

```go
        if n.Left == nil {
			t = types.NewSlice(r.Type)
```

- `[...]int`

```go
           } else if n.Left.Op == ODDD {
			if top&ctxCompLit == 0 {
				if !n.Diag() {
					n.SetDiag(true)
					yyerror("use of [...] array outside of array literal")
				}
				n.Type = nil
				return n
			}
			// t.Extra = &Array{Elem: r.Type, Bound: -1}
			t = types.NewDDDArray(r.Type)
```

- `[3]int`

```go
        } else {
			n.Left = indexlit(typecheck(n.Left, ctxExpr))
			l := n.Left
			v := l.Val()
			bound := v.U.(*Mpint).Int64()
			// t.Extra = &Array{Elem: r.Type, Bound: bound}
			t = types.NewArray(r.Type, bound)
        }
```

通过对这个片段的分析，我们发现数组的长度是类型检查期间确定的，而 [...]int 这种声明形式也只是 Go 语言为我们提供的语法糖

#### 哈希 OTMAP

与 slice 的一个重要差异是， 会检查map的key类型可否哈希。检查方法是初步完成类型检查之后把节点添加到 mapqueue 队列中， 编译器会在后面的阶段对哈希键的类型进行再次检查，而检查键类型调用的其实就是上面提到的 `checkMapKeys`

```go
func checkMapKeys() {
	for _, n := range mapqueue {
		k := n.Type.MapType().Key
		if !k.Broke() && !IsComparable(k) {
			yyerrorl(n.Pos, "invalid map key type %v", k)
		}
	}
	mapqueue = nil
}
```

#### 关键字 OMAKE

`make`关键字会在类型检查期间会根据创建的类型将`make`转换成特定的函数，后面生成中间代码的过程就不再会处理`OMAKE`类型的节点了，而是会根据这里生成的更加细分的操作类型进行处理

- make
    - makechan 用于创建信道
    - makeslice  用于创建切片
    - makemap    用于创建哈希空间

### 小结

类型检查是 Go 语言编译的第二个阶段，在词法和语法分析之后我们得到了每个文件对应的抽象语法树，随后的类型检查会遍历抽象语法树中的节点，对每个节点的类型进行检验，找出其中存在的语法错误，在这个过程中也可能会对抽象语法树进行改写，这不仅能够去除一些不会被执行的代码、对代码进行优化以提高执行效率，而且也会修改 make、new 等关键字对应节点的操作类型。

make 和 new 这些内置函数其实并不直接对应某些函数的实现，它们会在编译期间被转换成真正存在的其他函数

## 中间代码

背景： 很多编译器可能需要将一种源代码翻译成多种机器码，想要在高级的编程语言上直接进行翻译是比较困难的，但是我们使用中间代码就可以将我们的问题简化

1. 将编程语言直接翻译成机器码的过程拆成两个简单步骤 -- 中间代码的生成和机器码的生成
2. 中间代码是一种更接近机器语言的表示形式，对中间代码的优化和分析相比直接分析高级编程语言更容易

### 配置初始化

SSA(静态单赋值)配置的初始化过程会缓存可能用到的类型指针、初始化SSA配置和一些之后会调用的运行时函数， 例如： 用于处理`defer`关键字的`deferproc`、用于创建Goroutine的`newproc`和扩容切片的`growslice`等除此之外还会根据当前的目标设备初始化特定的ABI（application binary interface）

1. 缓存程序需要用到的类型的指针， 并赋值给当前类型中，优化类型指针的获取效率
2. 根据当前的CPU架构初始化SSA配置， `NewConfig`函数会根据传入的CPU架构设置用于生成中间代码和机器码的函数，当前编译器使用的指针、寄存器大小、可用寄存器列表、掩码等编译选项。所有的配置项一旦被创建，在整个编译期间都是只读的并且被全部编译阶段共享，也就是中间代码生成和机器码生成这两部分都会使用这一份配置完成自己的工作。在`initssaconfig`方法调用的最后，会初始化一些编译器会用到的Go语言运行时的函数
```
    assertE2I = sysfunc("assertE2I")
	assertE2I2 = sysfunc("assertE2I2")
	assertI2I = sysfunc("assertI2I")
	assertI2I2 = sysfunc("assertI2I2")
	deferproc = sysfunc("deferproc")
	Deferreturn = sysfunc("deferreturn")
```


### 遍历和替换

在生成中间代码之前，我们还需要对抽象语法树中节点的一些元素进行替换，这个替换的过程就是通过`walk`和很多以`walk`开头的相关函数实现的，简单展示几个相关函数的签名

```go
func walk(fn *Node)
func walkappend(n *Node, init *Nodes, dst *Node) *Node
...
func walkrange(n *Node) *Node
func walkselect(sel *Node)
func walkselectcases(cases *Nodes) []*Node
func walkstmt(n *Node) *Node
func walkstmtlist(s []*Node)
func walkswitch(sw *Node)
```

这些用于遍历抽象语法树的函数会将一些关键字和内建函数转换成函数调用，例如:`panic` -> `gopanic`， `recover` -> `gorecover` ， 关键字 `new` 也会在这里被转换成对 `newobject` 函数的调用

### SSA生成

经过`walk`系列函数的处理之后， AST 结构就不再会改变了，Go语言的编译器会使用 `compileSSA` 函数将抽象语法树转换成中间代码

首先，从AST到SSA的转化过程中，编译器会生成将函数调用的参数放到栈上的中间代码，处理参数之后才会生成一条运行函数的命令 `ssa.OpStaticCall`：

1. 如果这里使用的是defer关键字，就会插入`deferproc`函数
2. 如果使用go创建新的Goroutine时会插入`newproc`函数符号
3. 在遇到其他情况时会插入表示普通函数对应的符号

## 机器码生成

Go语言编译的最后一个阶段就是根据SSA中间代码生成机器码了，这里谈的机器码就是在目标CPU架构上能够运行的二进制代码

机器码的生成过程其实就是对SSA中间代码的降级过程， 在SSA中间代码降级的过程中，编译器将一些值重写成了目标CPU架构的特定值，降级的过程处理了所有机器特定的重写规则并对代码进行了一定程度的优化；在SSA中间代码生成阶段的最后，Go函数题的代码会被转换成一系列的obj.Prog结构体

- `src/cmd/compile/internal/ssa` 主要负责对 SSA 中间代码进行降级、执行架构特定的优化和重写并生成 obj.Prog 指令
- `src/cmd/internal/obj` 作为一个汇编器会将这些指令最终转换成机器码完成这次的编译

### 小结

机器码生成作为 Go 语言编译的最后一步，其实已经到了硬件和机器指令这一层，其中对于内存、寄存器的处理非常复杂并且难以阅读，想要真正掌握这里的处理的步骤和原理还是需要非常多的精力，但是作为软件工程师来说，如果不是 Go 语言编译器的开发者或者需要经常处理汇编语言和机器指令，掌握这些知识的投资回报率实在太低，没有太多的必要，我们在这里只需要对这个过程有所了解，补全知识上的盲点，这样在遇到问题时能够快速定位