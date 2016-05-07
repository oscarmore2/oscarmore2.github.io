---
layout: article
categories: unity
title: "Unity 材质简易教程：compute shader简介"
---

本文转载并翻译自：
Coxlin's Blog 

http://www.lindsaygcox.co.uk/tutorials/unity-shader-tutorial-an-intro-to-compute-shaders

原标题为：
Unity Shader Tutorial: An intro to Compute Shaders


Are you ready to turn this up to 11? We are going to look at some real “Triple A” business now. The world of compute shaders. So what are these mysterious  creatures that you probably don’t know exist in Unity?

你准备好学更多的东西了吗？现在要带踏入你真正3A级领域了--可计算着色器(compute shaders)*的国度。好，这个你根本不知道其存在的怪物是一种怎么样的东东呢？

*译者注：关于“compute shaders”的翻译，我是纠结了很久。我取“可计算着色器”这个译名是因为compute shaders则提供了一个入口让GPU可以实现一般性的计算。

To be honest, I had completely forgot they were there and I was looking at a fur tutorial (that doesn’t actually seem to work by the way and was also a really dirty way of doing it ) and then remembered my mate had said you could probably do grass and fur in one. I think he actually meant geometry shaders, but compute shaders peeked my interest.

说真的，我基本忘记他们的存在了，而当我找一些关于毛发效果的教程(并没有想象中这么有用而且方法太糟糕) 的时候才想起我的基友说过草地和毛发的效果可以用同一种方法实现。我想它说的是(geometry shaders)几何着色器，但计算着色器戳中了我的兴趣点。

However, after digging around the net it turns out that info surrounding them when it comes to Unity seems quite scarce.

可是，当我翻遍整个网络发现针对Unity的计算着色器的信息少之又少。

Let’s start from the top!
来我们从头说起

#计算着色器是什么，为什么要了解它？
In Microsoft’s fancy terms, “a compute shader is a programmable shader  stage that expands Microsoft Direct3D11 beyond graphics programming” and “a compute shader provides high-speed general purpose computing and takes advantage of the large numbers of parallel processors on the GPU”.

在微软的花俏术语里面，它是这么说：“可计算着色器是一种可编程的着色处理过程，它扩展了微软Direct3D 11并且超越了图形编程了领域”，“计算着色器利用了GPU大量的并行处理的特性提供了高速的通用计算功能”。

In simple terms, a compute shader is a program that runs on the graphics card that does stuff outside of the normal rendering pipeline.

说得简单一点，可计算着色器是一个在显卡上运行的程序，并且完成可以在正常渲染管线之外的任务。

So you are probably thinking “OK, I kind of get it, you can run some logic and put some work onto the graphics card, but why would I want to do that?” Well these shaders are really good at maths and parallelization, i.e. they are really good at performing tasks where you are doing a lot of the same thing. In other words, they are really good at tasks that involve applying the same set of calcualtions to every element in a given data set.

你可能会说：“好吧，我好像明白了一丢丢，这玩意就是让一些逻辑和任务运行在显卡上面，但我干嘛要用它呢？”其实啊，这些着色器对数学(尤其浮点)和并行计算方面有很大的本事。也就是说，他们同时处理同一种处理任务有很大本事。换句话说，他们非常擅长处理数据集合每个元素的运算。

This is probably a kind of crappy exlanation, so lets wind the clock back a bit to when I was just gracing the planet with my presence. The 90s. It was a beautiful time with games like Doom, Final Fantasy 7, The Legend of Zelda: Ocarina of Time, Crash Bandicoot, Tekken 3… do I need to go on? Essentially lots of 3D games and PCs started going out with graphics cards. Stuff like this bad boy.

这个解释有点蹩脚，所以让我略穿越一下，回到我对着星球放空的时候，那是90年代(...)。那真是游戏的黄金岁月，DOOM，最终幻想7，塞尔达传说： 时之笛，古惑狼赛车，铁拳3...还要在列举吗？(当时好游戏太多，本人补刀：魔法门，古墓丽影)。大部分的3D游戏都是PC游戏，尤其是有了显卡之后。

http://img1.tuicool.com/rqia6ff.jpg!web

What a rush indeed! Getting that sweet 6MB of power all up in your grill. Anyway OpenGL and DirectX appeared and the magic of the programmable pipeline emerged. Developers just send geometry down to the graphics card and OpenGL/Direct X would figure it out. However, the pipeline was pretty rigid, and thus to make more interesting effects and push the boundaries it had to became more flexible. This led onto shaders, where devs could write their own programs to perform certain parts of the pipeline and make things look like the wizard’s tits.

This the opened up a lot of possibilities and this new system mean that the new pipline could deal with a lot of different types of algorithms and now the GPU can do stuff like crazy multi-threaded physics, etc.

What this means now is we can do crazy stuff like Nvidia’s Hair works .

