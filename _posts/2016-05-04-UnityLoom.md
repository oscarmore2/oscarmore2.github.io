---
layout: article
categories: unity
title: "Unity多线程（Thread）和主线程（MainThread）交互使用类——Loom工具分享"
---

#博主按：
无意中看到这篇文章，Loom这个插件的确让我震惊。想当时曾经做一个项目，里面有一个图片处理的模块非常耗时。当时只能在文件读写阶段使用多线程处理，其他图片处理过程都因为Unity本身的限制而没法使用多线程，导致用户体验下降。今日见此插件，算是了了心头的困惑。同时分享出来供大家学习。

###本文转载自：http://dsqiu.iteye.com/blog/2028503

#第一部分:问题描述
熟悉Unity的developer都知道在Unity中的线程不能使用Unity的对象，但可以使用Unity的值类型变量，如Vector3等。这样就使得线程在Unity中显的很鸡肋和蹩脚，因为很多函数很都是UnityEngine类或函数的调用的，对于哪些是可以在多线程使用，风雨冲进行了如下总结：
1. 变量(都能指向相同的内存地址)都是共享的
2. 不是UnityEngine的API能在分线程运行
3. UnityEngine定义的基本结构(int,float,Struct定义的数据类型)可以在分线程计算，如  Vector3(Struct) 可以 ， 但 Texture2d(class,根父类为Object)不可以。
4. UnityEngine定义的基本类型的函数可以在分线程运行，如：


`
int i = 99;  
print (i.ToString());  
Vector3 x = new Vector3(0,0,9);  
x.Normalize();
`

类的函数不能在分线程运行，如

`
obj.name  
`

实际是get_name函数，分线程报错误:get_name  can only be called from the main thread. 

`
Texture2D tt = new Texture2D(10,10); 
`

实际会调用UnityEngine里的Internal_Create，分线程报错误:Internal_Create  can only be called from the main thread.
其他transform.position，Texture.Apply()等等都不能在分线程里运行。
结论: 分线程可以做 基本类型的计算， 以及非Unity(包括.Net及SDK)的API。
D.S.Qiu(博客作者) 觉得Unity做了这个限制，主要是Unity的函数执行机制是帧序列调用，甚至连Unity的协程Coroutine的执行机制都是确定的，如果可以使用多线程访问UnityEngine的对象和api就得考虑同步问题了，也就是说Unity其实根本没有多线程的机制，协程只是达到一个延时或者是当指定条件满足是才继续执行的机制。

#第二部分：Threads on a Loom
我们的项目目前还有没有比较耗时的计算，所以还没有看到Thread的使用。本来一直没有太考虑着方面的事情，直到在UnityGems.com看到Loom这个类，叹为观止呀。这是他们官方的介绍：

Our class is called Loom.  Loom lets you easily run code on another thread and have that other thread run code on the main game thread when it needs to.

There are only two functions to worry about:

- RunAsync(Action) which runs a set of statements on another thread
- QueueOnMainThread(Action, [optional] float time) - which runs a set of statements on the main thread (with an optional delay).

You access Loom using Loom.Current - it deals with creating an invisible game object to interact with the games main thread

这个类主要有两个函数： RunAsync(Action) 和  QueueOnMainThread(Action, [optional] float time) 。原理也很简单：用线程池去运行RunAsync(Action)的函数，在Update中运行QueueOnMainThread(Acition, [optional] float time)传入的函数。
直接贴出源码，供拜读：


{% highlight CSharp %}
namespace Loom
{
	using UnityEngine;
	using System.Collections;
	using System.Collections.Generic;
	using System;
	using System.Threading;
	using System.Linq;

	public class Loom : MonoBehaviour
	{
	    public static int maxThreads = 8;
	    static int numThreads;
	    
	    private static Loom _current;
	    private int _count;
	    public static Loom Current
	    {
	        get
	        {
	            Initialize();
	            return _current;
	        }
	    }
	    
	    void Awake()
	    {
	        _current = this;
	        initialized = true;
	    }
	    
	    static bool initialized;
	    
	    static void Initialize()
	    {
	        if (!initialized)
	        {
	        
	            if(!Application.isPlaying)
	                return;
	            initialized = true;
	            var g = new GameObject("Loom");
	            _current = g.AddComponent<Loom>();
	        }
	            
	    }
	    
	    private List<Action> _actions = new List<Action>();
	    public struct DelayedQueueItem
	    {
	        public float time;
	        public Action action;
	    }
	    private List<DelayedQueueItem> _delayed = new  List<DelayedQueueItem>();

	    List<DelayedQueueItem> _currentDelayed = new List<DelayedQueueItem>();
	    
	    public static void QueueOnMainThread(Action action)
	    {
	        QueueOnMainThread( action, 0f);
	    }
	    public static void QueueOnMainThread(Action action, float time)
	    {
	        if(time != 0)
	        {
	            lock(Current._delayed)
	            {
	                Current._delayed.Add(new DelayedQueueItem { time = Time.time + time, action = action});
	            }
	        }
	        else
	        {
	            lock (Current._actions)
	            {
	                Current._actions.Add(action);
	            }
	        }
	    }
	    
