---
layout: post
title: demo
excerpt: demo ------------------------demo
---
There has been lots of buzz about many of the new features in PHP 5.4, like the traits support, the
short array syntax and all those other syntax improvements.

But one set of changes that I think is particularly important was largely overlooked: For PHP 5.4
cataphract ([Artefacto][1] on StackOverflow) heroically rewrote large parts of `htmlspecialchars`
thus fixing various quirks and adding some really nice new features.

(The changes discussed here apply not only to [`htmlspecialchars`][2], but also to the related
[`htmlentities`][3] and in parts to [`htmlspecialchars_decode`][4], [`html_entity_decode`][5] and
[`get_html_translation_table`][6].)

Here a quick summary of the most important changes:

 * UTF-8 as the default charset
 * Improved error handling (`ENT_SUBSTITUTE`)
 * Doctype handling (`ENT_HTML401`, ...)

UTF-8 as the default charset
----------------------------

<?php
/**
*	$this->调用本类的方法属性 ，静态方法中不可以用$this 调用属性
*	self::调用本类的静态属性方法
*	parent:: 调用父类的成员属性方法
*/
	
	class base{
		//public static $name;
		public $sex;
		public static $new;
		
		public function __construct(){
			echo $this->output();
		}
		public static function obb(){
			if(!(self::$new instanceof base)){
				return  self::$new = new self();	
			}else{
				return self::$new;
			}
		}
		
		public function output(){
			return 222;
		}
		
		public function sex($sex){
			echo  $sex;
			
		}
	}
	
	
	class jibase extends base{
		public function __construct($name){
			echo $name.'<br>';
			parent::sex('sun');
		}
		
		public static function ceshi(){
			echo '<br>'.'ok';
		}
		
		public function kk(){
			self::ceshi();
		}
		
	}
	

	echo base::obb()->sex('nv');
	
	$a =new jibase('3233');
	$a->kk();
?>
-----------------------
