# EventBus源码分析 #

[EventBus](https://github.com/greenrobot/EventBus) 是一款针对Android优化的发布/订阅事件总线。主要功能是替代Intent, Handler, BroadCast 在 Fragment，Activity，Service，线程之间传递消息.优点是开销小，使用方便,可以很大程度上降低它们之间的耦合，使得我们的代码更加简洁，耦合性更低，提升我们的代码质量。

虽然已经在使用了，但是源码还是没有怎么仔细的阅读过，每次刚开始读的时候，就晕了，果然放弃了。今天，决定认认真真的读一下源码，好好的了解一下其原理。

（本文参考了其他资料，如有雷同，那么恭喜你，我就是这么任性，>_<，不过，自己还是一步一步对照源码，重新梳理了一遍，写比看要理解的更深一些）

## 使用EventBus ##

五步曲：

1.首先需要定义一个消息类，该类可以不继承任何基类也不需要实现任何接口。如：
	
	public class MessageEvent {}

2.在需要订阅事件的地方注册事件

	//把当前类注册为订阅者(接收者)
	EventBus.getDefault().register(this);

3.产生事件，即发送消息

	EventBus.getDefault().post(messageEvent);

4.处理消息

	@Subscribe(threadMode = ThreadMode.PostThread)
		public void XXX(MessageEvent messageEvent) {
    	...
	}

在3.0之前，EventBus还没有使用注解方式。消息处理的方法也只能限定于onEvent、onEventMainThread、onEventBackgroundThread和onEventAsync，分别代表四种线程模型。而在3.0之后，消息处理的方法可以随便取名，但是需要添加一个注解@Subscribe，并且要指定线程模型（默认为PostThread，一共有四种线程模型）

注意，事件处理函数的访问权限必须为public，否则会报异常。

5.及时回收，取消消息订阅

	//解除注册当前类(同广播一样,一定要调用,否则会内存泄露)
	EventBus.getDefault().unregister(this);

上面五步，基本上就是一个完整的EventBus的使用规则了。



## 源码--痛苦的开始 ##

第一步中，是调用的EventBus.getDefault()，那么上代码：

 	/** Convenience singleton for apps using a process-wide EventBus instance. */
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }

可以看出，采用了单例模式，调用了EventBus的构造方法new EventBus(),so,一探构造方法：

	 public EventBus() {
        this(DEFAULT_BUILDER);
     }

这里：构造方法是public，而不是private！

这样的设计是因为不仅仅可以只有一条总线，还可以有其他的线 (bus) ，订阅者可以注册到不同的线上的 EventBus ，通过不同的 EventBus 实例来发送数据，不同的 EventBus 是相互隔离开的，订阅者都只会收到注册到该线上事件。

DEFAULT_BUILDER的真容：

	private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();

这个EventBusBuilder的内容为：

	/**使用自定义的参数创建EventBus实例，也可以使用默认的build()创建*/
	public class EventBusBuilder {
    	private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();

	    boolean logSubscriberExceptions = true;//监听异常
	    boolean logNoSubscriberMessages = true;//如果没有订阅者，显示一个log
	    boolean sendSubscriberExceptionEvent = true;//发送监听到异常事件
	    boolean sendNoSubscriberEvent = true;//如果没有订阅者，发送一条默认事件
	    boolean throwSubscriberException;//如果失败抛出异常
	    boolean eventInheritance = true;//event的子类是否也能响应订阅者
	    boolean ignoreGeneratedIndex;
	    boolean strictMethodVerification;
	    ExecutorService executorService = DEFAULT_EXECUTOR_SERVICE;
	    List<Class<?>> skipMethodVerificationForClasses;
	    List<SubscriberInfoIndex> subscriberInfoIndexes;
	
	    EventBusBuilder() {
		｝
		...
	｝

Builder类提供了这么多个可选的配置属性

	/**
	 * 根据参数创建对象,并赋值给EventBus.defaultInstance, 必须在默认的eventbus对象使用以前调用
	 *
	 * @throws EventBusException if there's already a default EventBus instance in place
	 */
	public EventBus installDefaultEventBus() {
    	synchronized (EventBus.class) {
        	if (EventBus.defaultInstance != null) {
            	throw new EventBusException("Default instance already exists." +
                    " It may be only set once before it's used the first time to ensure " +
                    "consistent behavior.");
        	}
        	EventBus.defaultInstance = build();
        	return EventBus.defaultInstance;
    	}
	}

	/**
	 * 根据参数创建对象
	 */
	public EventBus build() {
    	return new EventBus(this);
	}

EventBusBuilder类提供了两种建造方法,还记得之前的getDefault()方法吗,维护了一个单例对象,installDefaultEventBus() 方法建造的EventBus对象最终会赋值给那个单例对象,但是有一个前提就是我们之前并没有创建过那个单例对象. 


第二个方法就是默认的建造者方法了.

下面看看this里面的内容：

	EventBus(EventBusBuilder builder) {
        subscriptionsByEventType = new HashMap<>();//key为event，value为subscriber列表，这个map就是这个事件有多少的订阅者，也就是事件对应的订阅者
		typesBySubscriber = new HashMap<>();//key为subscriber，value为event列表，这个map就是这个订阅者有多少的事件，也就是订阅者订阅的事件列表
		stickyEvents = new ConcurrentHashMap<>();//粘性事件
		mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);//MainThread的poster
		backgroundPoster = new BackgroundPoster(this);//Backgroud的poster
		asyncPoster = new AsyncPoster(this);//Async的poster
		indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
		subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
		builder.strictMethodVerification, builder.ignoreGeneratedIndex);//订阅者方法寻找类，默认情况下参数是(null, false, false)
		logSubscriberExceptions = builder.logSubscriberExceptions;
		logNoSubscriberMessages = builder.logNoSubscriberMessages;
		sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
		sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
		throwSubscriberException = builder.throwSubscriberException;
		eventInheritance = builder.eventInheritance;
		executorService = builder.executorService;
    }

根据提供的建造者初始化了一大堆属性

