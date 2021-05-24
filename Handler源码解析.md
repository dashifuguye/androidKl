# Handler源码解析

作为Android系统架构的核心，handler通信实现方案实际上是内存共享的方案。

## handler消息调度流程：

- handler-》sendMessage   -> messasgeQueue.enqueueMessage   //消息队列队列的插入节点

- looper.loop()->  messasgeQueue.next（）-》handler.dispatchMessage()->handler.handleMessage（）

  ![image-20210522204855063](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522204855063.png)

工作流程如上图：sentMessage或者postMessage时会把消息message发送至MessageQueue，MessageQueue是一个单链表实现的优先级队列，调用Lopper.loop()开启整个消息循环，不断去轮询MessageQueue里的next()函数，去获取MessageQueue里执行时间最早的消息，如果还未到执行时间，则阻塞，如果达到执行时间，则会调用handler.dispatch()方法来处理消息，这样就完成了整个消息的通信，一般情况下，发送消息在子线程，接收消息在主线程，这样就实现了线程间通信。

发送消息：sendMessage/postMessage

<img src="C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522162838665.png" alt="image-20210522162838665" style="zoom:50%;" />



内存共享：传递的是Message，使用new Message()或者Message.obtain()创建，实际上是一块内存，子线程发送message，主线程处理message，即实现了内存共享。



## MessageQueue

MessageQueue的数据结构是单链表实现的优先级队列。每个message对象都有一个next属性，指向下一个message。入队时会按照message.when进行排序，时间早的排在前面。插入排序

![image-20210522164638929](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522164638929.png)

队列先进先出，出队时直接调用next方法。

![image-20210522165116306](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522165116306.png)

## Lopper

核心：构造函数、loop()函数、ThreadLocal

1 构造函数私有，通过prepare()创建。

![image-20210522165403581](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522165403581.png)

![image-20210522165510124](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522165510124.png)

