## 1. AQS原理

### 1.1 AQS简介

AQS全名AbstractQueuedSynchronizer，是一个队列同步器，JUC的锁和同步组件都是基于AQS构建的。AQS的结构主要分两部分：

*   state：用了violate关键字修饰的int类型成员变量，表示同步状态，每次更新都能被所有线程可见；
  
*   head、tail：内置的Node类型（Node包含当前线程、等待状态等信息），组成了一个**FIFO的同步队列**。
  

AQS的主要使用方式是继承，它采用了模板方法的设计模式。什么是模板方法设计模式？简单来说，就是**一个类将一些不变行为封装成final方法（即模板方法），将剩余的可变方法（abstract）留给子类实现去拓展**。


### 1.2 AQS接口

AQS主要提供了如下三个方法来访问或修改同步状态state：

*   getState()：获取当前同步状态
  
*   setState(int newState)：设置当前同步状态
  
*   compareAndSetState(int expect, int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性。  



#### 1.2.1 AQS模板方法

下面列举几个最为核心的模板方法：

| 方法                           | 描述                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| void acquire(int arg)          | 独占式调用tryAquire()尝试获取同步状态，获取失败则将该线程塞进同步队列队尾 |
| void acquireShared(int arg)    | 共享式调用tryAquireShared()获取同步状态，同一时刻可以有多个线程获取到同步状态 |
| boolean release(int arg)       | 独占式调用tryRelease()释放同步状态                           |
| boolean releaseShared(int arg) | 共享式调用tryReleaseShared()释放同步状态                     |

#### 1.2.2 AQS重写接口

而AQS可重写的方法如下：

| 方法                              | 描述                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| boolean tryAquire(int arg)        | 独占式获取同步状态                                           |
| boolean tryRelease(int arg)       | 独占式释放同步状态                                           |
| int tryAquireShared(int arg)      | 共享式获取同步状态，与其他接口不同的是该接口返回值是int，这是为了实现共享的语义，返回大于等于0时表示成功，反之失败 |
| boolean tryReleaseShared(int arg) | 共享式释放同步状态                                           |



可以看到，AQS利用模板方法主要做了两件事：

1.  尝试获取或设置同步状态
  
2.  维护同步队列
  

而留给开发者的可重写方法更多聚焦于如何**设置、获取同步状态**，这种设计思想与双亲委派机制有点相似：loadClass中实现了双亲委派机制，而用户可以通过重写findClass去实现自己加载类的具体逻辑。



## 2. Lock和AQS的关系

AQS是实现Lock的基础，两者之间的主要区别在于：

*   Lock面向锁的使用者，它聚焦的问题是**使用者如何更好地使用锁处理并发问题**，而使用者不需要知道锁的实现细节就可以实现互斥同步；
  
*   AQS面向锁的开发者，它关注的两个主要问题是**同步状态管理**以及**维护线程的同步等待队列**。
  



## 3. 自定义实现独占锁

该节通过自定义实现独占锁Demo，去更加清晰揭示AQS和`Lock`接口的区别。

首先，我们定义了一个独占锁—`Mutex`实现`Lock`接口，并实现该接口的所有方法。而Lock接口的方法实现中所做的是都是调用AQS的模板方法。

其次，我们创建了一个内部类—Sync，实现了一个自定义同步器，并重写了其中的所有非模板方法，主要思想是**通过getState()、setState(int newState)和compareAndSetState(int expect, int update)三个方法去设置和获取同步状态**，具体实现可看代码注释。

至此我们总结一下其中的一个核心调用链`lock --> acquire() --> tryAcquire()`：

*   `lock`：面向用户提供锁功能
  
*   `acquire`：尝试设置同步状态并维护同步队列
  
*   `tryAquire`：设置同步状态的真正逻辑
  

```java
import java.util.concurrent.TimeUnit;  
import java.util.concurrent.locks.AbstractQueuedSynchronizer;  
import java.util.concurrent.locks.Condition;  
import java.util.concurrent.locks.Lock;  
  
/**  
 * @author Lam  
 * @date 2020/7/27  
 */  
public class Mutex implements Lock {  
    private final Sync sync = new Sync();  
      
    /**  
     * 同步器实现，重写非模板方法  
     * 主要思想是使用CAS设置state的状态  
     */  
    private static class Sync extends AbstractQueuedSynchronizer {  

        @Override  
        protected boolean tryAcquire(int arg) {  
            // CAS设置state  
            if (compareAndSetState(0, 1)) {  
                // 设置该线程为成功获取同步状态的线程  
                setExclusiveOwnerThread(Thread.currentThread());  
                return true;  
            }  

            // 设置状态失败  
            return false;  
        }  
          
        @Override  
        protected boolean tryRelease(int arg) {  
            // 若此时同步状态为0说明没有线程设置过同步状态  
            if (getState()==0) {  
                throw new IllegalMonitorStateException();  
            }  

            // 清空成功获取同步状态的线程信息  
            setExclusiveOwnerThread(null);  

            // 清除状态  
            setState(0);  
            return true;  
        }  
          
        @Override  
        protected boolean isHeldExclusively() {  
            return getState() == 1;  
        }  
          
        /**  
         * 该方法不是AQS的重写方法，只是为了代码风格统一将逻辑都放到Sync中  
         */  
        Condition newCondition() {  
            // 返回一个等待通知组件  
            return new ConditionObject();  
        }  
    }  
  
    /**  
    * Lock接口的实现  
    */  
      
    @Override  
    public void lock() {  
        sync.acquire(1);  
    }  
      
    @Override  
    public void lockInterruptibly() throws InterruptedException {  
        sync.acquireInterruptibly(1);  
    }  
      
    @Override  
    public boolean tryLock() {  
        return sync.tryAcquire(1);  
    }  
      
    @Override  
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {  
        return sync.tryAcquireNanos(1, unit.toNanos(time));  
    }  
      
    @Override  
    public void unlock() {  
        sync.release(1);  
    }  
      
    @Override  
    public Condition newCondition() {  
        return sync.newCondition();  
    }  
}

```



## 4. 总结

本文大部分内容取自[Java并发编程的艺术](https://book.douban.com/subject/26591326/)，在一些小地方做了删改并加了自己的理解，如果想更完全深入了解AQS的模板方法和非模板方法有哪些可以查看原书第五章的1、2节。