---
title: Android事件传递三部曲：事件总线EventBus(下)
date: '2017-01-20'
spoiler: Part two of the EventBus
---

在上篇中，我们主要分析了EventBus的基本使用，从最基本的订阅和取消订阅，发送事件和接受事件开始，简单看了下EventBus的事件传递的过程，我们知道：在注册的过程中，EventBus将订阅者以及订阅方法保存到订阅列表中，当发送事件的时候，从订阅列表中取出符合要求的订阅信息，通过反射调用订阅者的订阅方法，至此完成事件的传递，但是在这过程中，我们还省去了很多部分，比如如何找到订阅者的订阅方法，最后订阅者的订阅方法执行在哪个线程，发送Sticky事件怎么处理，这些问题，我们将继续分析。
<!-- more -->

## SubscriberMethodFinder
首先还是看它的构造方法
```
	SubscriberMethodFinder(List<SubscriberInfoIndex> subscriberInfoIndexes, boolean strictMethodVerification,
                           boolean ignoreGeneratedIndex) {
        // 这个我们在这里暂时不解释太多，后面用到的时候，我们在展开
        this.subscriberInfoIndexes = subscriberInfoIndexes;
        // 是否开启方法严格检查，如果开启，后面在检查订阅方法找到不符规则的方法会直接抛出异常，否则不做任何处理。默认是false
        this.strictMethodVerification = strictMethodVerification;
        // 是否忽略EventBus AnnotationProcessor 生成的代码，后面用到的时候再提，同样默认是false
        this.ignoreGeneratedIndex = ignoreGeneratedIndex;
    }
    
    // 找到subscriber中所有的订阅方法
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        // 首先试图从缓存中读取，如果命中将直接返回
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        
        // 根据配置的不同，选择不同的获取方法
        if (ignoreGeneratedIndex) {
            // 忽略AnnotationProcessor生成的文件，直接通过反射获取
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            // 通过AnnotationProcessor生成的文件获取所有的订阅方法，效率更高
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        
        // 如果在subscriber中没有找到任何订阅方法，则抛出异常
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            // 放到缓存中以提高效率
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```
在`EventBus`的构造器中，根据`EventBusBuilder`中的配置，以及通过`AnnotationProcessor`生成的`SubscriberIndex`文件（后面单独解释），生成一个`SubscriberMethodFinder`，然后在后面的`register`方法中我们也提到了它，现在终于是时候揭开它面纱的时候了，看它到底是怎么获取`subscriber`的所有订阅方法：我们先避开缓存不谈，`SubscriberMethodFinder`根据一个变量将获取方法分成了两路：一个是通过反射获取，效率相对较低，另一种就和`AnnotationProcessor`根据`Subscribe`注解自动生成的`SubscriberIndex`文件相关了，效率更高一些，因为它是在编译期就生成好的，我们先来看一下通过反射获取的逻辑：
```
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            // 获取findState当前指向的Class中包含的所有订阅方法
            findUsingReflectionInSingleClass(findState);
            // 如果支持订阅继承，findState将指向当前Class的父类，否则指向null, 退出循环。实现EventBus宣称的Subscriber inheritance特性
            findState.moveToSuperclass();
        }
        // 将findState中的subscriberMethods取出返回，并将findSate回收掉
        return getMethodsAndRelease(findState);
    }

    // 获取单个Class中的所有订阅方法并将其保存到findState的subscriberMethods中
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // 相比getMethods，getDeclaredMethods只会获取当前类声明的方法，不包括其继承的方法，效率更高
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // 尝试用简单的方法失败之后，直接用gatMethods，
            // 这时候返回的方法中已经包含其父类的所有方法，所有就将skipSuperClasses置为true，提高效率
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            // 这里就解释了为什么我们的订阅方法不是public时，不生效，但是具体为什么EventBus要求方法为public，我们会再做解释
            // 当然，除了public，还有一些其他的要求，包括不能包括abstract以及static
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                // 订阅方法要求只能有一个参数
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        // 通过eventType判断是否已经添加过，如果已经添加过，这次将直接跳过
                        // 具体判断的逻辑，有兴趣的同学可以看下，还有有些条条道道的，分成了两步来判断：先检查eventType，在检查方法名
                        if (findState.checkAdd(method, eventType)) {
                            // 将注解中的信息取出：theadMode，priority以及sticky，并将其添加到方法列表中
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                }
                // 省略了部分代码，用于处理方法不合规
            }
        }
    }
    
    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        // 省略部分代码
        return subscriberMethods;
    }
```
没什么可说的，就是通过反射将`subscriber`及其父类中所有的订阅方法找出来并返回
```
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        // 获取一个FindState并初始化，具体获取是通过对象池实现的，避免了重复创建对象
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            // 获取AnnotationProcessor生成的subscriberInfo，如果获取失败，将使用反射获取
            // 后续的操作就和通过反射获取的方式类似
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```
通过反射获取和通过`SubscriberIndex`文件获取，这两种方式唯一的不同就是获取`SubscriberMethod`的方式，剩下的操作都是类似的。

## Subscriber Index

