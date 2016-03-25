---
layout: post-zh
title : Xdebug 之远程调试
category: howto
tags : xdebug
excerpt: Xdebug 是php开发的调试利器，支持断点、步调，结合IDE可以让开发人员方便地调试一些复杂的应用。它还可以调试CLI程序，本篇主要介绍其常见的使用IDE+HTTP请求的调试方式，即通过访问页面产生一个HTTP请求，启动调试会话，借助IDE和HTTP服务器通信来完成调试过程。
---

本文图片和参考来源均来自：[xdebug.org](http://xdebug.org) 。

你是否有过这样的经历：写了一大段代码，执行的时候无任何报错，有输出结果，只是部分数据不对，或者干脆没有数据。你猜测是某个文件的某一行的某个方法返回结果不对，为了验证自己的诊断，打印出这个结果以看个究竟，你飞奔到那一行，熟练的敲出了一行代码：
{% highlight php %}
<?php var_dump($result); die;
{% endhighlight %}
保存文件并再次执行，然而却发现这部分的返回结果并没有问题。这时，你又猜测到另一行代码可能有问题，接着重复前面的过程。反反复复数次以后，你终于找到问题所在并成功修复。

在一个不太复杂的程序中，用以上这种打印变量、使用中断的方式来调试问题也是基本可行的。只是它有很多缺点：

 - 需要在代码中书写打印语句，有破坏代码的风险
 - 不方便一次同时查看多个变量，打印太多变量时也不方便观察
 - 在多处设置中断语句后可能最终会忘了清理
 - 效率低，搞不好要加班

如果你有这样的困扰，是时候使用Xdebug结合IDE进行断点调试了。Xdebug 是php开发的调试利器，支持断点、步调，结合IDE可以让开发人员方便地调试一些复杂的应用。它还可以调试CLI程序，本篇主要介绍其常见的使用IDE+HTTP请求的调试方式，即通过访问页面产生一个HTTP请求，启动调试会话，借助IDE和HTTP服务器通信来完成调试过程。

远程调试可以分为单用户和多用户两种，单用户最常见的就是本机搭建HTTP服务器，同时在本机开发程序；另外一种就是本文重点介绍的多用户远程调试。我们先介绍单用户模式：

单用户模式
---------------------
单用户模式常用于本机开发、本机调试的场景，Xdebug 的工作流程如下：

 - 假设HTTP服务器的IP为：10.0.1.2，端口号80
 - 假设IDE的IP为：10.0.1.42，就将 `xdebug.remote_host` 设置为 10.0.1.42
 - 假设IDE监听端口为：9000，就将 `xdebug.remote_port` 设置为 9000
 - IDE所在的机器上发起HTTP请求
 - xdebug连接到10.0.1.42:9000
 - 调试启动，HTTP开始响应

动画演示图片：

![http://xdebug.org/images/docs/dbgp-setup.gif](http://xdebug.org/images/docs/dbgp-setup.gif)

多用户模式
--------------------
多用户共用一个HTTP服务器时，IDE所在的IP(通常也是发起HTTP请求的来源IP)不固定，需要HTTP服务器端确定了IDE的IP后才能启动会话，只要配置 `xdebug.remote_connect_back=1` 即可，过程大致如下：

 - 假设HTTP服务器的IP为：10.0.1.2，端口号80
 - IDE所在的IP未知，配置了 `xdebug.remote_connect_back=1`
 - 假设IDE监听端口为：9000，就将 `xdebug.remote_port` 设置为 9000
 - 发起HTTP请求，Xdebug 根据headers信息确定请求来源IP
 - xdebug连接到检测到的IP和端口(10.0.1.42:9000)
 - 调试启动，HTTP开始响应

动画演示图片：

![http://xdebug.org/images/docs/dbgp-setup2.gif](http://xdebug.org/images/docs/dbgp-setup2.gif)

多用户远程调试非常适合团队协作开发，甚至可以在测试环境下使用。最简单的配置如下：

{% highlight ini startinline %}
xdebug.remote_enable=1
xdebug.remote_connect_back=1
xdebug.remote_port=9000
xdebug.idekey=XXX
{% endhighlight %}

其中 `xdebug.idekey` 是会话id标识，在各种IDE里有对应的配置，比如netbeans中的Session ID，phpstorm里的idekey；`xdebug.remote_port` 是IDE的配置的端口，只要都配置一致即可。

由于配置简单通用，在本机开发本机调试的情况下，还是建议按多用户模式配置。

生成快捷链接启用/停用调试
----------
默认配置是不会自动创建调试会话的，我们可以在发起HTTP请求的时候需要携带上这个会话标识，用来触发调试会话、建立HTTP服务器和IDE的连接，携带该标识的方式有：

 - 在请求的url追加参数 `XDEBUG_SESSION_START=idekey`，POST请求里带上也可以
 - 在cookie中设置 `XDEBUG_SESSION_START` 或 `XDEBUG_SESSION` 变量

想要停用调试只需要去掉携带的参数或cookie即可。
为了更方便的启用/停用调试，建议使用收藏夹设置快捷链接来完成启用/停用调试，这里提供一个自动生成快捷链接的小工具：

IDE key <input type="text" value="idekey" id="idekey" onfocus="document.getElementById('output').innerHTML=''">
<input type="button" value="生成链接" onclick="document.getElementById('output').innerHTML = document.getElementById('tpl').value.replace('idekey', document.getElementById('idekey').value)"/>
<textarea id="tpl" style="display:none">
请用鼠标按住这两个链接并拖拽到收藏夹：<a id="enable-xdebug" href="javascript:(function(){document.cookie='XDEBUG_SESSION='+'idekey'+';path=/;';})()">启用XDebug</a>、
<a href="javascript:(function(){document.cookie='XDEBUG_SESSION='+''+';expires=Mon, 05 Jul 2000 00:00:00 GMT;path=/;';})()">停用XDebug</a>。
</textarea>
<p id="output"></p>

另外，还可以安装浏览器插件来辅助启用/停用调试，如Firefox的插件 https://addons.mozilla.org/en-US/firefox/addon/the-easiest-xdebug/ 。

远程代码目录映射
-------------------
多用户调试的情况下，IDE和HTTP服务器往往不在一台机器上，由于调试的是HTTP服务器上的代码，需要配置IDE，让本机（也就是IDE所在的机器）上的项目代码路径映射到HTTP服务器上的代码路径，这样在本机设置断点后，Xdebug才能知道HTTP服务上代码的断点位置，前提是本机和服务器上的代码要完全一致。

