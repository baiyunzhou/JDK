[TOC]

# 从同步到异步

## 如何创建一个新线程

```java
new Thread();
```

## 如何启动一个新线程

```java
new Thread().start();
```

## 如何在新线程运行下面方法？

```java
public class Task {

    public static final String HELLO_WORLD = "Hello,World!";

    public void say() {
        System.out.println(Thread.currentThread().getName() + " say : " + HELLO_WORLD);
    }
}
```

## 两种方式

```java
public class MultiThread {

    public static void main(String[] args) {

        new Thread(new Runnable() {
            public void run() {
                new Task().say();
            }
        }).start();

        new Thread() {
            public void run() {
                new Task().say();
            };
        }.start();

    }

}
```

## 应该使用哪种？

- 传入Runnable对象

  推荐。将需要执行的逻辑封装到Runnable子类中

- 重写run()方法

  需要扩展Thread的功能时使用。

## 我们干了什么？
- 我们执行了异步任务！
- 我们在主线程执行时，把一个任务交给了另一个线程执行。另一个任务执行的时候不会阻塞当前流程！

## 我们的任务通常不会如此简单

```java
public class Task {

    public String say(String param) {
      	try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        String result = Thread.currentThread().getName() + " say : " + param;
        System.out.println(result);
        return result;
    }
  
}
```

- sleep模拟任务耗时
- 有参数，有返回值的任务如何执行呢？
- 我们需要在主线程中指定不同的参数，同时获取到不同的结果
## 封装Runnable实现RunnableTask

```java
public class RunnableTask implements Runnable {
    private Task task;
    private String param;
    private volatile String result;

    public RunnableTask(Task task, String param) {
        this.task = task;
        this.param = param;
    }

    @Override
    public void run() {
        this.result = task.say(param);
    }

    public String getResult() {
        return result;
    }

}
```

- volatile用于排除线程可见性影响

## 奇怪的结果？

```java
    public static void main(String[] args) throws Exception {
        RunnableTask runnableTask1 = new RunnableTask(new Task(), "Hello,World!");
        new Thread(runnableTask1).start();
        RunnableTask runnableTask2 = new RunnableTask(new Task(), "Hello,zby!");
        new Thread(runnableTask2).start();
        // TimeUnit.SECONDS.sleep(15);
        System.out.println(Thread.currentThread().getName() + " say : " + runnableTask1.getResult());
        System.out.println(Thread.currentThread().getName() + " say : " + runnableTask2.getResult());
```

```console
main say : null
Thread-0 say : Hello,World!
main say : null
Thread-1 say : Hello,zby!

```

- 两个10s的任务，通过异步的方式10s左右就能完成
- 我没有获取到异步执行的结果
- 去掉注释的一行试试？

## 为什么呢？

- 我们的任务是异步执行的，执行打印结果的时候，我们异步任务可能还没执行完成，因此result尚未设置

- 主线程等待1s，异步线程都执行完毕，可以获取到异步执行结果

## 优化思路
- 任务本身是异步执行的，但是需要在主线程获取执行结果，那么主线程就需要阻塞等待执行完成

## 优化后的RunnableTask

```java
public class RunnableTask implements Runnable {
    private Task task;
    private String param;
    private volatile String result;
    private volatile boolean complated = false;

    public RunnableTask(Task task, String param) {
        this.task = task;
        this.param = param;
    }

    @Override
    public void run() {
        synchronized (this) {
            this.result = task.say(param);
            complated = true;
            this.notifyAll();
        }

    }

    public String getResult() {
        synchronized (this) {
            while (!complated) {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
        return result;
    }

}
```

- 获取结果的时候需要判断是否完成，没完成需要阻塞等待完成
- 使用新的标记complated而不判断result是否为null是因为，result可能执行完返回null
## 还存在的问题
1. 同步机制性能很低，编码灵活度低，如取消等操作无法响应
2. 无法获取或者控制异步执行过程
3. 我不想对每个任务都进行封装
4. 我的参数和返回值类型不确定
## 解决参数问题
- 可以看到我们的优化代码并没有对参数做处理，其实我们创建Runnable子类的时候就可以指定有参构造方法，并不需要额外处理
## 解决返回值和异步过程控制

- 抽象出执行结果获取和执行控制过程的接口

```java
package java.util.concurrent;

public interface Future<V> {

    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

- 这个接口是jdk提供的，可以满足普通需求
- 但是，功能有限，只能阻塞方式获取结果。如Netty对它进行了更丰富的扩展，使用观察者模式监听执行结果
## Runnable升级为RunnableFuture

- 需要获取异步结果的Runnable子类，应该同时实现Runnable接口和Future接口，并提供对于方法的实现

```java
package java.util.concurrent;

