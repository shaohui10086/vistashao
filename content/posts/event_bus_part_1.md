---
title: Android事件传递三部曲：事件总线EventBus(上)
date: '2017-01-20'
spoiler: First part of EventBus
---

常用的事件传递方式包括：Handler、BroadcastReceiver、Interface 回调、事件总线EventBus，除去回调这种相对简单的多的方式我们不讨论，Handler的原理已经在之前分析过，接下来要分析的就是EventBus以及BroadcastReceiver，然后最后分析他们各自有优劣以及适用场景。今天的主角就是EventBus
<!-- more -->
因为整个分析下来，EventBus涉及到的内容还是很多的，所以我将其分成了两个部分，分作上下篇，其中上篇主要简单分析事件具体是怎么在EventBus中传递的，怎么从发布者的手中到达订阅者手中，在这个分析中，我们会特意跳过一些比较难的部分，只是快速地了解整个EventBus的架构；而在下篇中，将详细解释上篇中跳过的部分，将EventBus中的每个特性解释清楚。

## 基本使用
首先还是从最基本的使用场景出发，一步步来跟踪：
```
public class Example {

    public void onCreate() {
        EventBus bus = EventBus.getDefault();
        bus.register(this);
        bus.post("给xxx的一封信");
    }

    @Subscribe
    public void subscribeEvent(String message) {
        Log.i("TAG", "收到：" + message);
    }
    
    public void onDestroy() {
        EventBus.unregister(this);
    }

}
```
这就是EventBus的最简单的使用方式了，从获得一个`EventBus`实例开始，然后添加订阅，发送消息，然后订阅方收到消息进行处理，再到最后在合适的时机取消订阅，主要就是：订阅，发送消息，接受消息，取消订阅这四个步骤，再抽象一下就是两对方法：订阅和取消订阅，发送和接受，接下来分析的大致思路也都是按照依次进行，
今天我们就来看下，最主要的就是三行代码，加一个Subscribe注解的public方法，这个消息是怎么从转回来的。首先我们从第一行代码看起：
```
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
```
没错，就是一个普通的单例，调用的就是默认的构造器，倒是看不出来有什么神奇之处，因为主要的处理都在它的构造器里：
```
	private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
	
	public EventBus() {
        this(DEFAULT_BUILDER);
    }

    EventBus(EventBusBuilder builder) {
        
        // 以事件类型为键，存放所有的由subscriber和subscriberMethod组成的subscription
        subscriptionsByEventType = new HashMap<>();
        // 已subscriber为键，存放subscriber订阅的所有事件类型
        typesBySubscriber = new HashMap<>();
        // 存放所有的sticky事件
        stickyEvents = new ConcurrentHashMap<>();
        
        // 不同的消息发送器，将根据ThreadMode决定使用哪种，下面会详细解释他们各自的功效
        mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        
        // 后面还有很多从EventBusBuilder取出的配置参数
        ...
    }
```
可以看到订阅方法列表以及订阅者列表都是`EventBus`的成员变量，所以各个`EventBus`之间的数据是不共享的，所以订阅者只会收到特定`EventBus`发送的事件，所以`new EventBus()`的适用范围比较小，因为你new出来的`EventBus`还是要保存起来，不然它没有任何用处，如果要全局使用的话，还得把它标记成静态变量，而这件事，EventBus已经帮我们做好了，那就是`EventBus.getDefault()`。当我们需要配置一些EventBus的属性时，这时候就得借助`EventBusBuilder`，具体能配置什么属性就不在这里展开，下面分析的时候涉及到我们会提及
有了`EventBus`具体实例之后我们就可以订阅它，接下来就是`register()`：
## 订阅
```
	public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        // 找出subscriber中的所有订阅方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);		
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                // 处理每一个订阅方法
                subscribe(subscriber, subscriberMethod);				
            }
        }
    }
```
在这里我们牵涉到了EventBus中很重要的一环，那就是：找出subscriber的所有订阅方法，在这里我们先行跳过，可以提前预告一下的是，它可以通过两种方式获取：一种是通过反射，一种是通过AnnotationProcessor。继续向下看就是处理每一个订阅方法

