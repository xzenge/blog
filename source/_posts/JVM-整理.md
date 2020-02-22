---
title: JVM 整理
date: 2020-02-22 15:03:19
tags: 
    - JVM
    - Java
categories :
    - Technology
---

# JVM相关知识整理，持续更新。。。

## 类的加载

加载->验证->准备->解析->初始化->使用->卸载
<!-- more -->
### 加载流程

- 加载

  代码中使用到这个类的时候，.class字节码文件就会加载这个类到JVM内存中。

- 链接

	- 验证

	  根据JVM规范，校验.class文件。

	- 准备

	  给加载的类分配内存空间，同时给类的static变量分配内存空间，并设置默认值0。
	  clint()

	- 解析

	  把符号引用替换为直接引用。

- 初始化

  为类变量赋值：静态方法，静态变量，静态块。
  初始化一个类的时候，如果发现有父类还没初始化，会先初始化他的父类。

### 类加载器

- 加载器

	- BootStrap ClassLoader

	  加载lib目录下的类库

	- Extension ClassLoader

	  加载lib\ext目录下的类库

	- Application ClassLoader

	  加载ClassPath下的类

	- 自定义加载器

- 双亲委派机制

  先找父亲加载，加载不了再找儿子加载

## 内存结构

### 方法区、元空间、永久代

存放类的定义，常量池等。
方法区垃圾回收：
1.该类的所有实例已经被回收
2.加载该类的ClassLoader已经被回收
3.该类的Class对象没有被任何引用

### 程序技术器

记录当前执行的字节码指令的位置。
线程私有。

### Java虚拟机栈

保存方法信息。
线程私有。

- 栈帧

	- 局部变量表
	- 操作数栈
	- 动态链接
	- 方法返回地址
	- 其他

### Java堆内存

存放类的实例对象。

- 分代

	- 年轻代

	  新生代进入老年代：默认15次

		- Eden区

		  默认新生代80%空间。
		  新建对象产生在Eden区。
		  Eden区满后会发生Minor GC。

		- Survivor0,1区

		  默认年轻代20%。Survivor 0:10%  Survivor 1:10%。
		  默认对象幸存15次后，会进入老年区。

			- 动态年龄判断

			  当一批对象的大小大于Survivor区大小的50%，这批对象将不询问年龄，直接进入老年代。

	- 老年代

		- 进入老年代的规则

			- 年龄超过MaxTenuring
			- 动态年龄判断
			- 超过PretenureSize的大对象
			- Eden区幸存对象超过Survivor区大小

- JVM参数

	- -Xms

	  Java堆内存的大小

	- -Xmx

	  Java堆内存的最大大小

	- -Xmn

	  Java对内存中的新生代大小。
	  老年代=总大小-新生代

	- -XX:PermSize

	  永久代大小

	- -XX:MaxPermSize

	  永久代最大大小

	- -Xss

	  Java栈内存大小

	- -XX:MaxTenuringThreshold

	  对象在年轻代中幸存的最大轮数。
	  默认15，且不能超过15.

	- -XX:PretenureSizeThreshold

	  当对象大小超过该值时，对象直接进入老年代，不会进过年轻代。
	  避免大对象在Survivor中多次复制，减少无谓的操作。

	- -XX:-HandlePromotionFailure

	  判断老年代内存大小是否大于之前每一次Minor GC后进入老年代的对象的平均大小。
	  判断失败进行Full GC，再执行Minor GC。

	- -XX:SurvivorRatio

	  调整Eden区和Survivor区内存比率
	  - XX:SurvivorRatio=8  ： Eden 80%

	- -XX:CMSInitiatingOccupancyFaction

	  设置老年代占用多少比例时触发CMS

	- -XX:+UseCMSCompactAtFullCollection

	  Full GC后STW，整理内存碎片。

	- -XX:CMSFullGCsBeforeCompaction

	  默认0，设置多少次Full GC后进行内存碎片整理。

- 垃圾回收

	- 可达性分析

	  对一个对象层层向上分析，判断是否有一个GC Root。

		- GC Root

			- 本地变量表中的局部变量
			- 方法区内的静态变量

	- java 引用类型

		- 强引用

		  GC不会回收强引用

		- 软引用

		  内存空间足够时，GC不会回收软引用

		- 弱引用

		  GC时必定回收弱引用

		- 虚引用

		  继承Object的finalize()方法，在该对象被回收的时候通知自己。

	- 垃圾回收算法

		- 年轻代

			- 复制

			  在新生代幸存者0,1区使用。将标记为可用的内存复制到另外一片连续的内存区域中，并删除当前内存区域中的全部数据。

				- 优点

				  不会产生内存碎片

				- 缺点

				  增加内存的消耗

		- 老年代

			- 标记整理

			  标记存活对象，删除死亡对象，整理内存碎片。

				- 优点

				  不会产生内存碎片。内存利用率高。

				- 缺点

				  比起标记清楚，执行效率慢。

	- JVM垃圾回收器

		- 年轻代

			- ParNew

			  多线程，复制算法。
			  开启：-XX:+UseParNewGC
			  调整回收线程数：-XX:ParallelGCThreads

		- 老年代

			- CMS

			  标记清理算法

				- 运行机制

					- 初始标记

					  STW。
					  标记所有GC Roots直接引用的对象，被直接引用的对象所引用的对象不会被标记。

					- 并发标记

					  寻找被GC Roots间接引用的对象。

					- 重新标记

					  STW。
					  重新标记并发标记时产生的新对象。

					- 并发清除

					  清理标记的对象。

				- 缺点

				  默认启动垃圾回收线程：（CPU核数 +3）/4。
				  本身会消耗CPU资源。
				  针对浮动垃圾，初始标记后产生的垃圾没法回收。
				  CMS回收期间，新生代进入老年代对象大于可用空间时，会发生Concurrent Mode Failure ，并自动使用Serial Old回收器替代CMS工作。

### 本地方法栈

存放native方法。

### 堆外内存空间

通过调用操作系统接口，直接分配对外内存空间。
NIO->allocateDirect(DirectByteBuffer)
