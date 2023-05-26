## 1. ThreadLocal作用

ThreadLocal的作用是使得每个线程**都能拥有各自独立的对象副本**，假设多个线程拥有同一个实例，ThreadLocal\<T\>类型的变量在每个线程中都有一个副本，从而为变量提供了线程间隔离的作用。



## 2. ThreadLocal实例

```java

public class ThreadLocalDemo implements Runnable{
    private static ThreadLocal<People> local = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        ThreadLocalDemo localDemo = new ThreadLocalDemo();
        Thread thread1 = new Thread(localDemo);
        thread1.start();

      	Thread.sleep(1000);
      
        Thread thread2 = new Thread(localDemo);
        thread2.start();
    }

    @Override
    public void run() {
        People people = local.get();
        if (people==null) {
            people = new People();
            people.setName("djh");
            people.setAge(new Random().nextInt());
            local.set(people);
        }
        System.out.println(people);
    }
}

public class People {
    String name;
    int age;

    public void setName(String name) {
        this.name = name;
    }

    public void setAge(int age) {
        this.age = age;
    }

}
```

在该例子中，`thread1`和`thread2`拥有同一个对象`localDemo`，如果不使用`ThreadLocal`包装，则两个线程访问的people应该是同一个。而在我们使用`ThreadLocal`包装后，结果如下：

```
bean.People@6fb35e7d
bean.People@614b201a
```

两者访问的people是不一样的两个对象，而这就是ThreadLocal的作用，提供了**变量线程间的隔离能力。**



## 3. 源码

查看ThreadLocal的set方法，

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

1. 先获取当前线程；
2. 获取当前线程的 threadLocals 变量（是一个 ThreadLocalMap 类型的 Map）；
3. 如果 map 不为空，为这个 map 插入<当前 ThreadLocal 对象，值>的键值对；
4. 如果该 map 为空，则创建一个一个 ThreadLocalMap 对象，`t.threadLocals = new ThreadLocalMap(this, firstValue)`

由于每个线程都有各自的 threadLocals，所以多个线程运行时经过第二步拿到的 map 是不一样的，这样就实现了线程间隔离。



## 4. ThreadLocal的内存泄漏问题

由第3节可以得知，数据实际存储在每个线程的 threadLocals 成员（即一个**ThreadLocalMap对象**）里，ThreadLocalMap 和普通的 map 一样使用 Entry 数组存储数据，这个Entry特殊的地方在于，它继承了弱引用，即该Entry 的 key 是一个 ThreadLocal 对象的弱引用。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

弱引用的特点是，如果某个对象只被弱引用关联，在下一次GC时该对象就会被回收。此时某个 ThreadLocal 类型变量使用完被我们置为 null，此时 Entry 中关联的 Key 会变成 null，但 Value 由于是强引用，且运行的线程指向他，因此 Value 部分发生了内存泄漏。

线程执行完毕被回收时 Value 会随之回收，但当我们使用线程池时，由于经常复用复用，因此可能导致 Value 一直存在，这就发生了更为严重的内存泄漏。



## 5. 参考资料

1. [正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/)
2. [ThreadLocal 内存泄漏问题深入分析](https://segmentfault.com/a/1190000022704085)

