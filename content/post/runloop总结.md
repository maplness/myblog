---
title: "Runloop总结"
date: 2020-08-16T08:12:28+08:00
tags: ["面试","底层原理"]
categories: ["iOS"]
draft: true
---

# Runloop总结

#### 1. RunLoop是什么

RunLoop实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行上面EventLoop的逻辑。线程执行了这个函数之后，就会一直处于这个函数内部“接收消息-》等待->处理“的循环中，直到循环结束（传入quit消息），函数返回。

### 2. RunLoop 与线程的关系

首先，iOS开发中能遇到两个线程对象: pthread_t 和 NSThread.线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。



### 3. RunLoop 对外的接口

一个RunLoop包含若干个Mode，每个Mode又包含若干个Source/Timer/Observer。每次调用RunLoop的主函数时，只能指定其中一个Mode，这个Mode成为CurrentMode。如果需要切换Mode，只能退出Loop，再重新制定一个Mode进入。这样做是为了分隔开不同组的Source/Timer/Observer,让其互不影响。

### 4. RunLoop 有几种Mode，RunLoop设置Mode作用是什么

* RunLoop 的运行模式共有5种，RunLoop只会运行在一个模式下，要切换模式，就要暂停当前的模式，重写启动一个运行模式
* kCFRunLoopDefaultMode,App的默认模式，通常主线程是在这个模式下运行的。

* UITrackingRunLoopMode,跟踪用户交互事件,(用于Scrollview追踪触摸滑动，保证界面滑动时不受其他Mode影响)
* kCFRunLoopCommonModes,伪模式，不是一种真正的运行模式
* UIInitializationRunLoopMode: 在刚启动App时进入的第一个Mode，启动完成后就不再使用
* GSEventReceiveRunLoopMode: 接受系统内部事件，通常用不到



设置Mode的作用： 指定事件在运行循环中的优先级，线程的运行需要不用的模式，去响应不同的事件，去处理不同的情景模式。