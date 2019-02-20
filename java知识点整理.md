# 语言基础
## transient关键字
用transient关键字修饰的变量，不参加序列化。

# 集合
## List
### LinkedList
普通双向链表。没有很复杂的地方
### 查找
```
public E get(int index)
```
index与数组长度/2比较，如果在前半部分就从头遍历。如果在后面，从尾遍历。

### ArrayList
基于数组的链表。
#### 根据下标删除数组元素
```
public E remove(int index)
```
因为底层数据结构是数组。删除下标为index的元素需要将index后面的元素全部向前移动。每次需要用System.arraycopy移动数组。

#### 扩容
扩容时，新数组的容量拓展为原来的1.5倍。然后使用Arrays.copyOf复制数组。


# 并发
## volatile关键字
[就是要你懂Java中volatile关键字实现原理](http://www.cnblogs.com/xrq730/p/7048693.html)
### 作用
- 去除对变量的指令重排
- 保证可见性
### 注意
volatile不能真正保证线程安全。比如说32位虚拟机对volatile long型的变量进行修改。long类型有64位，需要两步操作。volatile变量不能保证两步操作像系统原语一样，中间不被CPU中断。
## synchronized关键字
### synchronized对内存可见性的影响
- 一个线程在获取到监视器锁以后才能进入synchronized控制的代码块。
- **进入代码块，该线程对于共享变量的缓存就会失效**，因此 synchronized代码块中对于共享变量的读取需要从主内存中重新获取，也就能获取到最新的值。
- **退出代码块时，会将该线程写缓冲区中的数据刷到主内存中**，所以在 synchronized 代码块之前或 synchronized 代码块中对于共享变量的操作随着该线程退出 synchronized 块，会立即对其他线程可见（这句话的前提是其他读取共享变量的线程会从主内存读取最新值）。

###### 单例模式中的双重检查
这里是一种错误的单例模式写法。
```
public class Singleton {

    private static Singleton instance = null;

    private int v;
    private Singleton() {
        this.v = 3;
    }

    public static Singleton getInstance() {
        if (instance == null) { // 1. 第一次检查
            synchronized (Singleton.class) { // 2
                if (instance == null) { // 3. 第二次检查
                    instance = new Singleton(); // 4
                }
            }
        }
        return instance;
    }
}
```
错误的原因是：我们假设有两个线程 a 和 b 调用 getInstance() 方法，假设 a 先走，一路走到 4 这一步，执行  instance = new Singleton() 这句代码。

instance = new Singleton() 这句代码首先会申请一段空间，然后将各个属性初始化为零值(0/null)，执行构造方法中的属性赋值[1]，将这个对象的引用赋值给 instance[2]。**在这个过程中，[1] 和 [2] 可能会发生重排序。**

此时，线程 b 刚刚进来执行到 1（看上面的代码块），就有可能会看到 instance 不为 null，然后线程 b 也就不会等待监视器锁，而是直接返回 instance。问题是这个 instance 可能还没执行完构造方法（线程 a 此时还在 4 这一步），**所以线程b拿到的instance是不完整的**，它里面的属性值可能是初始化的零值(0/false/null)，而不是线程 a 在构造方法中指定的值。

## Timer和ScheduledExecutorService的区别
###### Timer的使用
```
public class TimerTest  
{  
    private static long start;  
  
    public static void main(String[] args) throws Exception  
    {  
  
        TimerTask task1 = new TimerTask()  
        {  
            @Override  
            public void run()  
            {  
  
                System.out.println(”task1 invoked ! ”  
                        + (System.currentTimeMillis() - start));  
                try  
                {  
                    Thread.sleep(3000);  
                } catch (InterruptedException e)  
                {  
                    e.printStackTrace();  
                }  
  
            }  
        };  
        TimerTask task2 = new TimerTask()  
        {  
            @Override  
            public void run()  
            {  
                System.out.println(”task2 invoked ! ”  
                        + (System.currentTimeMillis() - start));  
            }  
        };  
        Timer timer = new Timer();  
        start = System.currentTimeMillis();  
        timer.schedule(task1, 1000);  
        timer.schedule(task2, 3000);  
  
    }  
}  
```
输出
```
task1 invoked ! 1000  
task2 invoked ! 4000
```
1. Timer是单线程，前一个任务耗时长会影响下一个任务的执行时间。ScheduledExecutorService是线程池，没有这个问题。
2. Timer当任务抛出异常时的缺陷；如果TimerTask抛出RuntimeException，Timer会停止所有任务的运行：ScheduledExecutorService不会有这个问题。
3. Timer执行周期任务时依赖系统时间；
## synchronized和Lock的区别
1. Lock是一个接口，而synchronized是Java中的关键字，synchronized是内置的语言实现；
2. synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁；
3. Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断；
4. 通过Lock可以知道有没有成功获取锁，而synchronized却无法办到。
5. Lock可以提高多个线程进行读操作的效率。
## 锁的相关概念
### 可重入锁
简单说，拿到锁的线程可以执行运行加锁的代码，不必重新获得锁。

当一个线程执行到某个synchronized方法时，比如说method1，而在method1中会调用另外一个synchronized方法method2，此时线程不必重新去申请锁，而是可以直接执行方法method2。
### 可中断锁
顾名思义，就是可以相应中断的锁。主要为了避免等待时间过长。在Java中，synchronized就不是可中断锁，而Lock是可中断锁。
### 公平锁
公平锁即尽量以请求锁的顺序来获取锁。
在Java中，synchronized是非公平锁，它无法保证等待的线程获取锁的顺序。而对于ReentrantLock和ReentrantReadWriteLock，它默认情况下是非公平锁，但是可以设置为公平锁。

公平锁因为要维护等待锁的线程队列，效率低。好处是不会有线程饥饿。
### 读写锁
保证多个线程的读操作不冲突。

# 框架
## spring
### IOC
两种容器 BeanFactory 和 ApplicationContext
#### 两种容器的特点和区别
#### spring bean的生命周期
[Spring Bean的生命周期（非常详细）](https://www.cnblogs.com/zrtqsk/p/3735273.html)

[Spring中bean的作用域与生命周期](https://blog.csdn.net/fuzhongmin05/article/details/73389779)

Bean实例生命周期的执行过程如下：
1. Spring对bean进行实例化，默认bean是单例；
2. Spring对bean进行依赖注入；
3. 如果bean实现了BeanNameAware接口，spring将bean的id传给setBeanName()方法；
4. 如果bean实现了BeanFactoryAware接口，spring将调用setBeanFactory方法，将BeanFactory实例传进来；
5. 如果bean实现了ApplicationContextAware接口，它的setApplicationContext()方法将被调用，将应用上下文的引用传入到bean中；
6. 如果bean实现了BeanPostProcessor接口，它的postProcessBeforeInitialization方法将被调用；
7. 如果bean实现了InitializingBean接口，spring将调用它的afterPropertiesSet接口方法，类似的如果bean使用了init-method属性声明了初始化方法，该方法也会被调用；
8. 如果bean实现了BeanPostProcessor接口，它的postProcessAfterInitialization接口方法将被调用；
9. 此时bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁；
10. 若bean实现了DisposableBean接口，spring将调用它的distroy()接口方法。同样的，如果bean使用了destroy-method属性声明了销毁方法，则该方法被调用；
#### 代码中实现
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory类
```
Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd){
    invokeAwareMethods(beanName, bean);
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    invokeInitMethods(beanName, wrappedBean, mbd);
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    return wrappedBean;
}

private void invokeAwareMethods(final String beanName, final Object bean) {
	if (bean instanceof Aware) {
		if (bean instanceof BeanNameAware) {
			((BeanNameAware) bean).setBeanName(beanName);
		}
		if (bean instanceof BeanClassLoaderAware) {
			((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
		}
		if (bean instanceof BeanFactoryAware) {
			((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
		}
	}
}

public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
	invokeAwareInterfaces(bean);
	return bean;
}

private void invokeAwareInterfaces(Object bean) {
	if (bean instanceof Aware) {
		if (bean instanceof EnvironmentAware) {
			((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
		}
		if (bean instanceof EmbeddedValueResolverAware) {
			((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(
					new EmbeddedValueResolver(this.applicationContext.getBeanFactory()));
		}
		if (bean instanceof ResourceLoaderAware) {
			((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
		}
		if (bean instanceof ApplicationEventPublisherAware) {
			((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
		}
		if (bean instanceof MessageSourceAware) {
			((MessageSourceAware) bean).setMessageSource(this.applicationContext);
		}
		if (bean instanceof ApplicationContextAware) {
			((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
		}
	}
```
## AOP
[求求你，下次面试别再问我什么是 Spring AOP 和代理了！
](https://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651482306&idx=1&sn=38cb83c3b1b646814096e78a89db3cf1&chksm=bd2504bd8a528dab8bfaba33ba8b35edc92dcb9124fff107348d634e33b0a58db22de0300d80&scene=0&key=286da683aced0f45b40c779c43e4d09ee0f47a162d217bf649e630b6ab0ae6a58559f436f513251bb488143f16fc6211cf0bb6672955e7643e50f472da12e7441c5900452eed46dc913049b23ce24b4e&ascene=0&uin=MjQxMjUwNDA2MA%3D%3D&devicetype=iMac+MacBookPro12%2C1+OSX+OSX+10.10.5+build(14F2315)&version=12031210&nettype=WIFI&lang=zh_CN&fontScale=100&pass_ticket=RC2DvoAi3GMkKFk8YLAb%2FEvh7Bxe1PXXgV%2BoWNRgKvcbY25AEjjQnCuKz%2FrQhVwa#%23)

[Spring AOP的实现原理](http://listenzhangbin.com/post/2016/09/spring-aop-cglib/)

### 动态代理
2种实现方式
[Java Proxy 和 CGLIB 动态代理原理](http://www.importnew.com/27772.html)
#### jdk原生
只能代理接口类型
#### cglib
通过生成子类的方式。如果目标类是final类型不可用。
#### 区别
cglib不需要实现一个接口。



# 设计模式
## 单例模式
三个要点
1. 线程安全
2. 延迟加载
3. 序列化与反序列化安全
只能存在一个实例。严格讲，应该避免所有可以使得单例对象复制的场景。
1. 饿汉
2. 懒汉
3. 双检锁
4. 静态内部类
5. 防止反序列化(重写readResolve函数)和反射的攻击（构造函数中抛出运行时异常）
### 参考资料
[单例模式](https://blog.csdn.net/hacfox/article/details/69389412)


# 网络协议
## TCP
[TCP的三次握手与四次挥手（详解+动图）](https://blog.csdn.net/qzcsu/article/details/72861891)
