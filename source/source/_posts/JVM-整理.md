---
title: JVM 整理
date: 2020-02-22 15:03:19
tags: 
    - JVM
    - Java
categories :
    - Technology
---

# JVM 相关知识整理
1. 内存分配
2. 垃圾回收器的参数配置

## 类的加载
<!-- more -->
加载->验证->准备->解析->初始化->使用->卸载

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
matespace满后会触发Full GC。
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

	- -XX:MetaspaceSize

	  元空间大小

	- -XX:MaxMetaspaceSize

	  元空间最大大小

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

	- -XX:+PrintGCDetils

	  打印详细的GC日志。

	- -XX:+PrintGCTimeStamps

	  打印GC发生的时间。

	- -Xloggc:gc.log

	  将CG日志输入到文件。

	- -XX:TraceClassLoading

	  追踪加载的类

	- -XX:TraceClassUnloading

	  追踪卸载的类

	- -XX:+HeapDumpOnOutOfMemoryError  

	  OOM时自动dump快照

	- -XX:HeapDumpPath=/usr/local/app/oom

	  将快照存放至指定目录

	- CMS

		- -XX:CMSInitiatingOccupancyFaction

		  设置老年代占用多少比例时触发CMS

		- -XX:+UseCMSCompactAtFullCollection

		  Full GC后STW，整理内存碎片。

		- -XX:CMSFullGCsBeforeCompaction

		  默认0，设置多少次Full GC后进行内存碎片整理。

		- -XX:+CMSParallelInitialMarkEnabled

		  初始标记阶段开启并发执行

	- G1

		- -XX:+UseG1GC

		  指定G1垃圾回收器。

		- -XX:G1HeapRegionSize

		  手动指定G1的Region的大小。

		- -XX:G1NewSizePercent

		  调整新生代在堆中的占比。
		  默认5%。

		- -XX:G1MaxNewSizePercent

		  新生代占比堆空间的最大大小。

		- -XX:MaxGCPauseMills

		  默认200ms。
		  设置G1垃圾回收器GC时的最大停顿时间。

		- -XX:InitiatingHeapOccupancyPercent

		  默认45%。
		  老年代占据的堆内存超多该值，将会进行新老生代的混合回收。

		- -XX:G1MixedGCCountTarget

		  一次G1混合回收过程中，最后阶段混合回收阶段分几次完成。
		  默认8。

		- -XX:G1HeapWastePercent

		  默认值5%。
		  混合回收阶段中，空间Region占比堆空间超多该值，会停止混合回收。

		- -XX:G1MixedGCLiveThresholdPercent

		  默认值85%。
		  一个Region中的存活对象低于该值时才会被回收。

		- -XX:+CMSScavengeBeforeRemark

		  重新标记前，尽量执行一次Young GC。
		  提前回收年轻代没有引用的对象，减少重新标记扫描的对象数，提高效率。

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

				- 回收过程

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

		- G1

		  基于复制算法。
		  把Java堆内存划分为多个大小相等的Region。
		  可以设置STW的时间，追踪每个Region里可回收对象的预估时间。
		  G1中的分代是逻辑上的分代，同一个Region会随着分配的内存对象不同，而属于不同分代。
		  每个Region的大小：堆大小/2048。
		  新生代的Region同时分Eden和Survivor，可通过-XX:SurvivorRatio调整占比。
		  适合大内存的机器，G1可以设置停顿时间，来确保对服务不会有较大的影响。

			- 大对象

			  超多Region大小50%即为大对象。
			  大对象可能横跨多个Region。
			  空的Region都可以存放大对象。
			  新老生代GC时都会回收大对象。

			- 回收过程

				- 初始标记

				  STW。
				  标记所有GC Roots直接引用的对象，被直接引用的对象所引用的对象不会被标记。

				- 并发标记

				  进行GC Roots的追踪，追踪所有存活对象。
				  同时JVM会记录该阶段中新建或失去引用的对象。

				- 最终标记

				  STW。
				  通过多并发标记中JVM堆新增或失去引用的对象，进行重新标记。

				- 混合回收

				  分析Region大小，执行效率。
				  STW。
				  在指定时间内进行回收。

			- 问题

			  回收中发现Region空间不够时，将会SWT，并单线程进行标记、清除、整理，效率很低。

	- CG触发时间

		- Minor GC

		  Eden区满时采用复制算法。

		- Old GC

		  1.老年代连续可用空间 < 新生代历次GC后升入老年代对象内存和的平均大小。
		  2.老年代不够存放，Minor GC后升入老年代的对象时。
		  3.老年代内存使用率超过92%时（可设置）。

		- 永久代GC

		  永久代不够使产生Full GC。