2. ThreadLocal：线程隔离工具

   线程上下文变量，本身不存储数据，通过内部类ThreadLocalMap来保存数据。每个线程都有一个ThreadLocalMap类型的成员变量，用于存储线程相关的上下文环境。ThreadLocalMap内部是Entry，对ThreadLocal的一个弱引用，Entry是一个键值对，key是ThreadLocal，value是要保存的东西。在调用ThreadLocal的set()方法时，会首先获取当前线程对应的ThreadLocalMap，然后把ThreadLocal本身作为key和value存到map中。

   ![image-20210522171005399](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522171005399.png)

   在Lopper中，成员变量sThreadLocal 是static final类型，所以这个变量只有一个。当调用prepare()函数时，首先通过sThreadLocal的get()方法获取Lopper，如果不为null，则抛出异常，否则创建一个Lopper并调用sThreadLocal的set()方法将新建的Lopper作为Value保存到ThreadLocalMap中。由于每个线程有唯一的ThreadLocal，即ThreadLocalMap的键值唯一，所以value值也唯一，即lopper唯一。所以ThreadLocal可以保证一个线程只有一个looper。除此之外，每个Lopper都有一个final类型的MessageQueue mQueue，一旦创建就不能修改，mQueue是在Lopper的构造函数中创建的。所以一个线程只有一个MessageQueue。

   

 ## 常见面试题 

   1. 一个线程有几个Handler？ 

      可以通过new来创建任意个handler

   2. 一个线程有几个lopper？如何保证？

      见上文

   3. Handler内存泄漏原因？为什么其他的内部类没有这个问题？

      匿名内部类默认持有外部类的引用，所以handler持有activity的引用；

      msg.target = this，所以message持有handler的引用；![image-20210522180003600](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522180003600.png)

      如果message的生命周期比activity长，比如设置在20min后执行message，此时activity已经退出，根据GC的可达性算法，activity仍然可达，因为message没有回收，所以activity不能回收，于是就造成了内存泄漏。

      至于为什么其他的内部类没有这个问题，归根结底是生命周期问题，如果内部类被其他对象引用，而且生命周期长于外部类，就会出现内存泄漏。

   4. 为何主线程可以new Handler？如果想在子线程new Handler需要做哪些准备？

      主线程在启动的时候，即在ActivityThread的main()方法中，已经调用了Lopper.prepareMainLooper()和Lopper.loop()。

      ![image-20210522181042544](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522181042544.png)

      子线程必须调用Lopper.prepare()和Lopper.loop()。

   5. 子线程中维护的Lopper，消息队列无消息时的处理方案是什么？有什么用？

      在主线程中，消息队列无消息，会阻塞休眠；子线程中，消息队列无消息时，必须调用quit()，调用Lopper.quit()方法，会调用MessageQueue.quit()方法，此时把mQuitting赋值为true，并清空所有的消息，最后调用nativeWake()唤醒线程。由于mQuitting为true，MessageQueue.next()方法返回null。lopper开启后是一个死循环，一直在轮询，退出死循环的唯一方法是让msg==null。所以调用quit()可以退出lopper，整个lopper结束。

      ![image-20210522183819169](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522183819169.png)

      生产者消费者模式：子线程是生产者，通过enqueMessage()将消息放入消息队列；主线程是消费者，通过next()方法获取消息；MessageQueue是仓库，存储消息。

      两个方面的阻塞？
      
      入队：根据时间排序，当队列满的时候，阻塞，直到用户通过next取出消息。当next方法被调用，通知MessagQueue可以进行消息的入队。
      
      出队：由Looper.loop(),启动轮询器，对queue进行轮询。当消息达到执行时间就取出来。当message queue为空的时候，队列阻塞，等消息队列调用enqueuer Message的时候，通知队列，可以取出消息，停止阻塞。
      
      1） message 不到时间 ，自动唤醒
      2）  messageQueue为空，无限等待

   <img src="C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522184106225.png" alt="image-20210522184106225" style="zoom:67%;" />

   6. 既然可以存在多个 Handler 往 MessageQueue 中添加数据（发消息时各个 Handler 可能处于不同线程），那它内部是如何确保线程安全的？

      通过锁保证。enqueMessage()和next()方法中都加了synchronized关键字，synchronized是内置锁，synchronized可以修饰方法或者代码块，当方法或者代码块执行完就释放锁，整个上锁和解锁的过程都是JVM帮我们完成的，是内置到系统里面的锁，所以叫内置锁。

      ![image-20210522192933375](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522192933375.png)

      synchronized(this){}，使用this，锁住的是整个对象，使用同一个对象对MessageQueue的访问都会受限，由于一个线程只有一个lopper，只有一个messagequeue，所以一个线程只有一个地方可以操作messagequeue。

      引申问题：next()加锁原因：虽然都是直接取头部消息，但插入的消息有可能比队列头部的消息要早，所以对头的消息是可能变化的，所以需要加锁。所以quit()方法也加锁，因为quit()会清空消息。

      

      主线程调用quit()会抛出异常，而且程序会退出。

      

   7. 我们使用 Message 时应该如何创建它？

      涉及到享元设计模式，有两种方式：new Message()和obtain()。每次loop取出一个消息之后，调用dispatchMesssage()处理后，没有直接释放message内存，而是调用msg.recycleUnchecked()进行回收，即将msg的所有属性都至null，并把这个消息放到sPool里，sPool就是一个Message，形成了一个缓存message的缓存池，可以避免内存抖动，防止oom。如果使用new Message()，会申请一块内存，由于android里无时无刻不在产生消息，所以会产生很多不连续内存块被使用，形成大量的内存碎片，造成内存抖动，甚至OOM。bitmap的加载也可以这样，内存复用。

      ![image-20210522210948370](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522210948370.png)<img src="C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210522211146637.png" alt="image-20210522211146637" style="zoom:80%;" />

   8. Looper死循环为什么不会导致应用卡死

      ANR原因：

       - 点击事件5s没响应
       - 广播10s没响应
       - service 20s没响应

      looper死循环与ANR是两个毫不相关的问题，点击事件以及广播等都是message，将消息发送出去进行处理，处理完之后会进行callback，如果message没有及时处理，就会让handler发送一个anr的弹窗提醒，所以anr本身的弹窗就是由handler触发的，所以anr与looper的死循环是没有关系的。

      looper的block不会导致anr，因为anr也是一个消息，因为block只不过是线程没事做，没事做就应该交出cpu，来睡眠省电，等待就是让自身休眠，没有消息处理，包括anr、点击事件都没有形成消息，都不需要处理。只不过用户点击事件，就会发送出来一个消息，然后交给handler处理。

