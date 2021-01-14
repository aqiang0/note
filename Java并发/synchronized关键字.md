# synchronized关键字

### 1 概述

在多线程并发编程中synchronized一直是元老级角色, 很多人都会称呼它为重量级锁. 但是, 随着Java SE 1.6对synchronized进行了各种优化之后, 有些情况下它就并不那么重了. 本文详细介绍Java SE 1.6中为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁, 以及锁的存储结构和升级过程.

### 2 实现同步的基础

Java中的每个对象都可以作为锁. 具体变现为以下3中形式.

1. 对于普通同步方法, 锁是当前实例对象.
2. 对于静态同步方法, 锁是当前类的Class对象.
3. 对于同步方法块, 锁是synchronized括号里配置的对象.

一个线程试图访问同步代码块时, 必须获取锁. 在退出或者抛出异常时, 必须释放锁.

### 3 实现方式

JVM基于进入和退出Monitor对象来实现方法同步和代码块同步, 但是两者的实现细节不一样.

1. 代码块同步: 通过使用monitorenter和monitorexit指令实现的.
2. 同步方法: ACC_SYNCHRONIZED修饰

monitorenter指令是在编译后插入到同步代码块的开始位置, 而monitorexit指令是在编译后插入到同步代码块的结束处或异常处.

**示例代码**

为了证明JVM的实现方式, 下面通过反编译代码来证明.

```java
public class Demo {

    public void f1() {
        synchronized (Demo.class) {
            System.out.println("Hello World.");
        }
    }

    public synchronized void f2() {
        System.out.println("Hello World.");
    }

}
```

编译之后的字节码如下(只摘取了方法的字节码):

```text
public void f1();
  descriptor: ()V
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=3, args_size=1
       0: ldc           #2                  // class me/snail/base/Demo
       2: dup
       3: astore_1
       4: monitorenter
       5: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       8: ldc           #4                  // String Hello World.
      10: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      13: aload_1
      14: monitorexit
      15: goto          23
      18: astore_2
      19: aload_1
      20: monitorexit
      21: aload_2
      22: athrow
      23: return
    Exception table:
       from    to  target type
           5    15    18   any
          18    21    18   any
    LineNumberTable:
      line 6: 0
      line 7: 5
      line 8: 13
      line 9: 23
    StackMapTable: number_of_entries = 2
      frame_type = 255 /* full_frame */
        offset_delta = 18
        locals = [ class me/snail/base/Demo, class java/lang/Object ]
        stack = [ class java/lang/Throwable ]
      frame_type = 250 /* chop */
        offset_delta = 4

public synchronized void f2();
  descriptor: ()V
  flags: ACC_PUBLIC, ACC_SYNCHRONIZED
  Code:
    stack=2, locals=1, args_size=1
       0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #4                  // String Hello World.
       5: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
    LineNumberTable:
      line 12: 0
      line 13: 8
}
```

先说f1()方法, 发现其中一个monitorenter对应了两个monitorexit, 这是不对的. 但是仔细看#15: goto语句, 直接跳转到了#23: return处, 再看#22: athrow语句发现, 原来第二个monitorexit是保证同步代码块抛出异常时锁能得到正确的释放而存在的, 这就理解了.

综上: 发现同步代码块是通过monitorenter和monitorexit来实现的; 同步方法是加了一个ACC_SYNCHRONIZED修饰来实现的.

### 4 Java对象头(存储锁类型)

在HotSpot虚拟机中, 对象在内存中的布局分为三块区域: 对象头, 示例数据和对其填充.

对象头中包含两部分: MarkWord 和 类型指针.

如果是数组对象的话, 对象头还有一部分是存储数组的长度.

多线程下synchronized的加锁就是对同一个对象的对象头中的MarkWord中的变量进行CAS操作.

**MarkWord**

Mark Word用于存储对象自身的运行时数据, 如HashCode, GC分代年龄, 锁状态标志, 线程持有的锁, 偏向线程ID等等.
占用内存大小与虚拟机位长一致(32位JVM -> MarkWord是32位, 64位JVM->MarkWord是64位).

