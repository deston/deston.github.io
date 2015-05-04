---
layout: post
title: 开源图片加载库Picasso的一些零散记录
---

最近在看Picasso的源码，大致流程已弄清楚，有时间会写一遍整体的源码分析。现在这篇主要零散的记录一些从Picasso中得到的启发。

Picasso内的源码中不可避免的需要持有context作为成员变量，但Picasso又是基于单例模式，除非特意控制，否则单例模式一旦建立就不会销毁掉，因此如果将activity作为context对象传给Picasso中，这个activity就不会再被回收，会造成OOM。基于此，Picasso使用了这样的方法：

 	this.context = context.getApplicationContext();

在构造单例的时候，通过传入的context，获得整个应用的全局context，就能避免掉OOM的问题。

以前在做类似单例时也考虑过持有context对象造成的内存问题，不过那时的思路是另外一种，通过持有context的对象的弱引用，即：

	WeakReference<Context> contextWeakReference;

弱引用只要GC执行就会被回收，因此也可以避免OOM。

但用弱引用也会有个问题：假如传入的是activityA作为弱引用，如果activityA进入后台被回收掉了，再次调用该单例里的方法可能就会有空指针问题了，当然也可以每次调用单例里的方法都传入context，不过这样就比较麻烦。

因此，如果像picasso这样的使用全局的applicationContext就比较完美了。

除此之外，我在Picasso中还看到一种很好的思路：CleanupThread。一个做清理工作的线程，代码如下：

	private static class CleanupThread extends Thread {
    private final ReferenceQueue<Object> referenceQueue;
    private final Handler handler;

    CleanupThread(ReferenceQueue<Object> referenceQueue, Handler handler) {
      this.referenceQueue = referenceQueue;
      this.handler = handler;
      setDaemon(true);
      setName(THREAD_PREFIX + "refQueue");
    }

    @Override public void run() {
      Process.setThreadPriority(THREAD_PRIORITY_BACKGROUND);
      while (true) {
        try {
          // Prior to Android 5.0, even when there is no local variable, the result from
          // remove() & obtainMessage() is kept as a stack local variable.
          // We're forcing this reference to be cleared and replaced by looping every second
          // when there is nothing to do.
          // This behavior has been tested and reproduced with heap dumps.
          RequestWeakReference<?> remove =
              (RequestWeakReference<?>) referenceQueue.remove(THREAD_LEAK_CLEANING_MS);
          Message message = handler.obtainMessage();
          if (remove != null) {
            message.what = REQUEST_GCED;
            message.obj = remove.action;
            handler.sendMessage(message);
          } else {
            message.recycle();
          }
        } catch (InterruptedException e) {
          break;
        } catch (final Exception e) {
          handler.post(new Runnable() {
            @Override public void run() {
              throw new RuntimeException(e);
            }
          });
          break;
        }
      }
    }

    void shutdown() {
      interrupt();
    }
  	}


在picasso单例创建的时候这个线程就会被创建被直接执行起来，而中断它的办法就是调用shutdown方法，而这个CleanupThread线程只有picasso的使用者主动中断了picasso的执行才会中断，因此可以说只要picasso在执行，这个线程就一直在执行。

它做的事情从代码中很容易看出来是从一个队列中获取对象，随后通过handler发送消息。这个队列是ReferenceQueue，是Java提供的一个类。

引用队列的创造比较简单，直接就new了出来： 	

	this.referenceQueue = new ReferenceQueue<Object>();

随后，将它传给了另外一个类，代码如下：

	abstract class Action<T> {
	  static class RequestWeakReference<M> extends WeakReference<M> {
	    final Action action;
	
	    public RequestWeakReference(Action action, M referent, ReferenceQueue<? super M> q) {
	      super(referent, q);
	      this.action = action;
	    }
	  }
	
	  final Picasso picasso;
	  final Request request;
	  final WeakReference<T> target;
	  final boolean noFade;
	  final int memoryPolicy;
	  final int networkPolicy;
	  final int errorResId;
	  final Drawable errorDrawable;
	  final String key;
	  final Object tag;
	
	  boolean willReplay;
	  boolean cancelled;
	
	  Action(Picasso picasso, T target, Request request, int memoryPolicy, int networkPolicy,
	      int errorResId, Drawable errorDrawable, String key, Object tag, boolean noFade) {
	    this.picasso = picasso;
	    this.request = request;
	    this.target =
	        target == null ? null : new RequestWeakReference<T>(this, target, picasso.referenceQueue);
	    this.memoryPolicy = memoryPolicy;
	    this.networkPolicy = networkPolicy;
	    this.noFade = noFade;
	    this.errorResId = errorResId;
	    this.errorDrawable = errorDrawable;
	    this.key = key;
	    this.tag = (tag != null ? tag : this);
	  }
	
	  abstract void complete(Bitmap result, Picasso.LoadedFrom from);
	
	  abstract void error();
	
	  void cancel() {
	    cancelled = true;
	  }
	
	  Request getRequest() {
	    return request;
	  }
	
	  T getTarget() {
	    return target == null ? null : target.get();
	  }
	
	  String getKey() {
	    return key;
	  }
	
	  boolean isCancelled() {
	    return cancelled;
	  }
	
	  boolean willReplay() {
	    return willReplay;
	  }
	
	  int getMemoryPolicy() {
	    return memoryPolicy;
	  }
	
	  int getNetworkPolicy() {
	    return networkPolicy;
	  }
	
	  Picasso getPicasso() {
	    return picasso;
	  }
	
	  Priority getPriority() {
	    return request.priority;
	  }
	
	  Object getTag() {
	    return tag;
	  }
	} 

传给了一个叫Action的类里的一个内部类，这个Action可以理解为Picasso每次加载图片都是一次action，它的内部类RequestWeakReference继承于Java的WeakReference，因此是一个弱引用，这个Action的构造方法知道这个弱引用持有的是一个叫target的对象，在Picasso中这个target就是图片加载到的地方，一般就是imageview。

也就是说将这个引用队列传给了一个弱引用，另外一个线程不断的从队列中获取对象，然后做一些处理。

关于ReferenceQueue可以简单理解为：当弱引用被GC回收时，弱引用会被加入到队列中。

因此Picasso通过这样的用法，当imageview控件被回收（如activity进入后台被销毁等）时，做了一些及时的处理，将这次图片加载当作取消事件，调用封装好的取消图片加载请求方法。
