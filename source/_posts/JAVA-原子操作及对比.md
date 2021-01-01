---
title: JAVA 原子操作及对比
date: 2020-12-30 20:58:10
tags: 
    - JVM
    - Java
categories :
    - Technology
---

原子操作是并发同步的一种运用场景，Java提供了多种原子操作的工具类。这里我们总结下各个工具的使用方法及其优缺点。
<!-- more -->

## 1.AtomicXXXX类

AtomicXXXX全家桶提供了各种基础类型的包装类型的原子对象。通过声明一个Atomic类型的对象对变量进行线程安全的操作。

### 源码分析

Atomic对象内部持有Unsafe对象，并通过Unsafe对象获取操作数所在的Atomic对象在内存中的偏移量。（可以理解为在C中持有改变量的指针）

``` java
    private static final Unsafe U = Unsafe.getUnsafe();
    private static final long VALUE
        = U.objectFieldOffset(AtomicInteger.class, "value");
```

对变量的操作采用Unsafe提供的CAS接口，通过循环比较最终保证变量的更新成功

``` java
    public final int getAndSet(int newValue) {
        return U.getAndSetInt(this, VALUE, newValue);
    }

    public final int getAndSetInt(Object o, long offset, int newValue) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!weakCompareAndSetInt(o, offset, v, newValue));
        return v;
    }
```

linux_X86下
可以看到JVM内部提供了Atomic类下的cmpxchg的方法

```C++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```

cmpxchg的实现也比较巧妙，首先会判断运行环境为多进程还是单进程。单进程下不需要对内存加锁，对变量直接进行操作。
而多进程下使用了cmpxchgl指令保证了变量的原子操作。
这里我们可以看到JVM内部的实现采用了内嵌汇编的方式，同时最大的优化的单进程下的执行效率。不采用调用内核提供的同步接口，减少了内核态和用户态切换所导致的成本。

```C++
#define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "

inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```

### 优点

1. 使用简单，语义清晰；
2. 对各种类型支持较好。

### 缺点

1. 可移植性差，对非Atomic变量的改造成本较高;
2. 内存消耗较大，除了变量本身，实例变量需要占用额外的内存;
3. 高并发场景下由于lock指令需要锁缓存，导致性能下降。


## 2.XXXXAdder

LongAdder、DoubleAdder只提供了Long和Double类型的+1、-1操作。

### 源码分析

LongAdder的思想是，多核、多线程场景下。每个线程独立计算，最终合计所有线程的累计值。
在单线程场景下，通过cas保证base值的安全累加。
一旦cas失败，或者之前产多多核的场景时，通过longAccumulate方法分别合计不同线程下的值。

```java
    public void increment() {
        add(1L);
    }

    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }
```

longAccumulate作为LongAddre中的核心方法，主要是为每个线程维护了Cell类型的数组。具体实现我们下次另起一篇重讲。

### 优点

1. 使用简单，语义清晰；
2. 高并发场景下，性能比Atomic类强。

### 缺点

1. 对类型的支持有限，现只支持Long和Double类型；
2. 多并发场景下可能会导致统计出现误差。

## 3.XXXXFieldUpdater

XXXXFieldUpdater工具类通过反射获取对象内的变量，从而对对象进行原子操作。

### 源码分析

使用FieldUpdater时，方法通过对类的反射获取Class、Field的相应信息。
最后还是通过Unsafe方法获取变量在结构体中的偏移量，从而进行操作。
```java
        AtomicIntegerFieldUpdaterImpl(final Class<T> tclass,
                                      final String fieldName,
                                      final Class<?> caller) {
            final Field field;
            final int modifiers;
            try {
                field = AccessController.doPrivileged(
                    new PrivilegedExceptionAction<Field>() {
                        public Field run() throws NoSuchFieldException {
                            return tclass.getDeclaredField(fieldName);
                        }
                    });
                modifiers = field.getModifiers();
                sun.reflect.misc.ReflectUtil.ensureMemberAccess(
                    caller, tclass, null, modifiers);
                ClassLoader cl = tclass.getClassLoader();
                ClassLoader ccl = caller.getClassLoader();
                if ((ccl != null) && (ccl != cl) &&
                    ((cl == null) || !isAncestor(cl, ccl))) {
                    sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
                }
            } catch (PrivilegedActionException pae) {
                throw new RuntimeException(pae.getException());
            } catch (Exception ex) {
                throw new RuntimeException(ex);
            }

            if (field.getType() != int.class)
                throw new IllegalArgumentException("Must be integer type");

            if (!Modifier.isVolatile(modifiers))
                throw new IllegalArgumentException("Must be volatile type");

            // Access to protected field members is restricted to receivers only
            // of the accessing class, or one of its subclasses, and the
            // accessing class must in turn be a subclass (or package sibling)
            // of the protected member's defining class.
            // If the updater refers to a protected field of a declaring class
            // outside the current package, the receiver argument will be
            // narrowed to the type of the accessing class.
            this.cclass = (Modifier.isProtected(modifiers) &&
                           tclass.isAssignableFrom(caller) &&
                           !isSamePackage(tclass, caller))
                          ? caller : tclass;
            this.tclass = tclass;
            this.offset = U.objectFieldOffset(field);
        }
```

而提供的原子操作和Atomic类原理一样

```java
    public int incrementAndGet(T obj) {
        int prev, next;
        do {
            prev = get(obj);
            next = prev + 1;
        } while (!compareAndSet(obj, prev, next));
        return next;
    }
```

### 优点

1. 不需要对类，或者成员变量进行修改就能执行原子操作，对代码移植友好。

### 缺点

1. 和Atomic一样，会存在高并发下的性能问题。主要还是由于对缓存上锁导致的。
2. 变量只支持int类型，且必须为volatile修饰的。

## 4.VarHandle

变量句柄是JDK1.9提供的新特性。在替代Atomic和Unsafe方法的同时，提供更加细粒度的内存屏障API。

### 源码分析

可以看到，通过方法句柄获取到变量句柄的对象，并对对象进行原子操作。

```java
    public static void main(String[] args) {
        VarHandle varHandle;
        try {
            varHandle = MethodHandles.lookup().
                    in(Foo.class).
                    findVarHandle(Foo.class, "i", int.class);
            Foo f = new Foo();
            System.out.println("before add: "+f.i);
            for(int i = 0;i<10;i++){
                int o = (int) varHandle.getAndAdd(f, 1);
            }

            System.out.println("after add: "+f.i);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```

方法句柄和变量句柄的实现较为复杂，我们下次另开一篇详细介绍JVM如何实现MethodHandle和VarHandle，并且详细分析原子操作和内存屏障实现的细节。

### 优点

1. 不需要对类，或者成员变量进行修改就能执行原子操作，对代码移植友好；
2. 对各种类型支持完善；
3. 语义粒度更加细，提供更加细致、安全的内存操作方式。

### 缺点

1. 新东西吧。。。学习成本比较高。


## 总结

以上，我们大致总结了Java在原子操作上的相关处理。原子操作是多线程同步的一种实际运用，显示、或非显示的对临界区进行并发上的保护。关于并发，JVM做了相当多的优化，我们有时间慢慢抽丝剥茧，好好学习下JVM在并发下的各种处理。