这里，先看一下Map的定义及其含义（源码设计了很多的Map，一会就看晕了，先列出来）

	private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();//缓存Subscription

	private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;//subscriptionsByEventType是以eventType为key，Subscription的ArrayList为value的HashMap，事件订阅者的保存队列，找到该事件所订阅的订阅者以及订阅者的方法、参数等，这个很重要，就是EventBus存储方法的地方
	private final Map<Object, List<Class<?>>> typesBySubscriber;//typesBySubscriber以subscriber为key，eventType的ArrayList为value的HashMap，订阅者订阅的事件列表，找到改订阅者所订阅的所有事件，表示当前订阅者订阅了哪些事件。 
	private final Map<Class<?>, Object> stickyEvents;//stickyEvents以eventType为key，event为value的ConcurrentHashMap，Sticky事件保存队列


### ## 三个Poster类 ## ###

们先来看这三个Poster，需要说明的一点就是：Poster只负责处理粘滞事件。

	private final HandlerPoster mainThreadPoster;//前台发送者，extends Handler
	private final BackgroundPoster backgroundPoster;//后台发送者，implements Runnable
	private final AsyncPoster asyncPoster;//后台发送者(只让队列第一个待订阅者去响应)，implements Runnable

其实从类名我们就能看出个大概了,就是三个发送事件的方法。

我们来看看他们的内部实现. 

这几个Poster的设计可以说是整个EventBus的一个经典部分

下面我们按照：PendingPost->PendingPostQueuequeue->XXXPoster进行说明

每个XXXPoster中都有一个发送任务队列：

	private final PendingPostQueuequeue;

进到队列PendingPostQueuequeue里面再看定义了两个节点
	
	private PendingPost head;//待发送对象队列头节点
	private PendingPost tail;//待发送对象队列尾节点

从字面上理解就是队列的头节点和尾节点

再看这个PendingPost类的实现:(待发送的事件被封装成了PendingPost对象)

	private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();//单例池,复用对象
	
	Object event;//事件类型
	Subscription subscription;//订阅者
	PendingPostnext;//队列下一个待发送对象

首先是提供了一个池的设计，类似于我们的线程池，目的是为了减少对象创建的开销，当一个对象不用了，我们可以留着它，下次再需要的时候返回这个保留的而不是再去创建。

再看最后的变量，PendingPost next 非常典型的队列设计，队列中每个节点都有一个指向下一个节点的指针。

		/** * 首先检查复用池中是否有可用,如果有则返回复用,否则返回一个新的 
			*
			* @param subscription 订阅者 
			* 
			* @param event        订阅事件 
			* @return 待发送对象
			* /
		static PendingPost obtainPendingPost(Subscription subscription, Object event) {
			synchronized (pendingPostPool) {
				int size = pendingPostPool.size();
		        	if (size > 0) {
		            	PendingPost pendingPost = pendingPostPool.remove(size - 1);
						pendingPost.event = event;
						pendingPost.subscription = subscription;
						pendingPost.next = null;
		            	return pendingPost;
				}
		    }
			return new PendingPost(event, subscription);
		}

obtainPendingPost(),对池复用的实现，每次新创建的节点尾指针都为 null 。

		/** 
		* 回收一个待发送对象,并加入复用池 
		* @param pendingPost 待回收的待发送对象 
		* /
		static void releasePendingPost(PendingPost pendingPost) {
			pendingPost.event = null;
			pendingPost.subscription = null;
			pendingPost.next = null;
		    synchronized (pendingPostPool) {
				// Don't let the pool grow indefinitely 防止池无限增长
				if (pendingPostPool.size() < 10000) {
					pendingPostPool.add(pendingPost);
				}
		    }
		}

releasePendingPost()，回收pendingPost对象，既然有从池中取，当然需要有存。
这里，原作非常细心的加了一次判断，if (pendingPostPool.size() < 10000 code> 如果ArrayList里面存了上千条都没有取走，那么肯定是使用出错了。

PendingPost的代码我们就看完了，再回到上一级，队列的设计：

接着是PendingPostQueue的入队方法

		synchronized void enqueue(PendingPost pendingPost) {
			if (pendingPost == null) {
				throw new NullPointerException("null cannot be enqueued");
			}
			if (tail != null) {
				tail.next = pendingPost;
				tail = pendingPost;
			} else if (head == null) {
				head = tail = pendingPost;
			} else {
				throw new IllegalStateException("Head present, but no tail");
			}
		    notifyAll();
		}

首先将当前节点的上一个节点(入队前整个队列的最后一个节点)的尾指针指向当期正在入队的节点(传入的参数pendingPost)，并将队列的尾指针指向自己(自己变成队列的最后一个节点)，这样就完成了入队。

如果是队列的第一个元素(队列之前是空的),那么直接将队列的头尾两个指针都指向自身就行了。

出队也是类似的队列指针操作


		synchronized PendingPost poll() {
		    PendingPost pendingPost = head;
		    if (head != null) {
				head = head.next;
		   		if (head == null) {
					tail = null;
				}
		    }
			return pendingPost;
		}

首先将出队前的头节点保留一个临时变量(它就是要出队的节点),拿到这个将要出队的临时变量的下一个节点指针，将出队前的第二个元素(出队后的第一个元素)的赋值为现在队列的头节点，出队完成。
 
值得提一点的就是，PendingPostQueue的所有方法都声明了synchronized，这意味着在多线程下它依旧可以正常工作。

再回到上一级，接着是HandlerPoster的入队方法enqueue()

		/** 
		* @param subscription 订阅者 
		* @param event        订阅事件
		* /
		void enqueue(Subscription subscription, Object event) {
		    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
		    synchronized (this) {
			queue.enqueue(pendingPost);
		        if (!handlerActive) {//已经有任务在跑着了，就不需要再去sendMessage()唤起我们的handleMessage()
					handlerActive = true;
		            if (!sendMessage(obtainMessage())) {
						throw new EventBusException("Could not send handler message");
					}
		        }
		    }
		}

入队方法会根据参数创建 待发送对象 pendingPost 并加入队列,如果此时 handleMessage() 没有在运行中,则发送一条空消息让 handleMessage 响应 接着是handleMessage()方法

 		@Override
	    public void handleMessage(Message msg) {
	        boolean rescheduled = false;
	        try {
	            long started = SystemClock.uptimeMillis();
	            while (true) {
	                PendingPost pendingPost = queue.poll();//出队
	                if (pendingPost == null) {
	                    synchronized (this) {
	                        // Check again, this time in synchronized 双重校验,类似单例中的实现
	                        pendingPost = queue.poll();
	                        if (pendingPost == null) {
	                            handlerActive = false;
	                            return;
	                        }
	                    }
	                }
	                eventBus.invokeSubscriber(pendingPost);//如果订阅者没有取消注册,则分发消息
	                long timeInMethod = SystemClock.uptimeMillis() - started;
	                if (timeInMethod >= maxMillisInsideHandleMessage) {//如果在一定时间内仍然没有发完队列中所有的待发送者,则退出
	                    if (!sendMessage(obtainMessage())) {
	                        throw new EventBusException("Could not send handler message");
	                    }
	                    rescheduled = true;
	                    return;
	                }
	            }
	        } finally {
	            handlerActive = rescheduled;
	        }
	    }

handleMessage()不停的在待发送队列queue中去取消息。 需要说明的是在循环之外有个临时boolean变量rescheduled,最后是通过这个值去修改了handlerActive。

而 handlerActive 是用来判断当前queue中是否有正在发送对象的任务，看到上面的入队方法enqueue(),如果已经有任务在跑着了，就不需要再去sendMessage()唤起我们的handleMessage()

最终通过eventBus对象的invokeSubscriber()最终发送出去，并回收这个pendingPost，让注册了的订阅者去响应(相当于回调),至于这个发送方法,我们之后再看。

看完了HandlePoster类,另外两个异步的发送者实现代码也差不多,唯一的区别就是另外两个是工作在异步,实现的Runnable接口,大家自己类比,这里就不帖代码了.

## Poster工作原理 ##

最后我们再来回顾一下Poster、PendingPostQueue、PendingPost这三个类，再看看下面这张图，是不是有种似曾相识的感觉。

![](http://kymjs.com/images/blog_image/20151211_5.png)

### Subscribe流程 ###

首先了解一下[注解](http://www.cnblogs.com/yydcdut/p/4646454.html)

了解了注解，那么来看一看 Subscribe 的注解：

		@Documented
		@Retention(RetentionPolicy.RUNTIME)
		@Target({ElementType.METHOD})
		public @interface Subscribe {
		    ThreadMode threadMode() default ThreadMode.POSTING;//线程模式，订阅者在哪个线程接收到事件
		
			/**
		     * If true, delivers the most recent sticky event (posted with
		     * {@link EventBus#postSticky(Object)}) to this subscriber (if event available).
		     */
			boolean sticky() default false;//是否是粘性事件
		
			/** Subscriber priority to influence the order of event delivery.
		     * Within the same delivery thread ({@link ThreadMode}), higher priority subscribers will receive events before
		     * others with a lower priority. The default priority is 0. Note: the priority does *NOT* affect the order of
		     * delivery among subscribers with different {@link ThreadMode}s! */
			int priority() default 0;//优先级，默认为0
		}

