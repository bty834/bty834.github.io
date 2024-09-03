---
title: Java synchronized 原理
categories: [ 编程,Java ]
tags: [ java ]
---
## Synchronized使用
`synchronized`关键字可使用在方法上或代码块上表示一段同步代码块：
```java
public class SyncTest {
    public void syncBlock(){
        synchronized (this){
            System.out.println("hello block");
        }
    }
    public synchronized void syncMethod(){
        System.out.println("hello method");
    }
}
```
当在方法上指定`synchronized`时，编译后的字节码会在方法的flag上标记`ACC_SYNCHRONIZED`。
在代码块上指定`synchronized`时，编译后的字节码会使用`monitorenter`和`monitorexit`包裹代码块，通常包含一个`monitorenter`和两个`monitorexit`，有两个`monitorexit`指令的原因是：为了保证抛异常的情况下也能释放锁，所以javac为同步代码块添加了一个隐式的`try-finally`，在`finally`中会调用`monitorexit`命令释放锁。

上面Java代码编译后的字节码如下：

```java
{
  public void syncBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter				 	  // monitorenter指令进入同步块
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String hello block
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit						  // monitorexit指令退出同步块
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit						  // monitorexit指令退出同步块
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
 

  public synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED      //添加了ACC_SYNCHRONIZED标记
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String hello method
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
 
}
```

## 锁类型和对象头

### 锁类型
本文基于JDK 1.8。

锁类型可分为：
- 偏向锁
- 轻量级锁
- 重量级锁

偏向锁和轻量级锁在JDK 1.6引入：为了解决在没有多线程竞争或基本没有竞争的场景下因使用传统锁机制带来的性能开销问题。

### 对象头

对象的组成有3个部分：
- 对象头
- 实例数据
- 对齐填充字节: 保证对象大小是8byte的整数倍

其中对象头包含3个部分：
- `Mark Word`: 存储`hashcode`、年龄、锁类型等信息，32位机器上占4字节，64位占8字节
- `Klass Point`: 指向元空间中类元信息的指针，开启指针压缩占4字节，关闭占8字节
- 数组长度（只有数组有）

其中Mark Word在32位和64位的组成分别如下图：

![](/assets/2024/09/03/mark_word.png)

我们引入以下依赖实践一下：
```xml
<!--查看对象头工具-->
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.16</version>
</dependency>
```
```java
import org.openjdk.jol.info.ClassLayout;

public class Test {

    static class World {

    }

    static class Hello {
        boolean bool;
        boolean bool2;
        Boolean bool3;
        String string;
        boolean bool5;
        Integer integer;
        int i;
        World world = new World();
    }

    public static void main(String[] args) {
        System.out.println(ClassLayout.parseInstance(new Hello()).toPrintable());
    }
}
```
在64位机器上运行以上代码输出：

```text
OFF  SZ                TYPE DESCRIPTION               VALUE
  0   8                     (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4                     (object header: class)    0x00060a18
 12   4                 int Hello.i                   0
 16   1             boolean Hello.bool                false
 17   1             boolean Hello.bool2               false
 18   1             boolean Hello.bool5               false
 19   1                     (alignment/padding gap)   
 20   4   java.lang.Boolean Hello.bool3               null
 24   4    java.lang.String Hello.string              null
 28   4   java.lang.Integer Hello.integer             null
 32   4          Test.World Hello.world               (object)
 36   4                     (object alignment gap)    
Instance size: 40 bytes
```
可以看到对象头占用12个字节，实例数据占用24字节，对其填充占用4字节，共40个字节。(`boolean`占1个字节，但会padding到4字节)。

再看下数组：

```java
Hello[] hellos = {new Hello(), new Hello(), new Hello()};
System.out.println(ClassLayout.parseInstance(hellos).toPrintable());
```
在64位机器上运行以上代码输出：
```text
OFF  SZ         TYPE DESCRIPTION                VALUE
  0   8              (object header: mark)      0x0000000000000001 (non-biasable; age: 0)
  8   4              (object header: class)     0x00060c10
 12   4              (array length)             3
 12   4              (alignment/padding gap)    
 16  12   Test$Hello [LTest$Hello;.<elements>   N/A
 28   4              (object alignment gap)     
Instance size: 32 bytes
```
可以看到对象头加了4个字节`00 00 00 03`表示数组长度3，如果数组长度超过4个字节表示的范围会发生什么？会编译不通过，只能接受int类型做数组长度。

## 锁升级

### 偏向锁
当JVM启用了偏向锁模式（`-XX:-UseBiasedLocking` 1.6以上默认开启），当新建一个锁对象，
如果该对象所属的class没有关闭偏向锁模式（什么时候会关闭一个class的偏向模式下文会说，默认所有class的偏向模式都是是开启的），
则该对象的`Mark Word`标记为将是偏向锁状态， 此时`Mark Word`中的线程id为0，表示未偏向任何线程，也叫做匿名偏向(anonymously biased)。

下图展示了锁状态的转换流程：
![](/assets/2024/09/03/state_trans.png)

#### 加锁过程

1. 当该对象第一次被线程获得锁的时候，发现是匿名偏向状态，则会用CAS指令，将`Mark Word`中的线程id由0改成当前线程Id。如果成功，则代表获得了偏向锁，继续执行同步块中的代码。否则，将偏向锁撤销，升级为轻量级锁。