	    public static Thread RunAsync(Action a)
	    {
	        Initialize();
	        while(numThreads >= maxThreads)
	        {
	            Thread.Sleep(1);
	        }
	        Interlocked.Increment(ref numThreads);
	        ThreadPool.QueueUserWorkItem(RunAction, a);
	        return null;
	    }
	    
	    private static void RunAction(object action)
	    {
	        try
	        {
	            ((Action)action)();
	        }
	        catch
	        {
	        }
	        finally
	        {
	            Interlocked.Decrement(ref numThreads);
	        }
	            
	    }
	    
	    
	    void OnDisable()
	    {
	        if (_current == this)
	        {
	            
	            _current = null;
	        }
	    }
	    
	    

	    // Use this for initialization
	    void Start()
	    {
	    
	    }
	    
	    List<Action> _currentActions = new List<Action>();
	    
	    // Update is called once per frame
	    void Update()
	    {
	        lock (_actions)
	        {
	            _currentActions.Clear();
	            _currentActions.AddRange(_actions);
	            _actions.Clear();
	        }
	        foreach(var a in _currentActions)
	        {
	            a();
	        }
	        lock(_delayed)
	        {
	            _currentDelayed.Clear();
	            _currentDelayed.AddRange(_delayed.Where(d=>d.time <= Time.time));
	            foreach(var item in _currentDelayed)
	                _delayed.Remove(item);
	        }
	        foreach(var delayed in _currentDelayed)
	        {
	            delayed.action();
	        }
	        
	        
	        
	    }
	}
}
{% endhighlight %}

怎么实现一个函数内使用多线程计算又保持函数体内代码的顺序执行，印象中使用多线程就是要摆脱代码块的顺序执行，但这里是把原本一个函数分拆成为两部分：一部分在C#线程中使用，另一部还是得在Unity的MainThread中使用，怎么解决呢，还得看例子：

{% highlight CSharp %}
//Scale a mesh on a second thread
void ScaleMesh(Mesh mesh, float scale)
{
    //Get the vertices of a mesh
    var vertices = mesh.vertices;
    //Run the action on a new thread
    Loom.RunAsync(()=>{
        //Loop through the vertices
        for(var i = 0; i < vertices.Length; i++)
        {
            //Scale the vertex
            vertices[i] = vertices[i] * scale;
        }
        //Run some code on the main thread
        //to update the mesh
        Loom.QueueOnMainThread(()=>{
            //Set the vertices
            mesh.vertices = vertices;
            //Recalculate the bounds
            mesh.RecalculateBounds();
        });
 
    });
}
{% endhighlight %}

D.S.Qiu的上述代码，如他自己阐述，是使用了闭包（closure）和lambda表达式的一个很好例子。这里本人也写了一例子供大家参考：

{% highlight CSharp %}
    ///This Method is going to scale an Texture2D picture by bilinear interpolation function
    void ScaleTexture2DBilinear (Texture2D originalTexture, float Factor, out Texture2D NewTexture)
    {
        NewTexture = new Texture2D(Mathf.CeilToInt (originalTexture.width * Factor), Mathf.CeilToInt (originalTexture.height * Factor));  
        float scale = 1.0f / Factor;
        int maxX = originalTexture.width - 1;
        int maxY = originalTexture.height - 1;

        Loom.RunAsync (()=>{
            for (int y = 0; y < NewTexture.height; y++)  
            {
                for (int x = 0; x < NewTexture.width; x++)
                {
                    // Bilinear Interpolation
                    float targetX = x * scale;
                    float targetY = y * scale;
                    int x1 = Mathf.Min(maxX, Mathf.FloorToInt(targetX));
                    int y1 = Mathf.Min(maxY, Mathf.FloorToInt(targetY));
                    int x2 = Mathf.Min(maxX, x1 + 1);
                    int y2 = Mathf.Min(maxY, y1 + 1);

                    float u = targetX - x1;
                    float v = targetY - y1 ;
                    float w1 = (1 - u) * (1 - v);
                    float w2 = u * (1 - v);
                    float w3 = (1 - u) * v;
                    float w4 = u * v;
                    Color color1 = originalTexture.GetPixel(x1, y1);
                    Color color2 = originalTexture.GetPixel(x2, y1);
                    Color color3 = originalTexture.GetPixel(x1, y2);
                    Color color4 = originalTexture.GetPixel(x2,  y2);
                    Color color = new Color(Mathf.Clamp01(color1.r * w1 + color2.r * w2 + color3.r * w3+ color4.r * w4),
                        Mathf.Clamp01(color1.g * w1 + color2.g * w2 + color3.g * w3 + color4.g * w4),
                        Mathf.Clamp01(color1.b * w1 + color2.b * w2 + color3.b * w3 + color4.b * w4),
                        Mathf.Clamp01(color1.a * w1 + color2.a * w2 + color3.a * w3 + color4.a * w4)
                    );
                    NewTexture.SetPixel(x, y, color); 
                }
            }

            Loom.QueueOnMianThread(()=>{
                NewTexture.Apply();
            });
        });
    }
{% endhighlight %}

Loom只有100多行的代码，实现起来非常容易简单，是让本人大为惊叹。本人对多线程编程的认识还只是九牛一毛，学习的路还有很长，同时对Loom的做出表示最衷心的佩服。