### 本地方法栈

存放native方法。

### 堆外内存空间

通过调用操作系统接口，直接分配对外内存空间。
NIO->allocateDirect(DirectByteBuffer)

## 工具

### jstat

通过jps查找java进程，再用jstat -gc PID查看GC状况。

- 指标

	- S0C

	  From Survivor区大小

	- S1C

	  To Survivor区大小

	- S0U

	  From Survivor区当前使用大小

	- S1U

	  To Survivor区当前使用大小

	- EC

	  Eden区大小

	- EU

	  Eden区当前使用的大小

	- OC

	  老年代的大小

	- OU

	  老年代当前使用的大小

	- MC

	  方法区（永久代、元空间）的大小

	- MU

	  方法区（永久代、元空间）当前使用的大小

	- YGC

	  系统迄今为止Young GC的次数

	- YGCT

	  Young GC耗时

	- FGC

	  系统迄今为止Full GC的次数

	- FGCT

	  Full GC的耗时

	- GCT

	  所有GC的总耗时

- 命令

	- jstat -gc PID

	  查看GC状况

	- jstat -gccapacity PID

	  堆内存分析

	- jstat -gcnew PID

	  新生代分析

	- jstat -gcnewcapacity PID

	  年轻代内存分析

	- jstat -gcold PID

	  老年代GC分析

	- jstat -gcoldcapacity PID

	  老年代内存分析

	- jstat -gcmetacapacity PID

	  元数据内存分析

### jmap

了解运行时的内存区域

- 命令

	- jmap -heap PID

	  打印堆内存相关参数设置。
	  当前堆内各个区的基本情况。

	- jmap -histo PID

	  按照对象在内存中占用大小排序，降序。
	  可以了解哪个对象在堆中占用了大量的空间。

	- jmap -dump:live,format=b,file=dump.hprof PID

	  生成内存快照并输出到文件。

### jhat

分析堆内存快照，可以使用工具内置的浏览器查看。

- 命令

	- jhat dump.hprof -port 7000

	  分析快照文件dump.hprof ，访问端口7000。

### MAT

内存分析工具

## JVM优化总结

### 调整新生代的大小

尽量让每次Young GC后的存活对象⼩于Survivor区域的50%，都留存在年轻代⾥。尽量别让对象进 ⼊⽼年代。尽量减少Full GC的频率，避免频繁Full GC对JVM性能的影响

### 调整老年代碎片回收的频率

CMS一次Full GC会产生内存随便，调整碎片整理的触发轮数，可以有效的降低Full  GC产生的碎片从而降低Full GC的频次。

### JVM参数模板

-Xms4096M -Xmx4096M -Xmn3072M -Xss1M -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -XX:+UseParNewGC - XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFaction=92 -XX:+UseCMSCompactAtFullCollection - XX:CMSFullGCsBeforeCompaction=0 -XX:+CMSParallelInitialMarkEnabled -XX:+CMSScavengeBeforeRemark - XX:+DisableExplicitGC -XX:+PrintGCDetails -Xloggc:gc.log -XX:+HeapDumpOnOutOfMemoryError - XX:HeapDumpPath=/usr/local/app/oom

### -XX:SoftRefLRUPolicyMSPerMB

在有大量反射的情况下建议增大该值，保证软引用不会被马上回收

### -XX:+DisableExplicitGC

禁止显示执行GC
性质System.gc()

### 使用缓存淘汰机制

避免本地缓存无限增大，使得老年代常驻对象无法回收，从而导致的内存溢出和频繁的Full GC。

### 频繁Full GC的可能原因

1.系统压力大，频繁Young GC后存活对象大于Survivor区，导致直接进入老年代。
2.频繁产生大对象。
3.对象始终不释放，导致内存泄漏。
4.方法区频繁有类反射加载。
5.错误使用System.gc()。

*XMind: ZEN - Trial Version*