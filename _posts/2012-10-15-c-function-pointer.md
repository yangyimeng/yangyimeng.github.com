---
layout: post
title: "c function pointer"
description: ""
category: "clanguage work"
tags: []
sumarry: |
  对于c语言的指针,说实在的,虽说用了这么多年,但是却没有过认真的总结过,每次都只是脑海里想一想"哦,不就是指针吗!就那样用,不会出错!",却没有想过去理解一下函数名的本质意义.本篇文章的目的就在于理解一下函数名的本质以及函数调用.
---
{% include JB/setup %}

####摘要
对于c语言的指针,说实在的,虽说用了这么多年,但是却没有过认真的总结过,每次都只是脑海里想一想"哦,不就是指针吗!就那样用,不会出错!",却没有想过去理解一下函数名的本质意义.本篇文章的目的就在于理解一下函数名的本质以及函数调用.


####实例分析

为了一下子进入正题,我用下面的例子来演示一下c函数名和函数指针的情况.我想,诸位也会多少有些奇怪.

<pre class="brush: js;">
#include &lt;iostream&gt;
#include &lt;stdio.h&gt;
using namespace std;

int f(int value) {
    printf("value is %d \n", value);
    return 1;
}

int main()
{
    char**p=str+1;
    int (*pf) (int) = f;
    printf("main is %d *main is %d &amp;main is %d \n", main, *main, &amp;main);
    printf("f is %d  *f is %d &amp;f is %d\n", f, *f, &amp;f);
    printf("pf is %d *pf is %d &amp;pf is %d \n", pf, *pf, &amp;pf);
    printf("call f(2)\n");
    f(2);
    printf("call (*f)(2)\n");
    (*f)(2);
    printf("call (*pf)(2)\n");
    (*pf)(2);
    printf("call pf(2)\n");
    pf(2);
    return 0;
}
</pre>

这段程序很简单,但是里面包含了很多的平常没有注意到的事情.上面的代码的运行结果如下:

<pre class="brush: js;">
main is 134514452 *main is 134514452 &amp;main is 134514452 
f is 134514420  *f is 134514420 &amp;f is 134514420
pf is 134514420 *pf is 134514420 &amp;pf is -1081660104 
call f(2)
value is 2 
call (*f)(2)
value is 2 
call (*pf)(2)
value is 2 
call pf(2)
value is 2 
</pre>

如何, 运行的结果是不是没有想到呢,发现了没有,函数名的值,与函数名的\*,&amp;运算结果值都是一样的,这也就说明了为何(\*pf)(2)可以,pf(2)也可以的原因了.其中,关于&amp;pf的值不一样的原因很简单,因为pf是一个指针变量,而&amp;pf则是存放这个变量的值的地址而已,显然这个地址与函数没有任何的关系.




####总结
我想这里应该可以做一下总结吧,函数名其实本质上也是一个指针,且这个指针的地址与指针的值是一样的,这就可以合理说明上面的情况了.我想,这也就是c语言的关于函数的特殊设定吧.


####思考

我觉得事情不能就这样子就结束了，同理的，我们是否要思考一下上面的机制到底是怎样实现的，为何会如此呢？还有对于数组类型的数据呢？他们的情况又是如何的呢？假如说str是一个int型数组，则对这个进行取地址，取值以及各种操作的值又是如何的呢？
















