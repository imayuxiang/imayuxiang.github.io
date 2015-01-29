---
layout: post
title: cocos2d-x安卓视频OpenGL错误
category : misc
tags : [cocos2d-x]
---

这个其实是我很早之前遇到的一个问题，见[
Cannot resume from video play](http://discuss.cocos2d-x.org/t/cannot-resume-from-video-play/10571) 然后今儿有人发邮件来问当时是怎么解决的-.-

我印象中是因为Android下播放视频也是用的OpenGL，所以最后是将视频相关的代码挪到了GL Thread，大体就是这样包一层

{% highlight java %}
actInstance.runOnUiThread(new Runnable() {
    @Override
    public void run() {
       	Cocos2dxVideo.display(name);            
    }
});
{% endhighlight %}