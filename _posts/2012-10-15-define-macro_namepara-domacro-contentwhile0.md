---
layout: post
title: "#define MACRO_NAME(para) do{macro content}while(0)"
description: ""
category: "clanguage work"
tags: []
sumarry: |
  这篇文章主要的目的在于收集一些小的编程技巧,例如c语言常使用的宏技巧,目的在于提醒自己而已,也好以后让自己有个好的参考文章.
---
{% include JB/setup %}

####概述

这篇文章主要的目的在于收集一些小的编程技巧,例如c语言常使用的宏技巧,目的在于提醒自己而已,也好以后让自己有个好的参考文章.


####目录

1. [do{macro content} while(0)](#a1)
2. [c语言指针](#a2)





<a name="a1"> </a>

####1 do{macro content} while(0)

* [空的宏定义避免warning:](#b1)

<pre class="brush: js;">
#define foo() do{}while(0)
</pre>
<br>
* [存在一个独立的block，可以用来进行变量定义，进行比较复杂的实现](#b1)
* [如果出现在判断语句过后的宏，这样可以保证作为一个整体来是实现:](#b2)

<pre class="brush: js;">
#define foo(x) /
action1(); /
action2();
</pre>
在以下情况下：

<pre class="brush: js;">
if(NULL == pPointer)
   foo();
</pre>

就会出现action1和action2不会同时被执行的情况，而这显然不是程序设计的目的。<br/>
 
* [以上的第3种情况用单独的{}也可以实现，但是为什么一定要一个do{}while(0)呢，看以下代码](#b1)

<pre class="brush: js;">
#define switch(x,y) {int tmp; tmp="x";x=y;y=tmp;}
if(x&gt;y)
switch(x,y);
else       //error, parse error before else
otheraction();
</pre>
在把宏引入代码中，会多出一个分号，从而会报错。


<a name="a2"> </a>

####2. c语言指针

恩,没错,这里就是要对c语言的指针进行复习,c语言的指针非常的强大,因此有必要对指针进行详细的推敲.

<pre class="brush: js;">
用变量a给出下面的定义 
a) 一个整型数 
b)一个指向整型数的指针( A pointer to an integer)  
c)一个指向指针的的指针，它指向的指针是指向一个整型数( A pointer to a pointer to an intege)r  
d)一个有10个整型数的数组( An array of 10 integers)  
e) 一个有10个指针的数组，该指针是指向一个整型数的。(An array of 10 pointers to integers)  
f) 一个指向有10个整型数数组的指针( A pointer to an array of 10 integers) 
g) 一个指向函数的指针，该函数有一个整型参数并返回一个整型数(A pointer to a function that takes an integer as an argument and returns an integer)  
h) 一个有10个指针的数组，该指针指向一个函数，该函数有一个整型参数并返回一个整型数( An array of ten pointers to functions that take an integer argument and return an integer ) 
</pre>


<a class="hide_switch" onMouseOver="document.getElementById('code1').style.display='block'" onMouseOut="document.getElementById('code1').style.display='none';" >让我偷看答案吧! </a>
<div class="hide_div" id="code1">
<pre class="brush: js;">
a) int a; // 一个整型数 An integer  
b) int *a; // 一个指向整型数的指针 A pointer to an integer  
c) int **a; // 一个指向指针的的指针 A pointer to a pointer to an integer  
d) int a[10]; // 一个有10个整型数的数组 An array of 10 integers  
e) int *a[10]; // 一个有10个指针的数组 An array of 10 pointers to integers  
f) int (*a)[10]; // 一个指向有10个整型数数组的指针 A pointer to an array of 10 integers  
g) int (*a)(int); // 一个指向函数的指针 A pointer to a function a that takes an integer argument and returns an integer  
h) int (*a[10])(int); // 一个有10个指针的数组，指向一个整形函数并有一个整形参数 An array of 10 pointers to functions that take an integer argument and return an integer 
</pre>
</div>




