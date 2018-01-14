# blog
## Hander知识点梳理
在Android开发中，从子线程切换到主线程都是用的Handler，我们一般使用Handler都是这么用的:

1. handler.sendMessage(message)——>发送消息 
2. handleMessage(Message msg){}——>处理消息

我们都知道在子线程中发送消息，然后去主线程里处理收到的消息，但是这其中的原理又是怎么样的呢？
要研究里面的原理，首先得找一个切入点-ThreadLocal,它的作用是用来维护一个线程本地变量。在Looper中，
```Java
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        } 
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
sThreadLocal.set(looper) 中看出，我们当前线程中存了一个looper的本地变量。

下面再来看下我们建立一个Handler对象时里面构造函数里会怎么去执行，
```Java
public Handler(Callback callback, boolean async) {
      
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
他首先会去执行Looper.myLooper()获取一个Looper对象，看看myLooper()方法是如何实现的，
 ```Java 
 public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
  ```
  他实际上就是从我们之前所说的ThreadLocal里去取一个Looper对象。
  又给Handler中mQueue的变量赋值了，我们可以猜猜mQueue这个变量其实就是MessageQueue对象，我们可以看下Looper的构造函数
  ```Java 
   private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
  ```
 在它的构造函数中创建了MessageQueue的对象。由此可见，Handler中的mQueue对象就是由Looper构造函数中所创建的。
 
 一般我们用handler发送的消息最终都是由sendMessageAtTime这个方法去处理的，我们看一下它的源码。
  ```Java 
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
    
      private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

  ```
  在enqueueMessage中，我们把Message.target 给赋值，然后让把这条消息加入到消息队列中。
  那么我们消息队列中的消息到底谁去处理呢，这就交给我们的Looper了,我们可以看下其中的loop()方法。
  ```Java
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

            msg.recycleUnchecked();
        }
    }
```
只要启用loop方法的时候，就会不断从消息列表中取出消息，最后由Handler的dispatchMessage(msg)去处理这个消息。如果没有消息则阻塞。

通过以上分析，我们可知Handler这一块的体系主要由Looper，MessageQueue,Handler组成。
一个线程对应一个Looper
一个Looper对应一个消息队列
一个线程对应一个消息队列
线程，Looper，消息队列三者一一对应
他们各个的指责是：
1. Looper 在每个线程中存入looper对象，创建一个消息队列，以及不断遍历消息队列里消息。
2. MessageQueue 存入Handler发送过来的消息，处理消息队列的逻辑。
3. Handler 发送消息给messageQueue、处理消息。

#### 逻辑图是这个样子
![Alt text](https://github.com/check0927/blog/blob/master/image/handler.png?raw=true)
 
 
