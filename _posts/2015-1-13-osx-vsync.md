---
layout: post
title: OSX下设置垂直同步
category : misc
tags : [C++]
---

最近发现一个逗比知识，osx下设置垂直同步貌似靠代码是无效的

{% highlight Objective-C %}
GLint swapInt = 1;
[[self openGLContext] setValues:&swapInt forParameter:NSOpenGLCPSwapInterval];
{% endhighlight %}

后来找到了这个[How to disable vsync on Mac OsX](http://stackoverflow.com/questions/12345730/how-to-disable-vsync-on-mac-osx)，不过选项的位置改到了Window-Quartz Debug Settings里面的Beam Sync中

![osx_vsync]({{ site.img_url }}/osx_vsync.png)