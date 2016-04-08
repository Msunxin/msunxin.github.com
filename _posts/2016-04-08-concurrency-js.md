---
layout: post
title: 点击事件的并发
excerpt: 网络延时，或者用户操作速度过快引起的数据并发
---


html 原生的dom对象有个方法

etTimeout() 方法用于在指定的毫秒数后调用函数或计算表达式。

语法

setTimeout(code,millisec)

参数 	描述

code 	必需。要调用的函数后要执行的 JavaScript 代码串。

millisec 	必需。在执行代码前需等待的毫秒数。

setTimeout()方法可以很好的帮我们执行锁住用户的操作，然后定时放开。


```
$(".applyTx").live("click",function(){
	$(".applyTx").attr("disabled",true);					//锁住用户点击
	setTimeout("$('.applyTx').attr('disabled',false)",1000);  //一秒后释放
	var jsonobj = getParms();
	var reg2 = /^\d+(\.\d{1,2})?$/;
	if(!reg2.test(jsonobj.moneys)){
		$('.moneysinput').next('span').css('display','block');return;
	}
	$.post('withdraw.ajaxAddWithdraw.html',{data:data},function(res){

					},'json');
});

```

