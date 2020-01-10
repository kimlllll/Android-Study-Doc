### 准备知识
为什么要学习ThreadLocal呢？ 因为这个类是线程中存储数据的类，在Android处理消息的过程中，它其中存储了主线程的Looper，而Looper中又创建了MessageQueue。在App启动的过程中，主线程把Looper放入ThreadLocal中，在处理消息的过程中，子线程通过myLooper()方法获取到主线程中的Looper和MessageQueue，从而把主线程和子线程联系起来。
### 1.ThreadLocal 


##### 1.1 什么是ThreadLocal
ThreadLocal : 是一个线程内部的数据存储类，它其中维护了一个ThreadLocalMap。 

##### 1.2 ThreadLocal的初始化方法：initialValue()。
源码中对该方法的实现是：
```
     protected T initialValue() {
        return null;
       }
```
默认返回为空的。

##### 1.3 从ThreadLocal中取数据:get()
```
     public T get() {
        //拿到当前线程 
        Thread t = Thread.currentThread();
        //拿到当前线程中的 ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        //通过下面两步，追踪可以看到，map此时为空。
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
点击进入源码发现这个方法首先是获取到当前的线程，然后拿到当前线程中的ThreadLocalMap。我们追踪其中的getMap(t):
```
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
返回的是threadLocals，这是Thread类中的一个全局变量。追踪进去可以看到：
Thread.java
```
 ThreadLocal.ThreadLocalMap threadLocals = null;
  threadLocals = null;
```
在Thread.java中，对这个全局变量的定义均为null。因此在get()中，map为空，会走到setInitialValue()中。我们继续追踪到setInitialValue()中。看看setInitialValue()在源码中如何实现的：
```
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
首先调用初始化方法initialValue()，得到value，如果用户重写了initialValue()，那么获得到就是用户定义的返回值。再往下走，得到的Map仍然是null，因此会走到createMap(t, value)中。继续追踪下去，看看createMap()是怎么实现的。
```
  void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
在这个方法中，定义了threadLocals这个变量。创建了ThreadLocalMap。（后面补充）

由上面可以看出来，get()返回的是initialValue()返回的值。

#### 1.4 往ThreadLocal中添加数据：set()
```
   public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

```
仍然是这样：首先获取当前线程，再通过当前线程获取到 ThreadLocalMap，在之前通过对getMap(t)的分析可以知道，此时的map = null，因此set(t)最终也会走 createMap(t, value)。之前对 createMap(t, value)的源码进行分析过， createMap(t, value)会创建一个ThreadLocalMap，并将value放入ThreadLocalMap中。

#### 1.5 
调用完set(t)再调用get(),此时ThreadLocalMap已经被创建，在get()中会走map!=null的逻辑。也就是下面的一段代码：
```
       ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
```
可以很清楚的看到，这时返回的就是e.value，也就是通过set(t)放入的t。

总结：
没有往ThreadLocal中set数据之前，通过ThreadLocal的get()取数据，获得的是initialValue()返回的数据。
#### 使用set()往ThreadLocal中添加数据之后，再使用get()取数据，获得的就是set()进去的数据。具体的原因在前面已经给出了。


###  2.从App的启动开始分析，Android的消息机制。

当App启动时，会创建全局唯一的Looper和MessageQueue对象。
这句话如何验证呢？当然是去看源码。

App的程序入口在哪里呢？在ActivityThread的main()这里。这个main()就是我们Java当中的main(),也就是程序的入口。
在main()中，调用了Looper.prepareMainLooper()。这个方法创建了全局唯一的MainLooper。我们来看一下这个方法是如何实现的：
```
 public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```
首先看 prepare(false) 这句，追踪到 prepare(false)中：
```
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

```
在这个方法中往sThreadLocal中set了一个Looper，再往下走，myLooper()，看看myLooper()是如何实现的：
```
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
可以非常清楚的看到，这里调用了ThreaLoacl的get(),通过刚才对ThreadLocal的分析，在set()调用后，get()
获得的就是set进ThreaLocal中的数据。
接下来再进入Looper的构造方法中看一看：
```
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```
这里创建了MessageQueue。

这里的一切，都是在主线程中进行的。

### 3.发送消息
Handler登上舞台，发送消息离不开Handler。

如果要从子线程中，发送消息到主线程，那么，是这样的写法：
1. 创建Handler
2. 创建子线程，在子线程中进行网络请求等操作。
3. 子线程网络请求结束，通过handler发送消息给主线程，通知主线程。
MainActivity.java
```
public class MainActivity extends AppCompatActivity {

//1.创建Handler
Handler mHandler = new Handler(){
    @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
};

//2. 创建子线程，在子线程中进行网络请求等操作。
new Thread(new Runnable() {
            @Override
            public void run() {
                //.....网络请求等操作

                //3. 子线程网络请求结束，通过handler发送消息给主线程，通知主线程。
                Message message = Message.obtain();
                message.what = 1;
                mHandler.sendMessage(message);

            }
        }).start();
 
}

