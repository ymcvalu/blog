---
layout: go
title: ast
date: 2019-06-19 16:04:07
tags:
	- golang
---

# go ast

### 相关包

`go`的官方库提供了几个包，可以帮我们解析`go`的源文件，主要有：

- go/scanner：词法解析，将源代码分割成一个个token
- go/token：token类型及相关结构体定义
- go/ast：ast的结构定义及相关操作方法
- go/parser：语法分析，读取token流生成ast



### 为什么需要使用解析源文件

通过解析源文件，我们可以得到`ast`(抽象语法树)。

而通过遍历`ast`，我们可以得到源码中声明的结构体、方法、类型等等信息，并根据实际需要生成具体的代码，比如自动生成`tag`，模板方法、手动实现泛型效果等。而且，go的注释在解析时是可以保留的，这就可以实现`java`中类似`annotation`的功能，比如根据注释自动生成接口文档（beego的swagger），根据注释提取接口权限信息实现统一权限校验等。



### ast: 抽象语法树

ast是源代码结构的一种抽象表示，以树状形式来表达编程语言的语法结构。

比如表达式 `a+b`，对应的ast为：

![](/img/ast1.png)

对应使用go表示的结构：

```go
*ast.BinaryExpr { // a+b是一个二元表达式
.  X: *ast.Ident { // X表示第一个操作数
.  .  NamePos: 1
.  .  Name: "a"
.  .  }
.  }
.  OpPos: 2
.  Op: + // 操作符
.  Y: *ast.Ident { // Y表示第二个操作数
.  .  NamePos: 3
.  .  Name: "b"
.  }
}
```



### 源码解析

首先要知道具体的接口怎么用，才知道源码从哪个入口开始看是吧

```go
package main

import (
	"go/ast"
	"go/parser"
	"go/token"
	"log"
)

func main() {
	// 创建FileSet
	fset := token.NewFileSet()
	// 解析源文件main.go，返回ast.File代表一个源文件的node
	f, err := parser.ParseFile(fset, "./main.go", nil, parser.ParseComments)
	if err != nil {
		log.Fatal(err)
	}
	// 打印AST
	ast.Print(fset, f)
}
```

首先来看第12行代码，这里创建了一个`FileSet`，顾名思义，`FileSet`就是源文件集合，因为我们一次解析可能不止解析一个文件，而是一系列文件。

`FileSet`最主要的用途是用来保存`token`的位置信息，每个token在当前文件的位置可以用行号，列号，token在当前文件中的偏移量这三个属性来描述，使用`Position`这个结构体来描述，`FileSet`中保存所有`token`的`Position`信息，而在`ast`中，只保存一个`Pos`索引。当遍历`ast`的时候，我们需要使用`Pos`索引向`FileSet`获取`Position`。

现在来看一下14行`parser.ParseFile`这个方法，这个方法实现了语法分析。

词法分析是和语法分析同时进行的，`go`的语法分析，总体上基于`LL1`从左到右，自上而下，最多向前看一个token来推导文法，但是有个别地方存在文法二义性的情况，比如当遇到一个`<-`的时候，可能是`<- chan type`，表示声明一个只读channel，也可能是`<- expr`，表示从一个channel中读取，这时候会先解析`<-`后面的内容，然后根据具体`node`的类型来决定当前是使用哪个文法进行推到。

```go
func ParseFile(fset *token.FileSet, filename string, src interface{}, mode Mode) (f *ast.File, err error) {
	// 必须要传入fset，用来保存Position信息
    if fset == nil {
		panic("parser.ParseFile: no token.FileSet provided (fset == nil)")
	}

	// 读取源文件，如果src不为空，则从src读取，否则读取filename指定的文件
	text, err := readSource(filename, src)
	if err != nil {
		return nil, err
	}

	var p parser 
	defer func() {
		...
	}()

	// parse source
	p.init(fset, filename, text, mode) // 初始化parser
	f = p.parseFile() // 解析源文件，生成AST

	return
}
```

先来简单看一下`parser.init`方法：

