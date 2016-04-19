---
layout: post
title: php command
excerpt: the design pettern of php
---


```

<?php
/**将多个命令只提交给一个执行该命令的对象
* 命令链模式
*/

//抽象命令
interface onCommand{

	public function onCommand($name);
	
}

class Command{
	public $Command; //命令集合
	
	public function __construct(){
		$this->Command = new SplObjectStorage();
	}
	 //增加命令
	public function addCommand($args){
		$this->Command->attach($args);
	}
	
	//执行命令
	public function run($name){
		$this->Command->rewind();
		while($this->Command->valid()){
			$now = $this->Command->current();
			$now->onCommand($name);
			$this->Command->next();
		}
	}

}

//具体命令
class delCommand implements onCommand{

	public function onCommand($name){
	
		if($name == 'del'){
			echo 'The delete is ok ---';
			return true;
		}else{
			return false;
		}
	}
}

//具体命令
class addCommand implements onCommand{

	public function onCommand($name){
	
		if($name == 'add'){
			echo 'The add is ok ---';
			return true;
		}else{
			return false;
		}
	}
}

$comm = new Command();
$comm->addCommand(new delCommand);
$comm->addCommand(new addCommand);

$comm->run('add');

```
