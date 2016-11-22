---
title: WatchKit 之 导航
tags: []
date: 2014-11-20 09:52:13
---

在 Watch App 中，当涉及到多个界面跳转时，我们共有两种导航方式可使用：Hierarchical 与 Page-based。本文将对这两种导航方式以及 Modal 展现方式做简单的介绍。

# Page-based

Page-based 是类似现在很多 iOS 第一次启动的介绍页面的 paged scrollview 那样，可以左右滑动翻页。（题外话，我很反感第一次启动的介绍页的设计）
创建 Page-based 界面必须使用 storyboard。创建方式很简单，只需在 interfaceController 之间创建多个 next page 类型的 segue 就可以了。Page 的数量和顺序是你在 storyboard 中设计阶段决定的，不可以在运行时修改。

![image](/WatchKitNavigation/001.png) 

运行效果:
![image](/WatchKitNavigation/002.gif) 

# Hierarchical

Hierarchical 方式类似一般的 NavigationController，你可以 Push 和 Pop 界面。可以通过 push segue 实现，也可以在代码调用 pushControllerWithName:context: 方法，这个方法有两个参数，第一个 name 参数需要的是 push 的目标 interfaceController 的 identifier，可以在 storyboard 中设定。第二个 context 参数是传任何你想传递给第二个 interfaceController 的东西。这个 context 可以在第二个界面的 init 方式中取出来。所以如果你有任何想传递给第二个界面的东西，就放在 context 这里。

<div class="codehilite"><pre><span class="nb">self</span><span class="p">.</span><span class="n">pushControllerWithName</span><span class="p">(</span><span class="s">&quot;ic2&quot;</span><span class="p">,</span> <span class="nl">context</span><span class="p">:</span> <span class="nb">nil</span><span class="p">)</span>
</pre></div>

而 Pop 也很简单：

<div class="codehilite"><pre><span class="nb">self</span><span class="p">.</span><span class="n">popController</span><span class="p">()</span>
</pre></div>

运行效果：
![image](/WatchKitNavigation/003.gif) 

# Modal

Watch App 的 modal 方式与 iPhone 不同的是，他会自动在左上角添加 dismiss 按钮，默认文案是 &quot;Cancel&quot;，当然你也可以修改。另外你可以一次性 modal 多个界面上去，就会以 page 的方式展现。

首先演示单个界面 modal，可以通过 modal segue，也可以通过调用方法 presentControllerWithName:context:。两个参数与上面 Push 效果相同。

<div class="codehilite"><pre><span class="nb">self</span><span class="p">.</span><span class="n">presentControllerWithName</span><span class="p">(</span><span class="s">&quot;ic2&quot;</span><span class="p">,</span> <span class="nl">context</span><span class="p">:</span> <span class="nb">nil</span><span class="p">)</span>
</pre></div>

运行效果
![image](/WatchKitNavigation/004.gif) 

多个界面 Modal 调用 presentControllerWithNames:contexts: 方法。传入界面的 identifier 数组，context 数组与之对应。执行后。会以 page 的形式展现。

<div class="codehilite"><pre><span class="nb">self</span><span class="p">.</span><span class="n">presentControllerWithNames</span><span class="p">([</span><span class="s">&quot;ic2&quot;</span><span class="p">,</span><span class="s">&quot;ic3&quot;</span><span class="p">],</span> <span class="nl">contexts</span><span class="p">:</span> <span class="nb">nil</span><span class="p">)</span>
</pre></div>

运行效果
![image](/WatchKitNavigation/005.gif) 

如果想用 storyboard，则是 modal 到第二个界面，然后第二个界面以 next page 的方式创建到其他界面的 segue，就像 Page-based 结构一样：
![image](/WatchKitNavigation/006.png)