ThreadMode 是枚举：

		public enum ThreadMode {
			/**
		     * Subscriber will be called in the same thread, which is posting the event. This is the default. Event delivery
		     * implies the least overhead because it avoids thread switching completely. Thus this is the recommended mode for
		     * simple tasks that are known to complete is a very short time without requiring the main thread. Event handlers
		     * using this mode must return quickly to avoid blocking the posting thread, which may be the main thread.
		     */
			POSTING,//post的时候是哪个线程订阅者就在哪个线程接收到事件
		
			/**
		     * Subscriber will be called in Android's main thread (sometimes referred to as UI thread). If the posting thread is
		     * the main thread, event handler methods will be called directly. Event handlers using this mode must return
		     * quickly to avoid blocking the main thread.
		     */
			MAIN,//订阅者在主线程接收到事件
		
			/**
		     * Subscriber will be called in a background thread. If posting thread is not the main thread, event handler methods
		     * will be called directly in the posting thread. If the posting thread is the main thread, EventBus uses a single
		     * background thread, that will deliver all its events sequentially. Event handlers using this mode should try to
		     * return quickly to avoid blocking the background thread.
		     */
			BACKGROUND,//订阅者在主线程接收到消息，如果post的时候不是在主线程的话，那么订阅者会在post的时候那个线程接收到事件。适合密集或者耗时少的事件。
		
			/**
		     * Event handler methods are called in a separate thread. This is always independent from the posting thread and the
		     * main thread. Posting events never wait for event handler methods using this mode. Event handler methods should
		     * use this mode if their execution might take some time, e.g. for network access. Avoid triggering a large number
		     * of long running asynchronous handler methods at the same time to limit the number of concurrent threads. EventBus
		     * uses a thread pool to efficiently reuse threads from completed asynchronous event handler notifications.
		     */
			ASYNC //订阅者会在不同的子线程中收到事件。适合操作耗时的事件。
		}

### 线程模型 ###

在EventBus的事件处理函数中需要指定线程模型，即指定事件处理函数运行所在的想线程。在上面我们已经接触到了EventBus的四种线程模型。那他们有什么区别呢？

在EventBus中的观察者通常有四种线程模型，分别是PostThread（默认）、MainThread、BackgroundThread与Async。

1. **PostThread：**如果使用事件处理函数指定了线程模型为PostThread，那么该事件在哪个线程发布出来的，事件处理函数就会在这个线程中运行，也就是说发布事件和接收事件在同一个线程。在线程模型为PostThread的事件处理函数中尽量避免执行耗时操作，因为它会阻塞事件的传递，甚至有可能会引起ANR。
2. **MainThread：**如果使用事件处理函数指定了线程模型为MainThread，那么不论事件是在哪个线程中发布出来的，该事件处理函数都会在UI线程中执行。该方法可以用来更新UI，但是不能处理耗时操作。
3. **BackgroundThread：**如果使用事件处理函数指定了线程模型为BackgroundThread，那么如果事件是在UI线程中发布出来的，那么该事件处理函数就会在新的线程中运行，如果事件本来就是子线程中发布出来的，那么该事件处理函数直接在发布事件的线程中执行。在此事件处理函数中禁止进行UI更新操作。
4. ** Async：**如果使用事件处理函数指定了线程模型为Async，那么无论事件在哪个线程发布，该事件处理函数都会在新建的子线程中执行。同样，此事件处理函数中禁止进行UI更新操作。

