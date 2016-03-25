---
layout: post-zh
title : Windows 下使用 mergetool 解决合并冲突
category: howto
tags : git
---

本文将介绍在 Windows 平台下使用Vim + fugitive.vim 插件解决 git 分支合并冲突。

## 配置
以下假设 Vim 已经安装可用，而且 fugitive.vim 插件也已经下载安装完毕。

修改 .gitconfig 文件：

{% highlight ini startinline %}
[merge]
    tool = gvim
[mergetool "gvim"]
    cmd = gvim -f -c \"Gdiff\" \"$MERGED\"- 
{% endhighlight %}

## 用法示例
假如目前有两个待合并的分支：一个 master 分支和另一个要并入的分支 fix，当然这两个分支要有冲突。
接下来打开 git 命令行：

 - 运行 git checkout master ，回到 master 分支；
 - 运行 git merge fix 进行合并，之后应该会提示 merge failed 之类的令人沮丧的提示；
 - 运行 git mergetool ；会提示用 gvim 进行合并，敲回车继续后打开vim，包含三列窗口，中间一列是自己要解决冲突后保存的版本，左边是HEAD，右边是要并入的版本，所有diff的区域都有明显的背景颜色；
 - 如果需要保留左边的代码方案，切换到左边窗口，将光标移动到该代码块下，敲击 dp (=diff put，即将该代码放到中间的文件里)，冲突解决；如果保留右边，同理；
 - 所有冲突解决版本，切换到中间窗口，保存退出，左右两个版本可以直接丢弃；如果还有其他文件有冲突，退出这个文件后会自动进入下一个文件，直到全部解决。

## 其他
解决合并冲突过程中可能会产生一些备份文件，可以直接删除，也可以配置不保留备份文件；
另外每次运行 git mergetool 的时候会提示合并工具和待解决的文件等信息，可以设置成免提示，总的来说就像这样：

{% highlight ini startinline %}
[mergetool]
    keepBackup = false
    prompt = false
{% endhighlight %}

![/assets/images/mergetool-demo.gif](/assets/images/mergetool-demo.gif)