public interface RunnableFuture<V> extends Runnable, Future<V> {

    void run();
}

```

- 我们不需要每次都去实现两个接口
- 这个接口也是JDK提供的，可以执行任务并获取结果

## 我需要自己实现吗？

- Future接口提供的功能，是跟业务代码无关的

- run()方法执行的逻辑应该是独特的

- 我们可以使用装饰模式对Runnable进行实现。

- 可以对Future提供的方法进行统一的实现

## Runnable另一个问题
- 我们在Runnable接口的run()方法中写自己的逻辑代码，没办法直接返回结果
- 我们装饰的是Runnable接口，该接口无法获取返回值，需要从它的内部获取返回值
- 为了获取到我们任务执行的结果，可能需要这么做
```java
public interface ResultRunnable<V> extends Runnable {
    V getResult();
}


public class MyRunnableFuture<V> implements Runnable, Future<V> {
    private ResultRunnable<V> resultRunnable;

    public MyRunnableFuture(ResultRunnable<V> resultRunnable) {
        this.resultRunnable = resultRunnable;
    }

    @Override
    public boolean cancel(boolean mayInterruptIfRunning) {
        // TODO Auto-generated method stub
        return false;
    }

    @Override
    public boolean isCancelled() {
      	// TODO Auto-generated method stub
        return false;
    }

    @Override
    public boolean isDone() {
      	// TODO Auto-generated method stub
        return false;
    }

    @Override
    public V get() throws InterruptedException, ExecutionException {
      	// TODO Auto-generated method stub
        // 阻塞直到执行完成
        return null;
    }

    @Override
    public V get(long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
      	// TODO Auto-generated method stub
        return null;
    }

    @Override
    public void run() {
        resultRunnable.run();
        resultRunnable.getResult();//获取结果并设置到当前的属性，以便get()能获取
        // 通知阻塞结束

    }

}
```

- 这样编码的话，我们需要手动设置执行结果
- 或者抽象结果的设置，我们执行完调用set设置，还是很麻烦，忘了咋办

## 来一个更好写逻辑代码的接口Callable

```java
 package java.util.concurrent;

@FunctionalInterface
public interface Callable<V> {

    V call() throws Exception;
}

```

- 这个接口就符合我们编写业务代码的习惯

## Callable接口的优点

- 对Callable接口进行装饰，可以调用执行业务代码，直接获取到返回值
- 无需对业务代码再次封装

## 对RunnableFuture的实现大概就是这样

```java
class FutureTask<V> implements RunnableFuture<V> {

    private Callable<V> callable;

    public FutureTask(Callable<V> callable) {
        this.callable = callable;
    }

    @Override
    public boolean cancel(boolean mayInterruptIfRunning) {
        // TODO Auto-generated method stub
        return false;
    }

    @Override
    public boolean isCancelled() {
        // TODO Auto-generated method stub
        return false;
    }

    @Override
    public boolean isDone() {
        // TODO Auto-generated method stub
        return false;
    }

