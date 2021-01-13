## 什么是JMM



## Synchronized

## 共享的问题

自增自减在Java中是一行代码，但是实际上有4行指令。所以在多线程场景中，不能保证线程安全。

现在两个线程共享一个静态变量`i`。

==线程2==执行 `i--`

==线程1==执行 `i++`

==线程2==先进来，读到了`i = 0`，它现在需要对`i`进行4步操作，最终修改为了`i  = -1`，此时还没有写回去。

==线程1==进来了，读到了`i = 0`，速度太快，修改为 `i = 1`并且写了回去。

现在==线程2==终于把 `i = -1`写了回去。

最终结果为 `i = -1`。

```java
public class Test1 {
    static int i = 0;
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int j = 0; j < 5000; j++) {
                i++;
            }
        },"t1");
        Thread t2 = new Thread(() -> {
            for (int j = 0; j < 5000; j++) {
                i--;
            }
        },"t2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(i);
    }
}
```



最简单的方法就是加上`synchronized`。

两个线程首先要尝试拿到锁，如果线程2先拿到，那么它可以读到i，即使时间片上下文切换到线程1，因为线程2还没有释放锁，也得等到线程2 i--操作，释放锁并唤醒线程1去拿锁。

```java
public class Test2 {
    static Object o1 = new Object();
    static int i = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            synchronized (o1) {
                for (int j = 0; j < 5000; j++) {
                    i++;
                }
            }
        }, "t1");
        Thread t2 = new Thread(() -> {
            synchronized (o1) {
                for (int j = 0; j < 5000; j++) {
                    i--;
                }
            }
        }, "t2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(i);
    }
}
```



synchronized只能对对象进行加锁，即使在修饰方法也是针对对象加锁的。











