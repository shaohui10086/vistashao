---
title: Android事件传递三部曲：本地广播LocalBroadcastManager
date: '2017-01-21'
spoiler: 原生支持的消息通知
---

我们都知道Android的四大组件，分别是：`Activity`, `Service`，`ContentProvider`以及`BroadcastReceiver`，实际开发中前两者接触的更多一点，后面两个虽然不怎么常用但是偶尔也会接触到，今天我们要说的就和`BroadcastReceiver`有关<!-- more -->，当我们想要去使用`BroadcastReceiver`会看到官方的提示：如果你不需要应用间的通信，可以考虑使用`LocalBroadcastManager`，会有更高的执行效率，因为它不涉及进程间通讯，而且不用担心普通广播可能产生的一些安全性问题，
`LocalBroadcastManager`是何许人也，听着好像是普通广播的阉割版，实际使用上看，他们确实有些相似，只是`LocalBroadcast`不能实现跨进程，但当我们揭开它神秘面纱，你就会发现，它其实和普通的广播一点关系都没有，如果非得扯出点关系的话，那就是他们都借助了`BroadcastReceiver`这个类来担当`receiver`的角色，


## 基本使用
如果你之前有使用过普通的广播，你会发现在方法调用上，`LocalBroadcastManager`和普通的广播是一模一样，不同的`LocalBroadCastManager`的调用方不再是`context`，而是`LocalBroadCastManager`的实例，所以所有的逻辑都是在`LocalBroadCastManager`的掌控之内。首先，我们还是从最基本的使用场景出发，从最基本的使用方法开始跟踪，看它是怎么将消息传递的：
```
public class Test extends Activity {
    private static final String ACTION = "simple_action";
    private static final String DATA = "data";

    BroadcastReceiver mReceiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 新建一个receiver
        mReceiver = new MyReceiver();
        // 注册receiver
        LocalBroadcastManager.getInstance(this)
                .registerReceiver(mReceiver, new IntentFilter(ACTION));

        // 发送消息
        Intent messageIntent = new Intent(ACTION);
        messageIntent.putExtra(DATA, "给xxx的一封信");
        LocalBroadcastManager.getInstance(this).sendBroadcast(messageIntent);
    }

    @Override
    protected void onDestroy() {
        // 取消注册
        LocalBroadcastManager.getInstance(this).unregisterReceiver(mReceiver);
    }

    class MyReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            // 处理消息
            Log.i("TAG", "收到一封信：" + intent.getStringExtra(DATA));
        }
    }
}
```
上面这段代码，就是最常见的`LocalBroadcastManager`的使用方式了，有没有似曾相识的感觉，没错，在`EventBus`中用到的也是类似功能的方法，发现都是一样的套路，无非就是注册，取消注册，发送消息以及处理消息，知道了具体的套路，那
么我们就来看一下它是具体怎么实现的。
## 熟悉的套路
从注册开始，不过在注册之前，我们要拿到一个`LocalBroadcastManager`的实例，所以先从`LocalBroadcastManager.getInstance()`开始：
### 初始化
```
    public static LocalBroadcastManager getInstance(Context context) {
        synchronized (mLock) {
            if (mInstance == null) {
                // 在这里，我们看到不管我们传过来的是哪种类型的context，最后构造方法用的都只是applicationContext，所以不用担心内存泄露的问题
                mInstance = new LocalBroadcastManager(context.getApplicationContext());
            }
            return mInstance;
        }
    }

    private LocalBroadcastManager(Context context) {
        mAppContext = context;
        // 看到了熟悉的Handler，而且用的还是mainLooper，已经猜到一些眉目
        mHandler = new Handler(context.getMainLooper()) {
            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case MSG_EXEC_PENDING_BROADCASTS:
                        // 具体处理消息的方法，会在处理事件时展开
                        executePendingBroadcasts();
                        break;
                    default:
                        super.handleMessage(msg);
                }
            }
        };
    }
```
我们看到`LocalBroadcastManager`采用的是经典的单例实现，所以它并没有用我们传入的`context`，因为那样肯定会导致内存泄露，而是保留的`applicationContext`。
在`LocalBroadcastManager`的构造方法中，我们还看到了利用`context.getMainLooper()`创建了一个主线程的`Handler`，所以我们可以猜到，它的线程间通信很有可能是通过`Handler`实现的，后面的分析会慢慢证实我们的猜想，继续往下看注册：