根据EventBus官方文档中介绍：Subscriber Index是EventBus3.0新增的特性，是一个可选的优化加入subscriber的注册，为了开启这个还需要额外做一些工作，在`build.gradle`文件中添加如下代码：
```
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [ eventBusIndex : 'com.example.myapp.MyEventBusIndex' ]
            }
        }
    }
}

dependencies {
    compile 'org.greenrobot:eventbus:3.0.0'
    annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.0.1'
}
```
这样配置以后就再build项目就会生成自定义的`MyEventBusIndex`文件，然后将这个文件添加到`EventBus`配置中：
```
EventBus eventBus = EventBus.builder().addIndex(new MyEventBusIndex()).build();
```
这个就是`SubscriberMethodFinder`检查的`SubscriberIndex`文件，如果检查到这个文件存在，那么将使用这个文件中的信息注册`Subscriber`，反之则将用反射的方式，具体`SubscriberIndex`文件的结构就不在这里展开解释，具体有兴趣的朋友可以直接去看下EventBus源码，里面有一个单独的module`EventBusAnnotationProcessor`用于自动生成`SubscriberIndex`文件，大致的逻辑就是：找到每个`Subscribe`注解的方法，并将其中的信息读取出来，将类名，方法名等信息组成一个`SubscriberInfo`的结构，这样在注册的时候直接读取index文件中的`SubscriberInfo`，就知道了`Subscriber`的所有订阅方法及其相关信息，比通过反射获取效率更高。

## 线程调度:ThreadMode
可以说，到此为止`EventBus`已经很好的满足了我们的需求，就是提供一个简洁高效的组件间的通信方法，并且效率还算不错，但是`EventBus`的野心远不止于此，既然已经实现了通信，那么再进一步就是线程间通信，于是有了`threadMode`
说完了基础用法，聊点高级的用法，在基础使用中，我们可以看到当我们`post`一个事件时，虽然经过了好一番跟踪，还是找到了它的`subscriber`，然后通过`invoke`执行了它的订阅方法，默认没有进行线程的调度，如果我们在订阅方法中发生了一些比较耗时的操作，比如进行网路请求，这时候就会堵塞线程，最明显的表现就是跟在它后面的`subscriber`好久才能收到事件，怎么避免这样的事情发生呢，这就是`threadMode`的作用体现了，
它可以定义具体订阅方法执行的线程，分别是：
1. POSTING：默认就是它，即不进行任何调度，就发生在`post`事件所在的线程
2. MAIN：主线程，刚才我们已经看到了，它其实是通过`Handler`实现的，通过`Looper.mainLooper`将事件转发到主线程，所有最后订阅方法也是执行在主线程
3. BACKGROUND：和`ASYNC`类似，不同的是它会做一些思考，如果`post`事件就是发生在异步线程，则直接执行，如果不是异步线程，则新开线程执行，
它只占用一个线程，所以还是有可能堵塞消息，它只有在前一个事件执行完毕之后才会执行下一个事件，所以它不会产生很多线程
4. ASYNC：通过线程池实现，每来一个事件都会在新的线程中执行，保证不会堵塞消息

```
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            // 如果没有定义threadMode，默认就是这种，直接通过invoke执行subscriber中的订阅方法
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            // 如果已经在主线程了，效果和POSTING一样，否则将通过mainThreadPoster进行调度
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            // 和MAIN恰好相反，如果不在主线程则直接调用，如果在主线程则将通过backgroundPoster调度
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            // 不管在什么线程post事件，都会新开线程进行调度
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```
具体三个`poster`：`mainThreadPoster`、`backgroundPoster`以及`asyncPoster`就不在这里继续展开，大致解释一下就是：`mainThreadPoster`本质上只一个`Handler`，用于从异步线程调度到主线程；`backgroundPoster`和`asyncPoster`类似，都是继承`Runnable`而来，最后的执行都是通过`ExecutorService.execute()`方法在异步线程中执行，它们两者唯一的不同就是，前者是串行的，只有当上一个事件处理完毕以后，才会处理下一个，而后者则是完全并行的，可以同时进行多个事件的处理。
`EventBus`中涉及到线程调度的基本上就这些，通过这些分析，我们就能知道，当`post`一个事件的时候并不是完全是异步的，决定权还在于订阅的人采用的是哪种`threadMode`，所以不论是在`post`事件还是在选择`threadMode`上，都要慎重，以防出现界面卡顿和消息通知不及时等问题。

