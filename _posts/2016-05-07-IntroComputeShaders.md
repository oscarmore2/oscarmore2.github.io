---
layout: article
categories: unity
title: "Unity 材质简易教程：compute shader简介"
---

本文转载并翻译自：
Coxlin's Blog 

<a href="http://www.lindsaygcox.co.uk/tutorials/unity-shader-tutorial-an-intro-to-compute-shaders">http://www.lindsaygcox.co.uk/tutorials/unity-shader-tutorial-an-intro-to-compute-shaders</a>

原标题为：
Unity Shader Tutorial: An intro to Compute Shaders


#从头说起
你准备好学更多的东西了吗？现在要带踏入你真正3A级领域了--compute shaders(可计算着色器)*的国度。好，这个你根本不知道其存在的怪物是一种怎么样的东东呢？

*译者注：关于“compute shaders”的翻译，我是纠结了很久。我取“可计算着色器”这个译名是因为compute shaders则提供了一个入口让GPU可以实现一般性的计算。


说真的，我基本忘记他们的存在了，而当我找一些关于毛发效果的教程(并没有想象中这么有用而且方法太糟糕) 的时候才想起我的基友说过草地和毛发的效果可以用同一种方法实现。我想它说的是geometry shaders(几何着色器)，但compute shaders戳中了我的兴趣点。


可是，当我翻遍整个网络发现关于Unity的compute shaders的信息少之又少。

来我们从头说起

#Compute Shaders是什么，为什么要了解它？

在微软的花俏术语里面，它是这么说：“compute shaders是一种可编程的着色处理过程，它扩展了微软Direct3D 11并且超越了图形编程了领域”，“compute shaders利用了GPU大量的并行处理的特性提供了高速的通用计算功能”。

说得简单一点，compute shaders是一个在显卡上运行的程序，并且完成可以在正常渲染管线之外的任务。

你可能会说：“好吧，我好像明白了一丢丢，这玩意就是让一些逻辑和任务运行在显卡上面，但我干嘛要用它呢？”其实啊，这些Shader(着色器)对数学(尤其浮点)和并行计算方面有很大的本事。也就是说，他们同时处理同一种处理任务有很大本事。换句话说，他们非常擅长处理数据集合每个元素的运算。


这个解释有点蹩脚，所以让我略穿越一下，回到我对着星球放空的时候，那是90年代(...)。那真是游戏的黄金岁月，DOOM，最终幻想7，塞尔达传说： 时之笛，古惑狼赛车，铁拳3...还要在列举吗？(当时好游戏太多，本人补刀：魔法门，古墓丽影)。大部分的3D游戏都是PC游戏，尤其是有了显卡之后。


<figure>
	<img src="{{ site.url }}/images/2016-05-07-IntroComputeShaders/rqia6ff.jpg">
</figure>


6MB 的显存。不管怎么说，OpenGL 与 DirectX 的出现并展现了可编程渲染管线的威力。攻城狮尽管拿到把几何体数据传到显卡，OpenGL 和 DirectX就能自己干活了。可是，一般的渲染管线太死板，所以当我们想要更多的特效的时候，它得更加灵活才行。这使得着色器开始能让开发者通过代码控制管线的特定部位来实现魔法般的效果。


这个新系统让很多事情变得可能。并意味着新型的渲染管线能处理许多不同的数据类型和算法，如今的GPU甚至能处理如超级多线程的物理的计算了。


现在，我们可以像英伟达员工那样干很多牛逼的事情了。


你入坑了么？如果还没有，这没关系，你就带戴有色眼镜去用它就得了。


基本上，你要是这么渴望性能的话甚至还能驾驭CPU让它做非图形的任务。

#管他呢，入坑吧！

That’s the attitude I want!
这是我要的态度


开始之前你得有一台Windows机器。Mac不带玩。

在Unity，创建一个Shader。

首先你要意识到的这不是CG语言，这个是DirectX 11 的HLSL。绑好安全带哦，要飞代码了。

{% highlight C %}
// Each #kernel tells which function to compile; you can have many kernels
// 每个 #kernel (内核) 都指定了哪个方法需要编译，你可以指定多个内核
#pragma kernel CSMain

