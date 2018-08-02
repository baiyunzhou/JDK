# Thread

## 1.进程和线程

- 进程是操作系统分配资源的最小单元，线程是操作系统调度的最小单元。
- 一个运行的程序就是一个进程，程序每个并行执行的分支都是一个线程。
## 2.开启一个线程

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

## 3.为什么开启的新线程什么都没干
### 3.1 Runnable接口
```java
package java.lang;

@FunctionalInterface
public interface Runnable {

    public abstract void run();
}
```
### 3.2 Thread类【简】
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
- 我们已经知道如何开启一个新的线程
- Thread继承Runnable接口
- 当我们创建一个线程对象，并且调用线程对象的start()方法后，开启的新线程会调用线程对象的run()方法。
- 如果target  != null，Thread类的run()方法直接调用target的run()方法。
- 使用无参构造方法创建的Thread对象的target == null，所以，新开启的线程什么都没有干。
## 4.Thread的构造方法

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
## 5.让开启的线程干点什么

### 5.1 使用带Runnable参数的构造方法

```java
package com.zby.thread;

public class ThreadMain {

	public static void main(String[] args) {

		System.out.println("主线程："+Thread.currentThread().getName());

		Thread thread = new Thread(new Runnable() {
			@Override
			public void run() {
				System.out.println("自定义线程："+Thread.currentThread().getName());
			}
		});
		thread.start();
	}

}
```
输出：
``` console
主线程：main
自定义线程：Thread-0
```
构造Thread实例的时候传入一个Runnable实例，线程启动时调用自己的run()方法，run()方法内部调用Runnable实例的run()方法，因此在新线程中打印当前线程的名字。可以看到，同样的两个方法，打印出不一样的线程名称。
### 5.2 继承Thread类，重写run()方法
```java
package com.zby.thread;

public class ThreadMain {
	public static void main(String[] args) {

		System.out.println("主线程：" + Thread.currentThread().getName());
		Thread thread = new Thread() {
			@Override
			public void run() {
				System.out.println("自定义线程：" + Thread.currentThread().getName());
			}
		};
		thread.start();
	}
}
```
输出：
```console
主线程：main
自定义线程：Thread-0
```
- 这里直接使用的匿名内部类重写run()方法。线程启动时调用自己的run()方法，我们重写了run()方法，因此直接调用上面代码，结果也是打印出不同的线程名称。
### 5.3 扩展Thread类，实现一个可以传参的线程
- 定义一个带泛型的接口
```java
package com.zby.thread;

@FunctionalInterface
public interface Callable<T> {
	void call(T t);
}
```
- 定义一个Thread子类，重写run()方法
```java
package com.zby.thread;

public class CustomThread<T> extends Thread {
	private Callable<T> callable;
	private T t;

	public CustomThread(Callable<T> callable, T t) {
		this.callable = callable;
		this.t = t;
	}

	@Override
	public void run() {
		callable.call(t);
	}

}
```
- 启动自定义的线程
```java
package com.zby.thread;

public class ThreadMain {

	public static void main(String[] args) {
		System.out.println("主线程：" + Thread.currentThread().getName());
		Thread thread = new CustomThread<Thread>(new Callable<Thread>() {

			@Override
			public void call(Thread parent) {
				System.out.println("自定义线程：" + Thread.currentThread().getName());
				System.out.println("创建自定义线程的线程：" + parent.getName());
			}
		}, Thread.currentThread());
		thread.start();
	}
}
```
输出：
```console
主线程：main
自定义线程：Thread-0
创建自定义线程的线程：main
```
- 创建Thread子类实例，通过构造器传入Callable匿名对象，日期匿名对象
- 使用start启动线程
- 线程启动时调用自己的run()方法，子类重写了run()方法，因此直接调用上面代码，结果打印出不同的线程名称，并且打印出我们传入的参数。
- 我们实现了一个可以传入任意类型参数的自定义线程
### 5.4 扩展Thread类，实现一个有返回值的线程

## 6.线程的构造器

```java
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name.toCharArray();

        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {

            if (security != null) {
                g = security.getThreadGroup();
            }

            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        g.checkAccess();

        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        if (parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        this.stackSize = stackSize;

        tid = nextThreadID();
    }
```

- Thread类的构造器都调用了init(ThreadGroup g, Runnable target, String name, long stackSize, AccessControlContext acc)方法
- 线程名称不能为空，不带线程名称的构造器会给线程生成一个默认的线程名称
- 获取到创建线程对象的线程对象parent
- 获取安全管理器
- 如果没有指定线程组，首先使用安全管理器的线程组，为空则使用parent的线程组
- 依赖所属线程组进行安全检查
- 如果有安全管理器，检查子类是否覆盖了线程上下文类加载器
- 所属线程组内部未启动线程计数器加一
- 设置当前线程所属线程组
- 线程是否是守护线程设置为与parent一致
- 线程优先级设置为与parent一致
- 没有安全管理器，并且线程上下文类加载器被覆盖，设置为与parent一致
- 设置访问控制上下文
- 设置Runnable实例
- 设置优先级
- 设置继承自parent的线程本地变量Map
- 设置栈大小
- 设置线程ID
## 7.线程内部属性
### 7.1 contextClassLoader
```java
    private ClassLoader contextClassLoader;
    @CallerSensitive
    public ClassLoader getContextClassLoader() {
        if (contextClassLoader == null)
            return null;
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            ClassLoader.checkClassLoaderPermission(contextClassLoader,
                                                   Reflection.getCallerClass());
        }
        return contextClassLoader;
    }
    public void setContextClassLoader(ClassLoader cl) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("setContextClassLoader"));
        }
        contextClassLoader = cl;
    }
```
```java
package com.zby.thread;

public class ThreadMain {

	public static void main(String[] args) {
		System.out.println("主线程：" + Thread.currentThread().getContextClassLoader());
		Thread thread = new Thread(new Runnable() {

			@Override
			public void run() {
				System.out.println("自定义线程：" + Thread.currentThread().getContextClassLoader());

			}
		});
		thread.start();
	}
}
```
输出：
```java
主线程：sun.misc.Launcher$AppClassLoader@2a139a55
自定义线程：sun.misc.Launcher$AppClassLoader@2a139a55
```
- 默认我们自定义的线程和parent是同一个线程上下文类加载器
- 如果存在安全管理器，覆盖getter和setter会触发检查

