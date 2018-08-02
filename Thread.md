# Thread

## 进程和线程

- 进程是操作系统分配资源的最小单元，线程是操作系统调度的最小单元。
- 一个运行的程序就是一个进程，程序每个并行执行的分支都是一个线程。
## 开启一个线程

1. 创建一个Thread实例
2. 调用Thread实例的start()方法
``` java
package com.zby.thread;

public class ThreadMain {

	public static void main(String[] args) {
		Thread thread = new Thread();
		thread.start();
	}

}
```
- 运行main方法，一个新的线程已经创建并且启动完成，然后，看不出有什么不同。
## Runnable接口与Thread类
```java
package java.lang;

@FunctionalInterface
public interface Runnable {

    public abstract void run();
}
```
```java
package java.lang;

public class Thread implements Runnable {

    private Runnable target;

    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
}
```
## 开启的新线程干了什么
- 我们已经知道如何开启一个新的线程
- Thread继承Runnable接口
- 当我们创建一个线程对象，并且调用线程对象的start()方法后，开启的新线程会调用线程对象的run()方法。
- 如果target  != null，Thread类的run()方法直接调用target的run()方法。
- 使用无参构造方法创建的Thread对象的target == null，所以，新开启的线程什么都没有干。
## Thread的构造方法

``` java
    public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }
    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }
    public Thread(ThreadGroup group, Runnable target) {
        init(group, target, "Thread-" + nextThreadNum(), 0);
    }
    public Thread(String name) {
        init(null, null, name, 0);
    }
    public Thread(ThreadGroup group, String name) {
        init(group, null, name, 0);
    }
    public Thread(Runnable target, String name) {
        init(null, target, name, 0);
    }
    public Thread(ThreadGroup group, Runnable target, String name) {
        init(group, target, name, 0);
    }
    public Thread(ThreadGroup group, Runnable target, String name,
                  long stackSize) {
        init(group, target, name, stackSize);
    }
    Thread(Runnable target, AccessControlContext acc) {
        init(null, target, "Thread-" + nextThreadNum(), 0, acc, false);
    }
```
## 在启动的线程中执行