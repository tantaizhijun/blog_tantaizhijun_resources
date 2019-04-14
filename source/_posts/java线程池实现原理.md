---
layout: post
title: java线程池实现原理
date: 2019-04-14 14:04:36
tags:
categories: java
---

## 1.如何自己实现一个线程池

#### 实现线程池需要准备的东西
   - 存放线程的**集合**：
   - 存放任务的**队列**：每有一个任务分配给线程池，我们就从线程池中分配一个线程处理它。但当线程池中的线程都在运行状态，没有空闲线程时，我们还需要一个队列来存储提交给线程池的任务。
另外还有，初始化线程池，是否要指定线程池的大小？要保存当前运行线程的数目等。

```
/**存放线程的集合*/
private ArrayList<MyThead> threads;

/**任务队列*/
private ArrayBlockingQueue<Runnable> taskQueue;

/**线程池初始限定大小*/
private int threadNum;

/**已经工作的线程数目*/
private int workThreadNum;
```
#### 线程池的核心方法
接下来就是线程池的核心方法，每当向线程池提交一个任务时。如果 已经运行的线程<线程池大小，则创建一个线程运行任务，并把这个线程放入线程池；否则将任务放入缓冲队列中。
```
public void execute(Runnable runnable) {
        try {
            mainLock.lock();
            //线程池未满，每加入一个任务则开启一个线程
            if(workThreadNum < threadNum) {
                MyThead myThead = new MyThead(runnable);
                myThead.start();
                threads.add(myThead);
                workThreadNum++;
            }
            //线程池已满，放入任务队列，等待有空闲线程时执行
            else {
                //队列已满，无法添加时，拒绝任务
                if(!taskQueue.offer(runnable)) {
                    rejectTask();
                }
            }
        } finally {
            mainLock.unlock();
        }
    }
```
**到这里，一个线程池已经实现的差不多了，** 
我们还有最后一个**难点**要解决：从任务队列中取出任务，分配给线程池中“空闲”的线程完成.

##### 分配任务给线程的实现思路

   - 思路1：额外开启一个线程，时刻监控线程池的线程空余情况，一旦有线程空余，则马上从任务队列取出任务，交付给空余线程完成。
    这种思路理解起来很容易，但仔细思考，实现起来很麻烦(麻烦如下：), 而且使线程池的架构变的更复杂和不优雅。
        - 如何检测到线程池中的空闲线程 
        - 如何将任务交付给一个.start()运行状态中的空闲线程。
        
   - 思路2:线程池中的所有线程一直都是运行状态的，线程的空闲只是代表此刻它没有在执行任务而已；我们可以让运行中的线程，一旦没有执行任务时，就自己从队列中取任务来执行。
         
为了达到这种效果，我们要重写`run`方法，所以要写一个自定义`Thread`类，然后让线程池都放这个自定义线程类

```
class MyThead extends Thread{
        private Runnable task;

        public MyThead(Runnable runnable) {
            this.task = runnable;
        }
        @Override
        public void run() {
            //该线程一直启动着，不断从任务队列取出任务执行
            while (true) {
                //如果初始化任务不为空，则执行初始化任务
                if(task != null) {
                    task.run();
                    task = null;
                }
                //否则去任务队列取任务并执行
                else {
                    Runnable queueTask = taskQueue.poll();
                    if(queueTask != null)
                        queueTask.run();    
                }
            }
        }
    }
```

