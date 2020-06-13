---
title: SwiftSyntax详解
author: Roy
date: 2020-06-14 00:34:00 +0800
categories: [SwiftSyntax]
tags: [Swift]
pin: true
---



[SwiftSyntax](https://github.com/apple/swift-syntax)是基于[libSyntax](https://github.com/apple/swift/tree/master/lib/Syntax)构建的Swift库，利用它可以分析，生成和转换Swift代码。现在已经有一些基于它开源的库，比如[SwiftRewriter](https://github.com/inamiy/SwiftRewriter)针对代码进行自动格式化(其中包括基于代码规范进行简单的代码优化)。


# Swift 编译器

Swift编译器分为前端和后端，LLVM架构下的都是如此（Objective-C编译器的前端是Clang，后端是也是LLVM），下图是Swift的编译器结构：

![](/assets/img/2020/SwiftSyntax详解/swift-compilation-diagram.png)

下面来解释一下各个阶段

| 阶段 |解释| 作用 |
| ------ | ------ | ------ |
| Parse | 语法分析  | 语法分析器对Swift源码进行逐字分析，生成不包含语义和类型信息的抽象语法树，简称AST(Abstract Syntax Tree)。这个阶段生成的AST也不包含警告和错误的注入。|
| Sema| 语义分析| 语义分析器会进行工作并生成一个通过类型检查的AST，并且在源码中嵌入警告和错误等信息|
| SILGen | Swift中级语言生成 | Swift中级语言生成（SILGen）阶段将通过语义分析生成的AST转换为Raw SIL，再对Raw SIL进行了一些优化（例如泛型特化，ARC优化等）之后生成了Canonical SIL。SIL是Swift定制的中间语言，针对Swift进行了大量的优化，使得Swift性能得到提升。SIL也是Swift编译器的精髓所在。|
| IRGen | 生成LLVM的中间语言 | 将SIL降级为LLVM IR，LLVM的中间语言|
| LLVM | LLVM编译器架构下的后端 | 前面几个阶段属于Swift编译器，相当于OC中的Clang，属于LLVM编译器架构下的前端，这里的LLVM是编译器架构下的后端，对LLVM IR进一步优化并生成目标文件（.o） |


# SwiftSyntax

SwiftSyntax 的操作目标是编译过程第一步所生成的 AST，从上面了解到AST不包含语义和类型信息，本文的关注点也是AST，其实生成AST需要两步：

第一步，词法分析，也叫做扫描scanner（或者Lexer）。它读取我们的代码，然后把它们按照预定的规则合并成一个个的标识tokens。同时，它会移除空白符，注释等。最后，整个代码将被分割进一个tokens列表（或者说一维数组）。

当词法分析源代码的时候，它会一个一个字母地读取代码，所以很形象地称之为扫描-scans；当它遇到空格，操作符，或者特殊符号的时候，它会认为一个话已经完成了。


![](/assets/img/2020/SwiftSyntax详解/struct.png)

第二步，语法分析，也解析器。它会将词法分析出来的数组转化成树形的表达形式。当生成树的时候，解析器会删除一些没必要的标识tokens（比如不完整的括号），因此AST不是100%与源码匹配的，但是已经能让我们知道如何处理了。

```
xcrun swiftc -frontend -emit-syntax ./Cat.swift | python -m json.tool
```

可以在终端使用这个命令，结果为一串 JSON 格式的 AST，我截取了其中一部分，把开头import相关的移除了。

```
{
    "id": 28,
    "kind": "SourceFile",
    "layout": [
        {
            "id": 27,
            "kind": "CodeBlockItemList",
            "layout": [
                {
                    "id": 25,
                    "kind": "CodeBlockItem",
                    "layout": [
                        {
                            "id": 24,
                            "kind": "StructDecl",
                            "layout": [
                                null,
                                null,
                                {
                                    "id": 7,
                                    "leadingTrivia": [
                                        {
                                            "kind": "Newline",
                                            "value": 2
                                        }
                                    ],
                                    "presence": "Present",
                                    "tokenKind": {
                                        "kind": "kw_struct"
                                    },
                                    "trailingTrivia": [
                                        {
                                            "kind": "Space",
                                            "value": 1
                                        }
                                    ]
                                },
                                {
                                    "id": 8,
                                    "leadingTrivia": [],
                                    "presence": "Present",
                                    "tokenKind": {
                                        "kind": "identifier",
                                        "text": "Cat"
                                    },
                                    "trailingTrivia": [
                                        {
                                            "kind": "Space",
                                            "value": 1
                                        }
                                    ]
                                },
                                null,
                                null,
                                null,
                                {
                                    "id": 23,
                                    "kind": "MemberDeclBlock",
                                    "layout": [
                                        {
                                            "id": 9,
                                            "leadingTrivia": [],
                                            "presence": "Present",
                                            "tokenKind": {
                                                "kind": "l_brace"
                                            },
                                            "trailingTrivia": []
                                        }
         
```

## SwiftSyntax内部构造

### RawSyntax

RawSyntax是所有Syntax的原始不可变后备存储，表示语法树基础的原始树结构。这些节点没有身份的概念，仅提供树的结构。它们是不可变的，可以在语法节点之间自由共享，因此它们不维护任何父母关系。最终，RawSyntax在以TokenSyntax类表示的Token中达到最低点，也就是叶子节点。

* RawSyntax 是所有语法的不可变后备存储。
* RawSyntax 是不可变的。
* RawSyntax 建立语法的树结构。
* RawSyntax 不存储任何父母关系，因此如果语法节点具有相同的内容，则可以在语法节点之间共享它们。

```
final class RawSyntax: ManagedBuffer<RawSyntaxBase, RawSyntaxDataElement> {
	let data: RawSyntaxData
	var presence: SourcePresence
}

/// 特定树或者Token节点的数据
fileprivate enum RawSyntaxData {
  /// 一个token，包含tokenKind，leading trivia, and trailing trivia
  case token(TokenData)
  /// 一个树节点，包含syntaxKind和一个子节点数组
  case layout(LayoutData)
}
```

### Trivia

Trivia与程序的语义无关，以下是一些Trivia的“原子”例子： 

* 空格
* 标签
* 换行符
* // 注释
* /* ... */ 注释
* /// 注释
* /** ... */ 注释
* \` \` 反引号

解析或构造新的语法节点时，应遵循以下两个Trivia规则：  

1. Trailing trivia: 一个Token拥有它之后的所有Trivia，直到遇到下一个换行符，并且不包含这个换行符。
2. Leading trivia: 一个Token拥有它之前的所有Trivia，直到遇到第一个换行符，并且包含这个换行符。

**例子**

```swift
func foo() {
  var x = 2
}
```
我们来逐个Token分解

* `func`
 + Leading trivia: 无
 + Trailing trivia: 占有之后的一个空格（根据规则1）
 
	 ``` 
	 // Equivalent to:
	Trivia::spaces(1)
	 ```
	 
* `foo`
 + Leading trivia: 无，前一个`func`占有了这个空格
 + Trailing trivia: 无

*  `(`
 + Leading trivia: 无
 + Trailing trivia: 无

*  `)`
 + Leading trivia: 无
 + Trailing trivia: 占有之后的一个空格（根据规则1）

*  `{`
 + Leading trivia: 无，前一个`(`占有了这个空格
 + Trailing trivia: 无，不占用下一个换行符（根据规则1）
 
*  `var`
 + Leading trivia: 一个换行符和两个空格（根据规则2）  
 
	   ```    
		  // Equivalent to:
	     Trivia::newlines(1) + Trivia::spaces(2)
	   ```
 + Trailing trivia: 占有之后的一个空格（根据规则1）
 
* `x`
 + Leading trivia: 无，前一个`var`占有了这个空格
 + Trailing trivia: 占有之后的一个空格（根据规则1）
   
* `=`
 + Leading trivia: 无，前一个`x`占有了这个空格
 + Trailing trivia: 占有之后的一个空格（根据规则1）

* `2`
 + Leading trivia: 无，前一个`=`占有了这个空格
 + Trailing trivia: 无，不占用下一个换行符（根据规则1）

* `}`
 + Leading trivia: 一个换行符（根据规则2）
 + Trailing trivia: 无

* `EOF`
 + Leading trivia: 无
 + Trailing trivia: 无


### SyntaxData
它用一些附加信息包装RawSyntax节点：指向父节点的指针，该节点在其父节点中的位置以及缓存的子节点。可以将SyntaxData视为“具体“或“已实现”语法节点。它们代表特定的源代码片段，具有绝对的位置，行和列号等。SyntaxData是每个Syntax节点的基础存储，私有的，不对外暴露。

### Syntax
Syntax表示在叶子上带有Token的节点树，每个节点都有其已知子节点的访问器，并允许通过其`children`属性对子节点进行有效的迭代。

抽象语法树节点的类别有三个类：与声明有关、与表达式有关、与语句有关。Swift也是一样，只不过在实现的时候划分更加细

```
public protocol DeclSyntax: Syntax {}

public protocol ExprSyntax: Syntax {}

public protocol StmtSyntax: Syntax {}

public protocol TypeSyntax: Syntax {}

public protocol PatternSyntax: Syntax {}

```

* DeclSyntax：与声明有关，比如TypealiasDeclSyntax、ClassDeclSyntax、StructDeclSyntax、ProtocolDeclSyntax、ExtensionDeclSyntax、FunctionDeclSyntax、DeinitializerDeclSyntax、ImportDeclSyntax、VariableDeclSyntax、EnumCaseDeclSyntax等等。
* StmtSyntax：与语句有关，比如GuardStmtSyntax、ForInStmtSyntax、SwitchStmtSyntax、DoStmtSyntax、BreakStmtSyntax、ReturnStmtSyntax等等。
* ExprSyntax：与表达式有关，比如StringLiteralExprSyntax、IntegerLiteralExprSyntax、TryExprSyntax、FloatLiteralExprSyntax、TupleExprSyntax、DictionaryExprSyntax等等。
* TypeSyntax：与声明有关，表示类型，TupleTypeSyntax、FunctionTypeSyntax、DictionaryTypeSyntax、ArrayTypeSyntax、ClassRestrictionTypeSyntax、AttributedTypeSyntax等
* PatternSyntax：与模式匹配有关

swift中模式有以下几种：

* 通配符模式（WildcardPatternSyntax）
* 标识符模式（IdentifierPatternSyntax）
* 值绑定模式（ValueBindingPatternSyntax）
* 元组模式（TuplePatternSyntax）
* 枚举用例模式（EnumCasePatternSyntax）
* 可选模式（OptionalPatternSyntax）
* 类型转换模式（AsTypePatternSyntax）
* 表达式模式（ExpressionPatternSyntax）
* 未知模式（UnknownPatternSyntax）

除了以上几种大类型的Syntax，还有其他的Syntax：

* SourceFileSyntax
* FunctionParameterSyntax
* InitializerClauseSyntax
* MemberDeclListItemSyntax
* MemberDeclBlockSyntax
* TypeInheritanceClauseSyntax
* InheritedTypeSyntax
* ......

### SyntaxNode
表示语法树中的节点。这是比Syntax更有效的表示形式，因为它避免了对表示父层次结构的Syntax的强制转换。它提供一般信息，例如节点的位置，范围和`uniqueIdentifier`，同时在必要时仍允许获取关联的`Syntax`对象。`SyntaxParser`使用`SyntaxNode`来有效地报告在增量重新解析期间重新使用了哪些语法节点。

### 示例：{return 1}

这是`{return 1}`示例图的样子。

![](/assets/img/2020/SwiftSyntax详解/SyntaxExample.png)


* 绿色：RawSyntax类型（TokenSyntax也是RawSyntax），这个图图是从[Syntax](https://github.com/apple/swift/tree/master/lib/Syntax#internals)拿来的，图中RawTokenSyntax在SwiftSyntax是TokenSyntax。
* 红色：SyntaxData类型
* 蓝色：Syntax类型
* 灰色：Trivia
* 实心箭头：强引用
* 虚线箭头：弱引用

## SwiftSyntax API

### Make APIs

```swift
let returnKeyword = SyntaxFactory.makeReturnKeyword(trailingTrivia: .spaces(1))
let three = SyntaxFactory.makeIntegerLiteralExpr(digits: SyntaxFactory.makeIntegerLiteral(String(3)))
let returnStmt = SyntaxFactory.makeReturnStmt(returnKeyword: returnKeyword, expression: three)

```
输出  

```swift
return 3

```

### With APIs

with API用于将节点转换为其他节点。 假设我们不返回3，而是希望语句返回“hello”。我们将使用expression方法来调用它，然后传入字符串。

```
let returnHello = returnStmt.withExpression(SyntaxFactory.makeStringLiteralExpr("Hello"))

```


### Syntax Builders

对于每种语法，都有一个对应的构建器结构。这些提供了一种构建语法节点的增量方法。如果我们想从头开始构建该cat结构，只需要四个Token，struct关键字，cat标识符和两个大括号。

```
let structKeyword = SyntaxFactory.makeStructKeyword(trailingTrivia: .spaces(1))
let identifier = SyntaxFactory.makeIdentifier("Cat", trailingTrivia: .spaces(1))

let leftBrace = SyntaxFactory.makeLeftBraceToken()
let rightBrace = SyntaxFactory.makeRightBraceToken(leadingTrivia: .newlines(1))
let members = MemberDeclBlockSyntax { builder in
    builder.useLeftBrace(leftBrace)
    builder.useRightBrace(rightBrace)
}

let structureDeclaration = StructDeclSyntax { builder in
    builder.useStructKeyword(structKeyword)
    builder.useIdentifier(identifier)
    builder.useMembers(members)
}

```

### SyntaxVisitors  
使用SyntaxVisitor，我们可以遍历语法树。当我们想要提取一些信息以对源代码进行分析时，这很有用。

```swift
class FindPublicExtensionDeclVisitor: SyntaxVisitor {

    func visit(_ node: ExtensionDeclSyntax) -> SyntaxVisitorContinueKind {
        if node.modifiers?.contains(where: { $0.name.tokenKind == .publicKeyword }) == true {
            // Do something if you find a `public extension` declaration.
        }
        return .skipChildren
    }
}

```
返回值是一种延续类型，指示是继续并访问语法树上的子节点（.visitChildren）还是跳过它（.skipChildren）

```swift
public enum SyntaxVisitorContinueKind {

  /// The visitor should visit the descendents of the current node.
  case visitChildren

  /// The visitor should avoid visiting the descendents of the current node.
  case skipChildren
}

```

### SyntaxRewriters  
SyntaxRewriter使我们可以通过仅重写visit方法并基于规则返回新节点来修改树的结构。   
**注意**：所有节点都是不可变的，因此我们不修改节点，而是创建另一个节点并将其返回以替换当前节点。

```swift

class PigRewriter: SyntaxRewriter {

    override func visit(_ token: TokenSyntax) -> Syntax {
        guard case .stringLiteral = token.tokenKind else { return token }
        return token.withKind(.stringLiteral("\"🐷\""))
    }
}

```

在这个例子中，我们将代码中的所有字符串替换成表情🐷，官方示例是将所有数字加一，感兴趣可以去github上看看。

# SwiftSyntax的使用

### 生成代码

通常我们不会用SwiftSyntax来大量生成代码，因为这需要写大量代码，工作量巨大，简直让人崩溃😂。我们可以用[GYB](https://github.com/apple/swift/blob/master/utils/gyb.py)，[GYB](https://github.com/apple/swift/blob/master/utils/gyb.py)（模板生成）是一个 Swift 内部使用的工具，可以用模板生成源文件。Swift标准库中的源码的很多代码就是用[GYB](https://github.com/apple/swift/blob/master/utils/gyb.py)生成的，其实SwiftSyntax很多代码也是用[GYB](https://github.com/apple/swift/blob/master/utils/gyb.py)生成的，包括SyntaxBuilders、SyntaxFactory、SyntaxRewriter等等。开源社区中另一个很棒的工具是 [Sourcery](https://github.com/krzysztofzablocki/Sourcery)，它允许你在Swift（通过[Stencil](https://github.com/stencilproject/Stencil)）而不是Python中编写模板，[SwiftGen](https://github.com/SwiftGen/SwiftGen)也是使用[Stencil](https://github.com/stencilproject/Stencil)生成Swift代码的。

### 分析和转换代码
现在有两个不错的库在使用SwiftSyntax，一个是[periphery](https://github.com/peripheryapp/periphery)，检测未使用的Swift代码，比如未使用的Protocol和类，以及他们的方法和方法参数等等。另一个是[SwiftRewriter](https://github.com/inamiy/SwiftRewriter)，Swift代码格式化工具。你也可以写一个Swift语法高亮工具，[SwiftGG](https://swift.gg/2019/01/25/nshipster-swiftsyntax/)有这么个例子。


### 参考：  
[Improving Swift Tools with libSyntax](https://academy.realm.io/posts/improving-swift-tools-with-libsyntax-try-swift-haskin-2017/)   
[An overview of SwiftSyntax](https://medium.com/@lucianoalmeida1/an-overview-of-swiftsyntax-cf1ae6d53494)   
[Swift编译器结构分析](https://www.jianshu.com/p/7c894f9b7b02)  
[libSyntax](https://github.com/apple/swift/tree/master/lib/Syntax) 
[编程语言的实现，从AST（抽象语法树）开始](https://baijiahao.baidu.com/s?id=1626159656211187310&wfr=spider&for=pc)