---
layout: post
title: Take advantage of Stencil buffer in Post Process
category : misc
tags : [Unity]
---

As I posted in [SSSSS in Unity]({% post_url 2015-1-10-unity-sssss %}) before, I cannot find a way to take advantage of stencil buffer in `OnRenderImage`. This makes the post effect full screen all the time. Other guys have come up with the same question in the [formu](http://forum.unity3d.com/threads/using-the-stencil-buffer-in-a-post-fx.222983/) or [AnswerHub](http://answers.unity3d.com/questions/621279/using-the-stencil-buffer-in-a-post-process.html). After several days hard work, I finally find a way to use stencil.

Here is an example for fast Bloom:

W/O Stencil: 

![unity_pp_stencil_off]({{ site.img_url }}/unity_pp_stencil_off.png)

With Stencil(Ocean Only): 

![unity_pp_stencil_on]({{ site.img_url }}/unity_pp_stencil_on.png)

The key idea is keeping the depth buffer (along with stencil) when rendering, with the help of `Graphics.SetRenderTarget`. First of all, create and set the camera render target. (I also tried depth bit as 16, which makes `RenderTexture.SupportsStencil` return false, and it still works. I don't know why...)

{% highlight C# %}
RenderTextureFormat format = RenderTextureFormat.Default;
if(SystemInfo.SupportsRenderTextureFormat(RenderTextureFormat.ARGBHalf)){
	format = RenderTextureFormat.ARGBHalf;
}
RenderTexture renderTexture = new RenderTexture(Screen.width, Screen.height, 24, format);
renderTexture.Create();
camera.targetTexture = renderTexture;

RenderTexture bloomRT0  = new RenderTexture(renderTexture.width, renderTexture.height, 0, format);
bloomRT0.Create();	//Ensure these two RT have the same size
{% endhighlight %}

Instead of downsample in `OnRenderImage`, we have to `Blit` in `OnPostRender`
{% highlight C# %}
public void OnPostRender(){
	postProcessingMaterial.SetPass(1);

	bloomRT0.DiscardContents();
	Graphics.SetRenderTarget(bloomRT0);
	GL.Clear(true, true, new Color(0,0,0,0));	// clear the full RT
	// *KEY POINT*: Draw with the camera's depth buffer
	Graphics.SetRenderTarget(bloomRT0.colorBuffer, renderTexture.depthBuffer);
	Graphics.Blit(renderTexture, postProcessingMaterial, 1);
}
{% endhighlight %}

The shaderlab side is quite simple, and you can find many resources online. Here are two code snippets:

Ocean Shader
{% highlight GLSL %}
Stencil {
	Ref 
	Comp Always
	Pass replace
}
{% endhighlight %}

Post Processing Shader
{% highlight GLSL %}
Stencil{
	Ref 2
	ReadMask 2
	Comp Equal
}
{% endhighlight %}

ps. You could try draw to screen directly, without setting an extra `renderTexture`. Currently I am combining some Post Process together, along with dynamic resolution, which makes this unavoidable.

![unity_postprocesschain]({{ site.img_url }}/unity_postprocesschain.png)
