# 第2章 Java并行程序基础
## 2.2 线程的基本操作
### 2.2.1 新建线程

```
Thread t = new Thread();
t.start();
```
start()方法会新建一个线程并让这个线程执行run()方法。
### 2.2.2 中止线程
Thread类中提供了一个stop()方法，但不推荐使用。
Thread.stop()方法会直接终止线程，释放线程持有的锁，容易造成数据不一致。
### 2.2.3 线程中断
线程中断并不会使线程立即退出，而是给线程一个通知，线程接到通知后如何处理，完全由目标线程自己决定。
```
public void Thread.interrupt() //中断线程
public boolean Thread.isInterrupted() // 判断是否被中断
public static boolean Thread.interrupted() // 判断是否被中断,并清除当前中断状态
```
中断标志位表示当前线程是否已经被中断。
### 2.2.5 等待wait()和通知notify()
当在一个对象实例上调用了wait()方法后，当前线程就会在这个对象上等待，直到其他线程调用了该对象的notify()方法，该线程会被唤醒。

###### wait()和notify()如何工作？
线程调用obj.wait()之后，会进入obj的等待队列。当调用obj.notify()时，会**随机**唤醒一个在等待队列中的线程。
notifyAll()方法会唤醒这个等待队列中所有等待的线程。

无论是wait()或者notify都需要首先获得目标对象的一个监视器，必须包含在对应的synchronzied语句中。
注意：wait()方法执行后，会释放这个监视器。这样做的目的是使得其他等待在object对象上的线程不至于因为该线程的休眠而无法正常执行。


T1 | T2 
---|---
取得object监视器|-
object.wait()|-
释放object监视器|-
-| 取得object监视器
-| object.notify()
-| 释放object监视器
等待object监视器 | -
重获object监视器 | -
释放object监视器 | -

###### Object.wait()和Thread.sleep()的区别？
Object.wait()和Thread.sleep()都可以让线程等待若干时间。

区别：
1. wait()可以被唤醒。
2. wait()方法会释放目标对象的锁，而Thread.sleep()方法不会释放任何资源。

### 2.2.5挂起（suspend）和继续执行（resume）线程
挂起（suspend）和继续执行（resume）已被标记为废弃方法。
因为suspend()在导致线程暂停的同时，并不会释放任何锁资源。直到执行了resume()操作，被挂起的线程才能继续。但是，如果resume()操作意外地在suspend()前就执行了，那么被挂起的线程可能很难有机会被继续执行。更严重的是，它所占用的锁不会被释放。
### 2.2.6 等待线程结束(join)和谦让(yield)
join的作用是等待一个线程执行完毕。

```
public final void join() throws InterrupterException
public final synchronized void join(long millis) throws InterrupterException
```
第一个join方法表示无限等待，第二个方法给出了一个最大等待时间。

###### join的本质
join的本质是让调用线程wait()在当前线程实例上。当线程执行完成后，被等待的线程会在退出前调用notifyAll()通知所有的等待线程继续执行。因此，注意：不要在应用程序中，在Thread对象实例上使用类似wait()或者notify()等方法，因为这很有可能被影响系统api工作，或者被系统api影响。

```
public static native void yield();
```
执行后会使当前线程让出cpu。但不代表线程会停止运行，线程还会竞争cpu资源。
## 2.3 volatile与java内存模型（JMM）
volatile的作用是去除指令重排和保证变量的线程可见性。
## 2.7 线程安全的概念和synchronized
关键字synchronized的用法：

1. 指定加锁对象：相当于对指定对象加锁
2. 作用于实例方法：相当于对当前实例加锁
3. 作用于静态方法：相当于对当前类加锁

### 3.1.2 Condition条件
类似object.notify()和object.wait()