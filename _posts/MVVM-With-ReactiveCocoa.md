---
title: MVVM With ReactiveCocoa
tags: []
date: 2016-02-27 22:17:12
---

[MVVM](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel) 是一种软件架构模式，它是 [Martin Fowler](https://en.wikipedia.org/wiki/Martin_Fowler) 的 [Presentation Model](http://martinfowler.com/eaaDev/PresentationModel.html) 的一种变体，最先由微软的架构师 John Gossman 在 2005 年提出，并应用在微软的 [WPF](https://en.wikipedia.org/wiki/Windows_Presentation_Foundation) 和 [Silverlight](https://en.wikipedia.org/wiki/Microsoft_Silverlight) 软件开发中。`MVVM` 衍生于 [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) ，是对 `MVC` 的一种演进，它促进了 `UI` 代码与业务逻辑的分离。

**说明**：本文将采用理论与实践相结合的方式，重点介绍一个使用 `MVVM` 和 `RAC` 开发的 `iOS` 开源项目 [MVVMReactiveCocoa](https://github.com/leichunfeng/MVVMReactiveCocoa) ，目的是希望能为你实践 `MVVM` 提供帮助。不过，在正式开始介绍正文之前，请你先思考以下三个问题：

*   `MVC` 与 `MVVM` 有什么异同点，`MVC` 到 `MVVM` 是怎样演进的；
*   `RAC` 在 `MVVM` 中扮演什么样的角色，`MVVM` 是否一定要结合 `RAC` 使用；
*   如何将一个现有的 `MVC` 应用转变成一个 `MVVM` 应用，有哪些需要注意的地方。

带着以上问题，我们一起进入正文。

**名词解释**：本文中的 `RAC` 为 `ReactiveCocoa` 的缩写。

## MVC

`MVC` 是 `iOS` 开发中使用最普遍的架构模式，同时也是苹果官方推荐的架构模式。`MVC` 代表的是 `Model–view–controller` ，它们之间的关系如下：

![](http://blog.leichunfeng.com/images/M-V-C.png)

是的，`MVC` 看上去棒极了，`model` 代表数据，`view` 代表 `UI` ，而 `controller` 则负责协调它们两者之间的关系。然而，尽管从技术上看 `view` 和 `controller` 是相互独立的，但事实上它们几乎总是结对出现，一个 `view` 只能与一个 `controller` 进行匹配，反之亦然。既然如此，那我们为何不将它们看作一个整体呢：

![](http://blog.leichunfeng.com/images/M-VC.png)

因此，`M-VC` 可能是对 `iOS` 中的 `MVC` 模式更为准确的解读。在一个典型的 `MVC` 应用中，`controller` 由于承载了过多的逻辑，往往会变得臃肿不堪，所以 `MVC` 也经常被人调侃成 [Massive View Controller](https://twitter.com/Colin_Campbell/status/293167951132098560) ：

> iOS architecture, where MVC stands for Massive View Controller.

坦白说，有一部分逻辑确实是属于 `controller` 的，但是也有一部分逻辑是不应该被放置在 `controller` 中的。比如，将 `model` 中的 `NSDate` 转换成 `view` 可以展示的 `NSString` 等。在 `MVVM` 中，我们将这些逻辑统称为展示逻辑。

## MVVM

因此，一种可以很好地解决 `Massive View Controller` 问题的办法就是将 `controller` 中的展示逻辑抽取出来，放置到一个专门的地方，而这个地方就是 `viewModel` 。其实，我们只要在上图中的 `M-VC` 之间放入 `VM` ，就可以得到 `MVVM` 模式的结构图：

![](http://blog.leichunfeng.com/images/M-V-VM.png)

从上图中，我们可以非常清楚地看到 `MVVM` 中四个组件之间的关系。**注**：除了 `view` 、`viewModel` 和 `model` 之外，`MVVM` 中还有一个非常重要的隐含组件 `binder` ：

*   `view` ：由 `MVC` 中的 `view` 和 `controller` 组成，负责 `UI` 的展示，绑定 `viewModel` 中的属性，触发 `viewModel` 中的命令；
*   `viewModel` ：从 `MVC` 的 `controller` 中抽取出来的展示逻辑，负责从 `model` 中获取 `view` 所需的数据，转换成 `view` 可以展示的数据，并暴露公开的属性和命令供 `view` 进行绑定；
*   `model` ：与 `MVC` 中的 `model` 一致，包括数据模型、访问数据库的操作和网络请求等；
*   `binder` ：在 `MVVM` 中，声明式的数据和命令绑定是一个隐含的约定，它可以让开发者非常方便地实现 `view` 和 `viewModel` 的同步，避免编写大量繁杂的样板化代码。在微软的 `MVVM` 实现中，使用的是一种被称为 [XAML](https://en.wikipedia.org/wiki/Extensible_Application_Markup_Language) 的标记语言。

## ReactiveCocoa

尽管，在 `iOS` 开发中，系统并没有提供类似的框架可以让我们方便地实现 `binder` 功能，不过，值得庆幸的是，`GitHub` 开源的 `RAC` ，给了我们一个非常不错的选择。

`RAC` 是一个 `iOS` 中的函数式响应式编程框架，它受 [Functional Reactive Programming](https://en.wikipedia.org/wiki/Functional_reactive_programming) 的启发，是 [Justin Spahr-Summers](https://github.com/jspahrsummers) 和 [Josh Abernathy](https://github.com/joshaber) 在开发 [GitHub for Mac](https://desktop.github.com/) 过程中的一个副产品，它提供了一系列用来组合和转换值流的 `API` 。如需了解更多关于 `RAC` 的信息，可以阅读我的上一篇文章[《ReactiveCocoa v2.5 源码解析之架构总览》](http://blog.leichunfeng.com/blog/2015/12/25/reactivecocoa-v2-dot-5-yuan-ma-jie-xi-zhi-jia-gou-zong-lan/)。

在 `iOS` 的 `MVVM` 实现中，我们可以使用 `RAC` 来在 `view` 和 `viewModel` 之间充当 `binder` 的角色，优雅地实现两者之间的同步。此外，我们还可以把 `RAC` 用在 `model` 层，使用 `Signal` 来代表异步的数据获取操作，比如读取文件、访问数据库和网络请求等。**说明**，`RAC` 的后一个应用场景是与 `MVVM` 无关的，也就是说，我们同样可以在 `MVC` 的 `model` 层这么用。

## 小结

综上所述，我们只要将 `MVC` 中的 `controller` 中的展示逻辑抽取出来，放置到 `viewModel` 中，然后通过一定的技术手段，比如 `RAC` 来同步 `view` 和 `viewModel` ，就完成了 `MVC` 到 `MVVM` 的转变。

> Talk is cheap. Show me the code.

下面，我们直接上代码，一起来看一个 `MVC` 模式转换成 `MVVM` 模式的示例。首先是 `model` 层的代码 `Person` ：

<figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
<span class='line-number'>2</span>
<span class='line-number'>3</span>
<span class='line-number'>4</span>
<span class='line-number'>5</span>
<span class='line-number'>6</span>
<span class='line-number'>7</span>
<span class='line-number'>8</span>
<span class='line-number'>9</span>
<span class='line-number'>10</span>
</pre></td><td class='code'>

    <span class='line'><span class="k">@interface</span> <span class="nc">Person</span> : <span class="bp">NSObject</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">instancetype</span><span class="p">)</span><span class="nf">initwithSalutation:</span><span class="p">(</span><span class="bp">NSString</span> <span class="o">*</span><span class="p">)</span><span class="nv">salutation</span> <span class="nf">firstName:</span><span class="p">(</span><span class="bp">NSString</span> <span class="o">*</span><span class="p">)</span><span class="nv">firstName</span> <span class="nf">lastName:</span><span class="p">(</span><span class="bp">NSString</span> <span class="o">*</span><span class="p">)</span><span class="nv">lastName</span> <span class="nf">birthdate:</span><span class="p">(</span><span class="bp">NSDate</span> <span class="o">*</span><span class="p">)</span><span class="nv">birthdate</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">copy</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="bp">NSString</span> <span class="o">*</span><span class="n">salutation</span><span class="p">;</span>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">copy</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="bp">NSString</span> <span class="o">*</span><span class="n">firstName</span><span class="p">;</span>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">copy</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="bp">NSString</span> <span class="o">*</span><span class="n">lastName</span><span class="p">;</span>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">copy</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="bp">NSDate</span> <span class="o">*</span><span class="n">birthdate</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@end</span>
    </span>`</pre></td></tr></table></div></figure>

    然后是 `view` 层的代码 `PersonViewController` ，在 `viewDidLoad` 方法中，我们将 `Person` 中的属性进行一定的转换后，赋值给相应的 `view` 进行展示：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    <span class='line-number'>12</span>
    <span class='line-number'>13</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">viewDidLoad</span> <span class="p">{</span>
    </span><span class='line'>    <span class="p">[</span><span class="nb">super</span> <span class="n">viewDidLoad</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="k">if</span> <span class="p">(</span><span class="nb">self</span><span class="p">.</span><span class="n">model</span><span class="p">.</span><span class="n">salutation</span><span class="p">.</span><span class="n">length</span> <span class="o">&gt;</span> <span class="mi">0</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>        <span class="nb">self</span><span class="p">.</span><span class="n">nameLabel</span><span class="p">.</span><span class="n">text</span> <span class="o">=</span> <span class="p">[</span><span class="bp">NSString</span> <span class="nl">stringWithFormat</span><span class="p">:</span><span class="s">@&quot;%@ %@ %@&quot;</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">model</span><span class="p">.</span><span class="n">salutation</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">model</span><span class="p">.</span><span class="n">firstName</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">model</span><span class="p">.</span><span class="n">lastName</span><span class="p">];</span>
    </span><span class='line'>    <span class="p">}</span> <span class="k">else</span> <span class="p">{</span>
    </span><span class='line'>        <span class="nb">self</span><span class="p">.</span><span class="n">nameLabel</span><span class="p">.</span><span class="n">text</span> <span class="o">=</span> <span class="p">[</span><span class="bp">NSString</span> <span class="nl">stringWithFormat</span><span class="p">:</span><span class="s">@&quot;%@ %@&quot;</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">model</span><span class="p">.</span><span class="n">firstName</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">model</span><span class="p">.</span><span class="n">lastName</span><span class="p">];</span>
    </span><span class='line'>    <span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="bp">NSDateFormatter</span> <span class="o">*</span><span class="n">dateFormatter</span> <span class="o">=</span> <span class="p">[[</span><span class="bp">NSDateFormatter</span> <span class="n">alloc</span><span class="p">]</span> <span class="n">init</span><span class="p">];</span>
    </span><span class='line'>    <span class="p">[</span><span class="n">dateFormatter</span> <span class="nl">setDateFormat</span><span class="p">:</span><span class="s">@&quot;EEEE MMMM d, yyyy&quot;</span><span class="p">];</span>
    </span><span class='line'>    <span class="nb">self</span><span class="p">.</span><span class="n">birthdateLabel</span><span class="p">.</span><span class="n">text</span> <span class="o">=</span> <span class="p">[</span><span class="n">dateFormatter</span> <span class="nl">stringFromDate</span><span class="p">:</span><span class="n">model</span><span class="p">.</span><span class="n">birthdate</span><span class="p">];</span>
    </span><span class='line'><span class="p">}</span>
    </span>`</pre></td></tr></table></div></figure>

    接下来，我们引入一个 `viewModel` ，将 `PersonViewController` 中的展示逻辑抽取到这个 `PersonViewModel` 中：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    <span class='line-number'>12</span>
    <span class='line-number'>13</span>
    <span class='line-number'>14</span>
    <span class='line-number'>15</span>
    <span class='line-number'>16</span>
    <span class='line-number'>17</span>
    <span class='line-number'>18</span>
    <span class='line-number'>19</span>
    <span class='line-number'>20</span>
    <span class='line-number'>21</span>
    <span class='line-number'>22</span>
    <span class='line-number'>23</span>
    <span class='line-number'>24</span>
    <span class='line-number'>25</span>
    <span class='line-number'>26</span>
    <span class='line-number'>27</span>
    <span class='line-number'>28</span>
    <span class='line-number'>29</span>
    <span class='line-number'>30</span>
    <span class='line-number'>31</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">@interface</span> <span class="nc">PersonViewModel</span> : <span class="bp">NSObject</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">instancetype</span><span class="p">)</span><span class="nf">initWithPerson:</span><span class="p">(</span><span class="n">Person</span> <span class="o">*</span><span class="p">)</span><span class="nv">person</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">strong</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="n">Person</span> <span class="o">*</span><span class="n">person</span><span class="p">;</span>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">copy</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="bp">NSString</span> <span class="o">*</span><span class="n">nameText</span><span class="p">;</span>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">copy</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="bp">NSString</span> <span class="o">*</span><span class="n">birthdateText</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@end</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@implementation</span> <span class="nc">PersonViewModel</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">instancetype</span><span class="p">)</span><span class="nf">initWithPerson:</span><span class="p">(</span><span class="n">Person</span> <span class="o">*</span><span class="p">)</span><span class="nv">person</span> <span class="p">{</span>
    </span><span class='line'>    <span class="nb">self</span> <span class="o">=</span> <span class="p">[</span><span class="nb">super</span> <span class="n">init</span><span class="p">];</span>
    </span><span class='line'>    <span class="k">if</span> <span class="p">(</span><span class="nb">self</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>       <span class="n">_person</span> <span class="o">=</span> <span class="n">person</span><span class="p">;</span>
    </span><span class='line'>  
    </span><span class='line'>      <span class="k">if</span> <span class="p">(</span><span class="n">person</span><span class="p">.</span><span class="n">salutation</span><span class="p">.</span><span class="n">length</span> <span class="o">&gt;</span> <span class="mi">0</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>          <span class="n">_nameText</span> <span class="o">=</span> <span class="p">[</span><span class="bp">NSString</span> <span class="nl">stringWithFormat</span><span class="p">:</span><span class="s">@&quot;%@ %@ %@&quot;</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">person</span><span class="p">.</span><span class="n">salutation</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">person</span><span class="p">.</span><span class="n">firstName</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">person</span><span class="p">.</span><span class="n">lastName</span><span class="p">];</span>
    </span><span class='line'>      <span class="p">}</span> <span class="k">else</span> <span class="p">{</span>
    </span><span class='line'>          <span class="n">_nameText</span> <span class="o">=</span> <span class="p">[</span><span class="bp">NSString</span> <span class="nl">stringWithFormat</span><span class="p">:</span><span class="s">@&quot;%@ %@&quot;</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">person</span><span class="p">.</span><span class="n">firstName</span><span class="p">,</span> <span class="nb">self</span><span class="p">.</span><span class="n">person</span><span class="p">.</span><span class="n">lastName</span><span class="p">];</span>
    </span><span class='line'>      <span class="p">}</span>
    </span><span class='line'>  
    </span><span class='line'>      <span class="bp">NSDateFormatter</span> <span class="o">*</span><span class="n">dateFormatter</span> <span class="o">=</span> <span class="p">[[</span><span class="bp">NSDateFormatter</span> <span class="n">alloc</span><span class="p">]</span> <span class="n">init</span><span class="p">];</span>
    </span><span class='line'>      <span class="p">[</span><span class="n">dateFormatter</span> <span class="nl">setDateFormat</span><span class="p">:</span><span class="s">@&quot;EEEE MMMM d, yyyy&quot;</span><span class="p">];</span>
    </span><span class='line'>      <span class="n">_birthdateText</span> <span class="o">=</span> <span class="p">[</span><span class="n">dateFormatter</span> <span class="nl">stringFromDate</span><span class="p">:</span><span class="n">person</span><span class="p">.</span><span class="n">birthdate</span><span class="p">];</span>
    </span><span class='line'>    <span class="p">}</span>
    </span><span class='line'>    <span class="k">return</span> <span class="nb">self</span><span class="p">;</span>
    </span><span class='line'><span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@end</span>
    </span>`</pre></td></tr></table></div></figure>

    最终，`PersonViewController` 将会变得非常轻量级：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">viewDidLoad</span> <span class="p">{</span>
    </span><span class='line'>    <span class="p">[</span><span class="nb">super</span> <span class="n">viewDidLoad</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="nb">self</span><span class="p">.</span><span class="n">nameLabel</span><span class="p">.</span><span class="n">text</span> <span class="o">=</span> <span class="nb">self</span><span class="p">.</span><span class="n">viewModel</span><span class="p">.</span><span class="n">nameText</span><span class="p">;</span>
    </span><span class='line'>    <span class="nb">self</span><span class="p">.</span><span class="n">birthdateLabel</span><span class="p">.</span><span class="n">text</span> <span class="o">=</span> <span class="nb">self</span><span class="p">.</span><span class="n">viewModel</span><span class="p">.</span><span class="n">birthdateText</span><span class="p">;</span>
    </span><span class='line'><span class="p">}</span>
    </span>`</pre></td></tr></table></div></figure>

    怎么样？其实 `MVVM` 并没有想像中的那么难吧，而且更重要的是它也没有破坏 `MVC` 的现有结构，只不过是移动了一些代码，仅此而已。好了，说了这么多，那 `MVVM` 相比 `MVC` 到底有哪些好处呢？我想，主要可以归纳为以下三点：

*   由于展示逻辑被抽取到了 `viewModel` 中，所以 `view` 中的代码将会变得非常轻量级；
*   由于 `viewModel` 中的代码是与 `UI` 无关的，所以它具有良好的可测试性；
*   对于一个封装了大量业务逻辑的 `model` 来说，改变它可能会比较困难，并且存在一定的风险。在这种场景下，`viewModel` 可以作为 `model` 的适配器使用，从而避免对 `model` 进行较大的改动。

    通过前面的示例，我们对第一点已经有了一定的感触；至于第三点，可能对于一个复杂的大型应用来说，才会比较明显；下面，我们还是使用前面的示例，来直观地感受下第二点好处：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    <span class='line-number'>12</span>
    <span class='line-number'>13</span>
    <span class='line-number'>14</span>
    <span class='line-number'>15</span>
    <span class='line-number'>16</span>
    <span class='line-number'>17</span>
    <span class='line-number'>18</span>
    <span class='line-number'>19</span>
    <span class='line-number'>20</span>
    <span class='line-number'>21</span>
    <span class='line-number'>22</span>
    <span class='line-number'>23</span>
    <span class='line-number'>24</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">SpecBegin</span><span class="p">(</span><span class="n">Person</span><span class="p">)</span>
    </span><span class='line'>    <span class="bp">NSString</span> <span class="o">*</span><span class="n">salutation</span> <span class="o">=</span> <span class="s">@&quot;Dr.&quot;</span><span class="p">;</span>
    </span><span class='line'>    <span class="bp">NSString</span> <span class="o">*</span><span class="n">firstName</span> <span class="o">=</span> <span class="s">@&quot;first&quot;</span><span class="p">;</span>
    </span><span class='line'>    <span class="bp">NSString</span> <span class="o">*</span><span class="n">lastName</span> <span class="o">=</span> <span class="s">@&quot;last&quot;</span><span class="p">;</span>
    </span><span class='line'>    <span class="bp">NSDate</span> <span class="o">*</span><span class="n">birthdate</span> <span class="o">=</span> <span class="p">[</span><span class="bp">NSDate</span> <span class="nl">dateWithTimeIntervalSince1970</span><span class="p">:</span><span class="mi">0</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="n">it</span> <span class="p">(</span><span class="s">@&quot;should use the salutation available. &quot;</span><span class="p">,</span> <span class="o">^</span><span class="p">{</span>
    </span><span class='line'>        <span class="n">Person</span> <span class="o">*</span><span class="n">person</span> <span class="o">=</span> <span class="p">[[</span><span class="n">Person</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithSalutation</span><span class="p">:</span><span class="n">salutation</span> <span class="nl">firstName</span><span class="p">:</span><span class="n">firstName</span> <span class="nl">lastName</span><span class="p">:</span><span class="n">lastName</span> <span class="nl">birthdate</span><span class="p">:</span><span class="n">birthdate</span><span class="p">];</span>
    </span><span class='line'>        <span class="n">PersonViewModel</span> <span class="o">*</span><span class="n">viewModel</span> <span class="o">=</span> <span class="p">[[</span><span class="n">PersonViewModel</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithPerson</span><span class="p">:</span><span class="n">person</span><span class="p">];</span>
    </span><span class='line'>        <span class="n">expect</span><span class="p">(</span><span class="n">viewModel</span><span class="p">.</span><span class="n">nameText</span><span class="p">).</span><span class="n">to</span><span class="p">.</span><span class="n">equal</span><span class="p">(</span><span class="s">@&quot;Dr. first last&quot;</span><span class="p">);</span>
    </span><span class='line'>    <span class="p">});</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="n">it</span> <span class="p">(</span><span class="s">@&quot;should not use an unavailable salutation. &quot;</span><span class="p">,</span> <span class="o">^</span><span class="p">{</span>
    </span><span class='line'>        <span class="n">Person</span> <span class="o">*</span><span class="n">person</span> <span class="o">=</span> <span class="p">[[</span><span class="n">Person</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithSalutation</span><span class="p">:</span><span class="nb">nil</span> <span class="nl">firstName</span><span class="p">:</span><span class="n">firstName</span> <span class="nl">lastName</span><span class="p">:</span><span class="n">lastName</span> <span class="nl">birthdate</span><span class="p">:</span><span class="n">birthdate</span><span class="p">];</span>
    </span><span class='line'>        <span class="n">PersonViewModel</span> <span class="o">*</span><span class="n">viewModel</span> <span class="o">=</span> <span class="p">[[</span><span class="n">PersonViewModel</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithPerson</span><span class="p">:</span><span class="n">person</span><span class="p">];</span>
    </span><span class='line'>        <span class="n">expect</span><span class="p">(</span><span class="n">viewModel</span><span class="p">.</span><span class="n">nameText</span><span class="p">).</span><span class="n">to</span><span class="p">.</span><span class="n">equal</span><span class="p">(</span><span class="s">@&quot;first last&quot;</span><span class="p">);</span>
    </span><span class='line'>    <span class="p">});</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="n">it</span> <span class="p">(</span><span class="s">@&quot;should use the correct date format. &quot;</span><span class="p">,</span> <span class="o">^</span><span class="p">{</span>
    </span><span class='line'>        <span class="n">Person</span> <span class="o">*</span><span class="n">person</span> <span class="o">=</span> <span class="p">[[</span><span class="n">Person</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithSalutation</span><span class="p">:</span><span class="nb">nil</span> <span class="nl">firstName</span><span class="p">:</span><span class="n">firstName</span> <span class="nl">lastName</span><span class="p">:</span><span class="n">lastName</span> <span class="nl">birthdate</span><span class="p">:</span><span class="n">birthdate</span><span class="p">];</span>
    </span><span class='line'>        <span class="n">PersonViewModel</span> <span class="o">*</span><span class="n">viewModel</span> <span class="o">=</span> <span class="p">[[</span><span class="n">PersonViewModel</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithPerson</span><span class="p">:</span><span class="n">person</span><span class="p">];</span>
    </span><span class='line'>        <span class="n">expect</span><span class="p">(</span><span class="n">viewModel</span><span class="p">.</span><span class="n">birthdateText</span><span class="p">).</span><span class="n">to</span><span class="p">.</span><span class="n">equal</span><span class="p">(</span><span class="s">@&quot;Thursday January 1, 1970&quot;</span><span class="p">);</span>
    </span><span class='line'>    <span class="p">});</span>
    </span><span class='line'><span class="n">SpecEnd</span>
    </span>`</pre></td></tr></table></div></figure>

    对于 `MVVM` 来说，我们可以把 `view` 看作是 `viewModel` 的可视化形式，`viewModel` 提供了 `view` 所需的数据和命令。因此，`viewModel` 的可测试性可以帮助我们极大地提高应用的质量。

    ## MVVMReactiveCocoa

    接下来，我们进入本文的第二部分，重点介绍一个使用 `MVVM` 和 `RAC` 开发的开源项目 `MVVMReactiveCocoa` 。**说明**，本文将主要介绍这个应用的架构和设计思路，希望可以为你实践 `MVVM` 提供一个真实的参考案例，有些架构并非是 `MVVM` 所必须的，而是我们为了更顺畅地使用 `MVVM` 而引入的，特别是 `ViewModel-Based Navigation` 。所以，请你在实践的过程中能够结合自身应用的实际情况做出相应的取舍，灵活处理。最后，我们将以登录界面为例，一起探讨下 `MVVM` 的实践思路。

    **说明**，以下内容均基于 `MVVMReactiveCocoa` 的 [v2.1.1](https://github.com/leichunfeng/MVVMReactiveCocoa/tree/v2.1.1) 标签进行展开，并且对部分无关代码做了删减。

    ### 类图

    为了方便我们从宏观上了解 `MVVMReactiveCocoa` 的整体结构，我们先来看看它的类图：

    ![MVVMReactiveCocoa-v2.1.1](http://blog.leichunfeng.com/images/MVVMReactiveCocoa-v2.1.1.png)

    从上图中，我们可以看到，在 `MVVMReactiveCocoa` 中主要有两大继承体系：

*   用蓝色标识出来的 `viewModel` 的继承体系，基类为 `MRCViewModel` ；
*   用红色标识出来的 `view` 的继承体系，基类为 `MRCViewController` 。

    除了提供与系统基类 `UIViewController` 相对应的基类 `MRCViewModel/MRCViewController` 外，还提供了与系统基类 `UITableViewController` 和 `UITabBarController` 相对应的基类 `MRCTableViewModel/MRCTableViewController` 和 `MRCTabBarViewModel/MRCTabBarController` ，其中基类 `MRCTableViewModel/MRCTableViewController` 的使用最为普遍。

    **说明**，之所以通过基类的方式来组织 `MVVMReactiveCocoa` ，一方面是因为主要开发者只有我一个人，这个方案非常容易实施；另一方面是因为通过基类的方式可以尽可能简单地实现代码重用，提高开发效率。

    ### 服务总线

    经过前面的探讨，我们已经知道了 `MVVM` 中的 `viewModel` 的主要职责就是从 `model` 层获取 `view` 所需的数据，并且将这些数据转换成 `view` 能够展示的形式。因此，为了方便 `viewModel` 层调用 `model` 层中的所有服务，并且统一管理这些服务的创建，我使用抽象工厂模式将 `model` 层的所有服务集中管理了起来，结构图如下：

    ![](http://blog.leichunfeng.com/images/service-bus.png)

    从上图中，我们可以看出，在服务总线类 `MRCViewModelServices/MRCViewModelServicesImpl` 中，主要包括以下三个方面的内容：

*   应用自有的服务类，用柚黄色进行了标识，包括 `MRCAppStoreService/MRCAppStoreServiceImpl` 和 `MRCRepositoryService/MRCRepositoryServiceImpl` 两个服务类；
*   第三方 `GitHub` 提供的 `API` 框架，用天蓝色进行了标识，主要包括 `OCTClient` 服务类；
*   应用的导航服务，用藻绿色进行了标识，包括 `MRCNavigationProtocol` 协议和实现类 `MRCViewModelServicesImpl` 等。

    其中，前两者都是以信号的形式对 `viewModel` 层提供服务，代表异步的网络请求等数据获取操作，而我们在 `viewModel` 层则可以通过订阅信号的形式获取到所需的数据。此外，服务总线还实现了 `MRCNavigationProtocol` 协议，它的内容如下：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    <span class='line-number'>12</span>
    <span class='line-number'>13</span>
    <span class='line-number'>14</span>
    <span class='line-number'>15</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">@protocol</span> <span class="nc">MRCNavigationProtocol</span> <span class="o">&lt;</span><span class="bp">NSObject</span><span class="o">&gt;</span>
    </span><span class='line'>
    </span><span class='line'><span class="o">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nl">pushViewModel</span><span class="p">:(</span><span class="n">MRCViewModel</span> <span class="o">*</span><span class="p">)</span><span class="n">viewModel</span> <span class="nl">animated</span><span class="p">:(</span><span class="kt">BOOL</span><span class="p">)</span><span class="n">animated</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">popViewModelAnimated:</span><span class="p">(</span><span class="kt">BOOL</span><span class="p">)</span><span class="nv">animated</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">popToRootViewModelAnimated:</span><span class="p">(</span><span class="kt">BOOL</span><span class="p">)</span><span class="nv">animated</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">presentViewModel:</span><span class="p">(</span><span class="n">MRCViewModel</span> <span class="o">*</span><span class="p">)</span><span class="nv">viewModel</span> <span class="nf">animated:</span><span class="p">(</span><span class="kt">BOOL</span><span class="p">)</span><span class="nv">animated</span> <span class="nf">completion:</span><span class="p">(</span><span class="n">VoidBlock</span><span class="p">)</span><span class="nv">completion</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">dismissViewModelAnimated:</span><span class="p">(</span><span class="kt">BOOL</span><span class="p">)</span><span class="nv">animated</span> <span class="nf">completion:</span><span class="p">(</span><span class="n">VoidBlock</span><span class="p">)</span><span class="nv">completion</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">resetRootViewModel:</span><span class="p">(</span><span class="n">MRCViewModel</span> <span class="o">*</span><span class="p">)</span><span class="nv">viewModel</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@end</span>
    </span>`</pre></td></tr></table></div></figure>

    看上去是不是有点眼熟？是的，`MRCNavigationProtocol` 协议其实就是参照系统的导航操作定义出来的，用来实现 `ViewModel-Based` 的导航服务。**注意**，服务总线类 `MRCViewModelServicesImpl` 其实并没有真正实现 `MRCNavigationProtocol` 协议中声明的操作，只不过是实现了一些空操作而已：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">pushViewModel:</span><span class="p">(</span><span class="n">MRCViewModel</span> <span class="o">*</span><span class="p">)</span><span class="nv">viewModel</span> <span class="nf">animated:</span><span class="p">(</span><span class="kt">BOOL</span><span class="p">)</span><span class="nv">animated</span> <span class="p">{}</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">popViewModelAnimated:</span><span class="p">(</span><span class="kt">BOOL</span><span class="p">)</span><span class="nv">animated</span> <span class="p">{}</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">popToRootViewModelAnimated:</span><span class="p">(</span><span class="kt">BOOL</span><span class="p">)</span><span class="nv">animated</span> <span class="p">{}</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">presentViewModel:</span><span class="p">(</span><span class="n">MRCViewModel</span> <span class="o">*</span><span class="p">)</span><span class="nv">viewModel</span> <span class="nf">animated:</span><span class="p">(</span><span class="kt">BOOL</span><span class="p">)</span><span class="nv">animated</span> <span class="nf">completion:</span><span class="p">(</span><span class="n">VoidBlock</span><span class="p">)</span><span class="nv">completion</span> <span class="p">{}</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">dismissViewModelAnimated:</span><span class="p">(</span><span class="kt">BOOL</span><span class="p">)</span><span class="nv">animated</span> <span class="nf">completion:</span><span class="p">(</span><span class="n">VoidBlock</span><span class="p">)</span><span class="nv">completion</span> <span class="p">{}</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">resetRootViewModel:</span><span class="p">(</span><span class="n">MRCViewModel</span> <span class="o">*</span><span class="p">)</span><span class="nv">viewModel</span> <span class="p">{}</span>
    </span>`</pre></td></tr></table></div></figure>

    那么，我们是怎么实现 `ViewModel-Based` 的导航操作的呢？用 `MRCViewModelServicesImpl` 来实现这些空操作到底有什么用意？为什么要这么做，目的是为了什么？兄台，莫急，请接着看下一小节的内容。

    ### ViewModel-Based Navigation

    我们先来思考一个问题，就是我们为什么要实现 `ViewModel-Based` 的导航操作呢？直接在 `view` 层使用系统的 `push/present` 等操作来完成导航不就好了么？我总结了一下这么做的理由，主要有以下三点：

*   从理论上来说，`MVVM` 模式的应用应该是以 `viewModel` 为驱动来运转的；
*   根据我们前面对 `MVVM` 的探讨，`viewModel` 提供了 `view` 所需的数据和命令。因此，我们往往可以直接在命令执行成功后使用 `doNext` 顺带就把导航操作给做了，一气呵成；
*   这样可以使 `view` 更加轻量级，只需要绑定 `viewModel` 提供的数据和命令即可。

    既然如此，那我们究竟要如何实现 `ViewModel-Based` 的导航操作呢？我们都知道 `iOS` 中的导航操作无外乎两种，`push/pop` 和 `present/dismiss` ，前者是 `UINavigationController` 特有的功能，而后者是所有 `UIViewController` 都具备的功能。**注意**，`UINavigationController` 也是 `UIViewController` 的子类，所以它也同样具备 `present/dismiss` 的功能。因此，从本质上来说，不管我们要实现什么样的导航操作，最终都是离不开 `push/pop` 和 `present/dismiss` 的。

    目前，`MVVMReactiveCocoa` 的做法是在 `view` 层维护一个 `NavigationController` 的堆栈 `MRCNavigationControllerStack` ，不管是 `push/pop` 还是 `present/dismiss` ，都使用栈顶的 `NavigationController` 来执行导航操作，并且保证 `present` 出来的是一个 `NavigationController` 。

    接下来，我们一起来看看 `MVVMReactiveCocoa` 在执行了 `push/pop` 或 `present/dismiss` 操作后视图层次结构的变化过程。首先，我们来看看用户在登录成功后进入到首页时应用的视图层次结构图：

    ![](http://blog.leichunfeng.com/images/view-model-based1.png)

    此时，应用展示的界面是 `NewsViewController` 。在 `MRCNavigationControllerStack` 堆栈中只有 `NavigationController0` 一个元素；而 `NavigationController1` 并没有在 `MRCNavigationControllerStack` 堆栈中，这是因为需要支持 `TabBarController` 的滑动切换而设计的视图层次结构，是首页比较特殊的一个地方。更多信息可以查看 `GitHub` 开源库 [WXTabBarController](https://github.com/leichunfeng/WXTabBarController) ，在这里，我们不用太过于关心这个问题，只需要理解原理就好了。

    接下来，当用户在 `NewsViewController` 界面，点击了某一个 `cell` ，通过 `push` 的方式，进入到仓库详情界面时，应用的视图层次结构图如下：

    ![](http://blog.leichunfeng.com/images/view-model-based2.png)

    应用通过 `MRCNavigationControllerStack` 栈顶的元素 `NavigationController0` ，将仓库详情界面 `push` 到了自身的堆栈中。此时，应用展示的界面是被 `push` 进来的仓库详情界面 `RepoDetailViewController` 。最后，当用户在仓库详情界面，点击左下角的切换分支按钮，通过 `present` 的方式，弹出分支选择界面时，应用的视图层次结构图如下：

    ![](http://blog.leichunfeng.com/images/view-model-based3.png)

    应用通过 `MRCNavigationControllerStack` 栈顶的元素 `NavigationController0` ，将 `NavigationController5` 以 `present` 的方式弹出来。此时，应用展示的是 `NavigationController5` 的根视图 `SelectBranchOrTagViewController` 。**说明**，由于 `pop` 和 `dismiss` 与 `push` 和 `present` 互为逆操作，所以只要按照从下到上的顺序看上面的视图层次结构图即可，这里不再赘述。

    等等，如果我没有记错的话，`MRCNavigationControllerStack` 堆栈是在 `view` 层，而服务总线类 `MRCViewModelServicesImpl` 是在 `viewModel` 层的。据我所知，`viewModel` 层是不能引入 `view` 层的任何东西的，更严格的说，是不能引入任何 `UIKit` 中的东西的，否则就违背了 `MVVM` 的基本原则，并且也会散失 `viewModel` 的可测试性。在这个前提下，你要如何让这两者产生关联呢？

    没错，这就是 `MRCViewModelServicesImpl` 中之所以实现那些空操作的目的所在了。`viewModel` 通过调用 `MRCViewModelServicesImpl` 中的空操作来表明需要执行相应的导航操作，而 `MRCNavigationControllerStack` 则通过 `Hook` 来捕获这些空操作，然后使用栈顶的 `NavigationController` 来执行真正的导航操作：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    <span class='line-number'>12</span>
    <span class='line-number'>13</span>
    <span class='line-number'>14</span>
    <span class='line-number'>15</span>
    <span class='line-number'>16</span>
    <span class='line-number'>17</span>
    <span class='line-number'>18</span>
    <span class='line-number'>19</span>
    <span class='line-number'>20</span>
    <span class='line-number'>21</span>
    <span class='line-number'>22</span>
    <span class='line-number'>23</span>
    <span class='line-number'>24</span>
    <span class='line-number'>25</span>
    <span class='line-number'>26</span>
    <span class='line-number'>27</span>
    <span class='line-number'>28</span>
    <span class='line-number'>29</span>
    <span class='line-number'>30</span>
    <span class='line-number'>31</span>
    <span class='line-number'>32</span>
    <span class='line-number'>33</span>
    <span class='line-number'>34</span>
    <span class='line-number'>35</span>
    <span class='line-number'>36</span>
    <span class='line-number'>37</span>
    <span class='line-number'>38</span>
    <span class='line-number'>39</span>
    <span class='line-number'>40</span>
    <span class='line-number'>41</span>
    <span class='line-number'>42</span>
    <span class='line-number'>43</span>
    <span class='line-number'>44</span>
    <span class='line-number'>45</span>
    <span class='line-number'>46</span>
    <span class='line-number'>47</span>
    <span class='line-number'>48</span>
    <span class='line-number'>49</span>
    <span class='line-number'>50</span>
    <span class='line-number'>51</span>
    <span class='line-number'>52</span>
    <span class='line-number'>53</span>
    <span class='line-number'>54</span>
    <span class='line-number'>55</span>
    <span class='line-number'>56</span>
    <span class='line-number'>57</span>
    <span class='line-number'>58</span>
    <span class='line-number'>59</span>
    <span class='line-number'>60</span>
    <span class='line-number'>61</span>
    <span class='line-number'>62</span>
    <span class='line-number'>63</span>
    <span class='line-number'>64</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">registerNavigationHooks</span> <span class="p">{</span>
    </span><span class='line'>    <span class="p">@</span><span class="n">weakify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>    <span class="p">[[(</span><span class="bp">NSObject</span> <span class="o">*</span><span class="p">)</span><span class="nb">self</span><span class="p">.</span><span class="n">services</span>
    </span><span class='line'>        <span class="nl">rac_signalForSelector</span><span class="p">:</span><span class="k">@selector</span><span class="p">(</span><span class="nl">pushViewModel</span><span class="p">:</span><span class="nl">animated</span><span class="p">:)]</span>
    </span><span class='line'>        <span class="nl">subscribeNext</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="n">RACTuple</span> <span class="o">*</span><span class="n">tuple</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>            <span class="bp">UIViewController</span> <span class="o">*</span><span class="n">viewController</span> <span class="o">=</span> <span class="p">(</span><span class="bp">UIViewController</span> <span class="o">*</span><span class="p">)[</span><span class="n">MRCRouter</span><span class="p">.</span><span class="n">sharedInstance</span> <span class="nl">viewControllerForViewModel</span><span class="p">:</span><span class="n">tuple</span><span class="p">.</span><span class="n">first</span><span class="p">];</span>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span><span class="p">.</span><span class="n">navigationControllers</span><span class="p">.</span><span class="n">lastObject</span> <span class="nl">pushViewController</span><span class="p">:</span><span class="n">viewController</span> <span class="nl">animated</span><span class="p">:[</span><span class="n">tuple</span><span class="p">.</span><span class="n">second</span> <span class="n">boolValue</span><span class="p">]];</span>
    </span><span class='line'>        <span class="p">}];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">[[(</span><span class="bp">NSObject</span> <span class="o">*</span><span class="p">)</span><span class="nb">self</span><span class="p">.</span><span class="n">services</span>
    </span><span class='line'>        <span class="nl">rac_signalForSelector</span><span class="p">:</span><span class="k">@selector</span><span class="p">(</span><span class="nl">popViewModelAnimated</span><span class="p">:)]</span>
    </span><span class='line'>        <span class="nl">subscribeNext</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="n">RACTuple</span> <span class="o">*</span><span class="n">tuple</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>          <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span><span class="p">.</span><span class="n">navigationControllers</span><span class="p">.</span><span class="n">lastObject</span> <span class="nl">popViewControllerAnimated</span><span class="p">:[</span><span class="n">tuple</span><span class="p">.</span><span class="n">first</span> <span class="n">boolValue</span><span class="p">]];</span>
    </span><span class='line'>        <span class="p">}];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">[[(</span><span class="bp">NSObject</span> <span class="o">*</span><span class="p">)</span><span class="nb">self</span><span class="p">.</span><span class="n">services</span>
    </span><span class='line'>        <span class="nl">rac_signalForSelector</span><span class="p">:</span><span class="k">@selector</span><span class="p">(</span><span class="nl">popToRootViewModelAnimated</span><span class="p">:)]</span>
    </span><span class='line'>        <span class="nl">subscribeNext</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="n">RACTuple</span> <span class="o">*</span><span class="n">tuple</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span><span class="p">.</span><span class="n">navigationControllers</span><span class="p">.</span><span class="n">lastObject</span> <span class="nl">popToRootViewControllerAnimated</span><span class="p">:[</span><span class="n">tuple</span><span class="p">.</span><span class="n">first</span> <span class="n">boolValue</span><span class="p">]];</span>
    </span><span class='line'>        <span class="p">}];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">[[(</span><span class="bp">NSObject</span> <span class="o">*</span><span class="p">)</span><span class="nb">self</span><span class="p">.</span><span class="n">services</span>
    </span><span class='line'>        <span class="nl">rac_signalForSelector</span><span class="p">:</span><span class="k">@selector</span><span class="p">(</span><span class="nl">presentViewModel</span><span class="p">:</span><span class="nl">animated</span><span class="p">:</span><span class="nl">completion</span><span class="p">:)]</span>
    </span><span class='line'>        <span class="nl">subscribeNext</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="n">RACTuple</span> <span class="o">*</span><span class="n">tuple</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>          <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>            <span class="bp">UIViewController</span> <span class="o">*</span><span class="n">viewController</span> <span class="o">=</span> <span class="p">(</span><span class="bp">UIViewController</span> <span class="o">*</span><span class="p">)[</span><span class="n">MRCRouter</span><span class="p">.</span><span class="n">sharedInstance</span> <span class="nl">viewControllerForViewModel</span><span class="p">:</span><span class="n">tuple</span><span class="p">.</span><span class="n">first</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="bp">UINavigationController</span> <span class="o">*</span><span class="n">presentingViewController</span> <span class="o">=</span> <span class="nb">self</span><span class="p">.</span><span class="n">navigationControllers</span><span class="p">.</span><span class="n">lastObject</span><span class="p">;</span>
    </span><span class='line'>            <span class="k">if</span> <span class="p">(</span><span class="o">!</span><span class="p">[</span><span class="n">viewController</span> <span class="nl">isKindOfClass</span><span class="p">:</span><span class="bp">UINavigationController</span><span class="p">.</span><span class="k">class</span><span class="p">])</span> <span class="p">{</span>
    </span><span class='line'>                <span class="n">viewController</span> <span class="o">=</span> <span class="p">[[</span><span class="n">MRCNavigationController</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithRootViewController</span><span class="p">:</span><span class="n">viewController</span><span class="p">];</span>
    </span><span class='line'>            <span class="p">}</span>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span> <span class="nl">pushNavigationController</span><span class="p">:(</span><span class="bp">UINavigationController</span> <span class="o">*</span><span class="p">)</span><span class="n">viewController</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="p">[</span><span class="n">presentingViewController</span> <span class="nl">presentViewController</span><span class="p">:</span><span class="n">viewController</span> <span class="nl">animated</span><span class="p">:[</span><span class="n">tuple</span><span class="p">.</span><span class="n">second</span> <span class="n">boolValue</span><span class="p">]</span> <span class="nl">completion</span><span class="p">:</span><span class="n">tuple</span><span class="p">.</span><span class="n">third</span><span class="p">];</span>
    </span><span class='line'>        <span class="p">}];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">[[(</span><span class="bp">NSObject</span> <span class="o">*</span><span class="p">)</span><span class="nb">self</span><span class="p">.</span><span class="n">services</span>
    </span><span class='line'>        <span class="nl">rac_signalForSelector</span><span class="p">:</span><span class="k">@selector</span><span class="p">(</span><span class="nl">dismissViewModelAnimated</span><span class="p">:</span><span class="nl">completion</span><span class="p">:)]</span>
    </span><span class='line'>        <span class="nl">subscribeNext</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="n">RACTuple</span> <span class="o">*</span><span class="n">tuple</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span> <span class="n">popNavigationController</span><span class="p">];</span>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span><span class="p">.</span><span class="n">navigationControllers</span><span class="p">.</span><span class="n">lastObject</span> <span class="nl">dismissViewControllerAnimated</span><span class="p">:[</span><span class="n">tuple</span><span class="p">.</span><span class="n">first</span> <span class="n">boolValue</span><span class="p">]</span> <span class="nl">completion</span><span class="p">:</span><span class="n">tuple</span><span class="p">.</span><span class="n">second</span><span class="p">];</span>
    </span><span class='line'>        <span class="p">}];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">[[(</span><span class="bp">NSObject</span> <span class="o">*</span><span class="p">)</span><span class="nb">self</span><span class="p">.</span><span class="n">services</span>
    </span><span class='line'>        <span class="nl">rac_signalForSelector</span><span class="p">:</span><span class="k">@selector</span><span class="p">(</span><span class="nl">resetRootViewModel</span><span class="p">:)]</span>
    </span><span class='line'>        <span class="nl">subscribeNext</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="n">RACTuple</span> <span class="o">*</span><span class="n">tuple</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span><span class="p">.</span><span class="n">navigationControllers</span> <span class="n">removeAllObjects</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="bp">UIViewController</span> <span class="o">*</span><span class="n">viewController</span> <span class="o">=</span> <span class="p">(</span><span class="bp">UIViewController</span> <span class="o">*</span><span class="p">)[</span><span class="n">MRCRouter</span><span class="p">.</span><span class="n">sharedInstance</span> <span class="nl">viewControllerForViewModel</span><span class="p">:</span><span class="n">tuple</span><span class="p">.</span><span class="n">first</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="k">if</span> <span class="p">(</span><span class="o">!</span><span class="p">[</span><span class="n">viewController</span> <span class="nl">isKindOfClass</span><span class="p">:[</span><span class="bp">UINavigationController</span> <span class="k">class</span><span class="p">]])</span> <span class="p">{</span>
    </span><span class='line'>                <span class="n">viewController</span> <span class="o">=</span> <span class="p">[[</span><span class="n">MRCNavigationController</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithRootViewController</span><span class="p">:</span><span class="n">viewController</span><span class="p">];</span>
    </span><span class='line'>                <span class="p">((</span><span class="bp">UINavigationController</span> <span class="o">*</span><span class="p">)</span><span class="n">viewController</span><span class="p">).</span><span class="n">delegate</span> <span class="o">=</span> <span class="nb">self</span><span class="p">;</span>
    </span><span class='line'>                <span class="p">[</span><span class="nb">self</span> <span class="nl">pushNavigationController</span><span class="p">:(</span><span class="bp">UINavigationController</span> <span class="o">*</span><span class="p">)</span><span class="n">viewController</span><span class="p">];</span>
    </span><span class='line'>            <span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="n">MRCSharedAppDelegate</span><span class="p">.</span><span class="n">window</span><span class="p">.</span><span class="n">rootViewController</span> <span class="o">=</span> <span class="n">viewController</span><span class="p">;</span>
    </span><span class='line'>        <span class="p">}];</span>
    </span><span class='line'><span class="p">}</span>
    </span>`</pre></td></tr></table></div></figure>

    通过 `Hook` 的方式，我们最终实现了 `ViewModel-Based` 的导航操作，并且在 `viewModel` 层中也没有引入 `view` 层的任意东西，实现了解耦合。

    #### Router

    还有一点值得一提的是，我们在 `viewModel` 中调用导航操作的时候，只传入了 `viewModel` 的实例作为参数，那么我们在 `MRCNavigationControllerStack` 中执行真正的导航操作时，怎么才能知道要跳转到哪个界面呢？为此，我们配置了一个从 `viewModel` 到 `view` 的映射，并且约定了一个统一的初始化 `view` 的方法 `initWithViewModel:` ：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    <span class='line-number'>12</span>
    <span class='line-number'>13</span>
    <span class='line-number'>14</span>
    <span class='line-number'>15</span>
    <span class='line-number'>16</span>
    <span class='line-number'>17</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="p">-</span> <span class="p">(</span><span class="n">MRCViewController</span> <span class="o">*</span><span class="p">)</span><span class="nf">viewControllerForViewModel:</span><span class="p">(</span><span class="n">MRCViewModel</span> <span class="o">*</span><span class="p">)</span><span class="nv">viewModel</span> <span class="p">{</span>
    </span><span class='line'>    <span class="bp">NSString</span> <span class="o">*</span><span class="n">viewController</span> <span class="o">=</span> <span class="nb">self</span><span class="p">.</span><span class="n">viewModelViewMappings</span><span class="p">[</span><span class="n">NSStringFromClass</span><span class="p">(</span><span class="n">viewModel</span><span class="p">.</span><span class="k">class</span><span class="p">)];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="n">NSParameterAssert</span><span class="p">([</span><span class="n">NSClassFromString</span><span class="p">(</span><span class="n">viewController</span><span class="p">)</span> <span class="nl">isSubclassOfClass</span><span class="p">:[</span><span class="n">MRCViewController</span> <span class="k">class</span><span class="p">]]);</span>
    </span><span class='line'>    <span class="n">NSParameterAssert</span><span class="p">([</span><span class="n">NSClassFromString</span><span class="p">(</span><span class="n">viewController</span><span class="p">)</span> <span class="nl">instancesRespondToSelector</span><span class="p">:</span><span class="k">@selector</span><span class="p">(</span><span class="nl">initWithViewModel</span><span class="p">:)]);</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="k">return</span> <span class="p">[[</span><span class="n">NSClassFromString</span><span class="p">(</span><span class="n">viewController</span><span class="p">)</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithViewModel</span><span class="p">:</span><span class="n">viewModel</span><span class="p">];</span>
    </span><span class='line'><span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="bp">NSDictionary</span> <span class="o">*</span><span class="p">)</span><span class="nf">viewModelViewMappings</span> <span class="p">{</span>
    </span><span class='line'>    <span class="k">return</span> <span class="l">@{</span>
    </span><span class='line'>      <span class="s">@&quot;MRCLoginViewModel&quot;</span><span class="o">:</span> <span class="s">@&quot;MRCLoginViewController&quot;</span><span class="p">,</span>
    </span><span class='line'>        <span class="s">@&quot;MRCHomepageViewModel&quot;</span><span class="o">:</span> <span class="s">@&quot;MRCHomepageViewController&quot;</span><span class="p">,</span>
    </span><span class='line'>        <span class="s">@&quot;MRCRepoDetailViewModel&quot;</span><span class="o">:</span> <span class="s">@&quot;MRCRepoDetailViewController&quot;</span><span class="p">,</span>
    </span><span class='line'>        <span class="p">...</span>
    </span><span class='line'>    <span class="l">}</span><span class="p">;</span>
    </span><span class='line'><span class="p">}</span>
    </span>`</pre></td></tr></table></div></figure>

    ### 登录界面

    最后，我们一起来看看登录界面中 `viewModel` 和 `view` 的部分关键代码，探讨一下 `MVVM` 的具体实践过程。**说明**，我们将会尽可能地回避具体的业务逻辑，重点关注 `MVVM` 的实践思路。下面是登录界面的截图：

    ![](http://blog.leichunfeng.com/images/login.jpg)

    其中，主要的界面元素有：

*   一个用于展示用户头像的按钮 `avatarButton` ；
*   用于输入账号和密码的输入框 `usernameTextField` 和 `passwordTextField` ；
*   一个直接登录的按钮 `loginButton` 和一个跳转到浏览器授权登录的按钮 `browserLoginButton` 。

    **分析**：根据我们前面对 `MVVM` 的探讨，`viewModel` 需要提供 `view` 所需的数据和命令。因此，`MRCLoginViewModel.h` 头文件的内容大致如下：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">@interface</span> <span class="nc">MRCLoginViewModel</span> : <span class="nc">MRCViewModel</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">copy</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="bp">NSURL</span> <span class="o">*</span><span class="n">avatarURL</span><span class="p">;</span>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">copy</span><span class="p">)</span> <span class="bp">NSString</span> <span class="o">*</span><span class="n">username</span><span class="p">;</span>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">copy</span><span class="p">)</span> <span class="bp">NSString</span> <span class="o">*</span><span class="n">password</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">strong</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="n">RACSignal</span> <span class="o">*</span><span class="n">validLoginSignal</span><span class="p">;</span>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">strong</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="n">RACCommand</span> <span class="o">*</span><span class="n">loginCommand</span><span class="p">;</span>
    </span><span class='line'><span class="k">@property</span> <span class="p">(</span><span class="k">nonatomic</span><span class="p">,</span> <span class="k">strong</span><span class="p">,</span> <span class="k">readonly</span><span class="p">)</span> <span class="n">RACCommand</span> <span class="o">*</span><span class="n">browserLoginCommand</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@end</span>
    </span>`</pre></td></tr></table></div></figure>

    非常直观，其中需要特别说明的是 `validLoginSignal` 属性代表的是登录按钮是否可用，它将会与 `view` 中登录按钮的 `enabled` 属性进行绑定。接着，我们来看看 `MRCLoginViewModel.m` 的实现文件中的部分关键代码：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    <span class='line-number'>12</span>
    <span class='line-number'>13</span>
    <span class='line-number'>14</span>
    <span class='line-number'>15</span>
    <span class='line-number'>16</span>
    <span class='line-number'>17</span>
    <span class='line-number'>18</span>
    <span class='line-number'>19</span>
    <span class='line-number'>20</span>
    <span class='line-number'>21</span>
    <span class='line-number'>22</span>
    <span class='line-number'>23</span>
    <span class='line-number'>24</span>
    <span class='line-number'>25</span>
    <span class='line-number'>26</span>
    <span class='line-number'>27</span>
    <span class='line-number'>28</span>
    <span class='line-number'>29</span>
    <span class='line-number'>30</span>
    <span class='line-number'>31</span>
    <span class='line-number'>32</span>
    <span class='line-number'>33</span>
    <span class='line-number'>34</span>
    <span class='line-number'>35</span>
    <span class='line-number'>36</span>
    <span class='line-number'>37</span>
    <span class='line-number'>38</span>
    <span class='line-number'>39</span>
    <span class='line-number'>40</span>
    <span class='line-number'>41</span>
    <span class='line-number'>42</span>
    <span class='line-number'>43</span>
    <span class='line-number'>44</span>
    <span class='line-number'>45</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">@implementation</span> <span class="nc">MRCLoginViewModel</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">initialize</span> <span class="p">{</span>
    </span><span class='line'>    <span class="p">[</span><span class="nb">super</span> <span class="n">initialize</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="n">RAC</span><span class="p">(</span><span class="nb">self</span><span class="p">,</span> <span class="n">avatarURL</span><span class="p">)</span> <span class="o">=</span> <span class="p">[[</span><span class="n">RACObserve</span><span class="p">(</span><span class="nb">self</span><span class="p">,</span> <span class="n">username</span><span class="p">)</span>
    </span><span class='line'>        <span class="nl">map</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="bp">NSString</span> <span class="o">*</span><span class="n">username</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="k">return</span> <span class="p">[[</span><span class="n">OCTUser</span> <span class="nl">mrc_fetchUserWithRawLogin</span><span class="p">:</span><span class="n">username</span><span class="p">]</span> <span class="n">avatarURL</span><span class="p">];</span>
    </span><span class='line'>        <span class="p">}]</span>
    </span><span class='line'>        <span class="n">distinctUntilChanged</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="nb">self</span><span class="p">.</span><span class="n">validLoginSignal</span> <span class="o">=</span> <span class="p">[[</span><span class="n">RACSignal</span>
    </span><span class='line'>      <span class="nl">combineLatest</span><span class="p">:</span><span class="l">@[</span> <span class="n">RACObserve</span><span class="p">(</span><span class="nb">self</span><span class="p">,</span> <span class="n">username</span><span class="p">),</span> <span class="n">RACObserve</span><span class="p">(</span><span class="nb">self</span><span class="p">,</span> <span class="n">password</span><span class="p">)</span> <span class="l">]</span>
    </span><span class='line'>        <span class="nl">reduce</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="bp">NSString</span> <span class="o">*</span><span class="n">username</span><span class="p">,</span> <span class="bp">NSString</span> <span class="o">*</span><span class="n">password</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>          <span class="k">return</span> <span class="l">@(</span><span class="n">username</span><span class="p">.</span><span class="n">length</span> <span class="o">&gt;</span> <span class="mi">0</span> <span class="o">&amp;&amp;</span> <span class="n">password</span><span class="p">.</span><span class="n">length</span> <span class="o">&gt;</span> <span class="mi">0</span><span class="l">)</span><span class="p">;</span>
    </span><span class='line'>        <span class="p">}]</span>
    </span><span class='line'>        <span class="n">distinctUntilChanged</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">@</span><span class="n">weakify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>    <span class="kt">void</span> <span class="p">(</span><span class="o">^</span><span class="n">doNext</span><span class="p">)(</span><span class="n">OCTClient</span> <span class="o">*</span><span class="p">)</span> <span class="o">=</span> <span class="o">^</span><span class="p">(</span><span class="n">OCTClient</span> <span class="o">*</span><span class="n">authenticatedClient</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>        <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>        <span class="n">MRCHomepageViewModel</span> <span class="o">*</span><span class="n">viewModel</span> <span class="o">=</span> <span class="p">[[</span><span class="n">MRCHomepageViewModel</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithServices</span><span class="p">:</span><span class="nb">self</span><span class="p">.</span><span class="n">services</span> <span class="nl">params</span><span class="p">:</span><span class="nb">nil</span><span class="p">];</span>
    </span><span class='line'>        <span class="n">dispatch_async</span><span class="p">(</span><span class="n">dispatch_get_main_queue</span><span class="p">(),</span> <span class="o">^</span><span class="p">{</span>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span><span class="p">.</span><span class="n">services</span> <span class="nl">resetRootViewModel</span><span class="p">:</span><span class="n">viewModel</span><span class="p">];</span>
    </span><span class='line'>        <span class="p">});</span>
    </span><span class='line'>    <span class="p">};</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">[</span><span class="n">OCTClient</span> <span class="nl">setClientID</span><span class="p">:</span><span class="n">MRC_CLIENT_ID</span> <span class="nl">clientSecret</span><span class="p">:</span><span class="n">MRC_CLIENT_SECRET</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="nb">self</span><span class="p">.</span><span class="n">loginCommand</span> <span class="o">=</span> <span class="p">[[</span><span class="n">RACCommand</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithSignalBlock</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="bp">NSString</span> <span class="o">*</span><span class="n">oneTimePassword</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>      <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>        <span class="n">OCTUser</span> <span class="o">*</span><span class="n">user</span> <span class="o">=</span> <span class="p">[</span><span class="n">OCTUser</span> <span class="nl">userWithRawLogin</span><span class="p">:</span><span class="nb">self</span><span class="p">.</span><span class="n">username</span> <span class="nl">server</span><span class="p">:</span><span class="n">OCTServer</span><span class="p">.</span><span class="n">dotComServer</span><span class="p">];</span>
    </span><span class='line'>        <span class="k">return</span> <span class="p">[[</span><span class="n">OCTClient</span>
    </span><span class='line'>          <span class="nl">signInAsUser</span><span class="p">:</span><span class="n">user</span> <span class="nl">password</span><span class="p">:</span><span class="nb">self</span><span class="p">.</span><span class="n">password</span> <span class="nl">oneTimePassword</span><span class="p">:</span><span class="n">oneTimePassword</span> <span class="nl">scopes</span><span class="p">:</span><span class="n">OCTClientAuthorizationScopesUser</span> <span class="o">|</span> <span class="n">OCTClientAuthorizationScopesRepository</span> <span class="nl">note</span><span class="p">:</span><span class="nb">nil</span> <span class="nl">noteURL</span><span class="p">:</span><span class="nb">nil</span> <span class="nl">fingerprint</span><span class="p">:</span><span class="nb">nil</span><span class="p">]</span>
    </span><span class='line'>            <span class="nl">doNext</span><span class="p">:</span><span class="n">doNext</span><span class="p">];</span>
    </span><span class='line'>    <span class="p">}];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="nb">self</span><span class="p">.</span><span class="n">browserLoginCommand</span> <span class="o">=</span> <span class="p">[[</span><span class="n">RACCommand</span> <span class="n">alloc</span><span class="p">]</span> <span class="nl">initWithSignalBlock</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="kt">id</span> <span class="n">input</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>        <span class="k">return</span> <span class="p">[[</span><span class="n">OCTClient</span>
    </span><span class='line'>          <span class="nl">signInToServerUsingWebBrowser</span><span class="p">:</span><span class="n">OCTServer</span><span class="p">.</span><span class="n">dotComServer</span> <span class="nl">scopes</span><span class="p">:</span><span class="n">OCTClientAuthorizationScopesUser</span> <span class="o">|</span> <span class="n">OCTClientAuthorizationScopesRepository</span><span class="p">]</span>
    </span><span class='line'>            <span class="nl">doNext</span><span class="p">:</span><span class="n">doNext</span><span class="p">];</span>
    </span><span class='line'>    <span class="p">}];</span>
    </span><span class='line'><span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@end</span>
    </span>`</pre></td></tr></table></div></figure>

*   当用户输入的用户名发生变化时，调用 `model` 层的方法查询本地数据库中缓存的用户数据，并返回 `avatarURL` 属性;
*   当用户输入的用户名或密码发生变化时，判断用户名和密码的长度是否均大于 `0` ，如果是则登录按钮可用，否则不可用;
*   当 `loginCommand` 或 `browserLoginCommand` 命令执行成功时，调用 `doNext` 代码块，使用服务总线中的方法 `resetRootViewModel:` 进入首页。

    接下来，我们来看看 `MRCLoginViewController` 中的部分关键代码：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    <span class='line-number'>8</span>
    <span class='line-number'>9</span>
    <span class='line-number'>10</span>
    <span class='line-number'>11</span>
    <span class='line-number'>12</span>
    <span class='line-number'>13</span>
    <span class='line-number'>14</span>
    <span class='line-number'>15</span>
    <span class='line-number'>16</span>
    <span class='line-number'>17</span>
    <span class='line-number'>18</span>
    <span class='line-number'>19</span>
    <span class='line-number'>20</span>
    <span class='line-number'>21</span>
    <span class='line-number'>22</span>
    <span class='line-number'>23</span>
    <span class='line-number'>24</span>
    <span class='line-number'>25</span>
    <span class='line-number'>26</span>
    <span class='line-number'>27</span>
    <span class='line-number'>28</span>
    <span class='line-number'>29</span>
    <span class='line-number'>30</span>
    <span class='line-number'>31</span>
    <span class='line-number'>32</span>
    <span class='line-number'>33</span>
    <span class='line-number'>34</span>
    <span class='line-number'>35</span>
    <span class='line-number'>36</span>
    <span class='line-number'>37</span>
    <span class='line-number'>38</span>
    <span class='line-number'>39</span>
    <span class='line-number'>40</span>
    <span class='line-number'>41</span>
    <span class='line-number'>42</span>
    <span class='line-number'>43</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">@implementation</span> <span class="nc">MRCLoginViewController</span>
    </span><span class='line'>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">void</span><span class="p">)</span><span class="nf">bindViewModel</span> <span class="p">{</span>
    </span><span class='line'>    <span class="p">[</span><span class="nb">super</span> <span class="n">bindViewModel</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">@</span><span class="n">weakify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>    <span class="p">[</span><span class="n">RACObserve</span><span class="p">(</span><span class="nb">self</span><span class="p">.</span><span class="n">viewModel</span><span class="p">,</span> <span class="n">avatarURL</span><span class="p">)</span> <span class="nl">subscribeNext</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="bp">NSURL</span> <span class="o">*</span><span class="n">avatarURL</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>      <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>        <span class="p">[</span><span class="nb">self</span><span class="p">.</span><span class="n">avatarButton</span> <span class="nl">sd_setImageWithURL</span><span class="p">:</span><span class="n">avatarURL</span> <span class="nl">forState</span><span class="p">:</span><span class="n">UIControlStateNormal</span> <span class="nl">placeholderImage</span><span class="p">:[</span><span class="bp">UIImage</span> <span class="nl">imageNamed</span><span class="p">:</span><span class="s">@&quot;default-avatar&quot;</span><span class="p">]];</span>
    </span><span class='line'>    <span class="p">}];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="n">RAC</span><span class="p">(</span><span class="nb">self</span><span class="p">.</span><span class="n">viewModel</span><span class="p">,</span> <span class="n">username</span><span class="p">)</span>  <span class="o">=</span> <span class="nb">self</span><span class="p">.</span><span class="n">usernameTextField</span><span class="p">.</span><span class="n">rac_textSignal</span><span class="p">;</span>
    </span><span class='line'>    <span class="n">RAC</span><span class="p">(</span><span class="nb">self</span><span class="p">.</span><span class="n">viewModel</span><span class="p">,</span> <span class="n">password</span><span class="p">)</span>  <span class="o">=</span> <span class="nb">self</span><span class="p">.</span><span class="n">passwordTextField</span><span class="p">.</span><span class="n">rac_textSignal</span><span class="p">;</span>
    </span><span class='line'>    <span class="n">RAC</span><span class="p">(</span><span class="nb">self</span><span class="p">.</span><span class="n">loginButton</span><span class="p">,</span> <span class="n">enabled</span><span class="p">)</span> <span class="o">=</span> <span class="nb">self</span><span class="p">.</span><span class="n">viewModel</span><span class="p">.</span><span class="n">validLoginSignal</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">[[</span><span class="nb">self</span><span class="p">.</span><span class="n">loginButton</span>
    </span><span class='line'>        <span class="nl">rac_signalForControlEvents</span><span class="p">:</span><span class="n">UIControlEventTouchUpInside</span><span class="p">]</span>
    </span><span class='line'>        <span class="nl">subscribeNext</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="kt">id</span> <span class="n">x</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span><span class="p">.</span><span class="n">viewModel</span><span class="p">.</span><span class="n">loginCommand</span> <span class="nl">execute</span><span class="p">:</span><span class="nb">nil</span><span class="p">];</span>
    </span><span class='line'>        <span class="p">}];</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">[[</span><span class="nb">self</span><span class="p">.</span><span class="n">browserLoginButton</span>
    </span><span class='line'>        <span class="nl">rac_signalForControlEvents</span><span class="p">:</span><span class="n">UIControlEventTouchUpInside</span><span class="p">]</span>
    </span><span class='line'>        <span class="nl">subscribeNext</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="kt">id</span> <span class="n">x</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>            <span class="bp">NSString</span> <span class="o">*</span><span class="n">message</span> <span class="o">=</span> <span class="p">[</span><span class="bp">NSString</span> <span class="nl">stringWithFormat</span><span class="p">:</span><span class="s">@&quot;“%@” wants to open “Safari”&quot;</span><span class="p">,</span> <span class="n">MRC_APP_NAME</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="n">UIAlertController</span> <span class="o">*</span><span class="n">alertController</span> <span class="o">=</span> <span class="p">[</span><span class="n">UIAlertController</span> <span class="nl">alertControllerWithTitle</span><span class="p">:</span><span class="nb">nil</span>
    </span><span class='line'>                                                                                     <span class="nl">message</span><span class="p">:</span><span class="n">message</span>
    </span><span class='line'>                                                                              <span class="nl">preferredStyle</span><span class="p">:</span><span class="n">UIAlertControllerStyleAlert</span><span class="p">];</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="p">[</span><span class="n">alertController</span> <span class="nl">addAction</span><span class="p">:[</span><span class="n">UIAlertAction</span> <span class="nl">actionWithTitle</span><span class="p">:</span><span class="s">@&quot;Cancel&quot;</span> <span class="nl">style</span><span class="p">:</span><span class="n">UIAlertActionStyleCancel</span> <span class="nl">handler</span><span class="p">:</span><span class="nb">NULL</span><span class="p">]];</span>
    </span><span class='line'>            <span class="p">[</span><span class="n">alertController</span> <span class="nl">addAction</span><span class="p">:[</span><span class="n">UIAlertAction</span> <span class="nl">actionWithTitle</span><span class="p">:</span><span class="s">@&quot;Open&quot;</span> <span class="nl">style</span><span class="p">:</span><span class="n">UIAlertActionStyleDefault</span> <span class="nl">handler</span><span class="p">:</span><span class="o">^</span><span class="p">(</span><span class="n">UIAlertAction</span> <span class="o">*</span><span class="n">action</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>                <span class="p">@</span><span class="n">strongify</span><span class="p">(</span><span class="nb">self</span><span class="p">)</span>
    </span><span class='line'>                <span class="p">[</span><span class="nb">self</span><span class="p">.</span><span class="n">viewModel</span><span class="p">.</span><span class="n">browserLoginCommand</span> <span class="nl">execute</span><span class="p">:</span><span class="nb">nil</span><span class="p">];</span>
    </span><span class='line'>            <span class="p">}]];</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="p">[</span><span class="nb">self</span> <span class="nl">presentViewController</span><span class="p">:</span><span class="n">alertController</span> <span class="nl">animated</span><span class="p">:</span><span class="nb">YES</span> <span class="nl">completion</span><span class="p">:</span><span class="nb">NULL</span><span class="p">];</span>
    </span><span class='line'>        <span class="p">}];</span>
    </span><span class='line'><span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@end</span>
    </span>
</td></tr></table></div></figure>

*   观察 `viewModel` 中 `avatarURL` 属性的变化，然后设置 `avatarButton` 中的图片；
*   将 `viewModel` 中的 `username` 和 `password` 属性分别与 `usernameTextField` 和 `passwordTextField` 输入框中的内容进行绑定；
*   将 `loginButton` 的 `enabled` 属性与 `viewModel` 的 `validLoginSignal` 属性进行绑定；
*   在 `loginButton` 和 `browserLoginButton` 按钮被点击时分别执行 `loginCommand` 和 `browserLoginCommand` 命令。

综上所述，我们将 `MRCLoginViewController` 中的展示逻辑抽取到 `MRCLoginViewModel` 中后，使得 `MRCLoginViewController` 中的代码更加简洁和清晰。实践 `MVVM` 的关键点在于，我们要能够分析清楚 `viewModel` 需要暴露给 `view` 的数据和命令，这些数据和命令能够代表 `view` 当前的状态。

## 总结

首先，我们从理论出发介绍了 `MVC` 和 `MVVM` 各自的概念以及从 `MVC` 到 `MVVM` 的演进过程；接着，介绍了 `RAC` 在 `MVVM` 中的两个使用场景；最后，我们从实践的角度，重点介绍了一个使用 `MVVM` 和 `RAC` 开发的开源项目 `MVVMReactiveCocoa` 。总的来说，我认为 `iOS` 中的 `MVVM` 可以分为以下三种不同的实践程度，它们分别对应不同的适用场景：

*   `MVVM + KVO` ，适用于现有的 `MVC` 项目，想转换成 `MVVM` 但是不打算引入 `RAC` 作为 `binder` 的团队；
*   `MVVM + RAC` ，适用于现有的 `MVC` 项目，想转换成 `MVVM` 并且打算引入 `RAC` 作为 `binder` 的团队；
*   `MVVM + RAC + ViewModel-Based Navigation` ，适用于全新的项目，想实践 `MVVM` 并且打算引入 `RAC` 作为 `binder` ，然后也想实践 `ViewModel-Based Navigation` 的团队。

写在最后，希望这篇文章能够打消你对 `MVVM` 模式的顾虑，赶快行动起来吧。

## 参考链接

[https://www.objc.io/issues/13-architecture/mvvm/](https://www.objc.io/issues/13-architecture/mvvm/)

[https://msdn.microsoft.com/en-us/library/hh848246.aspx](https://msdn.microsoft.com/en-us/library/hh848246.aspx)

[https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)

[https://medium.com/ios-os-x-development/ios-architecture-patterns-ecba4c38de52#.p6n56kyc4](https://medium.com/ios-os-x-development/ios-architecture-patterns-ecba4c38de52#.p6n56kyc4)

[http://cocoasamurai.blogspot.ru/2013/03/basic-mvvm-with-reactivecocoa.html](http://cocoasamurai.blogspot.ru/2013/03/basic-mvvm-with-reactivecocoa.html)

[http://www.sprynthesis.com/2014/12/06/reactivecocoa-mvvm-introduction/](http://www.sprynthesis.com/2014/12/06/reactivecocoa-mvvm-introduction/)

![](http://blog.leichunfeng.com/images/wechat_pay.jpg)