    @Override
    public V get() throws InterruptedException, ExecutionException {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public V get(long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public void run() {
        try {
            V call = callable.call();// 把结果设置到当前类的字段中
            // 唤醒等待获取线程
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

## 可是我也有没有返回值的业务代码

- 显然Callable比Runnable更强大，多一个返回值
- 为了兼容Runnable，可以使用适配器模式进行封装
```java
class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
```

- Runnable接口可以完美转换为Callable

## JDK中FutureTask完整实现异步封装

```java
package java.util.concurrent;

public class FutureTask<V> implements RunnableFuture<V> {
    /*
     * Revision notes: This differs from previous versions of this
     * class that relied on AbstractQueuedSynchronizer, mainly to
     * avoid surprising users about retaining interrupt status during
     * cancellation races. Sync control in the current design relies
     * on a "state" field updated via CAS to track completion, along
     * with a simple Treiber stack to hold waiting threads.
     *
     * Style note: As usual, we bypass overhead of using
     * AtomicXFieldUpdaters and instead directly use Unsafe intrinsics.
     */

    /**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

    /** The underlying callable; nulled out after running */
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;

    /**
     * Returns result or throws exception for completed task.
     *
     * @param s completed state value
     */
    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }

    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Callable}.
     *
     * @param  callable the callable task
     * @throws NullPointerException if the callable is null
     */
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Runnable}, and arrange that {@code get} will return the
     * given result on successful completion.
     *
     * @param runnable the runnable task
     * @param result the result to return on successful completion. If
     * you don't need a particular result, consider using
     * constructions of the form:
     * {@code Future<?> f = new FutureTask<Void>(runnable, null)}
     * @throws NullPointerException if the runnable is null
     */
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }

    public boolean isCancelled() {
        return state >= CANCELLED;
    }

    public boolean isDone() {
        return state != NEW;
    }

    public boolean cancel(boolean mayInterruptIfRunning) {
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }

    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }

    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }

    /**
     * Protected method invoked when this task transitions to state
     * {@code isDone} (whether normally or via cancellation). The
     * default implementation does nothing.  Subclasses may override
     * this method to invoke completion callbacks or perform
     * bookkeeping. Note that you can query status inside the
     * implementation of this method to determine whether this task
     * has been cancelled.
     */
    protected void done() { }

    /**
     * Sets the result of this future to the given value unless
     * this future has already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon successful completion of the computation.
     *
     * @param v the value
     */
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }

    /**
     * Causes this future to report an {@link ExecutionException}
     * with the given throwable as its cause, unless this future has
     * already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon failure of the computation.
     *
     * @param t the cause of failure
     */
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }

    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

    /**
     * Executes the computation without setting its result, and then
     * resets this future to initial state, failing to do so if the
     * computation encounters an exception or is cancelled.  This is
     * designed for use with tasks that intrinsically execute more
     * than once.
     *
     * @return {@code true} if successfully run and reset
     */
    protected boolean runAndReset() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }

    /**
     * Ensures that any interrupt from a possible cancel(true) is only
     * delivered to a task while in run or runAndReset.
     */
    private void handlePossibleCancellationInterrupt(int s) {
        // It is possible for our interrupter to stall before getting a
        // chance to interrupt us.  Let's spin-wait patiently.
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield(); // wait out pending interrupt

        // assert state == INTERRUPTED;

        // We want to clear any interrupt we may have received from
        // cancel(true).  However, it is permissible to use interrupts
        // as an independent mechanism for a task to communicate with
        // its caller, and there is no way to clear only the
        // cancellation interrupt.
        //
        // Thread.interrupted();
    }

    /**
     * Simple linked list nodes to record waiting threads in a Treiber
     * stack.  See other classes such as Phaser and SynchronousQueue
     * for more detailed explanation.
     */
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }

    /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }

    /**
     * Awaits completion or aborts on interrupt or timeout.
     *
     * @param timed true if use timed waits
     * @param nanos time to wait, if timed
     * @return state upon completion
     */
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }

    /**
     * Tries to unlink a timed-out or interrupted wait node to avoid
     * accumulating garbage.  Internal nodes are simply unspliced
     * without CAS since it is harmless if they are traversed anyway
     * by releasers.  To avoid effects of unsplicing from already
     * removed nodes, the list is retraversed in case of an apparent
     * race.  This is slow when there are a lot of nodes, but we don't
     * expect lists to be long enough to outweigh higher-overhead
     * schemes.
     */
    private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long stateOffset;
    private static final long runnerOffset;
    private static final long waitersOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = FutureTask.class;
            stateOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("state"));
            runnerOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("runner"));
            waitersOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

}

```

- 使用int字段表示当前任务状态流转
- 使用链表对阻塞线程进行排队
- 提供通用的异步任务封装

## 再次执行有返回值的任务

```java
    public static void main(String[] args) throws Exception {
        FutureTask<String> futureTask = new FutureTask<String>(new Callable<String>() {

            @Override
            public String call() throws Exception {
                return new Task().say("Hello,World!");
            }
        });
        new Thread(futureTask).start();
      	System.out.println(futureTask.isDone());//直接返回
        System.out.println(Thread.currentThread().getName() + " say : " + futureTask.get());//等待结果
    }
```

- 自动阻塞等待获取结果
- 不需要处理同步、排队
- 可以对futureTask进行操作

## 总结

- JDK1.0开始对多线程只提供了Thread类和Runnable接口的简单支持
- JDK的创建线程本不支持获取线程执行结果的返回值，但是通过大师一系列封装，一切都变得简单了
- 另外可以看到Runnable接口是1.0，Callable接口是1.5，那么获取异步结果在这之间JDK是有空白的
- 如果带着Callable接口回到1.0，那么Thread类的启动线程直接运行Callable.call()方法，Thread.start()方法直接返回Future，会不会更简单容易理解了
- 要是Thread.start()支持参数传递，会不会更完美了