我们继续来看EventBus类，分析完了包含的属性，接下来我们看入口方法register()：

		/**
		 * Registers the given subscriber to receive events. Subscribers must call {@link #unregister(Object)} once they
		 * are no longer interested in receiving events.
		 * <p/>
		* Subscribers have event handling methods that must be annotated by {@link Subscribe}.
		 * The {@link Subscribe} annotation also allows configuration like {@link
		* ThreadMode} and priority.
		 */
		public void register(Object subscriber) {
		    Class<?> subscriberClass = subscriber.getClass();
			List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);//ignoreGeneratedIndex
		    synchronized (this) {//使用同步块，保证一个时候只有一个线程在进行订阅
				for (SubscriberMethod subscriberMethod : subscriberMethods) {
		            subscribe(subscriber, subscriberMethod);//遍历一个订阅者的所有订阅方法，进行订阅
				}
		    }
		}


#### SubscriberMethod类 ####

出现了一个SubscriberMethod类，看看它是干嘛的：
看字面意思是订阅者方法,看看类中的内容，除了复写的equals()和hashCode()就只有这些了。

		/** Used internally by EventBus and generated subscriber indexes. */
		public class SubscriberMethod {
			final Method method;//方法名
		    final ThreadMode threadMode;//工作在哪个线程
		    final Class<?> eventType;//参数类型
		    final int priority;//优先级
		    final boolean sticky;//是否粘性事件
			/** Used for efficient comparison */
			String methodString;
		
		    public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {
				this.method = method;
		        this.threadMode = threadMode;
		        this.eventType = eventType;
		        this.priority = priority;
		        this.sticky = sticky;
			}
		
			@Override
			public boolean equals(Object other) {
				if (other == this) {
					return true;
				} else if (other instanceof SubscriberMethod) {
			            checkMethodString();
					SubscriberMethod otherSubscriberMethod = (SubscriberMethod)other;
						otherSubscriberMethod.checkMethodString();
					// Don't use method.equals because of http://code.google.com/p/android/issues/detail?id=7811#c6
					return methodString.equals(otherSubscriberMethod.methodString);
				} else {
					return false;
				}
			}
			
			private synchronized void checkMethodString() {
				if (methodString == null) {
					// Method.toString has more overhead, just take relevant parts of the method
					StringBuilder builder = new StringBuilder(64);
					builder.append(method.getDeclaringClass().getName());
					builder.append('#').append(method.getName());
					builder.append('(').append(eventType.getName());
					methodString = builder.toString();
				}
			}
			
			@Override
			public int hashCode() {
				return method.hashCode();
			}
		}

这个类封装了以@Subscribe注解方法的默认的属性。

回到EventBus.register()又遇到了SubscriberMethodFinder，继续去看。

#### SubscriberMethodFinder ####

从字面理解，就是订阅者方法发现者。

回想一下，我们之前用 EventBus 的时候，需要在注册方法传的那个 this 对象。没错，SubscriberMethodFinder类就是查看传进去的那个 this 对象里面的注解的方法。里面使用到了反射。

先看他的变量声明

		/*
		 * In newer class files, compilers may add methods. Those are called bridge or synthetic methods.
		 * EventBus must ignore both. There modifiers are not public but defined in the Java class file format:
		 * http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6-200-A.1  在较新的类文件，编译器可能会添加方法。那些被称为BRIDGE或SYNTHETIC方法。 * EventBus必须忽略两者。有修饰符没有公开，但在Java类文件中有格式定义
		 */
		private static final int BRIDGE = 0x40;
		private static final int SYNTHETIC = 0x1000;
		
		private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;//需要忽略的修饰符
		private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();//key:类名,value:该类中需要相应的方法集合
		
		private List<SubscriberInfoIndex> subscriberInfoIndexes;
		private final boolean strictMethodVerification;
		private final boolean ignoreGeneratedIndex;
		
		private static final int POOL_SIZE = 4;
		private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

有一句注释

		In newer class files, compilers may add methods. Those are called bridge or synthetic methods. EventBus must ignore both. There modifiers are not public but defined in the Java class file format: http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6-200-A.1

翻译过来大概就是说java编译器在编译的时候，会额外添加一些修饰符，然后这些修饰符为了效率应该是被忽略的。

看看其findSubscriberMethods方法：

		List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
		    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);//METHOD_CACHE缓存之前已经查找过的订阅方法，进来先从缓存查询
		    if (subscriberMethods != null) {
				return subscriberMethods;  //缓存有，直接返回
			}
		    //ignoreGeneratedIndex默认是false
			if (ignoreGeneratedIndex) {
				subscriberMethods = findUsingReflection(subscriberClass); //通过反射去查找订阅方法
			} else {
				subscriberMethods = findUsingInfo(subscriberClass);/通过索引去查找订阅方法
			}
			if (subscriberMethods.isEmpty()) {
				throw new EventBusException("Subscriber " + subscriberClass
		                + " and its super classes have no public methods with the @Subscribe annotation");//检查订阅者中是否声明了订阅方法，没有则抛出异常
			} else {
				METHOD_CACHE.put(subscriberClass, subscriberMethods);//把查找到的订阅方法加入缓存，下次订阅时直接从缓存拿订阅方法
		        return subscriberMethods;
			}
		}

默认是执行findUsingInfo()方法：

		private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
		    FindState findState = prepareFindState();//1.得到一个FindState对象
			...
		}

分别看一下 prepareFindState 和 FindState :

		private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];//POOL_SIZE = 4
		private FindState prepareFindState() {
			synchronized (FIND_STATE_POOL) {
			for (int i = 0; i < POOL_SIZE; i++) {//遍历复用池
		            FindState state = FIND_STATE_POOL[i];
		            if (state != null) {//如果找到可复用state，将该位置清空，返回state
						FIND_STATE_POOL[i] = null;
		                return state;
					}
		        }
		    }
			return new FindState();//没找到的话自己new一个
		}