**类型指针**

类型指针指向对象的类元数据, 虚拟机通过这个指针确定该对象是哪个类的实例.

**对象头的长度**

| 长度     | 内容                   | 说明                           |
| -------- | ---------------------- | ------------------------------ |
| 32/64bit | MarkWord               | 存储对象的hashCode或锁信息等   |
| 32/64bit | Class Metadada Address | 存储对象类型数据的指针         |
| 32/64bit | Array Length           | 数组的长度(如果当前对象是数组) |

如果是数组对象的话, 虚拟机用3个字宽(32/64bit + 32/64bit + 32/64bit)存储对象头; 如果是普通对象的话, 虚拟机用2字宽存储对象头(32/64bit + 32/64bit).

### 5 优化后synchronized锁的分类

级别从低到高依次是:

1. 无锁状态
2. 偏向锁状态
3. 轻量级锁状态
4. 重量级锁状态

锁可以升级, 但不能降级. 即: 无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁是单向的.

下面看一下每个锁状态时, 对象头中的MarkWord这一个字节中的内容是什么. 以32位为例.

**无锁状态**

| 25bit          | 4bit         | 1bit(是否是偏向锁) | 2bit(锁标志位) |
| -------------- | ------------ | ------------------ | -------------- |
| 对象的hashCode | 对象分代年龄 | 0                  | 01             |

**偏向锁状态**

| 23bit  | 2bit  | 4bit         | 1bit | 2bit |
| ------ | ----- | ------------ | ---- | ---- |
| 线程ID | epoch | 对象分代年龄 | 1    | 01   |

**轻量级锁状态**

| 30bit                | 2bit |
| -------------------- | ---- |
| 指向栈中锁记录的指针 | 00   |

**重量级锁状态**

| 30bit                      | 2bit |
| -------------------------- | ---- |
| 指向互斥量(重量级锁)的指针 | 10   |

### 6 锁的升级(进化)

#### 6.1 偏向锁

偏向锁是针对于一个线程而言的, 线程获得锁之后就不会再有解锁等操作了, 这样可以省略很多开销. 假如有两个线程来竞争该锁话, 那么偏向锁就失效了, 进而升级成轻量级锁了.

为什么要这样做呢? 因为经验表明, 其实大部分情况下, 都会是同一个线程进入同一块同步代码块的. 这也是为什么会有偏向锁出现的原因.

在Jdk1.6中, 偏向锁的开关是默认开启的, 适用于只有一个线程访问同步块的场景.

##### 偏向锁的加锁

当一个线程访问同步块并获取锁时, 会在锁对象的对象头和栈帧中的锁记录里存储锁偏向的线程ID, 以后该线程进入和退出同步块时不需要进行CAS操作来加锁和解锁, 只需要简单的测试一下锁对象的对象头的MarkWord里是否存储着指向当前线程的偏向锁(线程ID是当前线程), 如果测试成功, 表示线程已经获得了锁; 如果测试失败, 则需要再测试一下MarkWord中偏向锁的标识是否设置成1(表示当前是偏向锁), 如果没有设置, 则使用CAS竞争锁, 如果设置了, 则尝试使用CAS将锁对象的对象头的偏向锁指向当前线程.

##### 偏向锁的撤销

偏向锁使用了一种等到竞争出现才释放锁的机制, 所以当其他线程尝试竞争偏向锁时, 持有偏向锁的线程才会释放锁. 偏向锁的撤销需要等到全局安全点(在这个时间点上没有正在执行的字节码). 首先会暂停持有偏向锁的线程, 然后检查持有偏向锁的线程是否存活, 如果线程不处于活动状态, 则将锁对象的对象头设置为无锁状态; 如果线程仍然活着, 则锁对象的对象头中的MarkWord和栈中的锁记录要么重新偏向于其它线程要么恢复到无锁状态, 最后唤醒暂停的线程(释放偏向锁的线程).