```

基本上一个Handler发送消息的过程就是这样。那么到底是怎样通过Handler把消息从子线程发送到主线程的呢？
看源码： 首先看Handler的源码。我们首先分析的是sendMessage()
```
public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
    
    
    ....
    
    
    
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
    ....
    
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

```
一步一步追踪下来，我们可以看到，最终调用的是sendMessageAtTime()，首先看这个方法的第一句：
MessageQueue queue = mQueue;mQueue在Handler中赋值的过程是这样的：

```
 public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    
```
可以通过 mQueue = mLooper.mQueue;mLooper = Looper.myLooper();这两句代码得到：mLooper是主线程中的Looper，而mQueue是主线程中的MessageQueue。
怎么得到这一结论的呢？ 通过mLooper = Looper.myLooper();这一关键代码得到的。前面已经分析过myLooper()。
```
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
myLooper()，就是获取ThreadLocal中的数据。而ThreadLocal中的数据就是主线程中的Looper。既然Looper是主线程中的Looper，那么mQueue当然就是Looper构造方法中创建的那一个MessageQueue了。分析到这里，我们就发现了，此时子线程和主线程已经联系起来了。


再接着分析sendMessageAtTime()，此时，我们已经知道，sendMessageAtTime()中的MessageQueue被主线程的mQueue赋值，也就是说sendMessageAtTime()中的MessageQueue就是主线程的MessageQueue。此时mQueue!=null 程序往下走到enqueueMessage(queue, msg, uptimeMillis);这里来。
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

```
分析enqueueMessage：


msg.target = this;这里的this就是此时这个Handler。首先点进Message的源码中，target就是一个Handler。此时将这个Handler赋值给msg，在处理消息的过程中，会使用这个Handler进行消息的分发。


queue.enqueueMessage(msg, uptimeMillis);
在这里就是往主线程的MessageQueue中插入Message了。
```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                //把msg赋值给全局变量mMessages
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
这里的关键代码,是一个死循环,MessageQueue就是一个链表。当有消息到来时，就会往链表中插入一条消息。也就是说，这时，主线程的MessageQueuq中已经有一条消息了。

### 4.处理消息：
在ActivityThread的main()中，下面有一句
Looper.loop();我们追踪进去看一下：
```
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // Allow overriding a threshold with a system prop. e.g.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        boolean slowDeliveryDetected = false;

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
            long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
            if (thresholdOverride > 0) {
                slowDispatchThresholdMs = thresholdOverride;
                slowDeliveryThresholdMs = thresholdOverride;
            }
            final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
            final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

            final boolean needStartTime = logSlowDelivery || logSlowDispatch;
            final boolean needEndTime = logSlowDispatch;

            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            try {
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (logSlowDelivery) {
                if (slowDeliveryDetected) {
                    if ((dispatchStart - msg.when) <= 10) {
                        Slog.w(TAG, "Drained");
                        slowDeliveryDetected = false;
                    }
                } else {
                    if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                            msg)) {
                        // Once we write a slow delivery log, suppress until the queue drains.
                        slowDeliveryDetected = true;
                    }
                }
            }
            if (logSlowDispatch) {
                showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```
代码很长 ，我们首先抽出最关键代码进行分析：
```
 final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
        
        
        
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
           
            try {
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            msg.recycleUnchecked();
        }
```
首先，还是通过myLooper()，取得ThreadLocal中的Looper，再取得Looper中的MessageQueue，此时获取的Looper和MessageQueue都是主线程中的。
接下来是一个死循环：for(;;) 在其中通过queue.next()，不断的从MessageQueue中取消息，接下来  if (msg == null) return;也就是说，当消息为空时，不执行下面的操作，只有消息不为null走到下面一句msg.target.dispatchMessage(msg);使用msg中的Handler，进行消息的分发。
继续跟踪到dispatchMessage(msg)中：
```
public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
可以看到，这里出现了handleMessage(msg); 
```
public void handleMessage(Message msg) {
    }
```
这个handleMessage(Message msg)就是我们重写的Handler中的handleMessage(Message msg)，而这个消息就被传递过来，供我们使用。

大致的流程就是这样。

