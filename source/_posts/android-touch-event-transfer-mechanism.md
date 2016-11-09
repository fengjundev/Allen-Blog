---
title: Android事件传递机制
date: 2015-08-30 21:02:01
tags:
- android
- 事件传递
categories: 
- android
---

![](http://ww2.sinaimg.cn/large/72f96cbagw1f9alsuwmd3j20fa0fagmg.jpg)

<!-- more -->

> 出处： *[Allen's Zone](http://allenfeng.com/)*
> 作者： *Allen Feng*

## 基本知识
1. 每个事件都被封装成为`MotionEvent`，包含了Touch的位置，点按的数量(手指的数量)，时间点等信息
2. 包含了用户的当前动作，主要有以下几种类型：
	- `ACTION_DOWN`
	- `ACTION_UP`
	- `ACTION_MOVE`
	- `ACTION_POINTER_DOWN`
	- `ACTION_POINTER_UP`
	- `ACTION_CANCEL`
3. Android事件响应涉及的主要方法有以下几个：
	- `dispatchTouchEvent()` 用于事件的分发，所有的事件都要通过此方法进行分发，决定是自己对事件进行消费还是交由子View处理
	- `onTouchEvent()`主要用于事件的处理，返回true表示消费当前事件
	- `onInterceptTouchEvent`是`ViewGroup`中独有的方法，若返回`true`表示拦截当前事件，交由自己的`onTouchEvent()`进行处理，返回`false`表示不拦截

## 流程

事件由Activity的`dispatchTouchEvent()`开始，将事件传递给当前Activity的根View，事件开始自上而下进行传递，直至被消费。

事件传递至`View`的`dispatchTouchEvent()`时， 首先会判断`OnTouchListener`是否存在，倘若存在者则由`OnTouchListener`进行消费，执行`onTouch()`，。若`onTouch()`未对事件进行消费，事件将继续交由`onTouchEvent`处理，通过查看源码可知，View的`onClick`事件是在`onTouchEvent`的`ACTION_UP`中触发的，因此，onTouch事件优先于`onClick`事件。


事件传递至`ViewGroup`时，调用`dispatchTouchEvent()`进行处理:
1. 检查送否应该对事件进行拦截:`onInterceptTouchEvent()`，若为true，跳过2步骤
2. 按照子View添加顺序的**逆序**，将事件依次分发给子View，若事件被前一个View消费了，将不再继续分发
3. 如果2中没有子View对事件进行消费或者子View的数量为零，事件的处理流程和View的处理流程一致

若事件在自上而下的传递过程中一直没有被消费，而且最底层的子View也没有对其进行消费，事件会**反向**向上传递，此时，父`ViewGroup`可以对事件进行消费，若仍然没有被消费的话，最后会回到Activity的`onTouchEvent`。


![事件传递流程](http://ww1.sinaimg.cn/large/72f96cbagw1f9aljfns45j20nv15on04.jpg)


## 总结
1. 事件总是由`Activity`的`dispatchTouchEvent`开始向下进行传递
2. 父View(`ViewGroup`)可将事件传递给子View，`ViewGroup`可将事件拦截，事件将停止向下传递(`onInterceptTouchEvent`中返回`true`)
3. 事件自上而下传递，直到被消费
4. 如果View或者ViewGroup没有消费`ACTION_DOWN`，其他事件也不会被传递进来
5. 所有未被消费的事件将会回传至Activity的`onTouchEvent()`(事件的最后一环)
6. `OnTouchListener` 优先于`onTouchEvent()`对事件进行消费


## 技巧

对于底层的View来说，可以使用`getParent().requestDisallowInterceptTouchEvent(true)`来阻止父View拦截Touch事件。实践过程中，常用于解决嵌套滑动事件冲突的问题：
如scrollView+listview方式，此时可重写onTouchEvent判断滑动位置。  
- 当子View需要滑动时，在子view中使用`requestDisallowInterceptTouchEvent(true);`阻止父View拦截消费事件   
- 当子view滑动到边界才`requestDisallowInterceptTouchEvent(false);`