##### 总结

偏向锁在Java6及更高版本中是默认启用的, 但是它在程序启动几秒钟后才激活. 可以使用-XX:BiasedLockingStartupDelay=0来关闭偏向锁的启动延迟, 也可以使用-XX:-UseBiasedLocking=false来关闭偏向锁, 那么程序会直接进入轻量级锁状态.

#### 6.2 轻量级锁

当出现有两个线程来竞争锁的话, 那么偏向锁就失效了, 此时锁就会膨胀, 升级为轻量级锁.

##### 轻量级锁加锁

线程在执行同步块之前, JVM会先在当前线程的栈帧中创建用户存储锁记录的空间, 并将对象头中的MarkWord复制到锁记录中. 然后线程尝试使用CAS将对象头中的MarkWord替换为指向锁记录的指针. 如果成功, 当前线程获得锁; 如果失败, 表示其它线程竞争锁, 当前线程便尝试使用自旋来获取锁, 之后再来的线程, 发现是轻量级锁, 就开始进行自旋.

##### 轻量级锁解锁

轻量级锁解锁时, 会使用原子的CAS操作将当前线程的锁记录替换回到对象头, 如果成功, 表示没有竞争发生; 如果失败, 表示当前锁存在竞争, 锁就会膨胀成重量级锁.

##### 总结

总结一下加锁解锁过程, 有线程A和线程B来竞争对象c的锁(如: synchronized(c){} ), 这时线程A和线程B同时将对象c的MarkWord复制到自己的锁记录中, 两者竞争去获取锁, 假设线程A成功获取锁, 并将对象c的对象头中的线程ID(MarkWord中)修改为指向自己的锁记录的指针, 这时线程B仍旧通过CAS去获取对象c的锁, 因为对象c的MarkWord中的内容已经被线程A改了, 所以获取失败. 此时为了提高获取锁的效率, 线程B会循环去获取锁, 这个循环是有次数限制的, 如果在循环结束之前CAS操作成功, 那么线程B就获取到锁, 如果循环结束依然获取不到锁, 则获取锁失败, 对象c的MarkWord中的记录会被修改为重量级锁, 然后线程B就会被挂起, 之后有线程C来获取锁时, 看到对象c的MarkWord中的是重量级锁的指针, 说明竞争激烈, 直接挂起.

解锁时, 线程A尝试使用CAS将对象c的MarkWord改回自己栈中复制的那个MarkWord, 因为对象c中的MarkWord已经被指向为重量级锁了, 所以CAS失败. 线程A会释放锁并唤起等待的线程, 进行新一轮的竞争.

#### 6.3 锁的比较

| 锁       | 优点                                                         | 缺点                                            | 适用场景                           |
| -------- | ------------------------------------------------------------ | ----------------------------------------------- | ---------------------------------- |
| 偏向锁   | 加锁和解锁不需要额外的消耗, 和执行非同步代码方法的性能相差无几. | 如果线程间存在锁竞争, 会带来额外的锁撤销的消耗. | 适用于只有一个线程访问的同步场景   |
| 轻量级锁 | 竞争的线程不会阻塞, 提高了程序的响应速度                     | 如果始终得不到锁竞争的线程, 使用自旋会消耗CPU   | 追求响应时间, 同步快执行速度非常快 |
| 重量级锁 | 线程竞争不适用自旋, 不会消耗CPU                              | 线程堵塞, 响应时间缓慢                          | 追求吞吐量, 同步快执行时间速度较长 |

### 7 总结

首先要明确一点是引入这些锁是为了提高获取锁的效率, 要明白每种锁的使用场景, 比如偏向锁适合一个线程对一个锁的多次获取的情况; 轻量级锁适合锁执行体比较简单(即减少锁粒度或时间), 自旋一会儿就可以成功获取锁的情况.

要明白MarkWord中的内容表示的含义.

原文地址：<https://www.cnblogs.com/wuqinglong/p/9945618.html>