2. 当被偏向的线程再次进入同步块时，发现锁对象偏向的就是当前线程，在通过一些额外的检查后，会往当前线程的栈中添加一条`Displaced Mark Word`为null的`Lock Record`，然后继续执行同步块的代码，因为操纵的是线程私有的栈，因此不需要用到CAS指令；由此可见偏向锁模式下，当被偏向的线程再次尝试获得锁时，仅仅进行几个简单的操作就可以了，在这种情况下，`synchronized`关键字带来的性能开销基本可以忽略。

3. 当其他线程进入同步块时，发现已经有偏向的线程了，则会进入到撤销偏向锁的逻辑里，一般来说，会在safe point中去查看偏向的线程是否还存活，如果存活且还在同步块中则将锁升级为轻量级锁，原偏向的线程继续拥有锁，当前线程则走入到锁升级的逻辑里；如果偏向的线程已经不存活或者不在同步块中，则将对象头的`Mark Word`改为无锁状态（unlocked），之后再升级为轻量级锁。

由此可见，偏向锁升级的时机为：**当锁已经发生偏向后，只要有另一个线程尝试获得偏向锁，则该偏向锁就会升级成轻量级锁**。当然这个说法不绝对，因为还有批量重偏向这一机制。

HotSpot JVM在第一次调用`Object.hashCode`或`System.identityHashCode`时计算身份`hashcode`，并将其存储在对象头中。
随后的调用只是从头中提取以前计算的值。如果`hashcode`已经存到了对象头，则偏向锁无效，当该锁对象处于非偏向状态其他线程进入同步代码块会直接上轻量级锁，当处于偏向状态时计算`hashcode`也要将偏向锁失效并升级为重量级锁。

#### 解锁过程

当有其他线程尝试获得锁时，是根据遍历偏向线程的`Lock Record`来确定该线程是否还在执行同步块中的代码。因此偏向锁的解锁很简单，仅仅将栈中的最近一条`Lock Record`的`_obj`字段设置为null。
需要注意的是，偏向锁的解锁步骤中并不会修改对象头中的线程id。

关于`Lock Record`的结构如下：

```c++
class BasicObjectLock {
  ...
  private:
  BasicLock _lock; // 锁, must be double word aligned
  oop       _obj;  // 锁对象指针
};

class BasicLock {
  private:
  volatile markOop _displaced_header; // 对象头里的mark word
};
```

另外，偏向锁默认不是立即就启动的，在程序启动后，通常有几秒的延迟，可以通过命令 -XX:BiasedLockingStartupDelay=0来关闭延迟。


### 轻量级锁
当存在多个线程访问一个同步代码块时，偏向锁会升级为轻量级锁。

线程在执行同步块之前，JVM会先在当前的线程的栈帧中创建一个`Lock Record`，其包括一个用于存储对象头中的 `Mark Word`（官方称之为`Displaced Mark Word`）以及一个指向锁对象的指针。如下图所示：

![](/assets/2024/09/03/lock_record.png)

#### 加锁过程

1. 在线程栈中创建一个`Lock Record`，将其`_obj`（即上图的Object reference）字段指向锁对象。

2. 直接通过CAS指令将`Lock Record`的地址存储在对象头的`Mark Word`中，如果对象处于无锁状态则修改成功，代表该线程获得了轻量级锁。如果失败，进入到步骤3。

3. 如果是当前线程已经持有该锁了，代表这是一次锁重入。设置`Lock Record`第一部分（`Displaced Mark Word`）为null，起到了一个重入计数器的作用。然后结束。

4. 走到这一步说明发生了竞争，需要膨胀为重量级锁。

#### 解锁过程

1. 遍历线程栈,找到所有obj字段等于当前锁对象的Lock Record。

2. 如果`Lock Record`的`Displaced Mark Word`为null，代表这是一次重入，将`_obj`设置为null后continue。

3. 如果`Lock Record`的`Displaced Mark Word`不为null，则利用CAS指令将对象头的`Mark Word`恢复成为`Displaced Mark Word`。如果成功，则continue，否则膨胀为重量级锁。

### 重量级锁

当线程CAS抢轻量级锁自旋10次失败后，则升级为重量级锁。

重量级锁的状态下，对象的`Mark Word为`指向一个堆中monitor对象的指针。

一个monitor对象包括这么几个关键字段：`cxq`，`EntryList` ，`WaitSet`，`owner`。

其中`cxq` ，`EntryList` ，`WaitSet`都是由`ObjectWaiter`的链表结构，`owner`指向持有锁的线程。

当一个线程尝试获得锁时，如果该锁已经被占用，则会将该线程封装成一个`ObjectWaiter`对象插入到`cxq`的队列尾部，然后暂停当前线程。当持有锁的线程释放锁前，会将`cxq`中的所有元素移动到`EntryList`中去，并唤醒`EntryList`的队首线程。

如果一个线程在同步块中调用了`Object#wait`方法，会将该线程对应的`ObjectWaiter`从`EntryList`移除并加入到`WaitSet`中，然后释放锁。当wait的线程被notify之后，会将对应的`ObjectWaiter`从`WaitSet`移动到`EntryList`中。

当调用一个锁对象的wait或notify方法时，如当前锁的状态是偏向锁或轻量级锁则会先膨胀成重量级锁。

参考：
- [死磕Synchronized底层实现](https://github.com/farmerjohngit/myblog/issues/12)
- [深入浅出偏向锁](https://javabetter.cn/thread/pianxiangsuo.html#%E5%81%8F%E5%90%91%E9%94%81%E4%B8%8E-hashcode)