##### 简单线程池的实现源码
```
/**
 * 自定义简单线程池
 */
public class MyThreadPool{
    /**存放线程的集合*/
    private ArrayList<MyThead> threads;
    /**任务队列*/
    private ArrayBlockingQueue<Runnable> taskQueue;
    /**线程池初始限定大小*/
    private int threadNum;
    /**已经工作的线程数目*/
    private int workThreadNum;

    private final ReentrantLock mainLock = new ReentrantLock();

    public MyThreadPool(int initPoolNum) {
        threadNum = initPoolNum;
        threads = new ArrayList<>(initPoolNum);
        //任务队列初始化为线程池线程数的四倍
        taskQueue = new ArrayBlockingQueue<>(initPoolNum*4);

        threadNum = initPoolNum;
        workThreadNum = 0;
    }

    public void execute(Runnable runnable) {
        try {
            mainLock.lock();
            //线程池未满，每加入一个任务则开启一个线程
            if(workThreadNum < threadNum) {
                MyThead myThead = new MyThead(runnable);
                myThead.start();
                threads.add(myThead);
                workThreadNum++;
            }
            //线程池已满，放入任务队列，等待有空闲线程时执行
            else {
                //队列已满，无法添加时，拒绝任务
                if(!taskQueue.offer(runnable)) {
                    rejectTask();
                }
            }
        } finally {
            mainLock.unlock();
        }
    }

    private void rejectTask() {
        System.out.println("任务队列已满，无法继续添加，请扩大您的初始化线程池！");
    }
    public static void main(String[] args) {
        MyThreadPool myThreadPool = new MyThreadPool(5);
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName()+"执行中");
            }
        };

        for (int i = 0; i < 20; i++) {
            myThreadPool.execute(task);
        }
    }

    class MyThead extends Thread{
        private Runnable task;

        public MyThead(Runnable runnable) {
            this.task = runnable;
        }
        @Override
        public void run() {
            //该线程一直启动着，不断从任务队列取出任务执行
            while (true) {
                //如果初始化任务不为空，则执行初始化任务
                if(task != null) {
                    task.run();
                    task = null;
                }
                //否则去任务队列取任务并执行
                else {
                    Runnable queueTask = taskQueue.poll();
                    if(queueTask != null)
                        queueTask.run();    
                }
            }
        }
    }
}
```
总结一下，这个自定义线程池的整个工作过程：
    
   - 初始化线程池，指定线程池的大小。
   - 向线程池中放入任务执行。
   - 如果线程池中创建的线程数目未到指定大小，则创建我们自定义的线程类放入线程池集合，并执行任务。执行完了后该线程会一直监听队列
   - 如果线程池中创建的线程数目已满，则将任务放入缓冲任务队列
   - 线程池中所有创建的线程，都会一直从缓存任务队列中取任务，取到任务马上执行


#----------------------第二部分：java线程池实现原理-----------------

## 2. java线程池实现原理

   java线程池的实现原理很简单，就是一个线程集合workerSet和一个阻塞队列workQueue。
   当用户向线程池提交一个任务(也就是线程)时，线程池会先将任务放入workQueue中。
   workerSet中的线程会不断的从workQueue中获取线程然后执行。
   当workQueue中没有任务的时候，worker就会阻塞，直到队列中有任务了就取出来继续执行。
   
   ![图片](./0.jpg)
   
####线程池的几个重要参数   
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler);
```
  注解：
  
  - corePoolSize: 规定线程池有几个线程(worker)在运行。
  - maximumPoolSize: 当workQueue满了,不能添加任务的时候，这个参数才会生效。规定线程池最多只能有多少个线程(worker)在执行。
  - keepAliveTime: 超出corePoolSize大小的那些线程的生存时间,这些线程如果长时间没有执行任务并且超过了keepAliveTime设定的时间，就会消亡。
  - unit: 生存时间对于的单位
  - workQueue: 存放任务的队列
  - threadFactory: 创建线程的工厂
  - handler: 当workQueue已经满了，并且线程池线程数已经达到maximumPoolSize，将执行拒绝策略。
 
#### 任务提交后的流程分析

用户通过submit提交一个任务。线程池会执行如下流程: 
1. 判断当前运行的worker数量是否超过corePoolSize,如果不超过corePoolSize。就创建一个worker直接执行该任务。—— 线程池最开始是没有worker在运行的 
2. 如果正在运行的worker数量超过或者等于corePoolSize,那么就将该任务加入到workQueue队列中去。 
3. 如果workQueue队列满了,也就是offer方法返回false的话，就检查当前运行的worker数量是否小于maximumPoolSize,如果小于就创建一个worker直接执行该任务。 
4. 如果当前运行的worker数量是否大于等于maximumPoolSize，那么就执行RejectedExecutionHandler来拒绝这个任务的提交。

#### 源码解析

##### ThreadPoolExecutor中的几个关键属性
```
//这个属性是用来存放 当前运行的worker数量以及线程池状态的
//int是32位的，这里把int的高3位拿来充当线程池状态的标志位,后29位拿来充当当前运行worker的数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

//存放任务的阻塞队列
private final BlockingQueue<Runnable> workQueue;