### 注册
```
    // 和EventBus如出一辙的套路，同样定义了两个HashMap
    // 一个以receiver为键，保存receiver相关的所有匹配规则，方便取消注册的时候处理
    // 一个以action为键，保存和这个action相关的所有ReceiverRecord，类比EventBus中的Subscription
    private final HashMap<BroadcastReceiver, ArrayList<IntentFilter>> mReceivers = new HashMap<>();
            
    private final HashMap<String, ArrayList<ReceiverRecord>> mActions = new HashMap<>();
            
    public void registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        synchronized (mReceivers) {
            // 新建一条订阅记录
            ReceiverRecord entry = new ReceiverRecord(filter, receiver);
            // 新增一条和这个receiver的匹配记录
            ArrayList<IntentFilter> filters = mReceivers.get(receiver);
            if (filters == null) {
                filters = new ArrayList<IntentFilter>(1);
                mReceivers.put(receiver, filters);
            }
            filters.add(filter);
            
            // 将filter中的所有action取出，给相应的action增加订阅记录
            for (int i=0; i<filter.countActions(); i++) {
                String action = filter.getAction(i);
                ArrayList<ReceiverRecord> entries = mActions.get(action);
                if (entries == null) {
                    entries = new ArrayList<ReceiverRecord>(1);
                    mActions.put(action, entries);
                }
                entries.add(entry);
            }
        }
    }
```
和EventBus一样的套路，保存了两个`HashMap`，一个用于缓存和这个`receiver`相关的所有匹配规则，一个用于缓存`action`相关的所有订阅记录，前者主要用在`unRegister`的时候，方便将和这个`receiver`相关的所有匹配规则一次性移除；而后者就主要用在匹配事件上，当收到新的广播，根据广播中的`action`取出所有和这个`action`有关的所有订阅记录，然后挨个通知订阅记录中的`receiver`处理事件。在这点处理上，`EventBus`和`LocalBroadcast`一模一样，趁热打铁，我们决定先看一下取消注册的逻辑来验证我们的猜想：
### 取消注册
```
    public void unregisterReceiver(BroadcastReceiver receiver) {
        synchronized (mReceivers) {
            // 首先移除和这个receiver相关的所有匹配规则
            ArrayList<IntentFilter> filters = mReceivers.remove(receiver);
            if (filters == null) {
                return;
            }
            // 然后把匹配规则中所有的action取出，在每个action的订阅列表中，把和这个receiver相关的订阅记录移除
            for (int i=0; i<filters.size(); i++) {
                IntentFilter filter = filters.get(i);
                for (int j=0; j<filter.countActions(); j++) {
                    String action = filter.getAction(j);
                    ArrayList<ReceiverRecord> receivers = mActions.get(action);
                    if (receivers != null) {
                        for (int k=0; k<receivers.size(); k++) {
                            if (receivers.get(k).receiver == receiver) {
                                receivers.remove(k);
                                k--;
                            }
                        }
                        if (receivers.size() <= 0) {
                            mActions.remove(action);
                        }
                    }
                }
            }
        }
    }
```
和我们猜测的一模一样，在注册的时候两个`HashMap`都在这里发挥了作用，将`receiver`的引用移除，并且把与之相关所有的订阅事件都移除
### 发送事件
```
    public boolean sendBroadcast(Intent intent) {
        synchronized (mReceivers) {
            // 取出intent中所有信息，用于下面的匹配
            final String action = intent.getAction();
            final String type = intent.resolveTypeIfNeeded(
                    mAppContext.getContentResolver());
            final Uri data = intent.getData();
            final String scheme = intent.getScheme();
            final Set<String> categories = intent.getCategories();
            
            // 先匹配action，将相应action下的订阅记录列表取出
            ArrayList<ReceiverRecord> entries = mActions.get(intent.getAction());
            if (entries != null) {

                ArrayList<ReceiverRecord> receivers = null;
                for (int i=0; i<entries.size(); i++) {
                    ReceiverRecord receiver = entries.get(i);
                    
                    // 这一步，我个人觉得有点多余，它的本意是筛除掉订阅记录中的重复，但是因为它的put方法都是在掌控之内，
                    // 每次放入的对象都是new出来的，不会出现放入重复的对象，所以这个字段根本不会生效
                    if (receiver.broadcasting) {
                        continue;
                    }
                    // 在匹配recevier的各项指标都符合，就将它添加到意向列表中
                    int match = receiver.filter.match(action, type, scheme, data,
                            categories, "LocalBroadcastManager");
                    if (match >= 0) {
                        if (receivers == null) {
                            receivers = new ArrayList<ReceiverRecord>();
                        }
                        receivers.add(receiver);
                        receiver.broadcasting = true;
                    } 
                }

                if (receivers != null) {
                    for (int i=0; i<receivers.size(); i++) {
                        receivers.get(i).broadcasting = false;
                    }
                    // 在等待广播列表中增加一条广播记录，每条广播记录都包含它的intent以及recevier列表
                    mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));
                    if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                        mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
                    }
                    return true;
                }
            }
        }
        return false;
    }
```
又是熟悉的一幕：两步匹配，先匹配`action`，找到所有符合`action`的订阅记录，然后再通过`IntentFilter.match()`方法精准匹配，确定找到的每条订阅记录都是符合要求的。然后把原始的`intent`和它的`receivers`组装成一条`BroadcastRecord`，并添加到待处理列表，然后检查`Handler`是否已经在处理事件，如果没有则发一条特定消息唤醒它，然后进入消息处理阶段。
由此，我们可以看到，不同于`EventBus`发送事件和处理事件有时候是同步的，`LocalBroadcast`都是异步的，`sendBroadcast`只是将根据要发送的`intent`找到它合适的`receivers`列表，然后添加到待处理列表，然后通过发送`Handler`消息的方式唤醒广播处理器，所以发送广播不用担心堵塞线程

