---
layout: post
title: Unity中矩阵变换、深度纹理及其他(╯‵□′)╯︵┻━┻
category : misc
tags : [Unity]
---

在移植UE4的Temporal AA到Unity过程中，计算Camera Motion、Jitter等遇到了若干坑，记录一下。

ps. 这些问题其实主要是……文档不全造成的

## 矩阵变换

在Unity里有Model/World/View/Projection Space，需要将模型的坐标一步步转化最终得到投影坐标；这个在DirectX/OpenGL里也大同小异，就不具体解释。对应Shader代码：

{% highlight GLSL %}
mul(UNITY_MATRIX_MVP, v.vertex);
{% endhighlight %}

如果说要拆开来写，就是先从Model到World，然后World到View，最后到Project(具体变量参考文档[ShaderLab built-in values](http://docs.unity3d.com/Manual/SL-BuiltinValues.html)):

{% highlight GLSL %}
mul(UNITY_MATRIX_P, mul(UNITY_MATRIX_V, mul(_Object2World, v.vertex)));
{% endhighlight %}

然后我费了老大的劲找到了这三个矩阵具体对应API，之后想算啥都行了([aras-p](https://gist.github.com/aras-p/1010683)应该是官方大大)

{% highlight C# %}
Matrix4x4 world = transform.localToWorldMatrix;
Matrix4x4 view = camera.worldToCameraMatrix;
Matrix4x4 proj = camera.projectionMatrix;
{% endhighlight %}

## Depth Texture

之前看文档[Camera’s Depth Texture](http://docs.unity3d.com/Manual/SL-CameraDepthTexture.html)和SSAO例子，我一直以为只能在`_CameraDepthNormalsTexture`使用`DecodeDepthNormal`来读取深度信息。后来看了[Unity Shaders – Depth and Normal Textures (Part 2)](http://willychyr.com/2013/11/unity-shaders-depth-and-normal-textures-part-2/)之后，我才发现可以直接读取：`tex2Dproj(_CameraDepthTexture, UNITY_PROJ_COORD(i.scrPos)).r`！这样就方便了很多，而且不是变换过之后的深度。

提一句题外话就是根据平台不同，有点平台是直接传入ZBuffer；而有的是利用`Camera.RenderWithShader`函数重新渲染一遍、画上深度信息。

## 执行流程

[Execution Order of Event Functions](docs.unity3d.com/Manual/ExecutionOrder.html)里面有一张图，是可用函数的调用顺序~然后在写代码的时候发现unity自己的渲染管线很容易被搞乱，应该是它代码里的各种检查不够多……总之写起来各种小心~ 譬如`RenderTexture.active`和`Graphics.Blit`之间就会影响。。。