---
layout: post-zh
title : PHP Parser 使用示例
category: study
tags : parser
excerpt: 继上一篇 PHP Parser 的介绍文之后，本篇接下来会以几个示例来演示它的一些基本用法，希望对于解决实际场景遇到的问题会有所帮助。
---

## 安装 PHP Parser

推荐使用 Composer 安装：

```
composer require nikic/php-parser
```

## 代码重构

PHP Parser 的代码生成组件(pretty printer)可以根据抽象语法树(Abstract Syntax Tree，简称 `AST`)生成 PHP 代码。

{% highlight php startinline %}
<?php

require __DIR__ . '/vendor/autoload.php';

use PhpParser\Error;
use PhpParser\ParserFactory;
use PhpParser\PrettyPrinter;

$code = "<?php echo 'Hi ', hi\\getTarget();";

$parser = (new ParserFactory)->create(ParserFactory::PREFER_PHP7);
$prettyPrinter = new PrettyPrinter\Standard;

try {
    // 解析成AST
    $stmts = $parser->parse($code);

    // 重构
    $stmts[0]         // echo 语句
          ->exprs     // 子表达式
          [0]         // 第一个元素，即字符串
          ->value     // 字符串的值，也就是 'Hi '
          = 'Hello '; // 修改它

    // 生成重构后的代码
    $code = $prettyPrinter->prettyPrint($stmts);

    echo $code;
} catch (Error $e) {
    echo 'Parse Error: ', $e->getMessage();
}
{% endhighlight %}

以上代码会输出：

{% highlight php startinline %}
<?php echo 'Hello ', hi\getTarget();
{% endhighlight %}

## 节点遍历

上面的列子中， `$code` 是一小段写死的代码，所以很容易访问具体节点并修改。现实情况是我们通常要分析大量的代码，语法树结构更是无从知晓。
幸好 PHP Parser 提供了遍历节点的组件，比如下面这段代码就用到了 `PhpParser\NodeTraverser` 来遍历节点：

{% highlight php startinline %}
<?php
use PhpParser\NodeTraverser;
use PhpParser\ParserFactory;
use PhpParser\PrettyPrinter;

$parser        = (new ParserFactory)->create(ParserFactory::PREFER_PHP7);
$traverser     = new NodeTraverser;
$prettyPrinter = new PrettyPrinter\Standard;

// 添加遍历者
$traverser->addVisitor(new MyNodeVisitor);

try {
    $code = file_get_contents($fileName);

    // parse
    $stmts = $parser->parse($code);

    // traverse
    $stmts = $traverser->traverse($stmts);

    // pretty print
    $code = $prettyPrinter->prettyPrintFile($stmts);

    echo $code;
} catch (PhpParser\Error $e) {
    echo 'Parse Error: ', $e->getMessage();
}
{% endhighlight %}

相应的节点访问者(Visitor)代码如下：

{% highlight php startinline %}
<?php
use PhpParser\Node;
use PhpParser\NodeVisitorAbstract;

class MyNodeVisitor extends NodeVisitorAbstract
{
    public function leaveNode(Node $node) {
        if ($node instanceof Node\Scalar\String_) {
            $node->value = 'foo';
        }
    }
}
{% endhighlight %}

上面这段小程序能够将源代码里所有的字符串节点的值替换成 `'foo'`。

所有的访问者都需要实现 `PhpParser\NodeVisitor` 的接口，它定义了四个方法：
{% highlight php startinline %}
<?php
public function beforeTraverse(array $nodes);
public function enterNode(\PhpParser\Node $node);
public function leaveNode(\PhpParser\Node $node);
public function afterTraverse(array $nodes);
{% endhighlight %}

`beforeTraverse()` 和 `afterTraverse()` 分别在遍历开始之前和结束之后执行，参数为包含遍历节点的数组。

`enterNode()` 和 `leaveNode()` 分别在具体节点的访问开始之前和结束以后执行，参数为当前访问的节点本身。