prepareFindState 主要是得到一个 FindState ，那么看一下 FindState 类的结构：

		static class FindState {
			final List<SubscriberMethod> subscriberMethods = new ArrayList<>();//订阅者的方法的列表
		    final Map<Class, Object> anyMethodByEventType = new HashMap<>();//以EventType为key，method为value
		    final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();/以method的名字生成一个methodKey为key，该method的类(订阅者)为value
		    final StringBuilder methodKeyBuilder = new StringBuilder(128);//构建methodKey的StringBuilder
		
			Class<?> subscriberClass;//构建methodKey的StringBuilder
			Class<?> clazz;//当前类
		    boolean skipSuperClasses;//是否跳过父类
			SubscriberInfo subscriberInfo;
		
		    void initForSubscriber(Class<?> subscriberClass) {//clazz为当前类
				this.subscriberClass = clazz = subscriberClass;
				skipSuperClasses = false;
				subscriberInfo = null;
			}
		}

继续findUsingInfo的流程：

		private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
		    FindState findState = prepareFindState();//1.得到一个FindState对象
			findState.initForSubscriber(subscriberClass);//2.subscriberClass赋值给findState
		    while (findState.clazz != null) {//3.findState的当前class不为null
		        findState.subscriberInfo = getSubscriberInfo(findState);//4.默认情况下，getSubscriberInfo()返回的是null
		        if (findState.subscriberInfo != null) {//5.那么这个if判断就跳过了
		            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
		            for (SubscriberMethod subscriberMethod : array) {
						if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
		                    findState.subscriberMethods.add(subscriberMethod);
						}
		            }
		        } else {//5.来到了这里
		            findUsingReflectionInSingleClass(findState);
			}
		        findState.moveToSuperclass();//3.将当前clazz变为该类的父类，然后再进行while循环的判断
			}
			return getMethodsAndRelease(findState);//3.将当前clazz变为该类的父类，然后再进行for循环的判断
		}

findUsingReflectionInSingleClass() 很重要，在这个方法中找到了哪些是订阅者订阅的方法和事件：

		private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
		private static final int BRIDGE = 0x40;
		private static final int SYNTHETIC = 0x1000;
		
		private void findUsingReflectionInSingleClass(FindState findState) {
		    Method[] methods;//通过反射，获取到订阅者的所有方法
		    try {
				// This is faster than getMethods, especially when subscribers are fat classes like Activities
				methods = findState.clazz.getDeclaredMethods();
			} catch (Throwable th) {
				// Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
				methods = findState.clazz.getMethods();
				findState.skipSuperClasses = true;
			}
			for (Method method : methods) {
				int modifiers = method.getModifiers();//拿到修饰符
		        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {//判断是否是public，是否有需要忽略修饰符（事件处理方法必须为public）
		            Class<?>[] parameterTypes = method.getParameterTypes();//获得方法的参数 
		            if (parameterTypes.length == 1) {//EventBus只允许订阅方法后面的订阅事件是一个
		Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
		                if (subscribeAnnotation != null) {	//判断该方法是不是被Subcribe的注解修饰着的
		                    Class<?> eventType = parameterTypes[0];//确定这是一个订阅方法
		                    if (findState.checkAdd(method, eventType)) {
		                        ThreadMode threadMode = subscribeAnnotation.threadMode(); 	//通过Annotation去拿一些数据
								findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
								subscribeAnnotation.priority(), subscribeAnnotation.sticky())); //添加到subscriberMethods中
							}
		                }
		            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
		                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
		                throw new EventBusException("@Subscribe method " + methodName +
		"must have exactly 1 parameter but has " + parameterTypes.length);
					}
		        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
		            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
		            throw new EventBusException(methodName +
		" is a illegal @Subscribe method: must be public, non-static, and non-abstract");
				}
		    }
		}

补充，里面涉及到了反射的相关知识：

