
#  Handler的运行机制

* [Handler的作用](#Handler的作用)
* [Handler的使用演示](#Handler的使用演示)
* [Handler的源码分析](#Handler的源码分析)
* [MessageQueue的工作原理](#MessageQueue的工作原理)
* [Looper的工作原理](#Looper的工作原理)
* [Handler的运行机制](#Handler的运行机制)
* [Handler使用案例](#Handler使用案例)
* [Handler存在的问题](#Handler存在的问题)
* [Handler的改进](#Handler的改进)
* [Handler的使用实现](#Handler的使用实现)

## Handler的作用

 　　当我们需要在子线程处理耗时的操作（例如访问网络，数据库的操作），而当耗时的操作完成后，需要更新UI，这就需要使用Handler来处理，因为子线程不能做更新UI的操作。Handler能帮我们很容易的把任务（在子线程处理）切换回它所在的线程。简单理解，Handler就是解决线程和线程之间的通信的。

## Handler的使用演示

使用的handler的两种形式： 

1. 在主线程使用handler； 
2. 在子线程使用handler。

在主线程使用handler的示例：

	public class TestHandlerActivity extends AppCompatActivity {
	
	
	        private static final String TAG = "TestHandlerActivity";
	
	        private Handler mHandler = new Handler(){
	            @Override
	            public void handleMessage(Message msg) {
	                super.handleMessage(msg);
	                //获得刚才发送的Message对象，然后在这里进行UI操作
	                Log.e(TAG,"------------> msg.what = " + msg.what);
	            }
	        };
	
	
	        @Override
	        protected void onCreate(Bundle savedInstanceState) {
	            super.onCreate(savedInstanceState);
	            setContentView(R.layout.activity_handler_test);
	            initData();
	        }
	
	        private void initData() {
	
	            //开启一个线程模拟处理耗时的操作
	            new Thread(new Runnable() {
	                @Override
	                public void run() {
	
	                    SystemClock.sleep(2000);
	                    //通过Handler发送一个消息切换回主线程（mHandler所在的线程）
	                    mHandler.sendEmptyMessage(0);
	                }
	            }).start();
	
	        }   

![显示结果](http://img.blog.csdn.net/20160417192159801)

在主线程使用handler很简单，只需在主线程创建一个handler对象，在子线程通过在主线程创建的handler对象发送Message，在handleMessage（）方法中接受这个Message对象进行处理。通过handler很容易的从子线程切换回主线程了。

那么来看看在子线程中使用是否也是如此。

	public class TestHandlerActivity extends AppCompatActivity {
	
	
	        private static final String TAG = "TestHandlerActivity";
	        //主线程中的handler
	        private Handler mHandler = new Handler(){
	            @Override
	            public void handleMessage(Message msg) {
	                super.handleMessage(msg);
	                //获得刚才发送的Message对象，然后在这里进行UI操作
	                Log.e(TAG,"------------> msg.what = " + msg.what);
	            }
	        };
	        //子线程中的handler
	        private Handler mHandlerThread = null;
	
	        @Override
	        protected void onCreate(Bundle savedInstanceState) {
	            super.onCreate(savedInstanceState);
	            setContentView(R.layout.activity_handler_test);
	            initData();
	        }
	
	        private void initData() {
	
	            //开启一个线程模拟处理耗时的操作
	            new Thread(new Runnable() {
	                @Override
	                public void run() {
	
	                    SystemClock.sleep(2000);
	                    //通过Handler发送一个消息切换回主线程（mHandler所在的线程）
	                    mHandler.sendEmptyMessage(0);
	                    //在子线程中创建Handler
	                    mHandlerThread = new Handler(){
	                        @Override
	                        public void handleMessage(Message msg) {
	                            super.handleMessage(msg);
	                            Log.e("sub thread","---------> msg.what = " + msg.what);
	                        }
	                    };
	
	                    mHandlerThread.sendEmptyMessage(1);
	                }
	            }).start();
	
	        }

![显示结果](http://img.blog.csdn.net/20160417192248099)

程序崩溃了。报的错误是没有在子线程调用Looper.prepare()的方法。而为什么在主线程中使用不会报错？通过源码的分析可以解析这个问题。

在子线程中正确的使用Handler应该是这样的。

	public class TestHandlerActivity extends AppCompatActivity {
	
	        private static final String TAG = "TestHandlerActivity";
	
	        //主线程的Handler
	        private Handler mHandler = new Handler(){
	            @Override
	            public void handleMessage(Message msg) {
	                super.handleMessage(msg);
	                //获得刚才发送的Message对象，然后在这里进行UI操作
	                Log.e(TAG,"------------> msg.what = " + msg.what);
	            }
	        };
	        //子线程中的Handler
	        private Handler mHandlerThread = null;
	
	        @Override
	        protected void onCreate(Bundle savedInstanceState) {
	            super.onCreate(savedInstanceState);
	            setContentView(R.layout.activity_handler_test);
	            initData();
	        }
	
	        private void initData() {
	
	            //开启一个线程模拟处理耗时的操作
	            new Thread(new Runnable() {
	                @Override
	                public void run() {
	
	                    SystemClock.sleep(2000);
	                    //通过Handler发送一个消息切换回主线程（mHandler所在的线程）
	                    mHandler.sendEmptyMessage(0);
	
	                    //调用Looper.prepare（）方法
	                    Looper.prepare();
	
	                    mHandlerThread = new Handler(){
	                        @Override
	                        public void handleMessage(Message msg) {
	                            super.handleMessage(msg);
	                            Log.e("sub thread","---------> msg.what = " + msg.what);
	                        }
	                    };
	
	                    mHandlerThread.sendEmptyMessage(1);
	
	                    //调用Looper.loop（）方法
	                    Looper.loop();
	                }
	            }).start();
	
	        }

![显示结果](http://img.blog.csdn.net/20160417192312084)

可以看到，通过调用Looper.prepare()运行正常，handleMessage方法中就可以接收到发送的Message。

至于为什么要调用这个方法呢？去看看源码。

## Handler的源码分析

Handler的消息处理主要有五个部分组成，Message，Handler，Message Queue，Looper和ThreadLocal。首先简要的了解这些对象的概念

Message：Message是在线程之间传递的消息，它可以在内部携带少量的数据，用于线程之间交换数据。Message有四个常用的字段，what字段，arg1字段，arg2字段，obj字段。what，arg1，arg2可以携带整型数据，obj可以携带object对象。

Handler：它主要用于发送和处理消息的发送消息一般使用sendMessage（）方法,还有其他的一系列sendXXX的方法，但最终都是调用了sendMessageAtTime方法，除了sendMessageAtFrontOfQueue（）这个方法

而发出的消息经过一系列的辗转处理后，最终会传递到Handler的handleMessage方法中。

Message Queue：MessageQueue是消息队列的意思，它主要用于存放所有通过Handler发送的消息，这部分的消息会一直存在于消息队列中，等待被处理。每个线程中只会有一个MessageQueue对象。

Looper：每个线程通过Handler发送的消息都保存在，MessageQueue中，Looper通过调用loop（）的方法，就会进入到一个无限循环当中，然后每当发现Message Queue中存在一条消息，就会将它取出，并传递到Handler的handleMessage（）方法中。每个线程中只会有一个Looper对象。

ThreadLocal：MessageQueue对象，和Looper对象在每个线程中都只会有一个对象，怎么能保证它只有一个对象，就通过ThreadLocal来保存。Thread Local是一个线程内部的数据存储类，通过它可以在指定线程中存储数据，数据存储以后，只有在指定线程中可以获取到存储到数据，对于其他线程来说则无法获取到数据。

了解了这些基本概念后，我们深入源码来了解Handler的工作机制。

## MessageQueue的工作原理

MessageQueue消息队列是通过一个单链表的数据结构来维护消息列表的。下面主要看enqueueMessage方法和next（）方法。如下：

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

可以看出，在这个方法里主要是根据时间的顺序向单链表中插入一条消息。

next（）方法。如下

    Message next() {
            // Return here if the message loop has already quit and been disposed.
            // This can happen if the application tries to restart a looper after quit
            // which is not supported.
            final long ptr = mPtr;
            if (ptr == 0) {
                return null;
            }

            int pendingIdleHandlerCount = -1; // -1 only during first iteration
            int nextPollTimeoutMillis = 0;
            for (;;) {
                if (nextPollTimeoutMillis != 0) {
                    Binder.flushPendingCommands();
                }

                nativePollOnce(ptr, nextPollTimeoutMillis);

                synchronized (this) {
                    // Try to retrieve the next message.  Return if found.
                    final long now = SystemClock.uptimeMillis();
                    Message prevMsg = null;
                    Message msg = mMessages;
                    if (msg != null && msg.target == null) {
                        // Stalled by a barrier.  Find the next asynchronous message in the queue.
                        do {
                            prevMsg = msg;
                            msg = msg.next;
                        } while (msg != null && !msg.isAsynchronous());
                    }
                    if (msg != null) {
                        if (now < msg.when) {
                            // Next message is not ready.  Set a timeout to wake up when it is ready.
                            nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                        } else {
                            // Got a message.
                            mBlocked = false;
                            if (prevMsg != null) {
                                prevMsg.next = msg.next;
                            } else {
                                mMessages = msg.next;
                            }
                            msg.next = null;
                            if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                            msg.markInUse();
                            return msg;
                        }
                    } else {
                        // No more messages.
                        nextPollTimeoutMillis = -1;
                    }

                    // Process the quit message now that all pending messages have been handled.
                    if (mQuitting) {
                        dispose();
                        return null;
                    }

                    // If first time idle, then get the number of idlers to run.
                    // Idle handles only run if the queue is empty or if the first message
                    // in the queue (possibly a barrier) is due to be handled in the future.
                    if (pendingIdleHandlerCount < 0
                            && (mMessages == null || now < mMessages.when)) {
                        pendingIdleHandlerCount = mIdleHandlers.size();
                    }
                    if (pendingIdleHandlerCount <= 0) {
                        // No idle handlers to run.  Loop and wait some more.
                        mBlocked = true;
                        continue;
                    }

                    if (mPendingIdleHandlers == null) {
                        mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                    }
                    mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
                }

                // Run the idle handlers.
                // We only ever reach this code block during the first iteration.
                for (int i = 0; i < pendingIdleHandlerCount; i++) {
                    final IdleHandler idler = mPendingIdleHandlers[i];
                    mPendingIdleHandlers[i] = null; // release the reference to the handler

                    boolean keep = false;
                    try {
                        keep = idler.queueIdle();
                    } catch (Throwable t) {
                        Log.wtf(TAG, "IdleHandler threw exception", t);
                    }

                    if (!keep) {
                        synchronized (this) {
                            mIdleHandlers.remove(idler);
                        }
                    }
                }

                // Reset the idle handler count to 0 so we do not run them again.
                pendingIdleHandlerCount = 0;

                // While calling an idle handler, a new message could have been delivered
                // so go back and look again for a pending message without waiting.
                nextPollTimeoutMillis = 0;
            }
        }

在next方法是一个无限循环的方法，如果有消息返回这条消息并从链表中移除，而没有消息则一直阻塞在这里。

## Looper的工作原理

每个程序都有一个入口，而Android程序是基于java的，java的程序入口是静态的main函数，因此Android程序的入口也应该为静态的main函数，在android程序中这个静态的main在ActivityThread类中。我们来看一下这个main方法，如下：

     public static void main(String[] args) {
            SamplingProfilerIntegration.start();

            // CloseGuard defaults to true and can be quite spammy.  We
            // disable it here, but selectively enable it later (via
            // StrictMode) on debug builds, but using DropBox, not logs.
            CloseGuard.setEnabled(false);

            Environment.initForCurrentUser();

            // Set the reporter for event logging in libcore
            EventLogger.setReporter(new EventLoggingReporter());

            Security.addProvider(new AndroidKeyStoreProvider());

            // Make sure TrustedCertificateStore looks in the right place for CA certificates
            final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
            TrustedCertificateStore.setDefaultUserDirectory(configDir);

            Process.setArgV0("<pre-initialized>");
            //######
            Looper.prepareMainLooper();

            ActivityThread thread = new ActivityThread();
            thread.attach(false);

            if (sMainThreadHandler == null) {
                sMainThreadHandler = thread.getHandler();
            }

            if (false) {
                Looper.myLooper().setMessageLogging(new
                        LogPrinter(Log.DEBUG, "ActivityThread"));
            }

            Looper.loop();

            throw new RuntimeException("Main thread loop unexpectedly exited");
        }

在main方法中系统调用了 Looper.prepareMainLooper();来创建主线程的Looper以及MessageQueue，并通过Looper.loop（）来开启主线程的消息循环。来看看Looper.prepareMainLooper()是怎么创建出这两个对象的。如下：

     public static void prepareMainLooper() {
            prepare(false);
            synchronized (Looper.class) {
                if (sMainLooper != null) {
                    throw new IllegalStateException("The main Looper has already been prepared.");
                }
                sMainLooper = myLooper();
            }
        }

可以看到，在这个方法中调用了 prepare(false);方法和 myLooper();方法，我在进入这个两个方法中，如下：

     private static void prepare(boolean quitAllowed) {
            if (sThreadLocal.get() != null) {
                throw new RuntimeException("Only one Looper may be created per thread");
            }
            sThreadLocal.set(new Looper(quitAllowed));
        }

在这里可以看出，sThreadLocal对象保存了一个Looper对象，首先判断是否已经存在Looper对象了，以防止被调用两次。sThreadLocal对象是ThreadLocal类型，因此保证了每个线程中只有一个Looper对象。Looper对象是什么创建的，我们进入看看，如下：

	  private Looper(boolean quitAllowed) {
	            mQueue = new MessageQueue(quitAllowed);
	            mThread = Thread.currentThread();
	        }

可以看出，这里在Looper构造函数中创建出了一个MessageQueue对象和保存了当前线程。从上面可以看出一个线程中只有一个Looper对象，而Message Queue对象是在Looper构造函数创建出来的，因此每一个线程也只会有一个MessageQueue对象。

对prepare方法还有一个重载的方法：如下

	  public static void prepare() {
	            prepare(true);
	        }

prepare()仅仅是对prepare(boolean quitAllowed) 的封装而已，在这里就很好解释了在主线程为什么不用调用Looper.prepare（）方法了。因为在主线程启动的时候系统已经帮我们自动调用了Looper.prepare()方法。

在Looper.prepareMainLooper（）方法中还调用了一个方法myLooper()，我们进去看看，如下：

    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }

在调用prepare（）方法中在当前线程保存一个Looper对象sThreadLocal.set(new Looper(quitAllowed));my Looper（）方法就是取出当前线程的Looper对象，保存在sMainLooper引用中。

在main（）方法中还调用了Looper.loop()方法，如下：

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

            for (;;) {
                Message msg = queue.next(); // might block
                if (msg == null) {
                    // No message indicates that the message queue is quitting.
                    return;
                }

                // This must be in a local variable, in case a UI event sets the logger
                Printer logging = me.mLogging;
                if (logging != null) {
                    logging.println(">>>>> Dispatching to " + msg.target + " " +
                            msg.callback + ": " + msg.what);
                }

                msg.target.dispatchMessage(msg);

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

                msg.recycle();
            }
    }


在这个方法里，进入一个无限循环，不断的从MessageQueue的next方法获取消息，而next方法是一个阻塞操作，当没有消息的时候一直在阻塞，当有消息通过 msg.target.dispatchMessage(msg);这里的msg.target其实就是发送给这条消息的Handler对象。

## Handler的运行机制

看看Handler的构造方法。如下：

    public Handler(Callback callback) {
            this(callback, false);
        }

        public Handler(Looper looper) {
            this(looper, null, false);
        }

        public Handler(Looper looper, Callback callback) {
            this(looper, callback, false);
        }

我们去看看没有Looper 对象的构造方法：

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
                    "Can't create handler inside thread that has not called Looper.prepare()");
            }
            mQueue = mLooper.mQueue;
            mCallback = callback;
            mAsynchronous = async;
        }

可以看到，到looper对象为null，抛出 “Can’t create handler inside thread that has not called Looper.prepare()”异常由这里可以知道，当我们在子线程使用Handler的时候要手动调用Looper.prepare()创建一个Looper对象，之所以主线程不用，是系统启动的时候帮我们自动调用了Looper.prepare()方法。

handler的工作主要包含发送和接收过程。消息的发送主要通过post和send的一系列方法，而post的一系列方法是最终是通过send的一系列方法来实现的。而send的一系列方法最终是通过sendMessageAtTime方法来实现的，除了sendMessageAtFrontOfQueue（）这个方法。去看看这些一系列send的方法，如下：

    public final boolean sendMessage(Message msg)
        {
            return sendMessageDelayed(msg, 0);
        }

        public final boolean sendEmptyMessage(int what)
        {
            return sendEmptyMessageDelayed(what, 0);
        }  

        public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
            Message msg = Message.obtain();
            msg.what = what;
            return sendMessageAtTime(msg, uptimeMillis);
        }

        public final boolean sendMessageDelayed(Message msg, long delayMillis)
        {
            if (delayMillis < 0) {
                delayMillis = 0;
            }
            return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
        }

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

        public final boolean sendMessageAtFrontOfQueue(Message msg) {
            MessageQueue queue = mQueue;
            if (queue == null) {
                RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
                Log.w("Looper", e.getMessage(), e);
                return false;
            }
            return enqueueMessage(queue, msg, 0);
        }

        private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
            msg.target = this;
            if (mAsynchronous) {
                msg.setAsynchronous(true);
            }
            return queue.enqueueMessage(msg, uptimeMillis);
        }


可以看出，handler发送一条消息其实就是在消息队列插入一条消息。在Looper的loop方法中，从Message Queue中取出消息调msg.target.dispatchMessage(msg);这里其实就是调用了Handler的dispatchMessage(msg)方法，进去看看，如下：

      /**
         * Handle system messages here.
         */
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

判断msg.callback是否为空，不为空调用 handleCallback(msg);来处理消息。其实callback是一个Runnable对象，就是Handler发送post消息传过来的对象。

     public final boolean post(Runnable r)
        {
           return  sendMessageDelayed(getPostMessage(r), 0);
        }

         public final boolean postAtTime(Runnable r, long uptimeMillis)
        {
            return sendMessageAtTime(getPostMessage(r), uptimeMillis);
        }

        public final boolean postAtTime(Runnable r, Object token, long uptimeMillis)
        {
            return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
        }


        public final boolean postDelayed(Runnable r, long delayMillis)
        {
            return sendMessageDelayed(getPostMessage(r), delayMillis);
        }


        public final boolean postAtFrontOfQueue(Runnable r)
        {
            return sendMessageAtFrontOfQueue(getPostMessage(r));
        }

        private static Message getPostMessage(Runnable r) {
            Message m = Message.obtain();
            m.callback = r;
            return m;
        }

进去handleCallback方法看看怎么处理消息的，如下：

      private static void handleCallback(Message message) {
            message.callback.run();
        }

可以看出，其实就是回调Runnable对象的run方法。Activity的runOnUiThread，View的postDelayed方法也是同样的原理，我们先看看runOnUiThread方法，如下：

    public final void runOnUiThread(Runnable action) {
            if (Thread.currentThread() != mUiThread) {
                mHandler.post(action);
            } else {
                action.run();
            }
        }

View的postDelayed方法。如下：

	 public boolean postDelayed(Runnable action, long delayMillis) {
	            final AttachInfo attachInfo = mAttachInfo;
	            if (attachInfo != null) {
	                return attachInfo.mHandler.postDelayed(action, delayMillis);
	            }
	            // Assume that post will succeed later
	            ViewRootImpl.getRunQueue().postDelayed(action, delayMillis);
	            return true;
	        }

实质上都是在UI线程中执行了Runnable的run方法。

如果msg.callback是否为null，判断mCallback是否为null？mCallback是一个接口，如下：

       /**
         * Callback interface you can use when instantiating a Handler to avoid
         * having to implement your own subclass of Handler.
         *
         * @param msg A {@link android.os.Message Message} object
         * @return True if no further handling is desired
         */
        public interface Callback {
            public boolean handleMessage(Message msg);
        }

CallBack其实提供了另一种使用Handler的方式，可以派生子类重写handleMessage（）方法，也可以通过设置CallBack来实现。

我们梳理一下我们在主线程使用Handler的过程。

首先在主线程创建一个Handler对象 ，并重写handleMessage（）方法。然后当在子线程中需要进行更新UI的操作，我们就创建一个Message对象，并通过handler发送这条消息出去。之后这条消息被加入到MessageQueue队列中等待被处理，通过Looper对象会一直尝试从Message Queue中取出待处理的消息，最后分发会Handler的handler Message（）方法中。

![](http://img.blog.csdn.net/20160417192621814)

## Handler使用案例

方式一： post(Runnable)

1. 创建一个工作线程，实现 Runnable 接口，实现 run 方法，处理耗时操作
2. 创建一个 handler，通过 handler.post/postDelay，投递创建的 Runnable，在 run 方法中进行更新 UI 操作。

		new Thread(new Runnable() {
		@Override
		public void run() {
		  /**
		     耗时操作
		   */
		 handler.post(new Runnable() {
		     @Override
		     public void run() {
		         /**
		           更新UI
		          */
		     }
		 });
		}
		}).start();

方式二： sendMessage(Message)

1. 创建一个工作线程，继承 Thread，重新 run 方法，处理耗时操作
2. 创建一个 Message 对象，设置 what 标志及数据
3. 通过 sendMessage 进行投递消息
4. 创建一个handler，重写 handleMessage 方法，根据 msg.what 信息判断，接收对应的信息，再在这里更新 UI。


		private Handler handler = new Handler(){
		@Override
		public void handleMessage(Message msg) {
		    super.handleMessage(msg);
		    switch (msg.what) {      //判断标志位
		        case 1:
		            /**
		              获取数据，更新UI
		             */
		            break;
		    }
		 }
		};

		public class WorkThread extends Thread {
		@Override
		 public void run() {
		   super.run();
		/**
		  耗时操作
		 */
		   
		Message msg =Message.obtain(); //从全局池中返回一个message实例，避免多次创建message（如new Message）
		msg.obj = data;
		msg.what=1; //标志消息的标志
		handler.sendMessage(msg);
		}
		new WorkThread().start();


## Handler存在的问题

1. 内存方面

	- Handler 被作为 Activity 引用，如果为非静态内部类，则会引用外部类对象。当 Activity finish 时，Handler可能并未执行完，从而引起 Activity 的内存泄漏。故而在所有调用 Handler 的地方，都用静态内部类。

2. 异常方面
 	- 当 Activity finish 时,在 onDestroy 方法中释放了一些资源。此时 Handler 执行到 handlerMessage 方法,但相关资源已经被释放,从而引起空指针的异常。 

3. 避免

	- 如果是使用 handlerMessage，则在方法中加try catch。
	- 如果是用 post 方法，则在Runnable方法中加try catch。

## Handler的改进

1. 内存方面：使用静态内部类创建 handler 对象，且对 Activity 持有弱引用
2. 异常方面：不加 try catch，而是在 onDestory 中把消息队列 MessageQueue 中的消息给 remove 掉。
	- 则使用如下方式创建 handler 对象：

        
		    /**
		     为避免handler造成的内存泄漏
		     1、使用静态的handler，对外部类不保持对象的引用
		     2、但Handler需要与Activity通信，所以需要增加一个对Activity的弱引用
		    */
	      	private static class MyHandler extends Handler {
		        private final WeakReference<Activity> mActivityReference;    
		
		        MyHandler(Activity activity) {
		            this.mActivityReference = new WeakReference<Activity>(activity);
		        }
		
		        @Override
		            public void handleMessage(Message msg) {
		            super.handleMessage(msg);
		            MainActivity activity = (MainActivity) mActivityReference.get();  //获取弱引用队列中的activity
		            switch (msg.what) {    //获取消息，更新UI
		                case 1:
		                    byte[] data = (byte[]) msg.obj;
		                    activity.threadIv.setImageBitmap(activity.getBitmap(data));
		                    break;
		            }
		        }
	  	  	}

	- 并在 onDesotry 中销毁：

			@Override
			protected void onDestroy() {
			    super.onDestroy();
			    //避免activity销毁时，messageQueue中的消息未处理完；故此时应把对应的message给清除出队列
			    handler.removeCallbacks(postRunnable);   //清除runnable对应的message
			    //handler.removeMessage(what)  清除what对应的message
			}

## Handler的使用实现

1. 耗时操作采用从网络加载一张图片
2. 继承 Thread 或实现 Runnable 接口的线程，与 UI 线程进行分离，其中 Runnable 与主线程通过回调接口进行通信，降低耦合，提高代码复用性。


在 Activity 中创建 handler 对象，调用工作线程执行

    
	public class MainActivity extends AppCompatActivity {
	
		ImageView threadIv;
		ImageView runnableIv;
		SendThread sendThread;
		PostRunnable postRunnable;
		private final MyHandler handler = new MyHandler(this);
		
		@Override
		protected void onCreate(Bundle savedInstanceState) {
		    super.onCreate(savedInstanceState);
		    setContentView(R.layout.activity_main);
		    threadIv = (ImageView) findViewById(R.id.thread_iv);
		    runnableIv = (ImageView) findViewById(R.id.runnable_iv);
		
		    sendThread = new SendThread(handler);
		    sendThread.start();
		
		    postRunnable = new PostRunnable(handler);
		    postRunnable.setRefreshUI(new PostRunnable.RefreshUI() {
		        @Override
		        public void setImage(byte[] data) {
		            runnableIv.setImageBitmap(getBitmap(data));
		        }
		    });
		    new Thread(postRunnable).start();
		}
		
		/**
		  为避免handler造成的内存泄漏
		  1、使用静态的handler，对外部类不保持对象的引用
		  2、但Handler需要与Activity通信，所以需要增加一个对Activity的弱引用
		 /
		private static class MyHandler extends Handler {
		    private final WeakReference<Activity> mActivityReference;
		
		    MyHandler(Activity activity) {
		        this.mActivityReference = new WeakReference<Activity>(activity);
		    }
		
		    @Override
		    public void handleMessage(Message msg) {
		        super.handleMessage(msg);
		        MainActivity activity = (MainActivity) mActivityReference.get();  //获取弱引用队列中的activity
		        switch (msg.what) {    //获取消息，更新UI
		            case 1:
		                byte[] data = (byte[]) msg.obj;
		                activity.threadIv.setImageBitmap(activity.getBitmap(data));
		                break;
		        }
		    }
		}
		
		private Bitmap getBitmap(byte[] data) {
		    return BitmapFactory.decodeByteArray(data, 0, data.length);
		}
		
		@Override
		protected void onDestroy() {
		    super.onDestroy();
		    //避免activity销毁时，messageQueue中的消息未处理完；故此时应把对应的message给清除出队列
		    handler.removeCallbacks(postRunnable);   //清除runnable对应的message
		    //handler.removeMessage(what)  清除what对应的message
		}
	}

- 方式一：实现 runnable 接口，通过 post（Runnable）通信，并通过给定的回调接口通知 Activity 更新

		public class PostRunnable implements Runnable {
		
			private Handler handler;
			private RefreshUI refreshUI;
			byte[] data = null;
			
			public PostRunnable(Handler handler) {
			    this.handler = handler;
			}
			
			@Override
			public void run() {
			    /**
			      耗时操作
			     */
			    final Bitmap bitmap = null;
			    HttpClient httpClient = new DefaultHttpClient();
			    HttpGet httpGet = new HttpGet("http://i3.17173cdn.com/2fhnvk/YWxqaGBf/cms3/FNsPLfbkmwgBgpl.jpg");
			    HttpResponse httpResponse = null;
			    try {
			        httpResponse = httpClient.execute(httpGet);
			        if (httpResponse.getStatusLine().getStatusCode() == 200) {
			            data = EntityUtils.toByteArray(httpResponse.getEntity());
			        }
			    } catch (IOException e) {
			        e.printStackTrace();
			    }
			
			    //返回结果给UI线程
			    handler.post(new Runnable() {
			        @Override
			        public void run() {
			            refreshUI.setImage(data);
			        }
			    });
			}
			
			public interface RefreshUI {
			    public void setImage(byte[] data);
			}
			
			public void setRefreshUI(RefreshUI refreshUI) {
			    this.refreshUI = refreshUI;
			}
		}
- 方式二:继承Thread，通过handler的sendMessage通信

		public class SendThread extends Thread {
		
			private Handler handler;
			
			public SendThread(Handler handler) {
			    this.handler = handler;
			}
			
			@Override
			public void run() {
			    super.run();
			    /**
			      耗时操作
			     */
			    byte[]data=null;
			    HttpClient httpClient = new DefaultHttpClient();
			    HttpGet httpGet = new HttpGet("https://d36lyudx79hk0a.cloudfront.net/p0/descr/pc27/3095587d8c4560d8.png");
			    HttpResponse httpResponse = null;
			    try {
			        httpResponse = httpClient.execute(httpGet);
			        if(httpResponse.getStatusLine().getStatusCode()==200){
			            data= EntityUtils.toByteArray(httpResponse.getEntity());
			        }
			    } catch (IOException e) {
			        e.printStackTrace();
			    }
			
			    //返回结果给UI线程
			    doTask(data);
			}
			
			/**
			  通过handler返回消息
			  @param data
			 */
			private void doTask(byte[] data) {
			    Message msg =Message.obtain();  //从全局池中返回一个message实例，避免多次创建message（如new Message）
			    msg.obj = data;
			    msg.what=1;   //标志消息的标志
			    handler.sendMessage(msg);
			}
		}