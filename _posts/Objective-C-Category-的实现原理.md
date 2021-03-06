---
title: Objective-C Category 的实现原理
tags: []
date: 2015-05-18 00:20:57
---

对设计模式有一定了解的朋友应该听说过装饰模式，Objective-C 中的 Category 就是对装饰模式的一种具体实现。它的主要作用是在不改变原有类的前提下，动态地给这个类添加一些方法。在 Objective-C 中的具体体现为：实例（类）方法、属性和协议。是的，在 Objective-C 中可以用 Category 来实现协议。本文将结合 [runtime](http://opensource.apple.com/tarballs/objc4/)（我下载的是当前的最新版本 `objc4-646.tar.gz`) 的源码来探究它实现的原理。

## 使用场景

根据苹果官方文档对 [Category](https://developer.apple.com/library/ios/documentation/General/Conceptual/DevPedia-CocoaCore/Category.html#//apple_ref/doc/uid/TP40008195-CH5-SW1) 的描述，它的使用场景主要有三个：

1.  给现有的类添加方法；
2.  将一个类的实现拆分成多个独立的源文件；
3.  声明私有的方法。

其中，第 `1` 个是最典型的使用场景，应用最广泛。

**注**：Category 有一个非常容易误用的场景，那就是用 Category 来覆写父类或主类的方法。虽然目前 Objective-C 是允许这么做的，但是这种使用场景是**非常**不推荐的。[使用 Category 来覆写方法](http://stackoverflow.com/questions/5272451/overriding-methods-using-categories-in-objective-c)有很多缺点，比如不能覆写 Category 中的方法、无法调用主类中的原始实现等，且很容易造成无法预估的行为。

## 实现原理

我们知道，无论我们有没有主动引入 Category 的头文件，Category 中的方法都会被添加进主类中。我们可以通过 `- performSelector:` 等方式对 Category 中的相应方法进行调用，之所以需要在调用的地方引入 Category 的头文件，只是为了“照顾”编译器同学的感受。

下面，我们将结合 runtime 的源码探究下 Category 的实现原理。打开 runtime 源码工程，在文件 `objc-runtime-new.mm` 中找到以下函数：

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
<span class='line-number'>65</span>
<span class='line-number'>66</span>
<span class='line-number'>67</span>
<span class='line-number'>68</span>
<span class='line-number'>69</span>
</pre></td><td class='code'>

    <span class='line'><span class="kt">void</span> <span class="nf">_read_images</span><span class="p">(</span><span class="n">header_info</span> <span class="o">**</span><span class="n">hList</span><span class="p">,</span> <span class="kt">uint32_t</span> <span class="n">hCount</span><span class="p">)</span>
    </span><span class='line'><span class="p">{</span>
    </span><span class='line'>    <span class="p">...</span>
    </span><span class='line'>        <span class="n">_free_internal</span><span class="p">(</span><span class="n">resolvedFutureClasses</span><span class="p">);</span>
    </span><span class='line'>    <span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="c1">// Discover categories. </span>
    </span><span class='line'>    <span class="k">for</span> <span class="p">(</span><span class="n">EACH_HEADER</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>        <span class="kt">category_t</span> <span class="o">**</span><span class="n">catlist</span> <span class="o">=</span>
    </span><span class='line'>            <span class="n">_getObjc2CategoryList</span><span class="p">(</span><span class="n">hi</span><span class="p">,</span> <span class="o">&amp;</span><span class="n">count</span><span class="p">);</span>
    </span><span class='line'>        <span class="k">for</span> <span class="p">(</span><span class="n">i</span> <span class="o">=</span> <span class="mi">0</span><span class="p">;</span> <span class="n">i</span> <span class="o">&lt;</span> <span class="n">count</span><span class="p">;</span> <span class="n">i</span><span class="o">++</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="kt">category_t</span> <span class="o">*</span><span class="n">cat</span> <span class="o">=</span> <span class="n">catlist</span><span class="p">[</span><span class="n">i</span><span class="p">];</span>
    </span><span class='line'>            <span class="kt">Class</span> <span class="n">cls</span> <span class="o">=</span> <span class="n">remapClass</span><span class="p">(</span><span class="n">cat</span><span class="o">-&gt;</span><span class="n">cls</span><span class="p">);</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="k">if</span> <span class="p">(</span><span class="o">!</span><span class="n">cls</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>                <span class="c1">// Category&#39;s target class is missing (probably weak-linked).</span>
    </span><span class='line'>                <span class="c1">// Disavow any knowledge of this category.</span>
    </span><span class='line'>                <span class="n">catlist</span><span class="p">[</span><span class="n">i</span><span class="p">]</span> <span class="o">=</span> <span class="nb">nil</span><span class="p">;</span>
    </span><span class='line'>                <span class="k">if</span> <span class="p">(</span><span class="n">PrintConnecting</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>                    <span class="n">_objc_inform</span><span class="p">(</span><span class="s">&quot;CLASS: IGNORING category \?\?\?(%s) %p with &quot;</span>
    </span><span class='line'>                                 <span class="s">&quot;missing weak-linked target class&quot;</span><span class="p">,</span>
    </span><span class='line'>                                 <span class="n">cat</span><span class="o">-&gt;</span><span class="n">name</span><span class="p">,</span> <span class="n">cat</span><span class="p">);</span>
    </span><span class='line'>                <span class="p">}</span>
    </span><span class='line'>                <span class="k">continue</span><span class="p">;</span>
    </span><span class='line'>            <span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="c1">// Process this category. </span>
    </span><span class='line'>            <span class="c1">// First, register the category with its target class. </span>
    </span><span class='line'>            <span class="c1">// Then, rebuild the class&#39;s method lists (etc) if </span>
    </span><span class='line'>            <span class="c1">// the class is realized. </span>
    </span><span class='line'>            <span class="kt">BOOL</span> <span class="n">classExists</span> <span class="o">=</span> <span class="nb">NO</span><span class="p">;</span>
    </span><span class='line'>            <span class="k">if</span> <span class="p">(</span><span class="n">cat</span><span class="o">-&gt;</span><span class="n">instanceMethods</span> <span class="o">||</span>  <span class="n">cat</span><span class="o">-&gt;</span><span class="n">protocols</span>
    </span><span class='line'>                <span class="o">||</span>  <span class="n">cat</span><span class="o">-&gt;</span><span class="n">instanceProperties</span><span class="p">)</span>
    </span><span class='line'>            <span class="p">{</span>
    </span><span class='line'>                <span class="n">addUnattachedCategoryForClass</span><span class="p">(</span><span class="n">cat</span><span class="p">,</span> <span class="n">cls</span><span class="p">,</span> <span class="n">hi</span><span class="p">);</span>
    </span><span class='line'>                <span class="k">if</span> <span class="p">(</span><span class="n">cls</span><span class="o">-&gt;</span><span class="n">isRealized</span><span class="p">())</span> <span class="p">{</span>
    </span><span class='line'>                    <span class="n">remethodizeClass</span><span class="p">(</span><span class="n">cls</span><span class="p">);</span>
    </span><span class='line'>                    <span class="n">classExists</span> <span class="o">=</span> <span class="nb">YES</span><span class="p">;</span>
    </span><span class='line'>                <span class="p">}</span>
    </span><span class='line'>                <span class="k">if</span> <span class="p">(</span><span class="n">PrintConnecting</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>                    <span class="n">_objc_inform</span><span class="p">(</span><span class="s">&quot;CLASS: found category -%s(%s) %s&quot;</span><span class="p">,</span>
    </span><span class='line'>                                 <span class="n">cls</span><span class="o">-&gt;</span><span class="n">nameForLogging</span><span class="p">(),</span> <span class="n">cat</span><span class="o">-&gt;</span><span class="n">name</span><span class="p">,</span>
    </span><span class='line'>                                 <span class="n">classExists</span> <span class="o">?</span> <span class="s">&quot;on existing class&quot;</span> <span class="o">:</span> <span class="s">&quot;&quot;</span><span class="p">);</span>
    </span><span class='line'>                <span class="p">}</span>
    </span><span class='line'>            <span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'>            <span class="k">if</span> <span class="p">(</span><span class="n">cat</span><span class="o">-&gt;</span><span class="n">classMethods</span>  <span class="o">||</span>  <span class="n">cat</span><span class="o">-&gt;</span><span class="n">protocols</span>
    </span><span class='line'>                <span class="cm">/* ||  cat-&gt;classProperties */</span><span class="p">)</span>
    </span><span class='line'>            <span class="p">{</span>
    </span><span class='line'>                <span class="n">addUnattachedCategoryForClass</span><span class="p">(</span><span class="n">cat</span><span class="p">,</span> <span class="n">cls</span><span class="o">-&gt;</span><span class="n">ISA</span><span class="p">(),</span> <span class="n">hi</span><span class="p">);</span>
    </span><span class='line'>                <span class="k">if</span> <span class="p">(</span><span class="n">cls</span><span class="o">-&gt;</span><span class="n">ISA</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">isRealized</span><span class="p">())</span> <span class="p">{</span>
    </span><span class='line'>                    <span class="n">remethodizeClass</span><span class="p">(</span><span class="n">cls</span><span class="o">-&gt;</span><span class="n">ISA</span><span class="p">());</span>
    </span><span class='line'>                <span class="p">}</span>
    </span><span class='line'>                <span class="k">if</span> <span class="p">(</span><span class="n">PrintConnecting</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>                    <span class="n">_objc_inform</span><span class="p">(</span><span class="s">&quot;CLASS: found category +%s(%s)&quot;</span><span class="p">,</span>
    </span><span class='line'>                                 <span class="n">cls</span><span class="o">-&gt;</span><span class="n">nameForLogging</span><span class="p">(),</span> <span class="n">cat</span><span class="o">-&gt;</span><span class="n">name</span><span class="p">);</span>
    </span><span class='line'>                <span class="p">}</span>
    </span><span class='line'>            <span class="p">}</span>
    </span><span class='line'>        <span class="p">}</span>
    </span><span class='line'>    <span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="c1">// Category discovery MUST BE LAST to avoid potential races </span>
    </span><span class='line'>    <span class="c1">// when other threads call the new category code before </span>
    </span><span class='line'>    <span class="c1">// this thread finishes its fixups.</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="c1">// +load handled by prepare_load_methods()</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="p">...</span>
    </span><span class='line'><span class="p">}</span>
    </span>`</pre></td></tr></table></div></figure>

    从第 27-58 行的关键代码，我们可以知道在这个函数中对 Category 做了如下处理：

1.  将 Category 和它的主类（或元类）注册到哈希表中；
2.  如果主类（或元类）已实现，那么重建它的方法列表。

    在这里分了两种情况进行处理：Category 中的实例方法和属性被整合到主类中；而类方法则被整合到元类中（关于对象、类和元类的更多细节，可以参考我前面的博文[《Objective-C 对象模型》](http://blog.leichunfeng.com/blog/2015/04/25/objective-c-object-model/)）。另外，对协议的处理比较特殊，Category 中的协议被同时整合到了主类和元类中。

    我们注意到，不管是哪种情况，最终都是通过调用 `static void remethodizeClass(Class cls)` 函数来重新整理类的数据的。

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
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">static</span> <span class="kt">void</span> <span class="nf">remethodizeClass</span><span class="p">(</span><span class="kt">Class</span> <span class="n">cls</span><span class="p">)</span>
    </span><span class='line'><span class="p">{</span>
    </span><span class='line'>    <span class="p">...</span>
    </span><span class='line'>                         <span class="n">cls</span><span class="o">-&gt;</span><span class="n">nameForLogging</span><span class="p">(),</span> <span class="n">isMeta</span> <span class="o">?</span> <span class="s">&quot;(meta)&quot;</span> <span class="o">:</span> <span class="s">&quot;&quot;</span><span class="p">);</span>
    </span><span class='line'>        <span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'>        <span class="c1">// Update methods, properties, protocols</span>
    </span><span class='line'>
    </span><span class='line'>        <span class="n">attachCategoryMethods</span><span class="p">(</span><span class="n">cls</span><span class="p">,</span> <span class="n">cats</span><span class="p">,</span> <span class="nb">YES</span><span class="p">);</span>
    </span><span class='line'>
    </span><span class='line'>        <span class="n">newproperties</span> <span class="o">=</span> <span class="n">buildPropertyList</span><span class="p">(</span><span class="nb">nil</span><span class="p">,</span> <span class="n">cats</span><span class="p">,</span> <span class="n">isMeta</span><span class="p">);</span>
    </span><span class='line'>        <span class="k">if</span> <span class="p">(</span><span class="n">newproperties</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="n">newproperties</span><span class="o">-&gt;</span><span class="n">next</span> <span class="o">=</span> <span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">properties</span><span class="p">;</span>
    </span><span class='line'>            <span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">properties</span> <span class="o">=</span> <span class="n">newproperties</span><span class="p">;</span>
    </span><span class='line'>        <span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'>        <span class="n">newprotos</span> <span class="o">=</span> <span class="n">buildProtocolList</span><span class="p">(</span><span class="n">cats</span><span class="p">,</span> <span class="nb">nil</span><span class="p">,</span> <span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">protocols</span><span class="p">);</span>
    </span><span class='line'>        <span class="k">if</span> <span class="p">(</span><span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">protocols</span>  <span class="o">&amp;&amp;</span>  <span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">protocols</span> <span class="o">!=</span> <span class="n">newprotos</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>            <span class="n">_free_internal</span><span class="p">(</span><span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">protocols</span><span class="p">);</span>
    </span><span class='line'>        <span class="p">}</span>
    </span><span class='line'>        <span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">protocols</span> <span class="o">=</span> <span class="n">newprotos</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'>        <span class="n">_free_internal</span><span class="p">(</span><span class="n">cats</span><span class="p">);</span>
    </span><span class='line'>    <span class="p">}</span>
    </span><span class='line'><span class="p">}</span>
    </span>`</pre></td></tr></table></div></figure>

    这个函数的主要作用是将 Category 中的方法、属性和协议整合到类（主类或元类）中，更新类的数据字段 `data()` 中 `method_lists（或 method_list）`、`properties` 和 `protocols` 的值。进一步，我们通过 `attachCategoryMethods` 函数的源码可以找到真正处理 Category 方法的 `attachMethodLists` 函数：

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
    </pre></td><td class='code'><pre>`<span class='line'><span class="k">static</span> <span class="kt">void</span>
    </span><span class='line'><span class="nf">attachMethodLists</span><span class="p">(</span><span class="kt">Class</span> <span class="n">cls</span><span class="p">,</span> <span class="kt">method_list_t</span> <span class="o">**</span><span class="n">addedLists</span><span class="p">,</span> <span class="kt">int</span> <span class="n">addedCount</span><span class="p">,</span>
    </span><span class='line'>                  <span class="kt">bool</span> <span class="n">baseMethods</span><span class="p">,</span> <span class="kt">bool</span> <span class="n">methodsFromBundle</span><span class="p">,</span>
    </span><span class='line'>                  <span class="kt">bool</span> <span class="n">flushCaches</span><span class="p">)</span>
    </span><span class='line'><span class="p">{</span>
    </span><span class='line'>    <span class="p">...</span>
    </span><span class='line'>        <span class="n">newLists</span><span class="p">[</span><span class="n">newCount</span><span class="o">++</span><span class="p">]</span> <span class="o">=</span> <span class="n">mlist</span><span class="p">;</span>
    </span><span class='line'>    <span class="p">}</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="c1">// Copy old methods to the method list array</span>
    </span><span class='line'>    <span class="k">for</span> <span class="p">(</span><span class="n">i</span> <span class="o">=</span> <span class="mi">0</span><span class="p">;</span> <span class="n">i</span> <span class="o">&lt;</span> <span class="n">oldCount</span><span class="p">;</span> <span class="n">i</span><span class="o">++</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>        <span class="n">newLists</span><span class="p">[</span><span class="n">newCount</span><span class="o">++</span><span class="p">]</span> <span class="o">=</span> <span class="n">oldLists</span><span class="p">[</span><span class="n">i</span><span class="p">];</span>
    </span><span class='line'>    <span class="p">}</span>
    </span><span class='line'>    <span class="k">if</span> <span class="p">(</span><span class="n">oldLists</span>  <span class="o">&amp;&amp;</span>  <span class="n">oldLists</span> <span class="o">!=</span> <span class="n">oldBuf</span><span class="p">)</span> <span class="n">free</span><span class="p">(</span><span class="n">oldLists</span><span class="p">);</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="c1">// nil-terminate</span>
    </span><span class='line'>    <span class="n">newLists</span><span class="p">[</span><span class="n">newCount</span><span class="p">]</span> <span class="o">=</span> <span class="nb">nil</span><span class="p">;</span>
    </span><span class='line'>
    </span><span class='line'>    <span class="k">if</span> <span class="p">(</span><span class="n">newCount</span> <span class="o">&gt;</span> <span class="mi">1</span><span class="p">)</span> <span class="p">{</span>
    </span><span class='line'>        <span class="n">assert</span><span class="p">(</span><span class="n">newLists</span> <span class="o">!=</span> <span class="n">newBuf</span><span class="p">);</span>
    </span><span class='line'>        <span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">method_lists</span> <span class="o">=</span> <span class="n">newLists</span><span class="p">;</span>
    </span><span class='line'>        <span class="n">cls</span><span class="o">-&gt;</span><span class="n">setInfo</span><span class="p">(</span><span class="n">RW_METHOD_ARRAY</span><span class="p">);</span>
    </span><span class='line'>    <span class="p">}</span> <span class="k">else</span> <span class="p">{</span>
    </span><span class='line'>        <span class="n">assert</span><span class="p">(</span><span class="n">newLists</span> <span class="o">==</span> <span class="n">newBuf</span><span class="p">);</span>
    </span><span class='line'>        <span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">method_list</span> <span class="o">=</span> <span class="n">newLists</span><span class="p">[</span><span class="mi">0</span><span class="p">];</span>
    </span><span class='line'>        <span class="n">assert</span><span class="p">(</span><span class="o">!</span><span class="p">(</span><span class="n">cls</span><span class="o">-&gt;</span><span class="n">data</span><span class="p">()</span><span class="o">-&gt;</span><span class="n">flags</span> <span class="o">&amp;</span> <span class="n">RW_METHOD_ARRAY</span><span class="p">));</span>
    </span><span class='line'>    <span class="p">}</span>
    </span><span class='line'><span class="p">}</span>
    </span>`</pre></td></tr></table></div></figure>

    这个函数的代码量看上去比较多，但是我们并不难理解它的目的。它的主要作用就是将类中的旧有方法和 Category 中新添加的方法整合成一个新的方法列表，并赋值给 `method_lists` 或 `method_list` 。通过探究这个处理过程，我们也印证了一个结论，那就是主类中的方法和 Category 中的方法在 runtime 看来并没有区别，它们是被同等对待的，都保存在主类的方法列表中。

    不过，类的方法列表字段有一点特殊，它的结构是联合体，`method_lists` 和 `method_list` 共用同一块内存地址。当 `newCount` 的个数大于 1 时，使用 `method_lists` 来保存 `newLists` ，并将方法列表的**标志位**置为 `RW_METHOD_ARRAY` ，此时类的方法列表字段是 `method_list_t` 类型的指针数组；否则，使用 `method_list` 来保存 `newLists` ，并将方法列表的**标志位**置空，此时类的方法列表字段是 `method_list_t` 类型的指针。

    <figure class='code'><figcaption><span></span></figcaption><div class="highlight"><table><tr><td class="gutter"><pre class="line-numbers"><span class='line-number'>1</span>
    <span class='line-number'>2</span>
    <span class='line-number'>3</span>
    <span class='line-number'>4</span>
    <span class='line-number'>5</span>
    <span class='line-number'>6</span>
    <span class='line-number'>7</span>
    </pre></td><td class='code'><pre>`<span class='line'><span class="c1">// class&#39;s method list is an array of method lists</span>
    </span><span class='line'><span class="cp">#define RW_METHOD_ARRAY       (1&lt;&lt;20)</span>
    </span><span class='line'>
    </span><span class='line'><span class="k">union</span> <span class="p">{</span>
    </span><span class='line'>    <span class="kt">method_list_t</span> <span class="o">**</span><span class="n">method_lists</span><span class="p">;</span>  <span class="c1">// RW_METHOD_ARRAY == 1</span>
    </span><span class='line'>    <span class="kt">method_list_t</span> <span class="o">*</span><span class="n">method_list</span><span class="p">;</span>    <span class="c1">// RW_METHOD_ARRAY == 0</span>
    </span><span class='line'><span class="p">};</span>
    </span>
</td></tr></table></div></figure>

看过我上一篇博文[《Objective-C +load vs +initialize》](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)的朋友可能已经有所察觉了。我们注意到 runtime 对 Category 中方法的处理过程并没有对 +load 方法进行什么特殊地处理。因此，严格意义上讲 Category 中的 +load 方法跟普通方法一样也会对主类中的 +load 方法造成覆盖，只不过 runtime 在自动调用主类和 Category 中的 +load 方法时是直接使用各自方法的指针进行调用的。所以才会使我们觉得主类和 Category 中的 +load 方法好像互不影响一样。因此，当我们手动给主类发送 +load 消息时，调用的一直会是分类中的 +load 方法，you should give it a try yourself 。

## 总结

Category 是 Objective-C 中非常强大的技术之一，使用得当的话可以给我们的开发带来极大的便利。很多著名的开源库或多或少都会通过给系统类添加 Category 的方式提供强大功能，比如 [AFNetworking](https://github.com/AFNetworking/AFNetworking) 、[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) 、 [SDWebImage](https://github.com/rs/SDWebImage) 等。但是凡事有利必有弊，正因为 Category 非常强大，所以一旦误用就很可能会造成非常严重的后果。比如覆写系统类的方法，这是 iOS 开发新手经常会犯的一个错误，不管在任何情况下，切记一定不要这么做，No zuo no die 。

## 参考链接

[https://developer.apple.com/library/ios/documentation/General/Conceptual/DevPedia-CocoaCore/Category.html#//apple_ref/doc/uid/TP40008195-CH5-SW1](https://developer.apple.com/library/ios/documentation/General/Conceptual/DevPedia-CocoaCore/Category.html#//apple_ref/doc/uid/TP40008195-CH5-SW1)
[http://stackoverflow.com/questions/5272451/overriding-methods-using-categories-in-objective-c](http://stackoverflow.com/questions/5272451/overriding-methods-using-categories-in-objective-c)