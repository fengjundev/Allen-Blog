---
title: Android Binder
date: 2017-02-23 13:15:15
tags:
- android
- binder
---

[为何选择Binder](https://www.zhihu.com/question/39440766/answer/89210950)

什么是Binder
1. 直观来说，Binder是Android中的一个类，它继承了IBinder接口
2. 从IPC角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux中没有
3. 从Android Framework角度来说，Binder是ServiceManager连接各种Manager（ActivityManager、WindowManager，etc）和相应ManagerService的桥梁
4. 从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当你bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务