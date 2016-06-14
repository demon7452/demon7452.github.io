---
layout: post
title: Java学习笔记-Object类
category: 技术
tags: java
keywords: Java学习笔记-Object类
description: Java学习笔记-Object类
---
# Java学习笔记-Object类
*Note：参阅书籍《Core Java，Volume I：Fundamentals》*

## Object类的说明
- Object类是Java中所有类的最终祖先，在Java中每个类都是由它扩展而来；
- 如果没有明确的指出超类，Object就被认为是这个类的超类。class A extends Object是不必要的；
- 可以使用Object类型的变量引用任何类型的对象，Object obj = new A();
- 在Java中，只有**基本类型(primitive types)**不是对象，例如：数值、字符和布尔类型的值都不是对象。所有的数组类型，不管是对象数组还是基本类型的数组都扩展于Object类。

## Object类中的方法
- **boolean equals(Object otherObject)**<br>比较两个对象是否相等，如果两个对象指向同一块存储区域，方法返回true;否则返回false。在自定义的类中，应该覆盖这个方法。
- **int hashCode()**<br>返回对象的散列码。散列码可以是任意的整数，包括正数或负数。两个相等的对象要求返回相等的散列码，但是散列码相同不一定是相等的对象。
- **Class getClass()**<br>返回包含对象信息的类对象。
- **String toString()**<br>返回描述该对象值的字符串。在自定义的类中，应该覆盖这个方法。
- **Object clone()**<br>创建一个对象的副本。JRE将为新实例分配存储空间，并将当前的对象复制到这块存储区域中。
- **void notifyAll()**<br>解除那些在该对象上调用wait方法的线程的阻塞状态。该方法只能在同步方法或同步块内部调用。如果当前线程不是对象锁的持有者，该方法抛出一个IllegalMonitorStateException异常。
- **void notify()**<br>随机选择一个在该对象上调用wait方法的线程，解除其阻塞状态。该方法只能在一个同步方法或同步块这中调用。如果当前线程不是对象锁的持有者，该方法抛出一个IlegalMonitorStateException异常。
- **void wait()**<br>导致线程进入等待状态直到它被通知。该方法只能在一个同步方法中调用。如果当前线程不是对象锁的持有者，该方法抛出一个IllegalMonitorStateException异常。
- **void wait(long millis) / wait(long millis, int nanos)**<br>导致线程进入等待状态直到它被通知或者经过指定的时间。millis 毫秒数； nanos 纳秒数。