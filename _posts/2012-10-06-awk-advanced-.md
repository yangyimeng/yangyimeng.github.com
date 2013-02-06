---
layout: post
title: "awk sed  grep 应用(转)"
description: ""
category: "work awk sed grep"
tags: []
sumarry: |
  1. 不显示文件中的空行  
  2. 删除文件的1到5行     
  3. 删除文件注释行   
  4. 打印匹配行 
  5. 显示从字符1到字符2的中间行 
  6. 匹配特别表达式   
  7. 替代文本
---
{% include JB/setup %}


目录：

<br/>
<br/>
1. [不显示文件中的空行](http://)
2. [删除文件的1到5行](http://)
3. [删除文件注释行](http://)
4. [打印匹配行](http://)
5. [显示从字符1到字符2的中间行](http://)
6. [匹配特别表达式](http://)
7. [替代文本](http://)
         

##1.不显示文件中的空行

<pre class="brush: js;">
[guo@guo ~]$ cat rc.local 
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.

touch /var/lock/subsys/local
[guo@guo ~]$ grep -v '^$' rc.local 
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
touch /var/lock/subsys/local
[guo@guo ~]$ sed -e '/^$/d' rc.local 
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
touch /var/lock/subsys/local
[guo@guo ~]$ awk '!/^$/{print $0 }' rc.local 
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
touch /var/lock/subsys/local
</pre>

##2.删除文件的1到5行

<pre class="brush: js;">
[guo@guo ~]$ cat test 
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
[guo@guo ~]$ sed -e '1,5d' test 
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
[guo@guo ~]$ awk '{if(NR>5 ) print $0} ' test 
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
</pre>

##3.删除文件注释行

<pre class="brush: js;">
[guo@guo ~]$ sed -e "/^#/d" rc.local 

touch /var/lock/subsys/local
[guo@guo ~]$ awk '!/^#/{print $0}' rc.local 

touch /var/lock/subsys/local
[guo@guo ~]$ grep -v '^#' rc.local 

touch /var/lock/subsys/local
</pre>

##4.打印匹配行

<pre class="brush: js;">
[guo@guo ~]$ grep '^#' rc.local 
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
[guo@guo ~]$ sed -n -e '/^#/p' rc.local 
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
[guo@guo ~]$ awk ' /^#/ { print $0 }' rc.local 
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
</pre>

##5.显示从字符1到字符2的中间行

<pre class="brush: js;">
[guo@guo ~]$ cat test1 
good
hello
hi  shell
sorry 
goodbye c 
[guo@guo ~]$ sed -n -e '/hello/,/sorry/p' test1
hello
hi  shell
sorry 
[guo@guo ~]$ awk '/hello/,/sorry/ {print $0 }' test1
hello
hi  shell
sorry 
</pre>

##6.匹配特别表达式

<pre class="brush: js;">
[guo@guo ~]$ cat for.c
#include&lt;stdio.h&gt;
main() 
{
    printf("Hello Shell");
}
[guo@guo ~]$ sed -n -e '/main[[:space:]]*(/,/^}/p' for.c
main() 
{
    printf("Hello Shell");
}
</pre>

##7.替代文本

<pre class="brush: js;">
[guo@guo ~]$ cat test |tr 'root' 'good'
good:x:0:0:good:/good:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/vag/adm:/sbin/nologin
lp:x:4:7:lp:/vag/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shuddown:x:6:0:shuddown:/sbin:/sbin/shuddown
hald:x:7:0:hald:/sbin:/sbin/hald
mail:x:8:12:mail:/vag/spool/mail:/sbin/nologin
uucp:x:10:14:uucp:/vag/spool/uucp:/sbin/nologin
[guo@guo ~]$ cat test |tr 'root' 'god'
gddd:x:0:0:gddd:/gddd:/bin/bash
bin:x:1:1:bin:/bin:/sbin/ndldgin
daemdn:x:2:2:daemdn:/sbin:/sbin/ndldgin
adm:x:3:4:adm:/vag/adm:/sbin/ndldgin
lp:x:4:7:lp:/vag/spddl/lpd:/sbin/ndldgin
sync:x:5:0:sync:/sbin:/bin/sync
shudddwn:x:6:0:shudddwn:/sbin:/sbin/shudddwn
hald:x:7:0:hald:/sbin:/sbin/hald
mail:x:8:12:mail:/vag/spddl/mail:/sbin/ndldgin
uucp:x:10:14:uucp:/vag/spddl/uucp:/sbin/ndldgin
[guo@guo ~]$ sed 's/root/good/p' test
good:x:0:0:root:/root:/bin/bash
good:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
[guo@guo ~]$ sed 's/root/god/p' test
god:x:0:0:root:/root:/bin/bash
god:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync

shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
[guo@guo ~]$ awk '{gsub(/root/,"good");print $0}' test
good:x:0:0:good:/good:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
[guo@guo ~]$ awk '{gsub(/root/,"god");print $0}' test
god:x:0:0:god:/god:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
通过对比可知tr的替换两个字串必须等长度，而sed不能把说有的root替换
[guo@guo ~]$ sed 's/root/good/g' test
good:x:0:0:good:/good:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
通过参数g实现把每个root都替换
</pre>
