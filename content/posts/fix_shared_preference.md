---
title: 关于SharedPreference踩的那些坑
date: '2016-10-20'
spoiler: 看文章之前，首先思考一个问题：SharedPreference支持多进程吗？如果你知道的话，下面的内容可能对你没有太大的帮助，可以快读阅读，复习一下或者指教我写的不对的地方，如果你犹豫不决，或者不知道的话，下面的内容可能会对你有些帮助。
---

### SharedPreferences多进程

为什么会突然写这篇文章呢，主要是我昨天在用`SharedPreferences`的时候，涉及到多进程访问，写的时候没注意，然后导致数据不对，既然踩了坑，肯定要自己反思一下总结，所以也就有了这篇文章，简单总结一下，
首先直截了当得先回答文章开头那个问题，答案是：能但是不行。

为什么这么说呢，因为`SharedPreferences`确实可以通过设置`MODE_MULTI_PROCESS`实现多进程访问，而且是SDK2.3 之前是默认的，连这个标志都不用设置，SDK2.3之后就需要手动设置，既然这么说了，那就肯定是能了，但是为什么说不行呢？因为这个标志在 SDK6.0 的时候已经被Deprecated，Android为什么要把这么一个简单易用的进程间通讯方式废弃掉呢，看下源码就可以知道，原因就是它并不能保证数据的安全性和准确性，并没有实现并发修改的机制，所以官方也就不推荐了，而是推荐使用 ContentProvider。总结一句话：爱过。

### 基础知识大普及

既然看到了源码，索性就看完整一点，这样以后用的时候也更顺手一点，于是简单的把`SharedPreferences`相关的内容简单都看了下，下面就是自己简单的一些笔记：

我平时在使用 `SharedPreferences` 一般会直接用:

```
SharedPreferences sp2 = PreferenceManager.getDefaultSharedPreferences(this);
```

来获得一个`SharedPreferences`的引用来操作，正好借着这个机会好好看了下官方的介绍，终于算是对`SharedPreferences`有一个更完善的认识了，

```
SharedPreferences sp1 = getSharedPreferences(getPackageName() + "test", MODE_PRIVATE);
SharedPreferences sp2 = PreferenceManager.getDefaultSharedPreferences(this);
SharedPreferences sp3 = getPreferences(MODE_PRIVATE);
```

上面的三行代码都是为了获得一个`SharedPreferences` 引用，其中第三个方法是`Activity`专有的，其实我们看源码的话，就会发现其实后两个方法其实都是第一个方法的封装

```
 public static SharedPreferences getDefaultSharedPreferences(Context context) {
        return context.getSharedPreferences(getDefaultSharedPreferencesName(context),
                getDefaultSharedPreferencesMode());
    }
    
 private static String getDefaultSharedPreferencesName(Context context) {
        return context.getPackageName() + "_preferences";
    }
    
 private static int getDefaultSharedPreferencesMode() {
        return Context.MODE_PRIVATE;
    }
```

正如你所见，`getDefaultSharedPreferences()`就是返回了一个名字为`getPackageName() + "_preferences"`，mode 为`MODE_PRIVATE`的实例。

```
public SharedPreferences getPreferences(int mode) {
        return getSharedPreferences(getLocalClassName(), mode);
    }
```

而`getPreferences`就是返回了一个名字是以当前类名命名的，mode自定义的实例。


简单说完获得一个`SharedPreferences`引用的三个方法之后，简单看下这两个参数： name， mode, 分别是干嘛用的，首先说下`name`这个参数，正如源码中所说的，你只要知道了`name`,你就能访问到这个`SharedPreferences`中的所有数据，就是说`name`是`SharedPreferences`在系统的唯一标示，从这里，我想到的是，以后做多用户登录，保存多用户配置信息的时候，就可以用包名+用户名，设置不同的`name`来实现，这样用户注销或者删除之后，直接把对应`name`的`SharedPreferences`清空掉就OK了，不同`name`对应不同的`SharedPreferences`文件，在这里强插一句，`SharedPreferences`在手机中的保存方式其实是以xml文件的形式存储的，会在：data\data\程序包名\shared_prefs目录下，生成一个以`name`命名的xml文件。

