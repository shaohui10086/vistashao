---
title: Android消息处理机制：Handler|Message
date: '2016-07-15'
spoiler: Just because we can, doesn’t mean we should.
---

在日常开发中，不管出于什么目的，我们可能都会用到Handler来异步更新UI，有时是为了将一些费时的操作放到异步线程去处理，然后通过Handler将数据更新到UI线程，有时是为了在子线程里更新UI，种种原因，反正我们最后都是选择了直接的Handler+Message组合或者AsyncTask，而了解AsyncTask的同学都知道，AsyncTask内部就是通过Handler和Message实现的线程间通信，所以我们还是要好好熟悉一下这位老朋友


#### 为什么要使用Handler
我们使用Handler更多是为了异步操作，那么为什么非得要异步操作呢，直接在子线程更新UI不行吗？答案当然是不行的，不然Handler存在的意义是什么，这一切都是因为UI的控件都不是线程安全的，如果允许并发访问，那控件的状态就是未知的了，可能你刚获取完一个控件的状态，准备进行一些操作，这时候另一个线程改变了这个控件的状态，那就很麻烦了，解决这种问题的方法一般有两种：1.加锁，在对控件进行操作的时候先加锁，不允许其他人访问，这样的问题是会导致UI更新的效率会很差，而且容易堵塞某些线程，因为他需要等上一个访问这个控件的线程释放锁；所以Android选择的是第二种方法，只允许在一个线程内对UI控件进行更新，这个线程就是创建View时的线程，默认状态下，这个线程就是主线程，这也就是为什么我们在对UI组件进行更新的时候，必须回到主线程去操作。

