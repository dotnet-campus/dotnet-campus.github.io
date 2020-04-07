---
layout: post
title: "2020-3-8-MVC、MVP、MVVM模式演变简析"
date: 2020-3-8 13:54:32
date_modified: 2020-3-8 13:54:37
categories: 架构
tags: 架构
description:
---

今天和大家简单介绍下GUI设计中MVC、MVP 以及 MVVM 架构模式的演变。

由于MVC等相关模式的定义，实现都各有不同，加之作者认识水平有限，如有纰漏或不足，万望指正。

-----

## 从GUI开始

MVC、MVP 以及 MVVM都是GUI设计中的架构模式。

那我们就先从GUI开始，思考下这些模式的本质目的。

什么是GUI？wiki的定义是用于操作计算机的图形界面。

或者使用英文单词（interface）接口能够更加清晰理解它的本质。

GUI就是提供了一种图形化的方式便于用户操作计算机。

![image-20200308142131368](../media/image-20200308142131368.png)

当然这里的操作计算机不仅仅是狭义上的操作计算机硬件，还可以是操作存储，数据库，运行时程序的内存模型等等。

其实从这里我们就可以提取出两个概念。界面（View）和模型（Model）。

GUI程序就是为了解决用户通过View处理Model的需求。

![image-20200308141659491](../media/image-20200308141659491.png)

## 第一个设计——“MV”模式

既然我们刚刚分析了GUI程序中天然存在View和Model的两个概念，那我们在进行设计时，自然会想到的第一个模型就是上一个小节提出的View-Model模型。

用户通过View上的操作更新Model的数据。Model的数据改变后，更新View的显示状态。

很好，我们有了第一个GUI的设计结构

![image-20200308141659491](../media/image-20200308141659491.png)

我们已经有了一个“MV”模式，但是它真的足够好么？

模式的目的是为了提高复用性，减少开发工作。

我们可以分析下GUI中，哪些是变化的，哪些是不变的？然后把不变的部分抽出。当然我们在处理其他软件设计时，也可以采用类似方式操作。

OK，大部分情况下，Model是不变的，而View是多变的。比如不同主题配色，根据用户操作状态，显示部分数据等等，都会改变View，或者有些软件可以使用一个Model对应多个View

![image-20200308144327452](../media/image-20200308144327452.png)

这样我们每次变更都需要重写整个View。

## MVC模式——复用

我们再看一下“MV”模式中，各个部分的职责。

Model是完全被动的，他不知道外面世界的存在，只需要通过观察者模式，再自身数据变更时向外发出通知。

因此可以适应于任意种类，数量的View。

而View，承担了显示Model的数据，以及接收用户输入，并且更新显示状态以及Model数据的功能。

所以"MV"模式中的依赖关系是这样的。

![image-20200308151905840](../media/image-20200308151905840.png)

那么这里面有没有什么是相对来说不变的呢？

有。`接收用户输入，并且更新显示状态以及Model数据`就是一个相对不变的功能。

试想一个社交类应用。用户可以在注册界面，个人空间等多个地方（View）更改自己的用户名（操作更新Model数据）。但是这类操作是通用逻辑，没有必要每个View都进行实现。

此外例如点击跳转，页面切换等业务，如果写在View中，也会造成View之间的相互耦合，不利于复用。

所以我们可以把这部分业务逻辑抽取到一个单独的模块叫做Controller。

这样我们就更新了三者的职责：

- Model：存储数据，在变更时发出通知
- View：根据Model的数据进行显示
- Controller：接收用户输入，并操作View，以及更新Model

于是我们得到了一个新的模式——MVC模式，它的依赖关系如下。

相比于“MV”模式，一部分通用业务逻辑移动到了Controller，View变得更加轻量，易于扩展更新。

![image-20200308152313340](../media/image-20200308152313340.png)

当然，MVC在各个端的定义和实现也没有统一。比如如果没有View切换，Controller也不一定要依赖View。

