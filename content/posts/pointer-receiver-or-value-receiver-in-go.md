---
title: "【译】Go语言的方法Receiver使用Pointer还是Value？"
date: 2019-04-19T08:00:00+08:00
draft: false
author: 严子怡
categories:
tags:
---

## TL;DR

-   方法的receiver通常使用pointer。如果不确定用什么，就用pointer。（参考[Go Code Review Guidelines - Receiver Type](https://github.com/golang/go/wiki/CodeReviewComments#receiver-type)）
-   slices，maps，channels，strings，functions value，interface的内部实现使用了pointer，所以不需要使用指向这些类型的pointer来避免copy。
-   对于很大的struct或者需要修改其内容的struct，使用pointer。否则，使用value。

## 经常会使用pointer的情况

-   一个方法的参数可能有各种不同的情况，但是方法的receiver一般都是pointer。方法通常都会对receiver做修改，并且receiver struct的size一般都比较大。

## 可以不使用pointer的情况

-   根据Go Code Review Guidelines，不推荐使用小的struct的传pointer参数。尽管struct可能不是特别的小，但是，除非你需要对一个struct做in-place的修改，一般都传value参数。

-   value的语意可以避免aliasing（比如，Java里面的对象都是reference，传参数就是aliasing，名字不同但是其实是一个对象）。这样，不会不小心改了一个变量的内容，另一个变量的内容也变了。
-   有的时候，对小的struct传value更加高效，可以避免[缓存未命中](https://en.wikipedia.org/wiki/Locality_of_reference)（cache miss）和heap allocation开销。

-   对于slice，你不需要传pointer就能修改其内容。例如，`io.Reader.Read(p []byte)`会写入小于p的长度的byte到p里。其实，这也是一种"对于小的struct传value"情况的一种特例。由于slice的内部实现（参考[Russ Cox的文章](http://research.swtch.com/godata)），你其实传递的是一个slice header的小struct。
-   对于需要进行切片操作的slice（通过`s[start:end]`修改其start/end/length/capacity），可模仿`append()`函数的设计。`append()`函数接受一个slice再返回一个新的slice。这样的接口避免了aliasing，并告诉调用方新的slice指向array可能是重新分配的。
-   map，channel，string，function value和interface value，和slice类似，内部实现都是pointer或者饱含一些pointer的struct（[interface实现](http://research.swtch.com/interfaces)）。对他们直接传value不会导致对其内容的拷贝。

## 可能需要使用pointer的情况

-   如果你编写了一个function并且接收一个struct的pointer来修改其内容，最好让这个function成为这个struct的method。大家一般都会默认struct的method可能会修改其内容，所以receiver参数使用pointer是最容易让调用方接受的方式。
-   对于需要修改其non-receiver参数内容的函数，最好在godoc或者函数命名上明确表达出来。例如，`reader.WriteTo(writer)`。
-   有时，使用传pointer来允许调用方复用struct，减少内存分配是一个好处。但是，当没有出现明显的内存开销的时候，我们可以采用一些方法来避免这种非常有技巧性的API设计：

    1.  为了避免内存分配开销，Go的逃逸分析（[escape analysis](http://en.wikipedia.org/wiki/Escape_analysis)）可以帮到你。你可以通过使用简单的构造函数、普通的字面值（literal）和有用的zero value（像bytes.Buffer）来最大化逃逸分析的优化效果。
    2.  考虑给对象加一个Reset()方法来让对象回到空值的状态。不在乎内存开销的用户也可以不使用这个方法。
    3.  考虑即实现一个modify-in-place的函数，又实现一个create-from-scratch的函数，以供用户使用。例如，`existingUser.LoadFromJSON(json []byte) error` （从加载内容到一个user对象中）和 `NewUserFromJSON(json []byte) (*User, error)`（创建一个新的user），其中后一个函数可以利用前一个函数实现。调用方可以根据需要来选择任何一种内存分配方式。
    4.  需要循环利用内存的人可以考虑使用`sync.Pool`来帮助你。如果某个特定的内存分配操作制造了大量的内存压力，你很清楚的知道合适分配的对象不再需要了，但是有没有更好的优化方案，`sync.Pool`可以很好的解决这个问题。

## 是否使用slice of pointers（s []*MyStruct)

一般情况下，slice of values够用了，也能减少heap allocation和cache miss。但是也有些情况导致必须使用slice of pointers。

-   构造API只允许你获取到struct的pointer。例如，`NewFoo() *Foo`。
-   slice中struct的生命周期不相同。由于slice内部的array是统一回收的。如果只有其中一部分被引用会导致整个array不会被释放。
-   拷贝某些struct可能会导致一些问题。使用`append()`函数时，array的内容可能会被拷贝到新分配的array上。拷贝大的struct会很慢，同时有些struct（例如`sync.Mutex`）是不允许拷贝的。在slice中插入和删除item也可能引发类似的拷贝。

总的来说，value slices在你不需要移动slice中的内容时，是比较好的选择。或者，你需要确保slice中的value在拷贝到新的内存上时，不会引发什么问题。

## 参考

本文内容整理自 <https://stackoverflow.com/a/23551970>
