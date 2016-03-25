---
layout: post-zh
title : PHP Parser 介绍
category: study
tags : parser
---

[PHP Parser](https://github.com/nikic/PHP-Parser) 是一个用于源代码解析的项目，值得一提的是它使用纯 PHP 编写，对于 PHP 程序员来说，能使用自己熟悉的语言来做静态分析等源码处理，无疑是一大便利。

PHP 是动态语言，性能不高，所以用 PHP Parser 分析 PHP 代码，性能也比较差。幸好代码分析这种场景，一般对性能要求也不高。

PHP 自带的 **token_get_all** 函数使用 Zend 引擎的语法分析器将源码切分成一连串的 token，虽然使用这些 token 可以完成很多代码分析及处理的任务，不过由于 token 的结构太原始，遍历和操作都十分不方便。同样是基于 **token_get_all** 分析的结果，著名的代码标准化工具 PHP CodeSniffer 就是在对 token 作了很多处理并提供了一系列查找和遍历的接口的前提下，才让代码分析变得简便了些。

PHP Parser 可以生成 PHP 代码对应的抽象语法树(`AST`，即 Abstract Syntax Tree)结构，极大地简化源代码的遍历等操作。

## PHP parser 的解析结果示例

对于以下一段 PHP 代码：

{% highlight php startinline %}
<?php
echo 'Hi', 'World';
{% endhighlight %}

解析后生成的树结构如下：

```
array(
    0: Stmt_Echo(
        exprs: array(
            0: Scalar_String(
                value: Hi
            )
            1: Scalar_String(
                value: World
            )
        )
    )
)
```

## PHP parser 生成的语法树的结构

为了进一步简化操作，PHP Parser 对语言节点(Node)进行分组：

 - `PhpParser\Node\Stmt` 是语句(statement)节点，包括无返回值和不会出现在表达式的语言结构，例如类的定义；
 - `PhpParser\Node\Expr` 是表达式(expression)节点，包括有返回值和能出现在表达式的语言结构，例如 `$var` (`PhpParser\Node\Expr\Variable`) 和 `func()` (`PhpParser\Node\Expr\FuncCall`) 等；
 - `PhpParser\Node\Scalar` 标量(Scalar)节点，比如：`'string'` (`PhpParser\Node\Scalar\String_`), `0` (`PhpParser\Node\Scalar\LNumber`) 和魔术常量如 `__FILE__` (`PhpParser\Node\Scalar\MagicConst\File`) 等。它们也算是表达式，所有都继承自表达式节点；
 - 其他节点，例如：名称节点 (`PhpParser\Node\Name`) 和参数节点 (`PhpParser\Node\Arg`)

凡是节点类名与 PHP 关键字有冲突的，该节点的类名都统一以 `_` 结尾，如 `PhpParser\Node\Scalar\String_`。

## PHP Parser 能做什么？

除了单纯的将源代码解析成抽象语法树以外，它还附带了以下特性：

 - 代码生成，可以将抽象语法树转换成 PHP 代码
 - 抽象语法树与 XML 的相互转换
 - 导出便于查看的语法树结构
 - 遍历与修改语法树结构的基类(节点遍历者`traverser` 和 节点访问者 `visitor`)
 - 支持命名空间的节点访问者

利用语法树的遍历，我们能够写程序分析代码问题。结合代码生成和语法树结构的遍历修改等特性，我们可以自动化代码重构等等。

下一篇我们将通过具体实例来了解语法树的遍历的应用。