所有接口方法可以返回修改后的节点，或者不返回值，表示节点未修改。

另外 `enterNode()` 方法可返回 `NodeTraverser::DONT_TRAVERSE_CHILDREN` ，用来跳过对子节点的遍历（提升效率有没有！）。

`leaveNode()` 方法可以返回 `NodeTraverser::REMOVE_NODE`，用来移除当前节点；还可以返回包含节点的数组，用来替换当前节点。

实现每个接口显得有点繁琐，这里还有一个抽象类 `NodeVisitorAbstract` 可供继承，它默认用四个空方法实现了所有接口。

## 稍微复杂一点的用法

首先看一段代码：

{% highlight php startinline %}
<?php
class Demo {
    public function index(){
        // ...
        switch ($type) {
            case 'simple':
                $this->_indexSimple();
                break;

            case 'verbose':
                $this->_indexVerbose();
                break;

            default:
                throw new Exception("unknown type.");
        }
    }
}
{% endhighlight %}

我们虚拟一个需求：指定一个 `$type`，找出其对应调用的所有私有方法。比如告诉你 `$type = 'simple'` ，返回 `_indexSimple`。

{% highlight php startinline %}
<?php

use PhpParser\Node;
use PhpParser\NodeVisitorAbstract;
use PhpParser\NodeTraverser;
use PhpParser\ParserFactory;

class MyMethodVisitor extends NodeVisitorAbstract {
    public $foundMethods;

    public function leaveNode(Node $node) {
        if ($node instanceof Node\Expr\MethodCall) {
            $this->foundMethods[] = $node->name;
        }
    }
}

class MyCaseMethodVisitor extends NodeVisitorAbstract {
    protected $caseValue;
    protected $methodVisitor;
    protected $traverser;

    public function __construct($caseValue, $methodVisitor) {
        $this->caseValue = $caseValue;
        $this->methodVisitor = $methodVisitor;
        $this->traverser = new NodeTraverser;
        $this->traverser->addVisitor($this->methodVisitor);
    }

    public function leaveNode(Node $node) {
        if ($node instanceof Node\Stmt\Case_
            && $node->cond instanceof Node\Scalar\String_
            && $node->cond->value === $this->caseValue
        ) {
            $this->traverser->traverse($node->stmts);
        }
    }

    public function getFoundMethods() {
        return $this->methodVisitor->foundMethods;
    }
}

$code = <<<EOF
<?php
class Demo {
    public function index(){
        switch (\$type) {
            case 'simple':
                \$this->_indexSimple();
                break;

            case 'verbose':
                \$this->_indexVerbose();
                break;

            case 'default':
                throw new Exception("unknown type.");
        }
    }
}
EOF;

$parser = (new ParserFactory)->create(ParserFactory::ONLY_PHP5);
$traverser = new NodeTraverser;
$visitor = new MyCaseMethodVisitor('verbose', new MyMethodVisitor); // 遍历出 case 是 'verbose' 的时候对应调用的私有方法
$traverser->addVisitor($visitor);

try {
    $nodes = $parser->parse($code);
    $traverser->traverse($nodes);
    var_dump($visitor->getFoundMethods());
} catch (Error $e) {
    echo 'Parse Error: ', $e->getMessage();
}
{% endhighlight %}

运行以上代码，就会打印出私有方法 `_indexVerbose`。

## 更多用法

解析源码是为了更好地分析代码问题，比如语法、常见的硬编码、逻辑不严密之类的规范问题，还可以写分析工具用来检测 SQL 语句是否做了安全处理。
由于附带了代码生成工具，写脚本来重构代码也是可能的。甚至还有人基于它做了一个PHP环境的VM：[https://github.com/ircmaxell/PHPPHP](https://github.com/ircmaxell/PHPPHP)，
还有其他用处，等到需要的时候，自然就会想到。
