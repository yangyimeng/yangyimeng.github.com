---
layout: post
title: "Linux Screen Resolution"
description: ""
category: 
tags: []
sumarry: |
 Actualy we want to change the screen resolution of our linux systems, and we do not want to use the trouble steps the system provide.Now here , I just provide a simple way to solve this problem.
---
{% include JB/setup %}


####Summary

 Actualy we want to change the screen resolution of our linux systems, and we do not want to use the trouble steps the system provide.Now here , I just provide a simple way to solve this problem.


####Solution

###Xrandr

The X Resize, Rotate and Reflect Extension, called RandR for short,
brings the ability to resize, rotate and reflect the root window of a
screen. It is based on the X Resize and Rotate Extension as specified
in the Proceedings of the 2001 Usenix Technical Conference


###How to use xrandr

type 'xrandr' command int the shell, you can see information listed below.

<pre class="brush: js;">
xxx@ubuntu:~$ xrandr 
xrandr: Failed to get size of gamma for output default
Screen 0: minimum 320 x 200, current 1280 x 1024, maximum 2560 x 2400
default connected 1280x1024+0+0 0mm x 0mm
   800x600        60.0     56.0      0.0  
   640x480        60.0      0.0  
   320x240         0.0  
   400x300         0.0  
   512x384         0.0  
   1024x768        0.0  
   1152x864        0.0  
   1280x960        0.0  
   1400x1050       0.0  
   1600x1200       0.0  
   1920x1440       0.0  
   2048x1536       0.0  
   2560x1920       0.0  
   854x480         0.0  
   1280x720        0.0  
   1366x768        0.0  
   1920x1080       0.0  
   1280x800        0.0  
   1440x900        0.0  
   1680x1050       0.0  
   1920x1200       0.0  
   2560x1600       0.0  
   720x480         0.0  
   720x576         0.0  
   320x200         0.0  
   640x400         0.0  
   800x480         0.0  
   1280x768        0.0  
   1280x1024       0.0* 
   2560x2400       0.0 
</pre>

Look at the data listed, there is many kinds of screen resolution you can use to replace the screen solution now you used.

###Change the Screen Resolution

Just type 'xrandr -s \[index\]',and index mean the index of the screen resolution you want to use.

<pre class="brush: js;">
xrandr -s 27
</pre>





