[菜鸟学Java（十四）——Java反射机制（一）](http://blog.csdn.net/liushuijinger/article/details/14223459)

[菜鸟学Java（十四）——Java反射机制（二）](http://blog.csdn.net/liushuijinger/article/details/15378475)

[Java反射机制知识点](http://blog.csdn.net/stevenhu_223/article/details/9286121)

[相关方法的API](http://tool.oschina.net/apidocs/apidoc?api=jdk_7u4)

[getDeclaredMethods()
getMethods()](http://blog.sina.com.cn/s/blog_605f5b4f0100i77k.html)

[getModifiers()](http://blog.csdn.net/zhangfei_jiayou/article/details/7341936)：返回int类型值表示该字段的修饰符

getParameterTypes()：按方法声明的顺序返回参数类型数组

getAnnotation()：返回这个元素的注解指定类型

getDeclaringClass()：到目标属性所在类对应的Class对象

该方法流程是：  

1、拿到当前 class 的所有方法 

2、过滤掉不是 public 和是 abstract、static、bridge、synthetic 的方法
 
3、过滤出方法参数只有一个的方法
 
4、过滤出被Subscribe注解修饰的方法 

5、将 method 方法和 event 事件添加到 findState 中 

6、将 EventBus 关心的 method 方法、event 事件、threadMode、priority、sticky 封装成 SubscriberMethod 对象添加到 findState.subscriberMethods 列表中

那么，先来看看 findState.checkAdd() :

		boolean checkAdd(Method method, Class<?> eventType) {
			// 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
		    // Usually a subscriber doesn't have methods listening to the same event type.
			Object existing = anyMethodByEventType.put(eventType, method);
		    if (existing == null) {
				return true;
			} else {
				if (existing instanceof Method) {
					if (!checkAddWithMethodSignature((Method) existing, eventType)) {
					// Paranoia check
						throw new IllegalStateException();// 此时的情况是这个订阅者有多个方法订阅的是同一事件
					}
					// Put any non-Method object to "consume" the existing Method
					anyMethodByEventType.put(eventType, this);
				}
			return checkAddWithMethodSignature(method, eventType);
			}
		}

checkAdd 分为两个层级的 check 
第一层级只判断 event type，这样速度快一些；
第二层级是多方面判断。 

anyMethodByEventType 是一个 HashMap ， HashMap.put() 方法返回的是之前的 value ，如果之前没有 value 的话返回的是 null 

通常一个订阅者不会有多个方法接收同一事件，但是可能会出现子类订阅这个事件的同时父类也订阅了此事件的情况，那么 checkAddWithMenthodSignature() 就排上了用场：

		private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
			methodKeyBuilder.setLength(0);
			methodKeyBuilder.append(method.getName());
			methodKeyBuilder.append('>').append(eventType.getName());
		
			String methodKey = methodKeyBuilder.toString();
			Class<?> methodClass = method.getDeclaringClass();
			Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);//存储到以methodKey为Key，method的类为value的map中，返回之前methodKey存储的值
		    if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
				// Only add if not already found in a sub class
			return true;//如果这个值不存在或者这个值是method的类的父类的话，返回true
			} else {
			// Revert the put, old class is further down the class hierarchy
			subscriberClassByMethodKey.put(methodKey, methodClassOld);
		        return false;
			}
		}

这里的稍微逻辑有一些乱，但是其主要思想就是不要出现一个订阅者有多个方法订阅的是同一事件。

现在回过头来看一下 SubscriberMethod 这个类：将EventBus所需要的全封装起来了。

总结一下 findSubscriberMethods 的流程：

1、从复用池中或者 new 一个，得到 findState

2、将 subscriberClass 复制给 findState

3、进入循环，判断当前 clazz 为不为null

4、不为 null 的话调用 findUsingReflectionInSingleClass() 方法得到该类的所有的 SubscriberMethod

5、将 clazz 变为 clazz 的父类，再次进行循环的判断

6、返回所有的 SubscriberMethod

现在一层层 return 返回到了 findSubscriberMethods() 方法中，将所有的 SubscriberMethod 存储到 METHOD_CACHE 当中。

		public void register(Object subscriber) {
        	Class<?> subscriberClass = subscriber.getClass();
        	List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        	synchronized (this) {
            	for (SubscriberMethod subscriberMethod : subscriberMethods) {
                	subscribe(subscriber, subscriberMethod);
            	}
        	}
    	}

### 事件的处理与发送subscribe() ###

subscribe()方法：

		// Must be called in synchronized block
		private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
		    Class<?> eventType = subscriberMethod.eventType;//根据传入的响应方法名获取到响应事件(参数类型)
			Subscription newSubscription = new Subscription(subscriber, subscriberMethod);//封装一个Subscription出来，Subscription是将订阅者和订阅方法封装类(包括threadMode、sticky等)封装一起来了
			CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);//subscriptionsByEventType是以eventType为key，Subscription的ArrayList为value的HashMap，事件订阅者的保存队列，找到该事件所订阅的订阅者以及订阅者的方法、参数等
		    if (subscriptions == null) {//如果没有数据，说明此事件是还没有注册过的
		        subscriptions = new CopyOnWriteArrayList<>();
				subscriptionsByEventType.put(eventType, subscriptions);
			} else {//说明此事件是已经有地方注册过了的
				if (subscriptions.contains(newSubscription)) {
					throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
					+ eventType);
				}
		    }
			//根据优先级添加到指定位置
			int size = subscriptions.size();
		    for (int i = 0; i <= size; i++) {
				if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
		            subscriptions.add(i, newSubscription);
		            break;
				}
		    }
			//typesBySubscriber以subscriber为key，eventType的ArrayList为value的HashMap，订阅者订阅的事件列表，找到改订阅者所订阅的所有事件
		    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
		    if (subscribedEvents == null) {
		        subscribedEvents = new ArrayList<>();
				typesBySubscriber.put(subscriber, subscribedEvents);
			}
		    subscribedEvents.add(eventType);//添加到该订阅者的所有订阅方法列表中
		
		    if (subscriberMethod.sticky) {//如果是粘性事件
				if (eventInheritance) {//是否支持继承关系，就是记录事件的父类(比如事件是ArrayList类型的，那么如果eventInheritance为true的话，会去找为List类型的事件。)
					// Existing sticky events of all subclasses of eventType have to be considered.
		            // Note: Iterating over all events may be inefficient with lots of sticky events,
		            // thus data structure should be changed to allow a more efficient lookup
		            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
					Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();//stickyEvents以eventType为key，event为value的ConcurrentHashMap，Sticky事件保存队列
		            for (Map.Entry<Class<?>, Object> entry : entries) {
		                Class<?> candidateEventType = entry.getKey();
		                if (eventType.isAssignableFrom(candidateEventType)) {//是否有继承关系
		                    Object stickyEvent = entry.getValue();
							checkPostStickyEventToSubscription(newSubscription, stickyEvent);//分发事件
						}
		            }
		        } else {
		            Object stickyEvent = stickyEvents.get(eventType);
					checkPostStickyEventToSubscription(newSubscription, stickyEvent); //分发事件
				}
		    }
		}

该方法的流程：

1、首先判断是否有注册过

2、然后再按照优先级加入到 subscriptionsByEventType 的 value 的 List 中

3、而 subscriptionsByEventType 是事件订阅者的保存队列，找到该事件所有的订阅者以及订阅者的方法、参数等

4、然后再添加到 typesBySubscriber 的 value 的 List 中，而 typesBySubscriber 是订阅者订阅的事件列表，也就是找到该订阅者所有的事件

5、最后判断一下是否是粘性事件，是的话判断事件是否需要考虑继承关系，再分发这个黏性事件。

最后是调用checkPostStickyEventToSubscription()做一次安全判断

		private void checkPostStickyEventToSubscription(Subscription newSubscription, 	Object stickyEvent) {
			if (stickyEvent != null) {
			// If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
		        // --> Strange corner case, which we don't take care of here.
			postToSubscription(newSubscription, stickyEvent, Looper.getMainLooper() == Looper.myLooper());
			}
		}