### 处理事件
发送广播的最后，我们看到它发送了一条用于唤醒广播处理过程的消息，
```
    private void executePendingBroadcasts() {
        // 这是一个无限循环，只有把待处理广播列表都处理完才会退出
        while (true) {
            BroadcastRecord[] brs = null;
            // 为了减少对mReceivers的加锁时间，把列表的元素转移到另一个数组中
            synchronized (mReceivers) {
                final int N = mPendingBroadcasts.size();
                if (N <= 0) {
                    return;
                }
                brs = new BroadcastRecord[N];
                mPendingBroadcasts.toArray(brs);
                mPendingBroadcasts.clear();
            }
            // 将待处理广播都取出来进行处理
            for (int i=0; i<brs.length; i++) {
                BroadcastRecord br = brs[i];
                // 通知receiver列表中每个receiver
                for (int j=0; j<br.receivers.size(); j++) {
                    // 终于在这里调用了receiver的onReceive方法
                    br.receivers.get(j).receiver.onReceive(mAppContext, br.intent);
                }
            }
        }
    }
```
### 同步发送
除了异步调用，`LocalBroadcastManager`还提供了同步调用的方法：
```
    public void sendBroadcastSync(Intent intent) {
        if (sendBroadcast(intent)) {
            executePendingBroadcasts();
        }
    }
```
类比异步的发送广播方法，同步发送只是增加了在找到订阅记录之后，不等`Handler`被唤醒，直接在发送广播的线程直接执行了广播事件的处理，所以`receiver`的`onReceive`方法可能执行在任意线程，决定权在于发送的是哪种广播，如果是异步广播，那么将在主线程执行，如果是同步广播，那么将会在发送广播所在的线程执行.

我们在构造`LocalBroadcastManager`的时候就已经看到`Handler`在`handlerMessage`方法中只处理一个特定的`message`类型，就是用于唤醒消息处理的，发送广播的最后就是发送的这条消息。因为`Handler`使用的是`Looper.mainLooper`，所以最后`receiver`的`onReceive`方法都是执行在主线程，所以要求我们不能在`onReceive`方法中完成太多的费时操作。