//worker的集合,用set来存放
private final HashSet<Worker> workers = new HashSet<Worker>();

//历史达到的worker数最大值
private int largestPoolSize;

//当队列满了并且worker的数量达到maxSize的时候,执行具体的拒绝策略
private volatile RejectedExecutionHandler handler;

//超出coreSize的worker的生存时间
private volatile long keepAliveTime;

//常驻worker的数量
private volatile int corePoolSize;

//最大worker的数量,一般当workQueue满了才会用到这个参数
private volatile int maximumPoolSize;
```

##### 提交任务相关源码
execute()方法的源码：
```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //workerCountOf(c)会获取当前正在运行的worker数量
        if (workerCountOf(c) < corePoolSize) {
            //如果workerCount小于corePoolSize,就创建一个worker然后直接执行该任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //isRunning(c)是判断线程池是否在运行中,如果线程池被关闭了就不会再接受任务
        //后面将任务加入到队列中
        if (isRunning(c) && workQueue.offer(command)) {
            //如果添加到队列成功了,会再检查一次线程池的状态
            int recheck = ctl.get();
            //如果线程池关闭了,就将刚才添加的任务从队列中移除
            if (! isRunning(recheck) && remove(command))
                //执行拒绝策略
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //如果加入队列失败,就尝试直接创建worker来执行任务
        else if (!addWorker(command, false))
            //如果创建worker失败,就执行拒绝策略
            reject(command);
}
```

##### 添加worker的方法`addWorker()`源码
```
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        //使用自旋+cas失败重试来保证线程竞争问题
        for (;;) {
            //先获取线程池的状态
            int c = ctl.get();
            int rs = runStateOf(c);

            // 如果线程池是关闭的,或者workQueue队列非空,就直接返回false,不做任何处理
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                //根据入参core 来判断可以创建的worker数量是否达到上限,如果达到上限了就拒绝创建worker
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //没有的话就尝试修改ctl添加workerCount的值。这里用了cas操作,如果失败了下一个循环会继续重试,直到设置成功
                if (compareAndIncrementWorkerCount(c))
                    //如果设置成功了就跳出外层的那个for循环
                    break retry;
                //重读一次ctl,判断如果线程池的状态改变了,会再重新循环一次
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            final ReentrantLock mainLock = this.mainLock;
            //创建一个worker,将提交上来的任务直接交给worker
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                //加锁,防止竞争
                mainLock.lock();
                try {
                    int c = ctl.get();
                    int rs = runStateOf(c);
                    //还是判断线程池的状态
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //如果worker的线程已经启动了,会抛出异常
                        if (t.isAlive()) 
                              throw new IllegalThreadStateException();
                        //添加新建的worker到线程池中
                        workers.add(w);
                        int s = workers.size();
                        //更新历史worker数量的最大值
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        //设置新增标志位
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                //如果worker是新增的,就启动该线程
                if (workerAdded) {
                    t.start();
                     //成功启动了线程,设置对应的标志位
                    workerStarted = true;
                }
            }
        } finally {
            //如果启动失败了,会触发执行相应的方法
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
}
```
#### Worker的结构
`Worker`是`ThreadPoolExecutor`内部定义的一个内部类。我们先看一下Worker的继承关系

```
private final class Worker extends AbstractQueuedSynchronizer implements Runnable
```

它实现了`Runnable`接口,所以可以拿来当线程用。同时它还继承了`AbstractQueuedSynchronizer`同步器类,主要用来实现一个不可重入的锁。

一些属性还有构造方法:
```
//运行的线程,前面addWorker方法中就是直接通过启动这个线程来启动这个worker
final Thread thread;
//当一个worker刚创建的时候,就先尝试执行这个任务
Runnable firstTask;
//记录完成任务的数量
volatile long completedTasks;
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    //创建一个Thread,将自己设置给他,后面这个thread启动的时候,也就是执行worker的run方法
    this.thread = getThreadFactory().newThread(this);
}
```

worker的`run()`方法：
```
public void run() {
    //这里调用了ThreadPoolExecutor的runWorker方法
    runWorker(this);
}
```

ThreadPoolExecutor的runWorker方法:
```
final void runWorker(Worker w) {
        //获取当前线程
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        //执行unlock方法,允许其他线程来中断自己
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            //如果前面的firstTask有值,就直接执行这个任务
            //如果没有具体的任务,就执行getTask()方法从队列中获取任务
            //这里会不断执行循环体,除非线程中断或者getTask()返回null才会跳出这个循环
            while (task != null || (task = getTask()) != null) {
                //执行任务前先锁住,这里主要的作用就是给shutdown方法判断worker是否在执行中的
                //shutdown方法里面会尝试给这个线程加锁,如果这个线程在执行,就不会中断它
                w.lock();
               //判断线程池状态,如果线程池被强制关闭了,就马上退出
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    //执行任务前调用。预留的方法,可扩展
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        //真正的执行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                       //执行任务后调用。预留的方法,可扩展
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    //记录完成的任务数量
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
}
```

下面来看一下`getTask()`方法，这里面涉及到`keepAliveTime`的使用，从这个方法我们可以看出线程池是怎么让超过corePoolSize的那部分worker销毁的。

```
private Runnable getTask() {
        boolean timedOut = false; 

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 如果线程池已经关闭了,就直接返回null,
            //如果这里返回null,调用的那个worker就会跳出while循环,然后执行完销毁线程
            //SHUTDOWN状态表示执行了shutdown()方法
            //STOP表示执行了shutdownNow()方法
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
            //获取当前正在运行中的worker数量
            int wc = workerCountOf(c);

            // 如果设置了核心worker也会超时或者当前正在运行的worker数量超过了corePoolSize,就要根据时间判断是否要销毁线程了
            //其实就是从队列获取任务的时候要不要设置超时间时间,如果超过这个时间队列还没有任务进来,就会返回null
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            //如果上一次循环从队列获取到的未null,这时候timedOut就会为true了
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                //通过cas来设置WorkerCount,如果多个线程竞争,只有一个可以设置成功
                //最后如果没设置成功,就进入下一次循环,说不定下一次worker的数量就没有超过corePoolSize了,也就不用销毁worker了
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                //如果要设置超时时间,就设置一下咯
                //过了这个keepAliveTime时间还没有任务进队列就会返回null,那worker就会销毁
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                //如果r为null,就设置timedOut为true
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
}
```
##### 添加Callable任务的实现源码
```
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```
要添加一个有返回值的任务的实现也很简单。其实就是对任务做了一层封装，将其封装成Future，然后提交给线程池执行，最后返回这个future。 
这里的 newTaskFor(task) 方法会将其封装成一个FutureTask类。 
外部的线程拿到这个future，执行get()方法的时候,如果任务本身没有执行完，执行线程就会被阻塞，直到任务执行完。 
下面是FutureTask的get方法
```
public V get() throws InterruptedException, ExecutionException {
        int s = state;
        //判断状态,如果任务还没执行完,就进入休眠,等待唤醒
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        //返回值
        return report(s);
}

```
FutureTask中通过一个state状态来判断任务是否完成。当run方法执行完后,会将state状态置为完成，同时唤醒所有正在等待的线程。
我们可以看一下`FutureTask`的`run()`方法:
```
public void run() {
        //判断线程的状态
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
                    //执行call方法
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    //这个方法里面会设置返回内容,并且唤醒所以等待中的线程
                    set(result);
            }
        } finally {
            runner = null;
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
}
```

##### `shutdown`和`shutdownNow`方法的实现

```
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //检查是否可以关闭线程
            checkShutdownAccess();
            //设置线程池状态
            advanceRunState(SHUTDOWN);
            //尝试中断worker
            interruptIdleWorkers();
             //预留方法,留给子类实现
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
}

private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
}

private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //遍历所有的worker
            for (Worker w : workers) {
                Thread t = w.thread;
                //先尝试调用w.tryLock(),如果获取到锁,就说明worker是空闲的,就可以直接中断它
                //注意的是,worker自己本身实现了AQS同步框架,然后实现的类似锁的功能
                //它实现的锁是不可重入的,所以如果worker在执行任务的时候,会先进行加锁,这里tryLock()就会返回false
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
}
```

`shutdownNow()`做的比较绝，它先将线程池状态设置为STOP，然后拒绝所有提交的任务。最后中断左右正在运行中的worker,然后清空任务队列。

```
public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            //检测权限
            advanceRunState(STOP);
            //中断所有的worker
            interruptWorkers();
            //清空任务队列
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
}

private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //遍历所有worker，然后调用中断方法
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
}
```





    