说完了`name`，说下第二个参数:`mode`, 源码中提示，`SharedPreferenced`支持三个不同的mode， 

1. MODE_PRIVATE
2. MODE_WORLD_READABLE
3. MODE_WORLD_WRITEABLE

之前无感知的，一般用的都是`MODE_PRIVATE`， 也是现在唯一的mode，因为另外两个mode已经被废弃掉了，`mode`指定的是其他应用是否可以访问或者修改指定的`SharedPreferences`, 从字面上就能看的出来，`MODE_PRIVATE`表示其他应用不可读，不可写，`MODE_WORLD_READABLE`表示其他应用可读但不可写，`MODE_WORLD_WRITEABLE`表示其他应用可读可写，但是现在后两个mode已经被废弃掉了，官方给的提示是：全局可见的文件是很为危险的方式，容易导致安全漏洞，推荐更普遍的通信方式，比如：`ContentProvider`， `BroadcastReceiver`或者`Service`, 而且因为这个权限设置是在生成`SharedPreferences`文件的时候就设置的，但是不能保证这个权限设置在备份或者恢复的时候保留。总而言之，一句话，还是用`MODE_PRIVATE`吧，应用间通信不要考虑`SharedPreferences`这种方式，如果看到老的一些文章或者书，有推荐这种方式的，自己留意就好。

获得一个`SharedPreferences`之后，我们就可以对它进行读取或者写入了，这些大家有用过的都会比较熟悉，所以不在这里多写什么，无非就是`SharedPreferences`支持几种基本的数据类型：`int`,`float`, `long`,`String`,`boolean`,还有一种在我看来比较特殊的：`Set<String>`,写入的方式：同步`commit()`和异步`apply()`。这些都是日常会用到的，下面主要说一些可能大家没有用过的（其实是我自己没用过，所以觉得比较神奇）：

1. 	contains(String key)
这个方法我是第一次见到，之前因为没有这方面的需求，所以不太了解，这次看到了，就记一下吧。这个方法一眼就能看出来干嘛用的，所以就不多解释了。

2. registerOnSharedPreferenceChangeListener(listener)
这个方法才是我要说的重点，因为之前有些需求就是更改了`SharedPreferences`之后，要通知相应的组件做出改变，我以前的处理方式是通过事件订阅实现的，发一个event出去，然后目标收到event再做出反应，当时觉得特别蛋疼，两边都要做些操作，显的特别啰嗦，当时就在想可不可以在`SharedPreferences`上设置一个观察者，一旦有什么风吹草动，就自动通知目标，不曾想，人家早已经实现了，只是我愚昧无知，今天去看了下源码发现了这个方法，相见恨晚。

```
        sp1.registerOnSharedPreferenceChangeListener(new SharedPreferences.OnSharedPreferenceChangeListener() {

            @Override
            public void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key) {
                   // do any thing you want 
            }
        });
```

就是这样，在你想监听的`SharedPreferences`的引用上设置你的`Listener`,这样，当它有更改的时候，会自动通知所有的`Listener`,然后在Listener中做你想做的事吧。

### 题外话

其实就是扯闲篇，和`SharedPreferences`相关的还有一个类`Preference`，虽然他们没有什么直接关系，前者是用来存储键值对，而后者是用来帮助开发者构建界面的，主要是设置界面，唯一的相关性可能就是`Preference`是用`SharedPreferences`来保存设置信息的，但是如果有时间的话a，你还是可以看下`Preference`，你就知道为什么有些应用的设置界面怎么做到的那么遵守 Material Design，其实是现成的...,看完你就懂了，[传送门(自备梯子)](https://developer.android.com/guide/topics/ui/settings.html)

### 参考链接
 
1. https://developer.android.com/reference/android/content/SharedPreferences.html
2. https://developer.android.com/training/basics/data-storage/shared-preferences.html
3. http://www.jianshu.com/p/4dd53e1be5ba/comments/2931815
4. http://www.jianshu.com/p/ae2c7004179d