```go
func (p *parser) init(fset *token.FileSet, filename string, src []byte, mode Mode) {
	// 添加当前文件到FileSet中
    p.file = fset.AddFile(filename, -1, len(src))
	var m scanner.Mode
    // 设置scanner的mode，如果指定了ast需要保留注释，那么词法解析的时候需要解析注释
	if mode&ParseComments != 0 {
		m = scanner.ScanComments
	}
    // 错误处理
	eh := func(pos token.Position, msg string) { p.errors.Add(pos, msg) }
	// 初始化词法解析器
    p.scanner.Init(p.file, src, eh, m)
	p.mode = mode
	p.trace = mode&Trace != 0 // for convenience (p.trace is used frequently)
	// parser.next会前进到下一个非注释的token，其中注释会被保留
	p.next()
}
```

其中，注释有两种：

```go
// this
// is
// doc
func foo(){
    
}
```

```go
var globalNum int // this is a comment
```

第一种是注释独自自己占一到多行的，后一种则是跟语句在同一行。`parser.next`方法中，读取`token`时，如果遇到第一种注释，会保存到`parser.leadComment`，如果是第二种注释，则保存到`parser.lineComment`中，最终会保留到具体的`ast`中的节点中。

接着来看一下`parser.parseFile`方法

```go
func (p *parser) parseFile() *ast.File {
	// 如果执行parser.next时有错误发生
	if p.errors.Len() != 0 {
		return nil
	}

	// package clause
	doc := p.leadComment // package前面的注释被认为是当前文件的doc
    // 期待第一个token是`package`关键字，该方法内会执行parser.next方法，前进到下一个token
	pos := p.expect(token.PACKAGE) 
	// 解析当前的token为标识符，也就是包名
	ident := p.parseIdent()
	if ident.Name == "_" && p.mode&DeclarationErrors != 0 {
		p.error(p.pos, "invalid package name _")
	}
    // 读取`;`，如果没有的话，需要插入一个`;`
    // 也就是说go会自动在语句末尾插入`;`
	p.expectSemi()

	// 如果前面解析标识符时失败
	if p.errors.Len() != 0 {
		return nil
	}

    // 设置topScope
    // scope用于保存当前作用域内声明的符号引用，比如声明的方法、类型或常/变量等
	p.openScope()
    // 设置包作用域
	p.pkgScope = p.topScope
    // 一个源文件是由一系列声明组成的:
    // import声明
    // 方法声明
    // 类型声明
    // 全局常量/变量声明
    // 这里的ast.Decl是这些声明的公共接口
	var decls []ast.Decl
	// 如果不是只解析包名
    if p.mode&PackageClauseOnly == 0 {
		// 解析导入声明
        // 确保当前token的`import`
		for p.tok == token.IMPORT {
            // p.parserImportSpec解析具体的导入声明
			decls = append(decls, p.parseGenDecl(token.IMPORT, p.parseImportSpec))
		}
		// 如果不是只解析导入声明
		if p.mode&ImportsOnly == 0 {
			// 解析源代码后面的其他内容
			for p.tok != token.EOF {
				decls = append(decls, p.parseDecl(declStart))
			}
		}
	}
    
    // 关闭作用域
	p.closeScope()
    // 确保topScope为nil，否则说明有多余的`{}`没有匹配
	assert(p.topScope == nil, "unbalanced scopes")
	assert(p.labelScope == nil, "unbalanced label scopes")

	// resolve global identifiers within the same file
	i := 0
    // 在包作用域内查找未解析的符号引用，比如在方法内引用了全局的方法，变量等
	for _, ident := range p.unresolved {
		// i <= index for current ident
		assert(ident.Obj == unresolved, "object already resolved")
		ident.Obj = p.pkgScope.Lookup(ident.Name) // also removes unresolved sentinel
        // 有的是在同一个包的其他文件中声明的
		if ident.Obj == nil {
			p.unresolved[i] = ident
			i++
		}
	}

	return &ast.File{
		Doc:        doc,
		Package:    pos,
		Name:       ident,
		Decls:      decls,
		Scope:      p.pkgScope,
		Imports:    p.imports,
		Unresolved: p.unresolved[0:i],
		Comments:   p.comments,
	}
}
```



### 来个例子

现在来实现一个自动生成`tag`的例子

