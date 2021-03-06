---
title: Functor、Applicative 和 Monad
tags: []
date: 2015-11-08 10:53:16
---

`Functor`、`Applicative` 和 `Monad` 是函数式编程语言中三个非常重要的概念，尤其是 `Monad` ，难倒了不知道多少英雄好汉。事实上，它们的概念是非常简单的，但是却很少有文章能够将它们描述清楚，往往还适得其反，越描越黑。与其它文章不同的是，本文将从结论出发，层层深入，一步步为你揭开它们的神秘面纱。

**说明**：本文中的主要代码为 [Haskell](https://www.haskell.org) 语言，它是一门纯函数式的编程语言。其中，具体的语法细节，我们不需要太过关心，因为这并不影响你对本文的理解。

## 结论

关于 `Functor`、`Applicative` 和 `Monad` 的概念，其实各用一句话就可以概括：

1.  一个 `Functor` 就是一种实现了 `Functor typeclass` 的数据类型；
2.  一个 `Applicative` 就是一种实现了 `Applicative typeclass` 的数据类型；
3.  一个 `Monad` 就是一种实现了 `Monad typeclass` 的数据类型。

当然，你可能会问那什么是 [typeclass](http://learnyouahaskell.com/types-and-typeclasses#typeclasses-101) 呢？我想当你在看到**实现**二字的时候，就应该已经猜到了：

> A typeclass is a sort of interface that defines some behavior. If a type is a part of a typeclass, that means that it supports and implements the behavior the typeclass describes. A lot of people coming from OOP get confused by typeclasses because they think they are like classes in object oriented languages. Well, they&rsquo;re not. You can think of them kind of as Java interfaces, only better.

是的，`typeclass` 就类似于 `Java` 中的接口，或者 `Objective-C` 中的协议。在 `typeclass` 中定义了一些函数，实现一个 `typeclass` 就是要实现这些函数，而所有实现了这个 `typeclass` 的数据类型都会拥有这些共同的行为。

那 `Functor`、`Applicative` 和 `Monad` 三者之间有什么联系吗，为什么它们老是结队出现呢？其实，`Applicative` 是增强型的 `Functor` ，一种数据类型要成为 `Applicative` 的前提条件是它必须是 `Functor` ；同样的，`Monad` 是增强型的 `Applicative` ，一种数据类型要成为 `Monad` 的前提条件是它必须是 `Applicative` 。**注**：这个联系，在我们看到 `Applicative typeclass` 和 `Monad typeclass` 的定义时，自然就会明了。

## Maybe

在正式开始介绍 `Functor`、`Applicative` 和 `Monad` 的定义前，我想先介绍一种非常有意思的数据类型，[Maybe](https://downloads.haskell.org/~ghc/latest/docs/html/libraries/base-4.8.1.0/Data-Maybe.html) 类型（可类比 `Swift` 中的 `Optional`）：

> The Maybe type encapsulates an optional value. A value of type Maybe a either contains a value of type a (represented as Just a), or it is empty (represented as Nothing). Using Maybe is a good way to deal with errors or exceptional cases without resorting to drastic measures such as error.

`Maybe` 类型封装了一个可选值。一个 `Maybe a` 类型的值要么包含一个 `a` 类型的值（用 `Just a` 表示）；要么为空（用 `Nothing` 表示）。我们可以把 `Maybe` 看作一个盒子，这个盒子里面可能装着一个 `a` 类型的值，即 `Just a` ；也可能是一个空盒子，即 `Nothing` 。或者，你也可以把它理解成泛型，比如 `Objective-C` 中的 `NSArray&lt;ObjectType&gt;` 。不过，最正确的理解应该是把 `Maybe` 看作一个上下文，这个上下文表示某次计算可能成功也可能失败，成功时用 `Just a` 表示，`a` 为计算结果；失败时用 `Nothing` 表示，这就是 `Maybe` 类型存在的意义：

![maybe](http://blog.leichunfeng.com/images/maybe.png)

<figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
</pre></td><td class='code'>

    <span class='line'><span class="n">data</span> <span class="n">Maybe</span> <span class="n">a</span> <span class="o">=</span> <span class="n">Nothing</span> <span class="o">|</span> <span class="n">Just</span> <span class="n">a</span>
    </span>`</pre></td></tr></table></div></figure>

    下面，我们来直观地感受一下 `Maybe` 类型：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">Nothing</span>
    </span><span class='line'><span class="n">Nothing</span>
    </span><span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">Just</span> <span class="mi">2</span>
    </span><span class='line'><span class="n">Just</span> <span class="mi">2</span>
    </span>`</pre></td></tr></table></div></figure>

    我们可以用盒子模型来理解一下，`Nothing` 就是一个空盒子；而 `Just 2` 则是一个装着 `2` 这个值的盒子：

    ![just2](http://blog.leichunfeng.com/images/just2.png)

    **提前剧透**：`Maybe` 类型实现了 `Functor typeclass`、`Applicative typeclass` 和 `Monad typeclass` ，所以它同时是 `Functor`、`Applicative` 和 `Monad` ，具体实现细节将在下面的章节进行介绍。

    ## Functor

    在正式开始介绍 `Functor` 前，我们先思考一个这样的问题，假如我们有一个值 `2` ：

    ![normal2](http://blog.leichunfeng.com/images/normal2.png)

    我们如何将函数 `(+3)` 应用到这个值上呢？我想上过小学的朋友应该都知道，这就是一个简单的加法运算：

    ![value_apply](http://blog.leichunfeng.com/images/value_apply.png)

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="p">(</span><span class="o">+</span><span class="mi">3</span><span class="p">)</span> <span class="mi">2</span>
    </span><span class='line'><span class="mi">5</span>
    </span>`</pre></td></tr></table></div></figure>

    分分钟搞定。那么问题来了，如果这个值 `2` 是在一个上下文中呢？比如 `Maybe` ，此时，这个值 `2` 就变成了 `Just 2` ：

    ![just2](http://blog.leichunfeng.com/images/just2.png)

    这个时候，我们就不能直接将函数 `(+3)` 应用到 `Just 2` 了。那么，我们如何将一个函数应用到一个在上下文中的值呢？

    ![fmap](http://blog.leichunfeng.com/images/fmap.png)

    是的，我想你应该已经猜到了，`Functor` 就是干这事的，欲知后事如何，请看下节分解。

    ### Functor typeclass

    首先，我们来看一下 `Functor typeclass` 的定义：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">class</span> <span class="n">Functor</span> <span class="n">f</span> <span class="n">where</span>
    </span><span class='line'>    <span class="n">fmap</span> <span class="o">::</span> <span class="p">(</span><span class="n">a</span> <span class="o">-&gt;</span> <span class="n">b</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">f</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">f</span> <span class="n">b</span>
    </span>`</pre></td></tr></table></div></figure>

    在 `Functor typeclass` 中定义了一个函数 `fmap` ，它将一个函数 `(a -&gt; b)` 应用到一个在上下文中的值 `f a` ，并返回另一个在相同上下文中的值 `f b` ，这里的 `f` 是一个类型占位符，表示任意类型的 `Functor` 。

    **注**：`fmap` 函数可类比 `Swift` 中的 `map` 方法。

    ### Maybe Functor

    我们知道 `Maybe` 类型就是一个 `Functor` ，它实现了 `Functor typeclass` 。我们将类型占位符 `f` 用具体类型 `Maybe` 代入可得：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">class</span> <span class="n">Functor</span> <span class="n">Maybe</span> <span class="n">where</span>
    </span><span class='line'>    <span class="n">fmap</span> <span class="o">::</span> <span class="p">(</span><span class="n">a</span> <span class="o">-&gt;</span> <span class="n">b</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">Maybe</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">Maybe</span> <span class="n">b</span>
    </span>`</pre></td></tr></table></div></figure>

    因此，对于 `Maybe` 类型来说，它要实现的函数 `fmap` 的功能就是将一个函数 `(a -&gt; b)` 应用到一个在 `Maybe` 上下文中的值 `Maybe a` ，并返回另一个在 `Maybe` 上下文中的值 `Maybe b` 。接下来，我们一起来看一下 `Maybe` 类型实现 `Functor typeclass` 的具体细节：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">instance</span> <span class="n">Functor</span> <span class="n">Maybe</span> <span class="n">where</span>
    </span><span class='line'>    <span class="n">fmap</span> <span class="n">func</span> <span class="p">(</span><span class="n">Just</span> <span class="n">x</span><span class="p">)</span> <span class="o">=</span> <span class="n">Just</span> <span class="p">(</span><span class="n">func</span> <span class="n">x</span><span class="p">)</span>
    </span><span class='line'>    <span class="n">fmap</span> <span class="n">func</span> <span class="n">Nothing</span>  <span class="o">=</span> <span class="n">Nothing</span>
    </span>`</pre></td></tr></table></div></figure>

    这里针对 `Maybe` 上下文的两种情况分别进行了处理：如果盒子中有值，即 `Just x` ，那么就将 `x` 从盒子中取出，然后将函数 `func` 应用到 `x` ，最后将结果放入一个相同类型的新盒子中；如果盒子为空，那么直接返回一个新的空盒子。

    看到这里，我想你应该已经知道如何将一个函数应用到一个在上下文中的值了。比如前面提到的将函数 `(+3)` 应用到 `Just 2` ：

    ![fmap_just](http://blog.leichunfeng.com/images/fmap_just.png)

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">fmap</span> <span class="p">(</span><span class="o">+</span><span class="mi">3</span><span class="p">)</span> <span class="p">(</span><span class="n">Just</span> <span class="mi">2</span><span class="p">)</span>
    </span><span class='line'><span class="n">Just</span> <span class="mi">5</span>
    </span>`</pre></td></tr></table></div></figure>

    另外，值得一提的是，当我们将函数 `(+3)` 应用到一个空盒子，即 `Nothing` 时，我们将会得到一个新的空盒子：

    ![fmap_nothing](http://blog.leichunfeng.com/images/fmap_nothing.png)

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">fmap</span> <span class="p">(</span><span class="o">+</span><span class="mi">3</span><span class="p">)</span> <span class="n">Nothing</span>
    </span><span class='line'><span class="n">Nothing</span>
    </span>`</pre></td></tr></table></div></figure>

    ## Applicative

    现在，我们已经知道如何将函数 `(+3)` 应用到 `Just 2` 了。那么问题又来了，如果函数 `(+3)` 也在上下文中呢，比如 `Maybe` ，此时，函数 `(+3)` 就变成了 `Just (+3)` ：

    ![justadd3](http://blog.leichunfeng.com/images/justadd3.png)

    那么，我们如何将一个在上下文中的函数应用到一个在上下文中的值呢？

    ![apply_justadd3_just2](http://blog.leichunfeng.com/images/apply_justadd3_just2.png)

    这就是 `Applicative` 要干的事，详情请看下节内容。

    ### Applicative typeclass

    同样的，我们先来看一下 `Applicative typeclass` 的定义：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">class</span> <span class="n">Functor</span> <span class="n">f</span> <span class="o">=&gt;</span> <span class="n">Applicative</span> <span class="n">f</span> <span class="n">where</span>
    </span><span class='line'>    <span class="n">pure</span> <span class="o">::</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">f</span> <span class="n">a</span>
    </span><span class='line'>    <span class="p">(</span><span class="o">&lt;*&gt;</span><span class="p">)</span> <span class="o">::</span> <span class="n">f</span> <span class="p">(</span><span class="n">a</span> <span class="o">-&gt;</span> <span class="n">b</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">f</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">f</span> <span class="n">b</span>
    </span>`</pre></td></tr></table></div></figure>

    我们注意到，与 `Functor typeclass` 的定义不同的是，在 `Applicative typeclass` 的定义中多了一个类约束 `Functor f` ，表示的意思是数据类型 `f` 要实现 `Applicative typeclass` 的前提条件是它必须要实现 `Functor typeclass` ，也就是说它必须是一个 `Functor` 。

    在 `Applicative typeclass` 中定义了两个函数：

*   `pure` ：将一个值 `a` 放入上下文中；
*   `(&lt;*&gt;)` ：将一个在上下文中的函数 `f (a -&gt; b)` 应用到一个在上下文中的值 `f a` ，并返回另一个在上下文中的值 `f b` 。

    **注**：`&lt;*&gt;` 函数的发音我也不知道，如果有同学知道的话还请告之，谢谢。

    ### Maybe Applicative

    同样的，我们将类型占位符 `f` 用具体类型 `Maybe` 代入，可得：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">class</span> <span class="n">Functor</span> <span class="n">Maybe</span> <span class="o">=&gt;</span> <span class="n">Applicative</span> <span class="n">Maybe</span> <span class="n">where</span>
    </span><span class='line'>    <span class="n">pure</span> <span class="o">::</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">Maybe</span> <span class="n">a</span>
    </span><span class='line'>    <span class="p">(</span><span class="o">&lt;*&gt;</span><span class="p">)</span> <span class="o">::</span> <span class="n">Maybe</span> <span class="p">(</span><span class="n">a</span> <span class="o">-&gt;</span> <span class="n">b</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">Maybe</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">Maybe</span> <span class="n">b</span>
    </span>`</pre></td></tr></table></div></figure>

    因此，对于 `Maybe` 类型来说，它要实现的 `pure` 函数的功能就是将一个值 `a` 放入 `Maybe` 上下文中。而 `(&lt;*&gt;)` 函数的功能则是将一个在 `Maybe` 上下文中的函数 `Maybe (a -&gt; b)` 应用到一个在 `Maybe` 上下文中的值 `Maybe a` ，并返回另一个在 `Maybe` 上下文中的值 `Maybe b` 。接下来，我们一起来看一下 `Maybe` 类型实现 `Applicative typeclass` 的具体细节：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">instance</span> <span class="n">Applicative</span> <span class="n">Maybe</span> <span class="n">where</span>
    </span><span class='line'>    <span class="n">pure</span> <span class="o">=</span> <span class="n">Just</span>
    </span><span class='line'>    <span class="n">Nothing</span> <span class="o">&lt;*&gt;</span> <span class="n">_</span> <span class="o">=</span> <span class="n">Nothing</span>
    </span><span class='line'>    <span class="p">(</span><span class="n">Just</span> <span class="n">func</span><span class="p">)</span> <span class="o">&lt;*&gt;</span> <span class="n">something</span> <span class="o">=</span> <span class="n">fmap</span> <span class="n">func</span> <span class="n">something</span>
    </span>`</pre></td></tr></table></div></figure>

    `pure` 函数的实现非常简单，直接等于 `Just` 即可。而对于 `(&lt;*&gt;)` 函数的实现，我们同样需要针对 `Maybe` 上下文的两种情况分别进行处理：当装函数的盒子为空时，直接返回一个新的空盒子；当装函数的盒子不为空时，即 `Just func` ，则取出 `func` ，使用 `fmap` 函数直接将 `func` 应用到那个在上下文中的值，这个正是我们前面说的 `Functor` 的功能。

    好了，我们接下来看一下将 `Just (+3)` 应用到 `Just 2` 的具体过程：

    ![applicative_just](http://blog.leichunfeng.com/images/applicative_just.png)

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">Just</span> <span class="p">(</span><span class="o">+</span><span class="mi">3</span><span class="p">)</span> <span class="o">&lt;*&gt;</span> <span class="n">Just</span> <span class="mi">2</span>
    </span><span class='line'><span class="n">Just</span> <span class="mi">5</span>
    </span>`</pre></td></tr></table></div></figure>

    同样的，当我们将一个空盒子，即 `Nothing` 应用到 `Just 2` 的时候，我们将得到一个新的空盒子：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">Nothing</span> <span class="o">&lt;*&gt;</span> <span class="n">Just</span> <span class="mi">2</span>
    </span><span class='line'><span class="n">Nothing</span>
    </span>`</pre></td></tr></table></div></figure>

    ## Monad

    截至目前，我们已经知道了 `Functor` 的作用就是应用一个函数到一个上下文中的值：

    ![fmap](http://blog.leichunfeng.com/images/fmap.png)

    而 `Applicative` 的作用则是应用一个上下文中的函数到一个上下文中的值：

    ![apply_justadd3_just2](http://blog.leichunfeng.com/images/apply_justadd3_just2.png)

    那么 `Monad` 又会是什么呢？其实，`Monad` 的作用跟 `Functor` 类似，也是应用一个函数到一个上下文中的值。不同之处在于，`Functor` 应用的是一个接收一个普通值并且返回一个普通值的函数，而 `Monad` 应用的是一个接收一个普通值但是返回一个在上下文中的值的函数：

    ![bind](http://blog.leichunfeng.com/images/bind.png)

    ### Monad typeclass

    同样的，我们先来看一下 `Monad typeclass` 的定义：

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
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">class</span> <span class="n">Applicative</span> <span class="n">m</span> <span class="o">=&gt;</span> <span class="n">Monad</span> <span class="n">m</span> <span class="n">where</span>
    </span><span class='line'>    <span class="k">return</span> <span class="o">::</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">a</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">(</span><span class="o">&gt;&gt;=</span><span class="p">)</span> <span class="o">::</span> <span class="n">m</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="n">a</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">b</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">b</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">(</span><span class="o">&gt;&gt;</span><span class="p">)</span> <span class="o">::</span> <span class="n">m</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">b</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">b</span>
    </span><span class='line'>    <span class="n">x</span> <span class="o">&gt;&gt;</span> <span class="n">y</span> <span class="o">=</span> <span class="n">x</span> <span class="o">&gt;&gt;=</span> <span class="err">\</span><span class="n">_</span> <span class="o">-&gt;</span> <span class="n">y</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="n">fail</span> <span class="o">::</span> <span class="n">String</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">a</span>
    </span><span class='line'>    <span class="n">fail</span> <span class="n">msg</span> <span class="o">=</span> <span class="n">error</span> <span class="n">msg</span>
    </span>`</pre></td></tr></table></div></figure>

    哇，这什么鬼，完全看不懂啊，太复杂了。兄台莫急，且听我细说。在 `Monad typeclass` 中定义了四个函数，分别是 `return`、`(&gt;&gt;=)`、`(&gt;&gt;)` 和 `fail` ，且后两个函数 `(&gt;&gt;)` 和 `fail` 给出了默认实现，而在绝大多数情况下，我们都不需要去重写它们。因此，去掉这两个函数后，`Monad typeclass` 的定义可简化为：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">class</span> <span class="n">Applicative</span> <span class="n">m</span> <span class="o">=&gt;</span> <span class="n">Monad</span> <span class="n">m</span> <span class="n">where</span>
    </span><span class='line'>    <span class="k">return</span> <span class="o">::</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">a</span>
    </span><span class='line'>    <span class="p">(</span><span class="o">&gt;&gt;=</span><span class="p">)</span> <span class="o">::</span> <span class="n">m</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="n">a</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">b</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">b</span>
    </span>`</pre></td></tr></table></div></figure>

    怎么样？现在看上去就好多了吧。跟 `Applicative typeclass` 的定义一样，在 `Monad typeclass` 的定义中也有一个类约束 `Applicative m` ，表示的意思是一种数据类型 `m` 要成为 `Monad` 的前提条件是它必须是 `Applicative` 。另外，其实 `return` 函数的功能与 `Applicative` 中的 `pure` 函数的功能是一样的，只不过换了一个名字而已，它们的作用都是将一个值 `a` 放入上下文中。而 `(&gt;&gt;=)` 函数的功能则是应用一个（接收一个普通值 `a` 但是返回一个在上下文中的值 `m b` 的）函数 `(a -&gt; m b)` 到一个上下文中的值 `m a` ，并返回另一个在相同上下文中的值 `m b` 。

    **注**：`&gt;&gt;=` 函数的发音为 `bind` ，学习 `ReactiveCocoa` 的同学要注意啦。另外，`&gt;&gt;=` 函数可类比 `Swift` 中的 `flatMap` 方法。

    ### Maybe Monad

    同样的，我们将类型占位符 `m` 用具体类型 `Maybe` 代入，可得：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">class</span> <span class="n">Applicative</span> <span class="n">Maybe</span> <span class="o">=&gt;</span> <span class="n">Monad</span> <span class="n">Maybe</span> <span class="n">where</span>
    </span><span class='line'>    <span class="k">return</span> <span class="o">::</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">Maybe</span> <span class="n">a</span>
    </span><span class='line'>    <span class="p">(</span><span class="o">&gt;&gt;=</span><span class="p">)</span> <span class="o">::</span> <span class="n">Maybe</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="n">a</span> <span class="o">-&gt;</span> <span class="n">Maybe</span> <span class="n">b</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">Maybe</span> <span class="n">b</span>
    </span>`</pre></td></tr></table></div></figure>

    相信你用盒子模型已经能够轻松地理解上面两个函数了，因此不再赘述。接下来，我们一起来看一下 `Maybe` 类型实现 `Monad typeclass` 的具体细节：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">instance</span> <span class="n">Monad</span> <span class="n">Maybe</span> <span class="n">where</span>
    </span><span class='line'>    <span class="k">return</span> <span class="n">x</span> <span class="o">=</span> <span class="n">Just</span> <span class="n">x</span>
    </span><span class='line'>    <span class="n">Nothing</span> <span class="o">&gt;&gt;=</span> <span class="n">func</span> <span class="o">=</span> <span class="n">Nothing</span>
    </span><span class='line'>    <span class="n">Just</span> <span class="n">x</span> <span class="o">&gt;&gt;=</span> <span class="n">func</span>  <span class="o">=</span> <span class="n">func</span> <span class="n">x</span>
    </span>`</pre></td></tr></table></div></figure>

    正如前面所说，`return` 函数的实现跟 `pure` 函数一样，直接等于 `Just` 函数即可，功能就是将一个值 `x` 放入 `Maybe` 盒子中，变成 `Just x` 。同样的，对于 `(&gt;&gt;=)` 函数的实现，我们需要针对 `Maybe` 上下文的两种情况分别进行处理，当盒子为空时，直接返回一个新的空盒子；当盒子不为空时，即 `Just x` ，则取出 `x` ，直接将 `func` 函数应用到 `x` ，而我们知道 `func x` 的结果就是一个在上下文中的值。

    下面，我们一起来看一个具体的例子。我们先定义一个 `half` 函数，这个函数接收一个数字 `x` 作为参数，如果 `x` 是偶数，则将 `x` 除以 `2` ，并将结果放入 `Maybe` 盒子中；如果 `x` 不是偶数，则返回一个空盒子：

    ![half](http://blog.leichunfeng.com/images/half.png)

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">half</span> <span class="n">x</span> <span class="o">=</span> <span class="k">if</span> <span class="n">even</span> <span class="n">x</span>
    </span><span class='line'>    <span class="n">then</span> <span class="n">Just</span> <span class="p">(</span><span class="n">x</span> <span class="err">`</span><span class="n">div</span><span class="err">`</span> <span class="mi">2</span><span class="p">)</span>
    </span><span class='line'>    <span class="k">else</span> <span class="n">Nothing</span>
    </span>`</pre></td></tr></table></div></figure>

    接下来，我们使用 `(&gt;&gt;=)` 函数将 `half` 函数应用到 `Just 20` ，假设得到结果 `y` ；然后继续使用 `(&gt;&gt;=)` 函数将 `half` 函数应用到上一步的结果 `y` ，以此类推，看看会得到什么样的结果：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">Just</span> <span class="mi">20</span> <span class="o">&gt;&gt;=</span> <span class="n">half</span>
    </span><span class='line'><span class="n">Just</span> <span class="mi">10</span>
    </span><span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">Just</span> <span class="mi">10</span> <span class="o">&gt;&gt;=</span> <span class="n">half</span>
    </span><span class='line'><span class="n">Just</span> <span class="mi">5</span>
    </span><span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">Just</span> <span class="mi">5</span> <span class="o">&gt;&gt;=</span> <span class="n">half</span>
    </span><span class='line'><span class="n">Nothing</span>
    </span>`</pre></td></tr></table></div></figure>

    看到上面的运算过程，不知道你有没有看出点什么端倪呢？上一步的输出作为下一步的输入，并且只要你愿意的话，这个过程可以无限地进行下去。我想你可能已经想到了，是的，就是链式操作。所有的操作链接起来就像是一条生产线，每一步的操作都是对输入进行加工，然后产生输出，整个操作过程可以看作是对最初的原材料 `Just 20` 进行加工并最终生产出成品 `Nothing` 的过程：

    ![monad_chain](http://blog.leichunfeng.com/images/monad_chain.png)

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">ghci</span><span class="o">&gt;</span> <span class="n">Just</span> <span class="mi">20</span> <span class="o">&gt;&gt;=</span> <span class="n">half</span> <span class="o">&gt;&gt;=</span> <span class="n">half</span> <span class="o">&gt;&gt;=</span> <span class="n">half</span>
    </span><span class='line'><span class="n">Nothing</span>
    </span>`</pre></td></tr></table></div></figure>

    **注**：链式操作只是 `Monad` 为我们带来的主要好处之一；另一个本文并未涉及到的主要好处是，`Monad` 可以为我们自动处理上下文，而我们只需要关心真正的值就可以了。

    ### ReactiveCocoa

    现在，我们已经知道 `Monad` 是什么了，它就是一种实现了 `Monad typeclass` 的数据类型。那么它有什么具体的应用呢？你总不能让我们都来做理论研究吧。既然如此，那我们就只好祭出 `Objective-C` 中的神器，`ReactiveCocoa` ，它就是根据 `Monad` 的概念搭建起来的。下面是 `RACStream` 的继承结构图：

    ![RACStream](http://blog.leichunfeng.com/images/RACStream.png)

    `RACStream` 是 `ReactiveCocoa` 中最核心的类，它就是一个 `Monad` ：

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
    </pre></td><td class='code'><pre>`<span class='line'><span class="c1">/// An abstract class representing any stream of values.</span>
    </span><span class='line'><span class="c1">///</span>
    </span><span class='line'><span class="c1">/// This class represents a monad, upon which many stream-based operations can</span>
    </span><span class='line'><span class="c1">/// be built.</span>
    </span><span class='line'><span class="c1">///</span>
    </span><span class='line'><span class="c1">/// When subclassing RACStream, only the methods in the main @interface body need</span>
    </span><span class='line'><span class="c1">/// to be overridden.</span>
    </span><span class='line'><span class="k">@interface</span> <span class="nc">RACStream</span> : <span class="bp">NSObject</span>
    </span><span class='line'>
    </span><span class='line'><span class="c1">/// Lifts `value` into the stream monad.</span>
    </span><span class='line'><span class="c1">///</span>
    </span><span class='line'><span class="c1">/// Returns a stream containing only the given value.</span>
    </span><span class='line'><span class="p">+</span> <span class="p">(</span><span class="kt">instancetype</span><span class="p">)</span><span class="nf">return:</span><span class="p">(</span><span class="kt">id</span><span class="p">)</span><span class="nv">value</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="c1">/// Lazily binds a block to the values in the receiver.</span>
    </span><span class='line'><span class="c1">///</span>
    </span><span class='line'><span class="c1">/// This should only be used if you need to terminate the bind early, or close</span>
    </span><span class='line'><span class="c1">/// over some state. -flattenMap: is more appropriate for all other cases.</span>
    </span><span class='line'><span class="c1">///</span>
    </span><span class='line'><span class="c1">/// block - A block returning a RACStreamBindBlock. This block will be invoked</span>
    </span><span class='line'><span class="c1">///         each time the bound stream is re-evaluated. This block must not be</span>
    </span><span class='line'><span class="c1">///         nil or return nil.</span>
    </span><span class='line'><span class="c1">///</span>
    </span><span class='line'><span class="c1">/// Returns a new stream which represents the combined result of all lazy</span>
    </span><span class='line'><span class="c1">/// applications of `block`.</span>
    </span><span class='line'><span class="p">-</span> <span class="p">(</span><span class="kt">instancetype</span><span class="p">)</span><span class="nf">bind:</span><span class="p">(</span><span class="n">RACStreamBindBlock</span> <span class="p">(</span><span class="o">^</span><span class="p">)(</span><span class="kt">void</span><span class="p">))</span><span class="nv">block</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">@end</span>
    </span>`</pre></td></tr></table></div></figure>

    我们可以看到，在 `RACStream` 中定义了两个看上去非常眼熟的方法：

1.  `+ (instancetype)return:(id)value;` ；
2.  `- (instancetype)bind:(RACStreamBindBlock (^)(void))block;` 。

    其中，`return:` 方法的功能就是将一个值 `value` 放入 `RACStream` 上下文中；而 `bind:` 方法的功能则是将一个 `RACStreamBindBlock` 类型的 `block` 应用到一个在 `RACStream` 上下文中的值（`receiver`），并返回另一个在 `RACStream` 上下文中的值。**注**，`RACStreamBindBlock` 类型的 `block` 就是一个接收一个普通值 `value` 但是返回一个在 `RACStream` 上下文中的值的“函数”：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="c1">/// A block which accepts a value from a RACStream and returns a new instance</span>
    </span><span class='line'><span class="c1">/// of the same stream class.</span>
    </span><span class='line'><span class="c1">///</span>
    </span><span class='line'><span class="c1">/// Setting `stop` to `YES` will cause the bind to terminate after the returned</span>
    </span><span class='line'><span class="c1">/// value. Returning `nil` will result in immediate termination.</span>
    </span><span class='line'><span class="k">typedef</span> <span class="n">RACStream</span> <span class="o">*</span> <span class="p">(</span><span class="o">^</span><span class="n">RACStreamBindBlock</span><span class="p">)(</span><span class="kt">id</span> <span class="n">value</span><span class="p">,</span> <span class="kt">BOOL</span> <span class="o">*</span><span class="n">stop</span><span class="p">);</span>
    </span>`</pre></td></tr></table></div></figure>

    接下来，为了加深理解，我们一起来对比一下 `Monad typeclass` 的定义：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">class</span> <span class="n">Applicative</span> <span class="n">m</span> <span class="o">=&gt;</span> <span class="n">Monad</span> <span class="n">m</span> <span class="n">where</span>
    </span><span class='line'>    <span class="k">return</span> <span class="o">::</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">a</span>
    </span><span class='line'>    <span class="p">(</span><span class="o">&gt;&gt;=</span><span class="p">)</span> <span class="o">::</span> <span class="n">m</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="n">a</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">b</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">m</span> <span class="n">b</span>
    </span>`</pre></td></tr></table></div></figure>

    同样的，我们将类型占位符 `m` 用 `RACStream` 代入，可得：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">class</span> <span class="n">Applicative</span> <span class="n">RACStream</span> <span class="o">=&gt;</span> <span class="n">Monad</span> <span class="n">RACStream</span> <span class="n">where</span>
    </span><span class='line'>    <span class="k">return</span> <span class="o">::</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="n">RACStream</span> <span class="n">a</span>
    </span><span class='line'>    <span class="p">(</span><span class="o">&gt;&gt;=</span><span class="p">)</span> <span class="o">::</span> <span class="n">RACStream</span> <span class="n">a</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="n">a</span> <span class="o">-&gt;</span> <span class="n">RACStream</span> <span class="n">b</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">RACStream</span> <span class="n">b</span>
    </span>`</pre></td></tr></table></div></figure>

    其中，`return :: a -&gt; RACStream a` 就对应 `+ (instancetype)return:(id)value;` ，而 `(&gt;&gt;=) :: RACStream a -&gt; (a -&gt; RACStream b) -&gt; RACStream b` 则对应 `- (instancetype)bind:(RACStreamBindBlock (^)(void))block;` 。**注**：我们前面已经提到过了，`&gt;&gt;=` 函数的发音就是 `bind` 。因此，`ReactiveCocoa` 便有了下面的玩法：

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="n">RACSignal</span> <span class="o">*</span><span class="n">signal2</span> <span class="o">=</span> <span class="p">[[[</span><span class="n">signal1</span>
    </span><span class='line'>    <span class="nl">bind</span><span class="p">:</span><span class="n">block1</span><span class="p">]</span>
    </span><span class='line'>    <span class="nl">bind</span><span class="p">:</span><span class="n">block2</span><span class="p">]</span>
    </span><span class='line'>    <span class="nl">bind</span><span class="p">:</span><span class="n">block3</span><span class="p">];</span>
    </span>
</td></tr></table></div></figure>

`Monad` 就像是 `ReactiveCocoa` 中的太极，太极生两仪，两仪生四象，四象生八卦。至此，我们已经知道了 `ReactiveCocoa` 中最核心的原理，而更多关于 `ReactiveCocoa` 的内容我们将在后续的源码解析中再进行介绍，敬请期待。

## 总结

`Functor`、`Applicative` 和 `Monad` 是什么：

1.  一个 `Functor` 就是一种实现了 `Functor typeclass` 的数据类型；
2.  一个 `Applicative` 就是一种实现了 `Applicative typeclass` 的数据类型；
3.  一个 `Monad` 就是一种实现了 `Monad typeclass` 的数据类型。

`Functor`、`Applicative` 和 `Monad` 三者之间的联系：

1.  `Applicative` 是增强型的 `Functor` ，一种数据类型要成为 `Applicative` 的前提条件是它必须是 `Functor` ；
2.  `Monad` 是增强型的 `Applicative` ，一种数据类型要成为 `Monad` 的前提条件是它必须是 `Applicative` 。

`Functor`、`Applicative` 和 `Monad` 三者之间的区别：

![recap](http://blog.leichunfeng.com/images/recap.png)

1.  `Functor` ：使用 `fmap` 应用一个函数到一个上下文中的值；
2.  `Applicative` ：使用 `&lt;*&gt;` 应用一个上下文中的函数到一个上下文中的值；
3.  `Monad` ：使用 `&gt;&gt;=` 应用一个接收一个普通值但是返回一个在上下文中的值的函数到一个上下文中的值。

此外，我们还介绍了一种非常有意思的数据类型 `Maybe` ，它实现了 `Functor typeclass`、`Applicative typeclass` 和 `Monad typeclass` ，所以它同时是 `Functor`、`Applicative` 和 `Monad` 。

以上就是本文的全部内容，希望可以对你有所帮助，Good luck !

## 参考链接

[http://learnyouahaskell.com/chapters](http://learnyouahaskell.com/chapters)

[https://downloads.haskell.org/~ghc/latest/docs/html/libraries](https://downloads.haskell.org/~ghc/latest/docs/html/libraries)

[http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)

![](http://blog.leichunfeng.com/images/wechat_pay.jpg)