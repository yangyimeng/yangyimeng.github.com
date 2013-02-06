---
layout: post
title: "Android socket权限问题"
description: ""
category: "work android linux" 
tags: []
sumarry: |
  本文主要是要说明在android系统里,通过java的socket接口来实现套接字,是需要系统
  提供的权限的,而系统提供的权限则是在应用程序安装的时候赋予的,当不是普通的应用程序安装,而是直接用dalvik执行jar或dex文件的话,则系统会报出perssion denied的错误警告.
  本文通过一个实际的例子,来说明这种情况.
---
{% include JB/setup %}

本文主要是要说明在android系统里,通过java的socket接口来实现套接字,是需要系统
提供的权限的,而系统提供的权限则是在应用程序安装的时候赋予的,当不是普通的应用程序安装,而是直接用dalvik执行jar或dex文件的话,则系统会报出perssion denied的错误警告.
本文通过一个实际的例子,来说明这种情况.

在android里面, 在执行
<pre class="brush: js;">
    sendSocket = new DatagramSocket(destPort);
</pre>
这种代码的时候,   在没有权限的情况下, 会报出

<pre class="brush: js;">
    java.net.SocketException: socket failed: EACCES (Permission denied)
          at libcore.io.IoBridge.socket(IoBridge.java:573)
          at java.net.PlainDatagramSocketImpl.create(PlainDatagramSocketImpl.java:91)
          at java.net.DatagramSocket.createSocket(DatagramSocket.java:131)
          at java.net.DatagramSocket.&lt;init&gt;(DatagramSocket.java:78)
          at hello.main(hello.java:28)
          at dalvik.system.NativeStart.main(Native Method)
    Caused by: libcore.io.ErrnoException: socket failed: EACCES (Permission denied)
          at libcore.io.Posix.socket(Native Method)
          at libcore.io.BlockGuardOs.socket(BlockGuardOs.java:181)
          at libcore.io.IoBridge.socket(IoBridge.java:558)
          ... 5 more
</pre>

具体的原因是因为 上面的java代码关于socket的创建, 其实本质上还是要通过linux的socket函数, 
而关于linux的socket函数, 有如下的情况说明:
socket函数的返回值有如下几种错误的情况会发生

    ERRORS
           EACCES Permission to create a socket of the specified type and/or protocol is denied.
    EAFNOSUPPORT
          The implementation does not support the specified address family.
        
    EINVAL Unknown protocol, or protocol family not available.
    EINVAL Invalid flags in type.
          
    EMFILE Process file table overflow.
    ENFILE The system limit on the total number of open files has been reached.
           
    ENOBUFS or ENOMEM
          Insufficient memory is available.  The socket cannot be created until sufficient resources are freed.
           
    EPROTONOSUPPORT
          The protocol type or the specified protocol is not supported within this domain.

其中, 有一个EACCES的权限问题会发生, 这也就是上面之所以会报出的错误的原因.

也就是说, 说到底, 问题还是权限的问题, socket的创建也好, su命令的执行也好, 这些都是权限的问题. 

可是, 我们平常在执行android的程序的时候, 那些权限是由谁授予的呢, 这是个疑问.



