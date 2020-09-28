## ThreadLocal作用

ThreadLocal的作用是使得每个线程**都能拥有各自独立的对象副本**，假设多个线程拥有同一个实例，ThreadLocal\<T\>类型的变量在每个线程中都有一个副本，从而为变量提供了线程间隔离的作用。



## ThreadLocal实例

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



## 源码

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
2. 获取当前线程的threadLocals变量（是一个ThreadLocalMap类型的Map）；
3. 为这个map插入<当前ThreadLocal对象，值>的键值对；

由于每个线程都有各自的threadLocals，所以多个线程运行时经过第二步拿到的map是不一样的，这样就实现了线程间隔离。



## 参考资料

1. [正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/)

