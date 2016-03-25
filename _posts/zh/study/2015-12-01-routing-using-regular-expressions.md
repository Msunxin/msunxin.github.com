---
layout: post-zh
title : PHP正则实现快速路由
category: study
tags : route
---

[Pux](https://github.com/c9s/Pux) 是一个性能极高的 PHP 路由，使用 C 语言开发的 PHP 扩展，性能更高，还有更多特性不作赘述。
本文重点介绍的是另外一个路由项目：[FastRoute](https://github.com/nikic/FastRoute)，使用纯 PHP 编写，和很多路由一样，它也是使用正则实现，但是总体性能比 Pux 还要高。
接下来我们将分析 FastRoute 是如何逐步将正则路由的性能发挥到极致的。

术语
-------------------
 - 占位符：将要被变量替换的字符。例如本文中路由规则 *'/user/{name}'* 中的{name}。
 - 捕获组：正则中子表达式匹配到的内容。例如 */user/([0-9]+)* 中括弧中的内容。

路由的难题
-------------------
为了保证我们处在一个频道，首先说一下“路由”的定义。多数情况下，它指的是对设定的路由规则的处理的过程，其中路由规则的设定类似如下：

{% highlight php startinline %}
<?php
// 这里的路由涉及HTTP方法、URI规则以及对应处理函数/方法
$r->addRoute('GET', '/user/{name}/{id:\d+}', 'handler0');
$r->addRoute('GET', '/user/{id:\d+}', 'handler1');
$r->addRoute('GET', '/user/{name}', 'handler2');
{% endhighlight %}

根据这些规则，对URI进行匹配。例如：

{% highlight php startinline %}
<?php
$d->dispatch('GET', '/user/nikic/42');
// 匹配后得到了处理函数'handler0'和参数 ['name' => 'nikic', 'id' => '42']
{% endhighlight %}

为了抽象一些，HTTP方法以及路由的定义形式在此略过。本文只分析路由匹配环节，至于如何解析路由和匹配数据的生成都不属于本文讨论的范畴。

那么，路由究竟慢在什么地方呢？在一个大型系统中，通常开销是在许多的对象的实例化以及大量的方法调用上。Pux 对这种开销优化得不错。然而究其根本，不断地从一堆表达式中对指定的URI进行匹配的过程才是慢的根源。如何让这个过程快起来（或者说快一点）乃是本文的主题。

合并的正则表达式
-------------------
解决路由这种难题的基本思路是避免一条一条地去作正则匹配，而是将所有正则表达式合并为一条，这样只需要匹配一次。以上面的路由规则为例：

{% highlight php startinline %}
<?php
    // 多个单条正则
    ~^/user/([^/]+)/(\d+)$~
    ~^/user/(\d+)$~
    ~^/user/([^/]+)$~
{% endhighlight %}

{% highlight php startinline %}
<?php
    // 合并后的正则
    ~^(?:
        /user/([^/]+)/(\d+)
      | /user/(\d+)
      | /user/([^/]+)
    )$~x
{% endhighlight %}

这种合并很简单：基本就是将每条表达式用“或”的逻辑拼接起来。用合并后的正则进行匹配时，我们就不知道匹配到的是对应原来的哪条正则(路由规则)了，那如何定位匹配到的路由规则呢？
为了搞清这个问题，看一下 **preg_match** 一个示例：

{% highlight php startinline %}
<?php
// $regex 为上面合并后的正则
preg_match($regex, '/user/nikic', $matches);
=> [
    "/user/nikic",   # 匹配到的完整内容
    "", "",          # 第一条路由的捕获组内容 (未匹配成功，注意：本条路由规则包含两个捕获组)
    "",              # 第二条路由的捕获组内容 (未匹配成功)
    "nikic",         # 第三条路由的捕获组内容 (匹配到了)
]
{% endhighlight %}

定位匹配的路由规则的关键在于找到 **$matches** 数组第一个非空的匹配内容（当然，不算匹配到的完整内容），结合 **$matches** 数组的下标和路由的映射关系，就可以匹配到对应的路由规则。这种映射关系类似于：

{% highlight php startinline %}
<?php
[
  // 数组下标 => 路由信息
    1 => ['handler0', ['name', 'id']], # 注意：第一条路由规则包含两个捕获组
    3 => ['handler1', ['id']],
    4 => ['handler2', ['name']],
]
{% endhighlight %}

以下有一个程序实现的示例：

{% highlight php startinline %}
<?php
public function dispatch($uri) {
    if (!preg_match($this->regex, $uri, $matches)) {
        return [self::NOT_FOUND];
    }

    // 找到第一个非空匹配 (下标从1开始，即跳过匹配到的完整内容)
    for ($i = 1; '' === $matches[$i]; ++$i);

    list($handler, $varNames) = $this->routeData[$i];

    $vars = [];
    foreach ($varNames as $varName) {
        $vars[$varName] = $matches[$i++];
    }
    return [self::FOUND, $handler, $vars];
}
{% endhighlight %}

找到第一个非空匹配内容的下标 **$i** 以及对应的数据， 就可以从 **$matches** 里推算出占位符变量的值。

将多条正则(路由规则)合并后，表面上是匹配操作缩减成了一次。遗憾的是经过测试后，我们发现某些情况下性能反而下降了，比如在多占位符、多条路由规则的情况下命中比较靠后规则时。
而多占位符、多条路由规则都未命中的情况下，性能反而提高了。
作者推测的原因是：在多占位符、多条路由规则的情况下，假如匹配非常不理想，匹配到的是最后一条路由规则，该规则之前的所有正则都没匹配成功，它们的捕获组匹配内容都为空，也就是说，
在匹配结果 **$matches** 数组里，填充了很多空字符元素；所有规则都未匹配时，**$matches** 数组根本都不需要做那么多空字符的填充。但是最不理想的情况下反而性能更高，
可能是填充 **$matches** 影响了性能，如果占位符和路由规则非常多，匹配越不理想，生成的 **$matches** 越大。

如果假设成立，那么我们就需要减少 **$matches** 数组里那么多空字符元素的填充。


捕获组的序号重置
----------------
PCRE 正则语法有一个比较少见的非捕获组类型 (non-capturing group type) ： *(?| ... )* 。 **(?:** 和 **(?|** 的区别在于后者会对捕获组序号进行了重置。举个栗子：

{% highlight php startinline %}
<?php

preg_match('~(?:(Sat)ur|(Sun))day~', 'Saturday', $matches)
=> ["Saturday", "Sat", ""]   # 实际上 $matches 里最后这个 "" 元素是不存在的，补上它是为了让概念易于理解

preg_match('~(?:(Sat)ur|(Sun))day~', 'Sunday', $matches)
=> ["Sunday", "", "Sun"]

preg_match('~(?|(Sat)ur|(Sun))day~', 'Saturday', $matches)
=> ["Saturday", "Sat"]

preg_match('~(?|(Sat)ur|(Sun))day~', 'Sunday', $matches)
=> ["Sunday", "Sun"]
{% endhighlight %}

使用 **(?:** 将会在 **$matches** 中为 Sat 和 Sun 分别创建两个捕获组，尽管可以确定只有一个捕获组能匹配到。此时两个捕获组都占有一个序号，(Sat) 的序号是1，(Sun)的序号为2。
使用 **(?|** 时两个捕获组将共用序号 1。 关联的是 (Sat) 还是 (Sun) 取决于具体匹配到的内容。

但是，由于捕获组的序号被重置了，分辨出匹配到的是哪一个路由规则又成了一个问题。之前可以根据 **$matches** 里第一个非空下标来定位，现在捕获组始终是 **$matches[1]**，有帮助的信息就太少了。

这里用了另外一个奇淫技巧：让每个路由规则的捕获组的个数不一样。

上例三个路由规则中，第一条规则的匹配结果 **$matches** 数组有3元素 (一个完全匹配元素以及两个捕获组)，另外两条的数组有2个元素。

根据捕获组个数确定匹配到的路由，必须要让每条路由的捕获组个数互不相同。我们可以适当给路由虚拟一些捕获组：

{% highlight php startinline %}
<?php
~^(?|
    /user/([^/]+)/(\d+) # 第一条路由有2组
  | /user/(\d+)()()     # 第一条路由有3组，虚拟了2组
  | /user/([^/]+)()()() # 第一条路由有4组，虚拟了3组
)$~x
{% endhighlight %}

这样，就可以通过以下的映射关系匹配路由了：

{% highlight php startinline %}
<?php
[
  //组数 => 路由信息 
    3 => ['handler0', ['name', 'id']],
    4 => ['handler1', ['id']],
    5 => ['handler2', ['name']],
]
{% endhighlight %}

实现代码示例：

{% highlight php startinline %}
<?php
public function dispatch($uri) {
    if (!preg_match($this->regex, $uri, $matches)) {
        return [self::NOT_FOUND];
    }

    list($handler, $varNames) = $this->routeData[count($matches)];

    $vars = [];
    $i = 0;
    foreach ($varNames as $varName) {
        $vars[$varName] = $matches[++$i];
    }
    return [self::FOUND, $handler, $vars];
}
{% endhighlight %}

经过这样的改进之后，再次测试发现，多占位符多路由规则的情况下，命中比较靠后的规则时性能极大的提升了，不过，和单纯合并规则的方案相比，其他场景性能又有所下降。