## Sticky事件
之前在分析简单事件的时候，我们跳过了很多东西，其中就有一些是和`sticky`事件相关的，在这里，我们统一分析一下。在`EventBus`和`Otto`的对照表中，`EventBus`也将这条当做了很重要的一个属性写了出来，就是缓存最常用的Event，`EventBus`实现的方式就是通过`stickyEvent`，而`Otto`是没有这种特性的。具体它的使用场景呢，我们先不提前臆想，等我们分析完代码，自然就能想到合适它的使用场景
我们在定义订阅方法的时候，可以指定它的`sticky`属性，默认不设置是false，如果把它设置成true，那么将在`register`的时候进行一些额外的操作，前面简单分析的时候我们主要看的是subscribe方法的前半段，这次我们看下后半段
```
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        ...
        // 如果这个订阅方法的sticky属性为true的话，将在订阅的同时收到之前缓存的StickyEvent
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
    
    private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            postToSubscription(newSubscription, stickyEvent, Looper.getMainLooper() == Looper.myLooper());
        }
    }
```
如果`subscriberMethod`的`sticky`的属性为true，那么将在`register`的同时，检查缓存的`StickyEvent`中是否有符合的event，如果有，将直接`post`。`StickyEvent`的缓存是怎么来的呢？也是之前`post`的，但是在`post`方法中并没有发现和`StickyEvent`缓存相关的逻辑啊，没错，`post`方法中确实没有相关的逻辑，因为那部分的逻辑在`postSticky`方法里：
```
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        post(event);
    }
```
`postSticky`其实只是在`post`方法之上增加了一段将`event`添加到`stickyEvents`的缓存中的逻辑，但是`stickyEvents`是一个以事件的类型为key的`HashTable`，其实是一个特殊的`HashTable`，但在这里就不做展开，用`HashTable`保证同类型的`StickyEvent`只会缓存一个，而且缓存的是最新的。
这就是`sticky`的全部逻辑了，只是增加了一个`HashTable`用于保存想要缓存的事件，增加了一个特殊的`post`事件的方法以及在订阅的方法中增加一段重放缓存事件的逻辑处理，就实现了整个`Sticky`的逻辑

## 答疑

- EventBus怎么实现事件中途取消
这个我们在分析`postSingleEvent()`的方法时提到了，每次根据事件类型找到对应的订阅列表以后，然后就开始循环挨个分发，在循环中，没执行完一次就会检查一次`postingState`的`cancel`状态，如果已经取消，将直接终止循环，这个事件的分发到此结束，也就实现了事件分发过程中的中途取消。

- 订阅了同一个事件的`Subscriber`之间的分发顺序是怎么确定的？
我们在分析`register()`源码的时候，注意到将`subscriber`和`subscriberMethod`添加到订阅列表时，会检查`subscriberMethod`的`priority`属性，列表中的元素都是按照`priority`从大到小排序，相同`priority`会按照添加顺序排序，先加入的会在前面，主要代码就是这段：
```
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);			                                      
                break;
            }
        }
```
也就是说：如果定义了`priority`，那么分发顺序将会按照`priority`从大到小排序，没有定义`priority`，会按照订阅顺序排序，默认`priority`都是一样的，值为0。
- subscribe注解中的`threadMode`属性有什么作用？

这个我们前面已经花了大篇幅解释`threadMode`的作用：它将决定你的`subscriberMethod`执行在哪个线程，所以一定要慎重选择，因为一旦选择不当，比如在`ThreadMode.MAIN`进行了太多费时的操作可能会导致程序ANR，在`ThreadMode.ASYNC`或者`ThreadMode.BACKGROUND`对UI进行更新，很有可能会直接抛出异常。所以建议大家对此有所了解。
- 在不需要的时候需要手动解除绑定吗？不解除绑定会导致内存泄露吗？

这两个问答的答案都是肯定的，在`subscriber`被销毁之前一定好手动解除绑定，否则会导致内存泄露，因为我们在添加订阅的时候会在订阅列表中添加`subscriber`的引用，了解Java中GC机制的同学都知道，如果一个对象是在GC发生时依然是可到达的，那么它将不会被回收，这就导致了内存泄露，所以在不再需要的时候一定要调用取消订阅方法。
- 为什么订阅方法必须是public

这个问题的答案就在`SubscriberMethodFinder`中，首先说就是，是EventBus限制了订阅方法必须使用`public`，如果不是`public`，这个方法就不会被添加到订阅方法列表中，至于为什么`EventBus`为什么要限制这点，可以看下面这段代码：
```
        try {
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
```
这是在通过反射拿到`subscriber`的所有方法，`getDeclaredMethods()`确实是不管三七二十一，会把`private`方法也拿到，但是`getMethods()`则不然，它只能拿到标记为`public`的方法，所以我估计是`EventBus`为了规避风险，索性要求所有的订阅方法必须标记为`public`

至此，`EventBus`的所有特性就看完了，也分析完了，大家如果有兴趣的话，也还是可以自己下源码然后在跟一遍，`EventBus`源码中的注释还是很多的，许多细节都描述的很清楚，看起来还是很轻松

## 参考链接
1. [EventBus 源码解析](http://a.codekk.com/detail/Android/Trinea/EventBus%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)
2. [EventBus 3.0 源代码分析](http://skykai521.github.io/2016/02/20/EventBus-3-0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
3. [事件总线 —— otto的bus和eventbus对比分析](http://frodoking.github.io/2015/03/30/android-eventbus-otto-analysis/)


- [Android消息处理机制：Handler|Message](/2016/07/15/Android消息处理机制：Handler-Message/)
- [Android事件传递三部曲：事件总线EventBus(上)](/2017/01/20/android-messaging-2-eventbus/)
- Android事件传递三部曲：事件总线EventBus(下)
- [Android事件传递三部曲：本地广播LocalBroadcastManager](/2017/01/21/android-message-3-localBroadcast/)