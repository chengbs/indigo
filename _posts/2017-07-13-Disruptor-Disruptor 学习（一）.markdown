---
title: "Disruptor 学习（一）"
layout: post
date: 2017-07-13 12:44
image: /assets/images/markdown.jpg
headerImage: false
tag:
- Disruptor
- java
star: true
category: blog
author: sun
description: Markdown summary with different options
---

## Summary:

The LMAX Disruptor is a high performance inter-thread messaging library. It grew out of LMAX's research into concurrency, performance and non-blocking algorithms and today forms a core part of their Exchange's infrastructure.

### 使用 Disruptor
- [介绍](#basic-formatting)
- [开始](#headings)
- - [获取开始](#lists)
  - [基础的生产者和消费者](#paragraph-modifiers)
- [基本的调节选项](#基本的调节选项)
- [清理对象](#从环形缓冲区清理对象)
- [高级的方法](#images)
- [Code](#code)

---

## Basic formatting

This note **demonstrates** some of what [Markdown][1] is *capable of doing*.

And that's how to do it.

{% highlight html %}
This note **demonstrates** some of what [Markdown][some/link] is *capable of doing*.
{% endhighlight %}

---

## Headings

There are six levels of headings. They correspond with the six levels of HTML headings. You've probably noticed them already in the page. Each level down uses one more hash character. But we are using just 4 of them.

# Headings can be small

## Headings can be small

### Headings can be small

#### Headings can be small

{% highlight raw %}
# Heading
## Heading
### Heading
#### Heading
{% endhighlight %}

---

## Lists

### Ordered list

1. Item 1
2. A second item
3. Number 3

{% highlight raw %}
1. Item 1
2. A second item
3. Number 3
{% endhighlight %}

### Unordered list

* An item
* Another item
* Yet another item
* And there's more...

{% highlight raw %}
* An item
* Another item
* Yet another item
* And there's more...
{% endhighlight %}

---

## Paragraph modifiers

### Quote

> Here is a quote. What this is should be self explanatory. Quotes are automatically indented when they are used.

{% highlight raw %}
> Here is a quote. What this is should be self explanatory.
{% endhighlight raw %}

---

## 基本的调节选项

使用上述方法将在最广泛的部署场景中发挥作用。然而，如果你能够做出一定的假设，关于硬件和软件的干扰将运行在你可以利用一些优化选项来提高性能的环境。调优有2种主要方法：单对多生产者和可选等待策略。

### 单生产者 VS 多生产者

一个在concurrect系统提高性能的最好方式是按单个生产者的原则，这适用于干扰。如果你在这种情况下，永远只有一个单线程的事件产生的干扰，那么你可以利用这个来获得额外的性能。

{% highlight java %}
// Sticky Header
public class LongEventMain
{
    public static void main(String[] args) throws Exception
    {
        //.....
        // Construct the Disruptor with a SingleProducerSequencer
        Disruptor<LongEvent> disruptor = new Disruptor(
            factory, bufferSize, ProducerType.SINGLE, new BlockingWaitStrategy(), executor);
        //.....
    }
}
{% endhighlight %}

给多少性能优势的指示可以通过这个技术我们可以改变在onetoone性能测试的生产方式实现。测试运行在i5 Sandy Bridge MacBook Air。

### 多生产者

{% highlight raw %}
Starting Disruptor tests
Run 0, Disruptor=12,106,537 ops/sec
Run 1, Disruptor=12,285,012 ops/sec
Run 2, Disruptor=13,596,193 ops/sec
Run 3, Disruptor=13,745,704 ops/sec
Run 4, Disruptor=13,577,732 ops/sec
Run 5, Disruptor=13,020,833 ops/sec
Run 6, Disruptor=12,894,906 ops/sec

{% endhighlight %}

### 单生产者

{% highlight raw %}
Starting Disruptor tests
Run 0, Disruptor=43,572,984 ops/sec
Run 1, Disruptor=68,681,318 ops/sec
Run 2, Disruptor=76,569,678 ops/sec
Run 3, Disruptor=72,780,203 ops/sec
Run 4, Disruptor=75,987,841 ops/sec
Run 5, Disruptor=74,074,074 ops/sec
Run 6, Disruptor=73,746,312 ops/sec
{% endhighlight %}

### 可选择的等待策略

通过使用默认的干扰物等策略是blockingwaitstrategy。内部的blockingwaitstrategy采用典型的锁和条件变量的处理线程唤醒。的blockingwaitstrategy最慢的等待策略，但是是最保守的CPU使用率方面，将给予最一致的行为在最广泛的部署选项。然而，重新部署系统的知识可以允许额外的性能。

#### sleepingwaitstrategy

像blockingwaitstrategy试图保守的sleepingwaitstrategy CPU的使用，通过使用一个简单的忙等待循环，而是使用一个叫LockSupport。parknanos（1）在循环中。在典型的Linux系统将暂停线约60µ但是它有效益，生产线不需要采取任何行动，其他增加适当的计数器和不需要的信号条件可变成本。然而，在生产者和消费者线程之间移动事件的平均延迟时间会更高。它在不需要低延迟的情况下效果最好，但对生成线程的影响是低的。一个常见的用例是异步日志记录。

#### yieldingwaitstrategy

这是一个yieldingwaitstrategy 2等，可以使用低延迟的系统策略，哪里有机会与提高延迟的目的消耗CPU周期。yieldingwaitstrategy会忙的自旋等待序列增量为适当的值。内螺纹的环体。yield()将被允许其他队列的线程运行。当需要非常高的性能和事件处理线程的数量少于逻辑核心的总数时，这是推荐的等待策略，例如，启用了超线程。

#### busyspinwaitstrategy

是的busyspinwaitstrategy最高表演等策略，但提出了最高的约束对部署环境。只有当事件处理程序线程的数量小于盒子上物理内核的数量时，才应该使用这种等待策略。应禁用超线程。

* A named link to [Mark It Down][3].
* Another named link to [Mark It Down](http://markitdown.net/)
* Sometimes you just want a URL like <http://markitdown.net/>.

{% highlight raw %}
* A named link to [MarkItDown][3].
* Another named link to [MarkItDown](http://markitdown.net/)
* Sometimes you just want a URL like <http://markitdown.net/>.
{% endhighlight %}

---

## 从环形缓冲区清理对象

当通过数据通过干扰物，物体寿命比预期的可能。为了避免这种情况发生，可能需要在处理完事件后清除事件。如果有一个单独的事件处理程序，那么在同一个处理程序中清除值就足够了。如果您有一个事件处理程序链，那么您可能需要放置在链结束处的特定处理程序来处理清除对象。

{% highlight java %}
class ObjectEvent<T>
{
    T val;

    void clear()
    {
        val = null;
    }
}

public class ClearingEventHandler<T> implements EventHandler<ObjectEvent<T>>
{
    public void onEvent(ObjectEvent<T> event, long sequence, boolean endOfBatch)
    {
        // Failing to call clear here will result in the 
        // object associated with the event to live until
        // it is overwritten once the ring buffer has wrapped
        // around to the beginning.
        event.clear(); 
    }
}

public static void main(String[] args)
{
    Disruptor<ObjectEvent<String>> disruptor = new Disruptor<>(
        () -> ObjectEvent<String>(), bufferSize, executor);

    disruptor
        .handleEventsWith(new ProcessingEventHandler())
        .then(new ClearingObjectHandler());
}
{% endhighlight %}

---

## Horizontal rule

A horizontal rule is a line that goes across the middle of the page.
It's sometimes handy for breaking things up.

{% highlight raw %}
---
{% endhighlight %}

---

## Images

Markdown can also contain images. I'll need to add something here sometime.

{% highlight raw %}
![Markdowm Image][/image/url]
{% endhighlight %}

![Markdowm Image][6]

*Figure Caption*?

{% highlight raw %}
![Markdowm Image][/image/url]
<figcaption class="caption">Photo by John Doe</figcaption>
{% endhighlight %}

![Markdowm Image][6]
<figcaption class="caption">Photo by John Doe</figcaption>

*Bigger Images*?

{% highlight raw %}
![Markdowm Image][/image/url]{: class="bigger-image" }
{% endhighlight %}

![Markdowm Image][6]{: class="bigger-image" }

---

## Code

A HTML Example:

{% highlight html %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Document</title>
</head>
<body>
    <h1>Just a test</h1>
</body>
</html>
{% endhighlight %}

A CSS Example:

{% highlight css %}
pre {
    padding: 10px;
    font-size: .8em;
    white-space: pre;
}

pre, table {
    width: 100%;
}

code, pre, tt {
    font-family: Monaco, Consolas, Inconsolata, monospace, sans-serif;
    background: rgba(0,0,0,.05);
}
{% endhighlight %}

A JS Example:

{% highlight js %}
// Sticky Header
$(window).scroll(function() {

    if ($(window).scrollTop() > 900 && !$("body").hasClass('show-menu')) {
        $('#hamburguer__open').fadeOut('fast');
    } else if (!$("body").hasClass('show-menu')) {
        $('#hamburguer__open').fadeIn('fast');
    }

});
{% endhighlight %}

[1]: http://daringfireball.net/projects/markdown/
[2]: http://www.fileformat.info/info/unicode/char/2163/index.htm
[3]: http://www.markitdown.net/
[4]: http://daringfireball.net/projects/markdown/basics
[5]: http://daringfireball.net/projects/markdown/syntax
[6]: http://kune.fr/wp-content/uploads/2013/10/ghost-blog.jpg
