---
title: 提升Android应用性能
date: 2017-01-11 15:25:10
tags: android
---

# 为什么需要关注性能
糟糕的用户体验 --- 打嗝
页面需要很长时间才能启动
应用不响应用户输入，ANR
界面卡顿现象

失望 ---> 不耐烦 ---> 卸载

---

## 不要阻塞UI线程
- Runnable

```java
new Thread(new Runnable() {
	
	@Override
	public void run() {
		// do some heavy work
	}

}).start();
```

- AsyncTask


- Thread

- HandlerThread


- Future


- ExecutorService



# Part 1

## Performance

### Framework APIs

### UI performance

### I/O performance

### Scrolling performance

## Memory 

### How Android memory management works

#### OOM Killer and low memory killer

#### Memory measurement of an application

### Indentifying memory leaks

### Best practices

---

# Part 2

Android Performance Pattern

---

# Part 3

## Java代码优化

## 使用C/C++改进性能

## 高效使用内存

## 多线程与同步

## 性能评测与剖析

## 电量优化

## 图形优化