就调用postToSubscription()发送事件了。

		private void postToSubscription(Subscription subscription, Object event, 	boolean isMainThread) {
			switch (subscription.subscriberMethod.threadMode) {
				case POSTING:
					//如果该事件处理函数没有指定线程模型或者线程模型为PostThread
            		// 则调用invokeSubscriber在post的线程中执行事件处理函数
		            invokeSubscriber(subscription, event);
		            break;
		        case MAIN:
					// 如果该事件处理函数指定的线程模型为MainThread
            		// 并且当前post的线程为主线程，则调用invokeSubscriber在当前线程（主线程）中执行事件处理函数
            		// 如果post的线程不是主线程，将使用mainThreadPoster.enqueue该事件处理函数添加到主线程的消息队列中
					if (isMainThread) {//如果现在是UI线程，直接调用
		                invokeSubscriber(subscription, event);
					} else {//否则加入到mainThreadPoster队列中
						mainThreadPoster.enqueue(subscription, event);
					}
				break;
		        case BACKGROUND:
					// 如果该事件处理函数指定的线程模型为MainThread
            		// 并且当前post的线程为主线程，则调用invokeSubscriber在当前线程（主线程）中执行事件处理函数
            		// 如果post的线程不是主线程，将使用mainThreadPoster.enqueue该事件处理函数添加到主线程的消息队列中
					if (isMainThread) {//如果现在是UI线程，加入到backgroundPoster队列中
						backgroundPoster.enqueue(subscription, event);
					} else {//否则直接调用
					    invokeSubscriber(subscription, event);
					}
					break;
		        case ASYNC://无论如何都加入到asyncPoster队列中
					asyncPoster.enqueue(subscription, event);
				break;
				default:
					throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
			}
		}

这里就关联到了我们之前讲的Poster类的作用了。

		void invokeSubscriber(Subscription subscription, Object event) {
			try {
		        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);// 通过反射调用事件处理函数
			} catch (InvocationTargetException e) {
		        handleSubscriberException(subscription, event, e.getCause());
			} catch (IllegalAccessException e) {
				throw new IllegalStateException("Unexpected exception", e);
			}
		}

就是通过反射调用订阅者类subscriber的订阅方法，并将event作为参数传递进去

总体的register的流程如下：

![](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/event-bus/event-bus/image/register-flow-chart.png)

## post()方法调用流程 ##

刚才已经提到了Post的有关内容了，接下来就分析Post

可以从代码的任何地方发送事件，此时注册了的且匹配事件的订阅者能够接收到事件。通过 EventBus.post(event) 来发送事件。

		/** Posts the given event to the event bus. */
		public void post(Object event) {
		    PostingThreadState postingState = currentPostingThreadState.get();//1.得到PostingThreadState
			...		
		}

currentPostingThreadState 是一个 ThreadLocal 对象，而 ThreadLocal 是线程独有，不会与其他线程共享的。（ThreadLocal 是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，而这段数据是不会与其他线程共享的。其内部原理是通过生成一个它包裹的泛型对象的数组，在不同的线程会有不同的数组索引值，通过这样就可以做到每个线程通过 get() 方法获取的时候，取到的只能是自己线程所对应的数据)
其定义为：

		private final ThreadLocal<PostingThreadState> currentPostingThreadState = new 	ThreadLocal<PostingThreadState>() {
			@Override
			protected PostingThreadState initialValue() {
			return new PostingThreadState();
			}
		};

其实现是返回一个 PostingThreadState 对象，而 PostingThreadState 类的结构是：

		/** For ThreadLocal, much faster to set (and get multiple values). */
		final static class PostingThreadState {
			final List<Object> eventQueue = new ArrayList<Object>();
		    boolean isPosting;
		    boolean isMainThread;
			Subscription subscription;
			Object event;
		    boolean canceled;
		}

PostingThreadState 封装的是当前线程的 post 信息，包括事件队列、是否正在分发中、是否在主线程、订阅者信息、事件实例、是否取消。

那么回到 post 方法中：

		/** Posts the given event to the event bus. */
		public void post(Object event) {
		    PostingThreadState postingState = currentPostingThreadState.get();//1.得到PostingThreadState
			List<Object> eventQueue = postingState.eventQueue;//2.获取其中的队列
			eventQueue.add(event);//2.将该事件添加到队列中
		
		    if (!postingState.isPosting) {//3.如果postingState没有进行发送
		        postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();//4. 判断当前线程是否是主线程
				postingState.isPosting = true;//5.将isPosting状态改为true，表明正在发送中
		        if (postingState.canceled) {
					throw new EventBusException("Internal error. Abort state was not reset");//6.如果取消掉了，抛出异常
				}
				try {
					while (!eventQueue.isEmpty()) {//7.循环，直至队列为空
		                postSingleEvent(eventQueue.remove(0), postingState);//8.发送事件
					}
		    	} finally {
		            postingState.isPosting = false;
					postingState.isMainThread = false;
				}
		    }
		}

最后走到一个 while 循环，判断事件队列是否为空了，如果不为空，继续循环，进行 postSingleEvent 操作，从事件队列中取出一个事件进行发送。

再看 postSingleEvent() 方法

		private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
		    Class<?> eventClass = event.getClass();
		    boolean subscriptionFound = false;
		    if (eventInheritance) {//是否查看事件的继承关系
		        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);	//找到事件的所以继承关系的事件类型
		        int countTypes = eventTypes.size();
		        for (int h = 0; h < countTypes; h++) {
		            Class<?> clazz = eventTypes.get(h);
					subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);//发送事件
				}
		    } else {
		        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);	//直接发送事件
			}
			if (!subscriptionFound) {//如果没有任何事件
				if (logNoSubscriberMessages) {
		            Log.d(TAG, "No subscribers registered for event " + eventClass);
				}
				if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
		                eventClass != SubscriberExceptionEvent.class) {
		            post(new NoSubscriberEvent(this, event));//发送一个NoSubscriberEvent的事件出去
				}
		    }
		}

还记得 EventBusBuild 中的 eventInheritance是做什么的吗？它表示一个子类事件能否响应父类的 onEvent() 方法。

再往下看 lookupAllEventTypes() 它通过循环和递归一起用，将一个类的父类,接口,父类的接口,父类接口的父类,全部添加到全局静态变量 eventTypes 集合中。之所以用全局静态变量的好处在于用全局静态变量只需要将那耗时又复杂的循环+递归方法执行一次就够了，下次只需要通过 key:事件类名 来判断这个事件是否以及执行过 lookupAllEventTypes() 方法。