You on board now? If not, just know it is cool and you feel like a Game Development Maverick when you do it.

Basically, you can potentially harness the GPU to do none graphicsy stuff if you so desire and gain MOAR POWER.

##Sod it, lets jump in!
That’s the attitude I want!

Before you start though you need a WINDOWS machine. Macs don’t have it. And to be honest they are kinda crappy for big boy game development like this anyway :stuck_out_tongue:

Create a compute shader in Unity.

The first thing you will notice is that this is not CG. This is a Direct X 11 style HLSL bad boy. Yeah fasten your seat belts boys and girls.

{% hightlight C %}
// Each #kernel tells which function to compile; you can have many kernels
#pragma kernel CSMain

// Create a RenderTexture with enableRandomWrite flag and set it
// with cs.SetTexture
RWTexture2D<float4> Result;

[numthreads(8,8,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
     // TODO: insert actual code here!

     Result[id.xy] = float4(id.x & id.y, (id.x & 15)/15.0, (id.y & 15)/15.0, 0.0);
}
{% endhightlight %}

 The #pragma kernel CSMain  is the first thing we see in our shader. Kind of like the other shaders, this is us telling the program where our entry point is, in this case CSMain. A compute shader can have many a function and you can call a specific funciton from a script. More on that later.

The next bit is a RWTexture2D<float4> Result

Again, like our other shaders, this is just a variable declaration. However, as we aren’t using mesh data, we have to say what the shader will read and write to. In this case we have a RWTexture2D a read/write texture 2d object that the program is gonna use. Take a look at MSDN for reference:

https://msdn.microsoft.com/en-gb/library/windows/desktop/ff471505(v=vs.85).aspx

Finally, the last super different thing is the numthreads which is the the number of thread groups that are spawned by our shader. GPUs love the parallel processing business and create threads that run simultaneously.  This line is specifying the dimensions of the thread groups. These specify how the threads that are created are organised and in this case we are saying that we want to create 64 threads. Take a look at msdn for refrence: 

https://msdn.microsoft.com/en-us/library/windows/desktop/ff471442(v=vs.85).aspx

The size of your thread groups will be determined by a lot of factors and probably most notably your target hardware. For example, the PS4 may have  a different optimum size compared to the Xbox One.

The rest is kind of bog standard code. The kernel function determines what pixel it should be working on based on the uint3 ID of the thread running the function and writes some data to the result buffer.

Cool. We have our first compute shader! But how do we actually run this warlock? It doesn’t run on mesh data so we can’t attach it to a mesh. We need to grab it from a script.

But hold up, before we start lets change up our script that Unity spat out and make the compute shader do something different.

{% hightlight C %}
// Each #kernel tells which function to compile; you can have many kernels
#pragma kernel CSMain

RWStructuredBuffer<int> buffer1;
 
[numthreads(4,1,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
    buffer1[id.x] = mul(id.x, 2.0);
}
{% endhightlight %}

You can see we switched out the texture for a structured buffer. This is just an array of data consisting of a single data type, in this case it is an int. In the code you can see we are just taking the id of the thread and multiplying it by 2.

Cool lets write a new script.

{% hightlight CSharp %}
using UnityEngine;
using System.Collections;

public class RunComputeShader : MonoBehaviour
{
    [SerializeField]
    private ComputeShader _shader;

    void Start()
    {
       ComputeBuffer buffer = new ComputeBuffer(4, sizeof(int));

       _shader.SetBuffer(0, "buffer1", buffer);

       _shader.Dispatch(0, 1, 1, 1);

        int[] data = new int[4];

        buffer.GetData(data);

        for (int i = 0; i < 4; i++)
        {
             Debug.Log(data[i]);
        }

         buffer.Release();
     }
}
{% endhightlight %}

Firstly we are creating a compute buffer the size of an int, a buffer that ComputeShader programs use to store arbitrary data and then we are using the SetBuffer to tell the shader to dump data in there. We use the dispatch function to run our shader and then grab the work the shader has done.

If you set up the above you should see in the debug window it print out some numbers. Yeah it did that on the Graphics card.

http://img2.tuicool.com/i2iqIfj.gif

Alright fine, it wasn’t the most crazy thing in the world, but it is just showing you that work other than just rendering pretty images can be done.

##Round up

This is a post to show you compute shaders are there. I am not saying go out and use them everywhere. The GPU can be used to do some cool multi threaded tasks, however a word to the wise. The tasks that the GPU can do are going to be limited and you really have to look at the problem you are trying to solve before you go down this path. If your game is gonna be super pretty in, you porbably want to be maxing out the gpu on that first before offloading stuff the cpu can do onto it. If your GPU is jsut idling though… maybe on like some lower poly strategy game, etc then maybe consider offloading some of the logic to the GPU using a compute shader.

Well until next time!