// Create a RenderTexture with enableRandomWrite flag and set it
// with cs.SetTexture
// 创建一个设置了enableRandomWrite标识的RenderTexture，并通过cs.SetTexture设置像素
RWTexture2D<float4> Result;

[numthreads(8,8,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
     // TODO: insert actual code here!

     Result[id.xy] = float4(id.x & id.y, (id.x & 15)/15.0, (id.y & 15)/15.0, 0.0);
}
{% endhighlight %}


 “#pragma kernel CSMain” 标识是咱们Shader的首行。跟其他着色器代码相类似。这个标识告诉程序咱们Shader的入口点的位置，我们现在的情况就是“CSMain”。compute shaders可以有很多函数，而你可以从程序的脚本调用。以后会慢慢说。


接着就是“RWTexture2D<float4> Result”


这也跟其他shader类似。这是变量声明。可是我们并没用到模型网格数据。我们得指定Shader输入输出。在例子中，RWTexture2D就是从程序输入并输出数据的2维图像对象。你可以参考一下MSDN：

<a href="https://msdn.microsoft.com/en-gb/library/windows/desktop/ff471505(v=vs.85).aspx">https://msdn.microsoft.com/en-us/library/windows/desktop/ff471442(v=vs.85).aspx</a>

最后，相当大的大区别的是"[numthreads(8,8,1)]" -- 指定着色器所运行的线程数量。GPU都爱并行地处理多个事务，通过创建线程同步地运行。这行代码指定了线程组的维度，同时也指定了线程创建和组织的方式。在例子中，我们创建了64个线程。参考MSDN文档。

<a href="https://msdn.microsoft.com/en-us/library/windows/desktop/ff471442(v=vs.85).aspx">https://msdn.microsoft.com/en-us/library/windows/desktop/ff471442(v=vs.85).aspx</a>


线程组的数量能够决定很多因素，也许是最显著的是你的目标硬件。举例来说，针对PS4的的线程组的数量值跟Xbox One的就不一样。


接下来都是比较标准的代码。这个内核函数指派了代表线程ID的uint3 类型数据，让特定的线程写入数据以完成缓冲。


好咧，我们有自己的的第一个compute shader了。但我们要如何才能运行它呢？它并不处理网格数据，所以没法附加一个网格进来。我们需要从脚步那里获得。


不过且慢，在我们开始修改脚步之前，我们要需要先改一下我们的compute shader。

{% highlight C %}
// Each #kernel tells which function to compile; you can have many kernels
#pragma kernel CSMain

RWStructuredBuffer<int> buffer1;
 
[numthreads(4,1,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
    buffer1[id.x] = mul(id.x, 2.0);
}
{% endhighlight %}


你看到我们用纹理代替了。现在这个只是一个单精度类型组成的数组，在这里是Int。 我们将赋值为线程ID的两倍。

接下来我们创建一个Unity脚本。

{% highlight CSharp %}
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
{% endhighlight %}


首先我们创建了int长度的compute buffer，这是ComputeShader存储各种数据的缓冲区，将来我们使用SetBuffer方法让着色器程序把数据转储到那里。我们用派遣函数让着色器运作，并把计算好的数据拿回来。


如果你完成了上述的设定，你应该能够在Debug窗口看到它打印了一些数字。对的，这是在显卡上完成的。

<figure>
	<img src="{{ site.url }}/images/2016-05-07-IntroComputeShaders/i2iqIfj.gif">
</figure>


很不错吧。这虽然算不上世界上最厉害的技术，但告诉你的显卡除了渲染画面以外还能干别的东西。

#总结

这个文章向你们展示了compute shaders的技术。虽然我不认为它们哪里都能用得上，但GPU可以干很多很酷的多线程的计算。不过话说回来，GPU能够做的东西目前还是很有限的，如果你要朝这个方向发展，你需要解决这个限制性的问题。假如你的游戏对画面要求比较高，可以考虑最大化地把一些本应在CPU完成的运算放到GPU上。但如果你的游戏对画面的要求一般的话，通过Computer Shader GPU也可以完成一些游戏逻辑(尤其是类似AI Pathfinding 之类的大计算量的操作)。

下次再见吧！