## 同步屏障

消息是根据执行时间进行先后排序，然后消息是保存在队列中，因而消息只能从队列的队头取出来。

那么问题来了！需要紧急处理的消息怎么办？   

同步屏障就是阻碍同步消息，只让异步消息通过。如何开启同步屏障呢？调用 MessageQueue的postSyncBarrier(）方法。代码如下

```java
/**
*
@hide
**/
public int postSyncBarrier() {
return postSyncBarrier(SystemClock.uptimeMillis());
}
private int postSyncBarrier(long when) {
// Enqueue a new sync barrier token
synchronized (this) {
final int token = mNextBarrierToken++;
//从消息池中获取Message
final Message msg = Message.obtain();
msg.markInUse();
//就是这里！！！初始化Message对象的时候，并没有给target赋值，因此 target==null
msg.when = when;
msg.arg1 = token;
Message prev = null;
Message p = mMessages;
if (when != 0) {
while (p != null && p.when <= when) {
//如果开启同步屏障的时间（假设记为T）T不为0，且当前的同步消息里有时间小于T，则prev也不为null
prev = p;
p = p.next;
}
}
//根据prev是不是为null，将 msg 按照时间顺序插入到 消息队列（链表）的合适位置
if (prev != null) { // invariant: p == prev.next
msg.next = p;
prev.next = msg;
} else {
msg.next = p;
mMessages = msg;
}
return token;
}
}

```

message.target = null，即是同步屏障。

在MessageQueue的next()方法中，有这样的代码，

```java
//关键！！！
//如果target==null，那么它就是屏障，需要循环遍历，一直往后找到第一个异步的消息
if (msg != null && msg.target == null) {
do {
prevMsg = msg;
msg = msg.next;
} while (msg != null && !msg.isAsynchronous());
}

```

如果taget==null，则进行do while循环，直到找到第一个不为空且isAsynchronous为true的消息，即异步消息，将这个消息返回进行处理。当消息队列开启同步屏障的时候（即标识为 msg.target == null ），消息机制在处理消息的时候，优先处理异步消息。这样，同步屏障就起到了一种过滤和优先级的作用。由于同步屏障的存在，每次调用next()方法都会搜索一遍。那么这 些同步消息什么时候可以被处理呢？那就需要先移除这个同步屏障，即调用 removeSyncBarrier() 。

Android 系统中 的 UI 更新相关的消息即为异步消息，需要优先处理。60Hz刷新频率，16ms更新一次。 比如，在 View 更新时，draw、requestLayout、invalidate 等很多地方都调用了 ViewRootImpl#scheduleTraversals() ，如下

//ViewRootImpl.java

```java
void scheduleTraversals() {
if (!mTraversalScheduled) {
mTraversalScheduled = true;
//开启同步屏障
mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
//发送异步消息
mChoreographer.postCallback(
Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
if (!mUnbufferedInputDispatch) {
scheduleConsumeBatchedInput();
}
notifyRendererOfFramePending();
pokeDrawLockIfNeeded();
}
}
```

postCallback() 最终走到了 ChoreographerpostCallbackDelayedInternal() ：

```java
private void postCallbackDelayedInternal(int callbackType,
Object action, Object token, long delayMillis) {
if (DEBUG_FRAMES) {
Log.d(TAG, "PostCallback: type=" + callbackType- ", action=" + action + ",
token=" + token =" + delayMillis);
}
synchronized (mLock) {
final long now = SystemClock.uptimeMillis();
final long dueTime = now + delayMillis;
mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
if (dueTime <= now) {
scheduleFrameLocked(now);
} else {
Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
msg.arg1 = callbackType;
msg.setAsynchronous(true); //异步消息
mHandler.sendMessageAtTime(msg, dueTime);
}
}
}
```

这里就开启了同步屏障，并发送异步消息，由于 UI 更新相关的消息是优先级最高的，这样系统就会优先处理这些异 步消息。 最后，当要移除同步屏障的时候需要调用 ViewRootImpl#unscheduleTraversals() 。

```java
void unscheduleTraversals() {
if (mTraversalScheduled) {
mTraversalScheduled = false;
//移除同步屏障
mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
mChoreographer.removeCallbacks(
Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
}
}
```

小结 同步屏障的设置可以方便地处理那些优先级较高的异步消息。当我们调用 Handler.getLooper().getQueue().postSyncBarrier() 并设置消息的 setAsynchronous(true) 时，target 即 为 null ，也就开启了同步屏障。当在消息轮询器 Looper 在 loop() 中循环处理消息时，如若开启了同步屏障，会优 先处理其中的异步消息，而阻碍同步消息。



## HandlerThread

HandlerThread是Thread的子类，严格意义上来说就是一个线程，只是它在自己的线程里面帮我们创建了Looper HandlerThread 存在的意义如下：

1） 方便使用：a. 方便初始化，b，方便获取线程looper 2）保证了线程安全

 我们一般在Thread里面 线程Looper进行初始化的代码里面，必须要对Looper.prepare(),同时要调用Loop。 loop（）

```java
@Override
public void run() {
Looper.prepare();
Looper.loop();
}
```



 而我们要使用子线程中的Looper的方式是怎样的呢？看下面的代码

```java
Thread thread = new Thread(new Runnable() {
Looper looper;
@Override
public void run() {
// Log.d(TAG, "click2: " + Thread.currentThread().getName());
Looper.prepare();
looper =Looper.myLooper();
Looper.loop();
}
public Looper getLooper() {
return looper;
}
});
thread.start();
Handler handler = new Handler(thread.getLooper());
```

 上面这段代码有没有问题呢？

 1）在初始化子线程的handler的时候，我们无法将子线程的looper传值给Handler,解决办法有如下办法： 

	- 可以将Handler的初始化放到 Thread里面进行 
 - 可以创建一个独立的类继承Thread，然后，通过类的对象获取。 

这两种办法都可以，但是，这个工作 HandlerThread帮我们完成了

 2）依据多线程的工作原理，我们在上面的代码中，调用 thread.getLooper（）的时候，此时的looper可能还没有初始化，此时是不是可能会挂掉呢？ 以上问题 HandlerThread 已经帮我们完美的解决了，这就是 handlerThread存在的必要性了。 我们再看 HandlerThread源码 

```java
public void run() {
mTid = Process.myTid();
Looper.prepare();
synchronized (this) {
mLooper = Looper.myLooper();
notifyAll(); //此时唤醒其他等待的锁，但是
}
Process.setThreadPriority(mPriority);
onLooperPrepared();
Looper.loop();
mTid = -1;
}

```

![image-20210523102155001](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210523102155001.png)

它的优点就在于它的多线程操作，可以帮我们保证使用Thread的handler时一定是安全的。

注意：wait会立即释放锁；notify/notifyAll不会释放锁，而是会唤醒调用wait的线程，进入就绪状态，但这个线程不会马上继续执行，而是继续等待，等synchronized(){}中的代码块执行完成后才释放锁，然后wait的线程开始执行。





## IntentService

继承service，并且是一个抽象类，因此必须创建它的子类才能使用IntentService。它可用于执行后台耗时任务，任务执行后自动停止，同时由于IntentService是服务的原因，导致它的优先级比单纯的线程要高很多，所以比较适合执行一些高优先级的后台任务，因为优先级高不容易被系统杀死。在实现上，IntentService封装了HandlerThread和Handler。

![image-20210523103726522](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210523103726522.png)