```
	private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        // 创建一个由subscriber和subscriberMethod组成的对象
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        // 找到eventType对应的订阅列表中，如果没有则新建一个
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);		
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
        
        // 将新建的Subscription对象添加到对应的列表中，添加顺序由priority决定，后面会详细解释
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);			                                        // 3
                break;
            }
        }
        // 保存subscriber订阅的所有事件类型，方便取消订阅
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);                                        
        }
        subscribedEvents.add(eventType);
        
        // 省去数行代码，关于sticky
    }
```
别看这段代码有点长，其实主要就做了两件事，更新了两个最重要的Map：`subscriptionsByEventType`和`typesBySubscriber`，已备我们在后面`post`事件时使用。其中，前者保存了已`eventType`为键的所有相关的`subscription`。
到这里为止，整个事件的订阅就结束了，我们拿到了什么？两个填充了必要数据的HashMap，接下来就是这两个HashMap发挥作用的时候了，预告一下这两个HashMap的作用，一个是用于发送事件，一个是用于取消订阅。整个的流程图大概就是整个样子(图片来自codeKK)，最后的Sticky事件不用理会，会在下篇再解释:
![注册流程](https://img.vistashao.com/eventbus_register.png)
## 发送事件
```
    // 有关ThreadLocal，大家可以看其他的相关文章，不在这里展开，它是和线程绑定的
    private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
            @Override
            protected PostingThreadState initialValue() {
                return new PostingThreadState();                                  
            }
        };
        
    public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();         
        List<Object> eventQueue = postingState.eventQueue;
        // 将要发送的事件添加到事件队列
        eventQueue.add(event);                                                     

        if (!postingState.isPosting) {
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();   
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                // 开始分发，将当前线程event队列所有对象全部清空并分发出去
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);           
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```
首先在发送事件所在的线程创建一个`PostingThreadState`对象，这个对象有一个事件队列，将新的事件添加到队列中，然后开始分发并将发送过的事件移除队列，直到将队列中所有的事件发送完毕。
> 其实在这里，我有点疑惑，为什么要用到`ThreadLocal`和`eventQueue`，为什么不是直接调用`postSingleEvent()`，表面上看的是防止同一个线程中消息发送堵塞，所以利用`eventQueue`来缓存将要发送的事件，但是在同一个线程中，代码都是顺序执行的，也只有等上一个事件post完成之后，下一个post才会执行，所以在同一个线程中使用队列并不能达到防治堵塞的目的，欢迎有了解的朋友指教

```
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        // 检查是否支持事件继承，如果支持，则会找出当前事件的所有父类以及父类接口，
        // 订阅了它的父类或者父类接口的subscriber也会收到消息，默认是true
        if (eventInheritance) {                                                  
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);           
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);      
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);     
        }
    }
    
    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            // 我们在订阅时填充的订阅事件列表在这里使用
            subscriptions = subscriptionsByEventType.get(eventClass);                             
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    // 发送到目标订阅者的目标订阅方法
                    postToSubscription(subscription, event, postingState.isMainThread);           
                    // 每发送一次，检查一下事件是否被取消
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

```
在这里，终于将发送的时间和`subscriber`联系了起来，根据发送事件的类型找到订阅了这个事件的订阅列表，通知列表中每个订阅者：
```
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        // 省略线程调度相关代码，会在下面再详细解释
        invokeSubscriber(subscription, event);      
    }
    
    void invokeSubscriber(Subscription subscription, Object event) {
        // 省略一些代码
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);        // 2
    }
```
终于走到了最后，看到了`subscriber`的`subscriberMethod`最终通过反射的方式被执行，整个事件分发的过程也就到此结束了。整个函数流程图是这样的(图片来自codeKK):
![EventBus发送事件](https://img.vistashao.com/eventbus_post.png)

## 取消订阅
因为篇幅的原因，取消订阅的方法我们就不在这里展示，估计大家也能想到到：就是通过`typesBySubscriber`这个HashMap找到订阅者订阅的所有事件，然后在`subscriptionsByEventType`的各个事件订阅列表中将这个订阅者对应的订阅事件移除，最后再将订阅者从`typesBySubscriber`移除。

## 总结
至此，简单的EventBus事件流就算解释清楚了，主要就是两个方法，一个是`register`，一个是`post`，分别用于注册保存`subscriber`和消息通知`subscriber`， 在注册的过程中，通过`subscriberMethodFinder`将`subscriber`中所有的订阅方法都找出来，并根据两种不同的规则分别按照`eventType`和`subscriber`将订阅方法缓存在`Map`中，以供后续的步骤使用；在`post`事件中，是以线程为单位的，将事件放入当前线程的消息队列中，然后依次循环取出发送，直至队列为空，按照事件的类型找到当前事件类型相关的`Subscription`列表，如果当前bus还支持事件继承的话，还会找到当前事件类型的父类和父类接口对应的`Subscription`列表，找到这样的订阅列表之后，顺序取出每一个`Subscription`，然后通过反射调用`subscriber`的订阅方法，至此，整个事件的传递就结束了。分析完整个流程之后，我们在看EventBus的类图(图片来自codeKK):
![EventBus类图](https://img.vistashao.com/eventbus_class.png)
依照这个图中，再回忆我们分析的整个过程:首先`EventBus`的实例都是依据`EventBusBuilder`创建的，我们可以自定义`EventBusBuilder`来达到定制`EventBus`的目的，`SubscriberMethodFinder`是用于找出订阅者中的所有的订阅方法，然后订阅方法和订阅者组成一个叫做`Subscription`的订阅事件，`EventBus`保存的就是`Subscription`的列表，类图下半部分我们会在下篇详细解释，在这里简单介绍下，最后执行订阅者的订阅方法时，不是简单的当前线程执行调用反射，而是根据订阅方法不同的`threadMode`选择不同的`Poster`将传递到目标线程然后在异步线程或者主线程执行

大概就是这些内容，当然我们还省去三个很重要的点没有说清楚，那就是：SubscriberMethodFinder、ThreadMode、以及StickEvent，会在下篇详细介绍。

- [Android消息处理机制：Handler|Message](/2016/07/15/Android消息处理机制：Handler-Message/)
- Android事件传递三部曲：事件总线EventBus(上)
- [Android事件传递三部曲：事件总线EventBus(下)](/2017/01/20/android-messaging-2-eventbus-2/)
- [Android事件传递三部曲：本地广播LocalBroadcastManager](/2017/01/21/android-message-3-localBroadcast/)