例如Martin Fowler在这篇[GUI Architectures](https://martinfowler.com/eaaDev/uiArchs.html)文章中介绍的MVC就没有Controller和View的关键依赖。

![image-20200308154049617](../media/image-20200308154049617.png)

各个框架中MVC的实现方式，可以参考[浅谈 MVC、MVP 和 MVVM 架构模式](https://draveness.me/mvx)，其中有详细介绍，不再赘述。

## MVP——可测试

可测试性是软件设计的一个重要的非业务需求。

我们看下MVC的可测试性。

我们都知道UI测试是最困难的测试之一。

由于View自身处理了Model数据渲染的逻辑，而这部分处在View中的逻辑就变得不容易测试了。

这种情况下，最简单的解决方案就是将这部分逻辑移动至容易测试的部分——我们的Controller。

这里因为Controller承担了一部分显示的逻辑，所以为了区分，就将其改名为Presenter。

这样我们就更新了三者的职责：

- Model：存储数据，在变更时发出通知
- View：根据Presenter的指导显示
- Presenter：接收用户输入，决定View显示，以及更新Model

这里根据显示逻辑迁移的多少可以分为Supervising Controller和 Passive View模式

Supervising Controller承担了大部分的复杂逻辑，如下图所示。



![image-20200308161958492](../media/image-20200308161958492.png)

而Passive View则是将全部逻辑都交给了Presenter处理。

也正是View没有了渲染逻辑，所以他不需要从Model中拿数据，Model的数据更新只需要通知Presenter。

由Presenter处理后操作View渲染。

这样所有逻辑都可测，而且View同Model完成了解构。

![image-20200308162442260](../media/image-20200308162442260.png)

MVP解决了可测试性问题，但是也造成了Presenter的进一步庞大，而且和业务逻辑，甚至是显示逻辑也更加耦合。

## MVVM——不同层次的模型抽象

MVVM是MVC的另一个变种，也是目前广泛使用的一种GUI模型。我们常见的WPF框架就是建立在MVVM模式的基础之上。

试想下有这样一个问题，我们要显示用户的博客空间。

我们期望在界面上让用户的昵称显示宋体、加粗、红色。

那么这个宋体、加粗、红色的信息应该放在那里呢？

Model层显然不行，这里的数据属于UI，不属于领域模型，加入Model层只会使Model更加庞大，且冗余。

View层？UI数据随View层的确是个好主意，但是如果这个数据会变化呢？比如当用户点击头像时，用户昵称要加粗；点击简介时，用户昵称不加粗。（大家就当这个设计师没有审美好了，一时间想不出更好的场景。。）

更好的方式仍然是显示和数据分类。

所以我们引入一个新的概念，ViewModel，来赋予Controller存储UI相关数据的能力。

这样我们就更新了三者的职责：

- Model：存储数据，在变更时发出通知
- View：根据ViewModel的数据进行显示
- ViewModel：接收用户输入，更改Model数据，并更加Model的更新，更新自身数据并通知View

所以这三种的依赖变化如下。可以看到我们利用MVVM完成了根据可变性分离的单向依赖模型。

![image-20200308165301003](../media/image-20200308165301003.png)

对于WPF框架，还可以使用DataBinding和Command进行进一步解耦。

![image-20200308165905308](../media/image-20200308165905308.png)

## 后记

我们分析总结了GUI模式一步步的演变过程，以及各个模式解决的重点问题。

当然这里并不是说某个模式能够完全胜过另一个模式，而是根据具体场景选择合适的架构。

例如web应用这边使用MVVM就不是很合适，而更适合MVC。（Model和Controller在服务端，而View在用户端）

能够理解和明白各个架构模式的优劣才能够在使用时得心应手。



---

参考文档：

-  [浅谈 MVC、MVP 和 MVVM 架构模式](https://draveness.me/mvx)
-  [MVC - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/MVC)
-  [Applications Programming in Smalltalk-80 (TM): How to use Model-View-Controller (MVC)](http://www.dgp.toronto.edu/~dwigdor/teaching/csc2524/2012_F/papers/mvc.pdf)
-  [Model–view–presenter - Wikipedia](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter)
-  [Building MVP apps: MVP Part I](http://www.gwtproject.org/articles/mvp-architecture.html)
-  [Presentation Model](https://www.martinfowler.com/eaaDev/PresentationModel.html)
-  [GUI Architectures](https://martinfowler.com/eaaDev/uiArchs.html)
-  [Passive View](https://www.martinfowler.com/eaaDev/PassiveScreen.html)
-  [Supervising Controller](https://www.martinfowler.com/eaaDev/SupervisingPresenter.html)
-  [Model–view–presenter - Wikipedia](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter#cite_note-7)
-  [Model–view–viewmodel - Wikipedia](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)
-  [Graphical user interface - Wikipedia](https://en.wikipedia.org/wiki/Graphical_user_interface#History)


