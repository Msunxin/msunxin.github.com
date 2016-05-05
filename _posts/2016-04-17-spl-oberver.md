---
layout: post
title: SPL快速实现observer
excerpt: php SPL实现observer
---

-------------------
复习所用
-------------------

```

<?php 
/**
*观察者模式
*什么是 SPL
*SPL（Standard PHP Library）即标准 PHP 库，是 PHP 5 在面向对象上能力提升的真实写照，它由一系列内置的类、接口和函数构成。SPL 通过加入集合，迭代器，新的异常类型，文件和数据处理类等提升了 PHP 语言的生产力。它还提供了一些十分有用的特性，如本文要介绍的内置 Observer 设计模式。
*本文介绍如何通过使用 SPL 提供的 SplSubject和 SplObserver接口以及 SplObjectStorage类，快速实现 Observer 设计模式。
*SPL  在大多数PHP 5 系统上都是默认开启的，尽管如此，由于 SPL 的功能在 PHP 5.2 版本发生了引人注目的改进，所以建议读者在实践本文内容时，使用不低于 PHP 5.2 的版本。
*SplSubject 和 SplObserver 接口
*SplObjectStorage 类
*/
class  subject implements  SplSubject{
	
	public $loginNum;  
	public $security;  //subject
	
	protected $observers;	//观察者
	
	public function __construct($security){
		$this->security = $security;
		$this->loginNum =mt_rand(1,10);
		
		$this->observers = new SplObjectStorage();
	}
	
	public function notify(){
		$this->observers->rewind();
		while($this->observers->valid()){
			$now = $this->observers->current();
			$now->update($this);
			$this->observers->next();
		}
	}
	
	//添加观察者
	public function attach(SplObserver $observer) {
		$this->observers->attach($observer);
	}
	//删除观察者
	public function detach (SplObserver $observer){
		$this->observers->detach($observer);
	}
}

class loginnum implements SplObserver{
	public function update(SplSubject $subject){
		echo $subject->loginNum.PHP_EOL;
	}
}

class security implements SplObserver{
	public function update(SplSubject $subject){
		echo $subject->security.PHP_EOL;
	}
}

$sub = new subject('hhhhhhhhhh');
$sub->attach(new loginnum());
$sub->attach(new security());

$sub->notify();

```