## 适用场景
分析完之后，我们就大概能想到`LocalBroadcast`的适用场景了，就是当我们想进行一些费时操作，所有把它扔在了异步线程去处理，当我们想知道它具体执行的进度的时候就可以用发送`LocalBroadcast`的方式实现，为什么不是直接使用`Broadcast`，因为`Broadcast`会涉及到`Binder`机制，必然效率会大打折扣，而且还有广播使用不当的话，还可能产生一些安全问题。可能还会有同学问，为什么我不是直接用`Handler`实现，类似`AsyncTask`，就是直接使用的`Handler`实现的线程间通信，没错，用`Handler`是可以满足需求，但是如果在一些特定场景下需要通知多个`receiver`，比如在后台下载更新包，下载完成以后每个页面都可能需要响应，这时候怎么处理呢？如果单纯使用`Handler`肯定是不能满足需求的，因为我们在分析`Handler`的时候就提到过，`Handler`发送的每条`Message`都只有一个`target`，那就是发送这条消息的那个`Handler`实例，所以它只会被处理一次。而使用`LocalBroadcast`就省心很多，每个页面都对目标`action`进行监听，这样下载成功的事件就能被所有的页面感知到。总之，因为`LocalBroadcast`在`Handler`之上进行了一些封装，所以在对付特定需求时，用起来会更加得心应手，具体取舍，还是看需求，哪个用起来方便就用哪个。

## 对比EventBus
整篇文章，我们多次提到了`EventBus`，因为我们发现，他们的使用方式是如此相似，而且`LocalBroadcast`能实现的功能，用`EventBus`也能实现，比如我们继续讨论上面那个需求，用`EventBus`也能满足需求，而且使用上好像更方便一些，因为不用定义那么多`BroadcastReceiver`的子类，只需要在用到的地方加一个方法就行了。

如果联想到我们上篇文章对`EventBus`的分析，发现`LocalBroadCastManager`其实就是一个简化版的`EventBus`，它简化了两点:
1. 限制`Subscriber`的类型，这样就省去了`EventBus`中`SubscriberMethodFinder`这个重量级嘉宾，在`LocalBroadCastManager`中，所有的`Subscriber`都是`BroadcastReceiver`的子类，`SubscriberMethod`就是`BroadcastReceiver`的`onReceive()`方法，这样在代码的简洁程度上，`LocalBroadCastManager`完胜`EventBus`，但是同时带来的缺点就是不灵活。
2. 限制`threadMode`，`EventBus`提供了四种`threadMode`，分别是`MAIN`，`POSTING`，`BACKGROUND`以及`ASYNC`，`LocalBroadCastManager`通过发送同步广播和异步广播，实现了主线程处理和发送事件线程处理两种模式，类比到`EventBus`，就是`MAIN`和`POSTING`。
因为这两点限制，而且`EventBus`在使用上不仅不比`LocalBroadCastManager`复杂，而且还多了很多Feature，导致了`LocalBroadCastManager`只能闪在一边，没有任何登场的机会。

## 内存泄露
Handler、EventBus以及LocalBroadcastManager三者如果处理不当都有可能引发内存泄露，具体来说：`Handler`因为发送的每个`Message`对象有持有一个对发送者`Handler`的引用，方便到处理消息的时候调用`handler`的`handleMessage()`方法，所以在把`Handler`当做内部类使用时，一定要使用`static`，不然因为内部类会持有外部类的引用，会导致外部类泄露，直到消息被处理并回收；`EventBus`和`LocalBroadCast`就类似了，因为它们都会在注册的时候将订阅者保存在订阅者列表中，所以在不使用的时候一定要调用取消订阅方法将对象移除

在最后给一句话总结`LocalBroadcastManager`：披着广播外衣的`Handler`

- [Android消息处理机制：Handler|Message](/2016/07/15/Android消息处理机制：Handler-Message/)
- [Android事件传递三部曲：事件总线EventBus(上)](/2017/01/20/android-messaging-2-eventbus/)
- [Android事件传递三部曲：事件总线EventBus(下)](/2017/01/20/android-messaging-2-eventbus-2/)
- Android事件传递三部曲：本地广播LocalBroadcastManager