```go
func main() {
    // 使用时需要传入目标源代码路径，目标结构体包含的某个行号和列号
	args := os.Args[:len(os.Args)]
	if len(args) < 4 {
		log.Fatal("参数：文件路径，行号，列号")
	}
	fpath := args[1]
	lineNum, err := strconv.Atoi(args[2])
	if err != nil {
		log.Fatal("incorrect line number")
	}
	// columnNum, err := strconv.Atoi(args[3])
	if err != nil {
		log.Fatal("incorrect column number")
	}
	// 创建FileSet
	fset := token.NewFileSet()

	// 解析源文件
	f, err := parser.ParseFile(fset, fpath, nil, parser.ParseComments)
	if err != nil {
		log.Fatal("failed to parse file: ", err.Error())
	}
	// 全局变量，用来保存找到的目标结构体声明的具体node
	var target *ast.StructType
    // 使用Inspect方法遍历ast
	ast.Inspect(f, func(node ast.Node) bool {
		// 如果不是结构体类型声明，跳过，继续下一个遍历
		st, ok := node.(*ast.StructType)
        // 如果不是结构体类型或者类型声明未完成
		if !ok || st.Incomplete {
			return true
		}
		// 如果是结构体声明，需要包含指定的行和列，这里实际上只要包含实际行就行
		begin := fset.Position(st.Pos())
		end := fset.Position(st.End())

		// 找到目标struct，返回false，结束遍历
		if begin.Line <= lineNum && end.Line >= lineNum {
			target = st // 设置目标target
 			return false
		}

		return true
	})

    // 如果找到了目标trget
	if target != nil {
        // 生成tag，因为结构体声明是可以嵌套的，该方法会递归调用
		genTag(target)
        // 打开目标文件
		fd, err := os.OpenFile(fpath, os.O_TRUNC|os.O_RDWR, 0777)
		if err != nil {
			log.Fatal(err)
		}
		defer fd.Close()
        // 使用format.Node方法将ast转换为源文件
		err = format.Node(fd, fset, f)
		if err != nil {
			log.Fatal(err)
		}
	}
}
```

接着来看一下`genTag`方法，该放方法主要就是遍历声明的字段，为其生成tag然后设置到ast中对应的node上

```go
func genTag(st *ast.StructType) {
    // 遍历结构体声明的字段列表
	fs := st.Fields.List
	for i := range fs {
		var (
			tag string
		)

		fd := fs[i]
        // 如果有指定字段名
		if len(fd.Names) > 0 {
			name := fd.Names[0].Name
            // 只有导出字段才需要生成tag
			if !isExport(name) {
				continue
			}
            // 根据字段名生成tag中的名字，比如NodeId变成node_id
			tag = genKey(name)
		}
		// 判断字段的类型
		switch t := fd.Type.(type) {
        // 如果是标识符标识引用了其他声明类型
		case *ast.Ident:
            // 如果tag==""表示没有字段名，这时候默认字段名就是类型名
            // 如果类型导出，则生成tag
			if tag == "" && isExport(t.Name) {
				tag = genKey(t.Name)
			}
        // 嵌套结构体声明
		case *ast.StructType:
			// 递归生成tag
            genTag(t)
		}

		var tagStr string
        // 获取原来的tag
		if fd.Tag != nil {
			tagStr = fd.Tag.Value
		}
	
        // 解析tag字符串：`json:"sdf" form:"sdf"`成tag切片
		tags, err := parseTag(tagStr)
		if err != nil {
			log.Fatal(err)
		}
	
		change := false
		// 如果已经存在json这个tag，则跳过自动生成
        if _, ok := tags.Lookup("json"); !ok {
			tags.Append("json", tag)
			change = true
		}
        // 如果已经生成form这个tag，跳过
		if _, ok := tags.Lookup("form"); !ok {
			tags.Append("form", tag)
			change = true
		}
	
        // 如果自动生成了tag
		if change {
            // 根据新的tag切片生成tag字符串
			tagStr = tags.TagStr()
			if fd.Tag == nil {
				fd.Tag = &ast.BasicLit{}
			}
            // 设置到目标node中
			fd.Tag.Kind = token.STRING
			fd.Tag.Value = tagStr
		}
	}
}
```

[完整代码](<https://github.com/ymcvalu/auto-tag>)