### 7.2 tid
```java
    private long tid;
    public long getId() {
        return tid;
    }
```
- 自动生成的线程序列号
- 不能修改
- main线程是1，自定义线程从10开始自增
- 2~9去了哪里？
### 7.3 name
```java
    private volatile char  name[];
    public final String getName() {
        return new String(name, true);
    }
    public final synchronized void setName(String name) {
        checkAccess();
        this.name = name.toCharArray();
        if (threadStatus != 0) {
            setNativeName(name);
        }
    }
```
- 名称使用的字符数组存储
- getter和setter都是Final
- 可以在创建时指定
- 创建时没有指定，默认会生成线程名称
- 默认线程名称："Thread-" + nextThreadNum()
- nextThreadNum()返回从0自增的数字
### 7.4 priority
```java
    public final static int MIN_PRIORITY = 1;
    public final static int NORM_PRIORITY = 5;
    public final static int MAX_PRIORITY = 10;
    private int priority;
    public final int getPriority() {
        return priority;
    }
    public final void setPriority(int newPriority) {
        ThreadGroup g;
        checkAccess();
        if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
            throw new IllegalArgumentException();
        }
        if((g = getThreadGroup()) != null) {
            if (newPriority > g.getMaxPriority()) {
                newPriority = g.getMaxPriority();
            }
            setPriority0(priority = newPriority);
        }
    }
```
- 使用一个整数表示，范围是[1,10]，默认5

- 设置值如果超过所属线程组最大优先级，会被设置为所属线程组优先级

### 7.5 threadStatus
```java
    public enum State {
        NEW,
        RUNNABLE,
        BLOCKED,
        WAITING,
        TIMED_WAITING,
        TERMINATED;
    }
    private volatile int threadStatus = 0;
    public State getState() {
        return sun.misc.VM.toThreadState(threadStatus);
    }
```
- 是给虚拟机用的，不能拿来做同步控制
- 只能获取转换后的状态
```java
package com.zby.thread;

public class ThreadMain {
	private static volatile boolean blocked = false;
	private static volatile boolean waiting = false;
	private static volatile boolean timed_waiting = false;
	private static volatile boolean terminated = false;
	private static final Object LOCK = new Object();

	public static void main(String[] args) {
		Thread thread = new Thread(new Runnable() {
			@Override
			public void run() {

				while (true) {
					if (blocked) {
						synchronized (LOCK) {
						}
					}

					if (waiting) {
						synchronized (LOCK) {
							try {
								LOCK.wait();
							} catch (InterruptedException e) {
								e.printStackTrace();
							}
						}
					}
					if (timed_waiting) {
						try {
							Thread.sleep(1000);
						} catch (InterruptedException e) {
							e.printStackTrace();
						}
					}
					if (terminated) {
						break;
					}
				}
			}

		});
		// NEW
		System.out.println("1-新建自定义线程状态：" + thread.getState());
		// NEW->RUNNABLE
		thread.start();
		System.out.println("2-自定义线程启动状态：" + thread.getState());
		// RUNNABLE->BLOCKED->RUNNABLE
		synchronized (LOCK) {
			blocked = true;
			sleep();
			System.out.println("3-自定义线程等待获取锁状态：" + thread.getState());
		}
		blocked = false;
		sleep();
		System.out.println("4-自定义线程获取锁后状态：" + thread.getState());
		// RUNNABLE->WAITING->RUNNABLE
		waiting = true;
		sleep();
		System.out.println("5-自定义线程等待锁后状态：" + thread.getState());
		synchronized (LOCK) {
			LOCK.notify();
			waiting = false;
		}
		sleep();
		System.out.println("6-自定义线程等待锁后状态：" + thread.getState());
		// RUNNABLE->TIMED_WAITING->RUNNABLE
		timed_waiting = true;
		sleep();
		System.out.println("7-自定义线程等待状态：" + thread.getState());
		timed_waiting = false;
		sleep();
		System.out.println("8-自定义线程等待后状态：" + thread.getState());
		terminated = true;
		sleep();
		System.out.println("9-自定义线程等待后状态：" + thread.getState());
	}

	public static void sleep() {
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```
```console
1-新建自定义线程状态：NEW
2-自定义线程启动状态：RUNNABLE
3-自定义线程阻塞状态：BLOCKED
4-自定义线程获阻塞后状态：RUNNABLE
5-自定义线程休眠状态：WAITING
6-自定义线程休眠后状态：RUNNABLE
7-自定义线程定时阻塞状态：TIMED_WAITING
8-自定义线程定时阻塞后状态：RUNNABLE
9-自定义线程执行完成后状态：TERMINATED

```
```java

```