> 经[極ws丶琪](http://weibo.com/2kpurplerose) 指正，更新View的线程不一定必须是主线程，而是创建该View的线程，因为默认状态下，我们创建View都是在主线程，所以就慢慢有种只能在主线程更新View的错觉，其实是：只能在创建View是所在的线程更新一个View。

#### Handler、MessageQueue和Looper
我一直在用Handler，不过总是对它一知半解，一直停留在会用的程度，以致于别人在提到MessageQueue和Looper的时候，我竟是一脸懵逼，当时觉得好羞愧，连这些都不知道，你还用什么Handler，回去搬砖吧！我相信也有很多的Android初学者跟我有同样的问题，所以在这里，我想先跟大家详细介绍一下这几个概念

##### Looper
Looper是一个用于给线程提供消息循环处理的类，其实在没有感知的情况下，创建一个Handler，看似简单，但是最少需要三步：在目标线程中执行`Looper.prepare()`：
```
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
省去部分细节我们不谈，我们只看我们最关心的，那就是它新建了一个Looper实例并把它放在了它的一个成员变量里，具体`ThreadLocal`的作用我们先不在这里展开，只需要记住它是跟线程相关的，每个线程从它这里读出来的数据都是不一样的。创建好Looper之后，我们就可以创建我们最终需要的Handler了：
```
public Handler() {
    this(null, false);
}
public Handler(Callback callback, boolean async) {
    ...
    
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
现在知道为什么要在新建Handler之前先执行`Looper.prepare()`了吧，没有Looper的线程，我们创建Handler实例会直接抛出异常。整个过程到这里就结束了吗？没有，我们虽然已经有了Looper的实例和Handler的实例，但是还需要最后一步开启消息循环，执行`Looper.loop()`：
```
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;
    
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        
        try {
            msg.target.dispatchMessage(msg);
        }
        msg.recycleUnchecked();
    }
}
```
源码其实比这要长，为了方便理解，我删除了一些没什么相关性的代码，可以看到，这个方法其实就是执行了一个无限循环，具体执行逻辑先不在这里展开，我们只需要知道，它每次循环都从MessageQueue取出一条新的消息，然后执行handler的dispatchMessage方法，当取出的消息为null就退出循环。所以看到这里，我想大家也就知道如何在一个新的线程里创建Handler开始消息处理了吧
``` java
new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                Handler handler = new Handler();
                Looper.loop();
            }
        }).start();
```
可能会有人问，为什么我们平常使用的时候，没有感觉到这么复杂呢？因为在主线程中，系统自动帮我们完成了这些事，有兴趣的同学可以去看一下`ActivityThread`的`main()`方法，就会看到关于Looper的这两行代码也都是执行的了，主线程生而就已经开始了消息循环，用一个示意图表示:
![ActivityThread示意](https://img.vistashao.com/handler_activity_thread.jpg)
从上面这张图中，我们可以看到我们在Activity中写的代码也都是执行在一条最外层`Handler`的`dispatchMessage`方法里的，所以我们能随便创建Handler也就是理所当然的事情

分析完源码之后，我们就大概理解了Looper的作用：开始一个无限循环，每次循环都试图从消息队列中取出下一条消息，然后进行相应的处理，下面我们分析消息队列的时候就会看到，当没有消息的时候，消息队列就会堵塞在那里，不返回消息，直到有新的消息进来，这时候Looper也会被堵塞在这里，直到有新的消息返回。Looper的角色就是一个任劳任怨的搬运工，因为它处理消息的过程都是在Looper内部，所以消息执行也就就是在Looper所在的线程，即Looper创建时所在的线程，这点很重要，它将最终解答我们最大的疑惑：为什么Handler在其他线程发送的消息最后处理都是在主线程处理？

上面我们是分析了Looper主要的两个方法：prepare()用来初始化当前线程的Looper，loop()用来开始Looper的循环，其实还有两个不是很常用的方法：quit()和quitSafely()，这两个方法都是用来退出循环的，不同在于，quit会立即停止循环，而quitSafely会在消息队列为空以后才跳出循环，保证所有的消息都已经被处理。

##### MessageQueue

MessageQueue是一个消息队列，用来存放Handler发送的消息，主要有两个操作:添加和读取，读取的同时伴随着消息从消息队列的移除，分别对应的方法就是：enqueueMessage(Message msg, long when)和next()，enqueueMessage方法就是将一条消息插入MessageQueue，插入的过程，会检查消息的时间决定具体插入的位置，保证消息的先后顺序，next方法相对会复杂一点，前面我们在分析Looper的时候就有所涉及，它是一个无限循环，返回值是一个Message，它的返回值就是用作Looper处理用的，前面已经说过：在插入消息的是就是按照发送时间排序好的，那么next只需要无脑返回消息队列顶部的消息就可以了吗？答案显然不是，因为Handler还可以发送延迟消息，所以有可能有消息在消息队列的顶部，但现在还不是合适的时机返回它，因为一旦返回它将立刻回处理，这就违反了我们发送`delayMessage`的初衷，所以在next()的方法内部当拿到消息队列首部的消息，会再判断一下现在是不是合适的时机返回它，如果不是，那么它将继续等待，直到有合适的消息需要它返回，所以MessageQueue担当的角色像是一辆马车，负责将具体的消息保存并运送到Looper的面前，供Looper处理
Message算是一个比较简单的类，它担当的角色就是一个具体的包裹箱，记录了'寄件人'和'收件人'以及具体货物信息，然后在Handler，MessageQueue和Looper之前传递，主要有这么几个属性：

- what：int型，最主要的属性，用于指定消息的类型，这算是Message唯一一个必须指定的值
- arg0，arg1：两个int型值，一般用来传递一些progress，或者一些简单的整型
- obj：可以传递一些复杂一些的object
- data：Bundle型，这个就不用多解释了，传递较多种数据的时候肯定会用到它
- callback：Runnable型，Handler.post(Runnable)内部就是设置的这个属性，这个一般不会手动设置，但是也会用到，只是我们感觉不到，下面会详细解释，用于覆盖Handler的默认处理逻辑

大致就这些属性了，详细了解Message的基本结构还是有必要的，传递什么数据，用什么方法，对以后的实际使用会有一些帮助。

##### Handler

说了这么多铺垫，准备工作算是差不多了，Handler也就要上场了，其实Looper，MessageQueue和Message，除了Message有些接触以外，其他两个在普通开发中其实是见的不多的，而Handler则是开发者接触最多的，因为几乎所有的处理逻辑都是写在Handler里的，发送消息和处理消息也都算是Handler接手的，但是我们平常是怎么创建一个Handler的呢? 无外乎两种：

- 创建Handler子类，并重写handlerMessage方法。
- 直接使用默认的Handler类，但是在新建Handler对象时，传入一个Callback对象。

上面说的这两种方法，我用的比较多的是第二种方法，因为用起来更加灵活一点。

#### 获得一个Message：obtain()
在发送一个Message之前 ，我们肯定要先得到一个Message对象，可以直接new一个Message对象出来，但是这种方法并不推荐，更推荐用Message的obtain方法，它使用对象池模式，创建了一个Message池，如果有闲置的Message就直接返回，不然就新建一个，用完以后，返回消息池，这种方法大大减少了创建Message对象的成本，而且有obtain方法有多种形式，基本能满足我们的多数需求：

- obtain() 返回一个空的Message
- obtain(Message msg) copy一个message并返回
- obtain(Handler h) 返回一个设置了target的message，设置了这个属性以后message就可以使用sendToTarget()方法，其实内部调用的就是Handler.sendMessage(this)
- obtain(Handler h, Callback callback) 同上，并设置message的callback属性
- obtain(Handler h, int what) 同obtain(Handler h)，并设置message的what属性
- obtain(Handler h, int what, Object obj)
- obtain(Handler h ,int what, int arg0, int arg1)
- obtain(Handler h, int what, int arg0, int arg1, Object obj)

这些方法都大同小异，无非就是预设几个属性，以后尽量还是要用obtain方法，算不上方便快捷吧，但至少算的上优雅。

##### 发送一个Message：sendMessage()
一个Message已经准备好了，蓄势待发，接下来的工作就是把它发射出去，这时候就要交给Handler，调用它的sendMessage(Message msg)，其实我们也可以不去费心得到一个Message对象直接用Handler.sendEmptyMessage(int what)，发送一个空message，而且Handler还有sendMessageDelay和sendMessageAtTime方法，这些有什么不同点，可以直接看源码

``` java
public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```

``` java
public final boolean sendEmptyMessage(int what)
   {
       return sendEmptyMessageDelayed(what, 0);
   }
```

``` java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

从这些代码中，我们可以看出来，sendEmptyMessage和sendMessage都是调用的sendMessageDelay方法，只不过sendEmptyMessage是在方法内部用Message.obtain()方法得到了一个空message，然后sendMessageDelay方法内部又是调用的sendMessageAtTime，所以殊途同归，最后只要看sendMessageAtTime就好了：

``` java
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

没错，又回来了，看到了enqueueMessage方法，就是这样，一条message被插入MessageQueue队列，接下来的视情节就交给Looper，它会循环调用MessageQueue的next方法，而next会在合适的时机返回要处理的Message，交给Looper处理。

#### 消息处理的过程
具体消息处理的过程又是怎样的呢？Handler的post方法和sendMessage有什么不同？都将在这一节揭晓。其实post的Runnable对象，有没有觉得很熟悉，刚才提到的Message也有一个属性是Runnable，就是callback，正如我们所料想的那样，post其实也是发送的一个message，和sendMessage一样，只不过它给message设置了属性CallBack，还是贴最诚实的源码：

``` java
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

从源码中我们可以看到，当message的callback属性不为空的话，就会调用handlerCallback(msg)方法，而这个方法内部则是直接运行的message的callback，消息处理结束；当message的callback为空时，就是这个message是一个普通的message，会先查看mCallback是否存在，如果存在的话，尝试调用mCallback的handlerMessage方法，根据返回值决定是否再调用Handler的handlerMessage方法，而这个mCallback变量就是我们在新建handler时，传入的那个Runable对象：

``` java
Handler handler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                return false;
            }
        });
 ```
 
 现在终于理解为什么在这个地方传入一个Runable对象也能拦截下消息处理了，至此，消息处理的优先级就出来了：

> Message的callback>handler的Callback>Handler的handlerMessage

#### 可能发生的问题：内存泄露
在使用Handler的时候，有时候处理不当就会导致内存泄露，当我们试图使用内部类或者匿名内部类重写Handler的handlerMessage()方法，一定要保证他们是静态，因为内部类和匿名内部类默认都会持有外部类的引用，我们上面已经分析过，handler发送的每条Message都会持有一个对发送者的引用，即对handler的引用，如果我们发送了延迟消息，直到我们关掉Activity的时候，消息还在消息队列里，就会发生内存泄露，引用链是这样的：
> messageQueue->message->handler->activity

解决方案就是把内部类或者匿名内部类分别设置成静态内部类和静态成员，这样就没有没有了对外部activity的引用，如果需要对外部Activity的引用可以给handler添加一个队外部类的弱引用。当然从上面的引用关系中，我们还可以有另外一种解决方案，那就是在退出的时候清空消息队列，这样也可以达到我们的目的。

#### Message上限
之前我记得有个同事问我Message是否有上限，我当时答不上来，后来去搜索也没搜索到，只查到Message.obtain()的消息池上限是50个，但是MessageQueue得上限还是不知道，或许是他问错了，或者是我理解错了，问题还是存在的，只是我还没找到答案，有知道的看官可以留言指导一下。

#### The End
虽然现在RxJava和EventBus用的比较火，切换线程处理事情以及线程间通信变得触手可及，Handler作为元老级人物出场机会变得越来越少，但是关于Handler的一些东西，作为开发者，我们还是有必要知道的，通过看任玉刚大大的书，还有网上的各种资料，最后算是对Handler和Message有了个一知半解，有不对的地方，希望大家批评指正。

熬夜写博客，真是一种挑战，困得要死，想着都要放弃了，明天再写，但是我知道以我性格一旦拖下去，基本上就废掉了，于是冰箱拿了瓶啤酒，夜里3点，终于写完了。

- Android消息处理机制：Handler|Message
- [Android事件传递三部曲：事件总线EventBus(上)](/2017/01/20/android-messaging-2-eventbus/)
- [Android事件传递三部曲：事件总线EventBus(下)](/2017/01/20/android-messaging-2-eventbus-2/)
- [Android事件传递三部曲：本地广播LocalBroadcastManager](/2017/01/21/android-message-3-localBroadcast/)