然后我们继续往下，看发送方法 postSingleEventForEventType()：

		private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
		    CopyOnWriteArrayList<Subscription> subscriptions;
		    synchronized (this) {
		        subscriptions = subscriptionsByEventType.get(eventClass);//所有订阅了event的事件集合，register时已经保存了所有的方法
			}
			if (subscriptions != null && !subscriptions.isEmpty()) {
				for (Subscription subscription : subscriptions) {
		            postingState.event = event;
					postingState.subscription = subscription;
		            boolean aborted = false;
		            try {
		                postToSubscription(subscription, event, postingState.isMainThread);//这里调用的postToSubscription方法，上面有解析
						aborted = postingState.canceled;
					} finally {
		                postingState.event = null;
						postingState.subscription = null;
						postingState.canceled = false;
					}
					if (aborted) {
						break;
					}
		        }
				return true;
			}
			return false;
		}


它首先通过这一句

		subscriptions = subscriptionsByEventType.get(eventClass);

获取到所有订阅了 eventClass 的事件集合，之前有讲过， subscriptionsByEventType 是一个以 key:订阅的事件 value:订阅这个事件的所有订阅者集合 的 Map，在register的时候用到的。

最后通过循环，遍历所有订阅了 eventClass 事件的订阅者，并向每一个订阅者发送事件。

看它的发送事件的方法：

		postToSubscription(subscription, event, postingState.isMainThread);

又回到了和之前 Subscribe 流程中处理粘滞事件相同的方法里————对声明不同线程模式的事件做不同的响应方法，最终都是通过invokeSubscriber()反射订阅者类中的方法。


整个post过程，大致分为3步：

1、将事件对象添加到事件队列eventQueue中等待处理

2、遍历eventQueue队列中的事件对象并调用postSingleEvent处理每个事件

3、找出订阅过该事件的所有事件处理函数，并在相应的线程中执行该事件处理函数

![](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/event-bus/event-bus/image/post-flow-chart.png)

## unregister() ##

我们继续来看EventBus类，的最后一个入口方法unregister()

		/** Unregisters the given subscriber from all event classes. */
		public synchronized void unregister(Object subscriber) {
		    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);//typesBySubscriber以subscriber为key，eventType的ArrayList为value的HashMap，订阅者订阅的事件列表，找到改订阅者所订阅的所有事件
		    if (subscribedTypes != null) {
				for (Class<?> eventType : subscribedTypes) {
		            unsubscribeByEventType(subscriber, eventType); //取消注册subscriber对eventType事件的响应
				}
				typesBySubscriber.remove(subscriber); //当subscriber对所有事件都不响应以后,移除订阅者
			} else {
		        Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
			}
		}


		/** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
		private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
		    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);//subscriptionsByEventType是以eventType为key，Subscription的ArrayList为value的HashMap，事件订阅者的保存队列，找到该事件所订阅的订阅者以及订阅者的方法、参数等
		    if (subscriptions != null) {
				int size = subscriptions.size();
		        for (int i = 0; i < size; i++) {
		            Subscription subscription = subscriptions.get(i);
		            if (subscription.subscriber == subscriber) {
		                subscription.active = false;
						subscriptions.remove(i);
						i--;
						size--;
					}
		        }
		    }
		}

之前讲过typesBySubscriber key:订阅者对象 value:这个订阅者订阅的事件集合，表示当前订阅者订阅了哪些事件。 

取消过程：

1、首先遍历要取消注册的订阅者订阅的每一个事件，调用unsubscribeByEventType(),从这个事件的所有订阅者集合中将要取消注册的订阅者移除

2、最后将当前订阅者为 key 全部订阅事件集合为 value 的一个 Map 的 Entry 移除，就完成了取消注册的全部过程。

## EventBus工作原理 ##

最后我们再来从设计者的角度看一看EventBus的工作原理。

### 订阅的逻辑 ###

1、首先是调用register()方法注册一个订阅者A

2、遍历这个订阅者A的全部以onEvent开头的订阅方法

3、将A订阅的所有事件分别作为 key，所有能响应 key 事件的订阅者的集合作为 value，存入 Map< List>>

4、以A的类名为 key，所有Subscribe注解的参数类型的类名组成的集合为 value，存入 Map< List>>

5、如果是订阅了粘滞事件的订阅者，从粘滞事件缓存区获取之前发送过的粘滞事件，响应这些粘滞事件

### 发送事件的逻辑 ###

1、取当前线程的发送事件封装数据，并从封装的数据中拿到发送事件的事件队列

2、将要发送的事件加入到事件队列中去

3、循环，每次发送队列中的一条事件给所有订阅了这个事件的订阅者

4、如果是子事件可以响应父事件的事件模式，需要先将这个事件的所有父类、接口、父类的接口、父类接口的父类都找到，并让订阅了这些父类信息的订阅者也都响应这条事件

### 响应事件的逻辑 ###

1、发送事件处理完成后会将事件交给负责响应的逻辑部分

2、根据不同的响应模式响应


### 取消注册的逻辑 ###

1、首先是调用unregister()方法拿到要取消注册的订阅者B

2、从这个类订阅的时候存入的 Map< List>> 中，拿到这个类的订阅事件集合

3、遍历订阅时间集合，在注册的时候存入的 Map< List>> 中将对应订阅事件的订阅者集合中的这个订阅者移除

4、将步骤2中的 Map< List>> 中这个订阅者相关的 Entry 移除

### 工作原理图示 ###

![](http://kymjs.com/images/blog_image/20151211_8.png)


## 参考资料 ##

[EventBus 源码解析](http://www.codekk.com/open-source-project-analysis/detail/Android/Trinea/EventBus%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)

[EventBus源码研读(上)](http://kymjs.com/code/2015/12/12/01/)  

[EventBus源码研读(中)](http://kymjs.com/code/2015/12/13/01/ )

[EventBus源码研读(下)](http://kymjs.com/code/2015/12/16/01/)