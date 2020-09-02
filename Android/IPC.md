# IPC机制

# IPC简介

IPC是Inter-Process Communication的缩写，含义为**进程间通信**或者**跨进程通信**，是指两个进程之间进行数据交换的过程。

## 进程与线程

按照操作系统中的描述，线程是**CPU调度的最小单元**，同时线程是**一种有限的系统资源**。而进程一般指**一个执行单元**，在PC和移动设备上指**一个程序**或者**一个应用**。**一个进程可以包含多个线程，因此进程和线程是包含与被包含的关系**。

在Android中最有特色的进程间通信方式就是Binder了，通过Binder可以轻松地实现进程间通信。

**注意：**主线程中执行耗时的任务时容易发生ANR（Application Not Responding）。

延展：Android对单个应用所使用的最大内存做了限制，有时候为了提高应用所使用的内存我们使用多进程来申请更多的内存，详情参考[这里](https://www.notion.so/uncle404/Android-06660468667d4d0687a3001ab1dd309b)。

# Android中的多进程模式

## 开启多进程模式

在Android中SDK提供使用多进程只有一种方法，**那就是给四大组件（Activity、Service、Receiver、ContentProvider）在AndroidMenifest中指定android:process属性**，除此之外没有其他办法，也就是说我们无法给一个线程或者一个实体类指定其运行时所在的进程。其实还有**另一种非常规的多进程方法，那就是通过JNI在native层去fork一个新的进程**，但是这种方法属于特殊情况，也不是常用的创建多进程的方式，因此我们暂时不考虑这种方式。

```xml
<activity android:name=".ipc.IPCSecondActivity"
	android:configChanges="screenLayout"
  android:process="cn.isif.reviewandroid.remote"/>
<activity android:name=".ipc.IPCFirstActivity"
  android:configChanges="screenLayout"
  android:process=":remote"/>
```

## 查看进程

我们可以通过Android Studio的LogCat查看进程信息

![Untitled](.assets\Untitled-1586920996922.png)

除了使用LogCat查看外，我们还可以通过命令：`adb shell ps` 或 `adb shell ps | grep cn.isif.reviewandroid`

":remote"与"cn.isif.reviewandroid.remote"的区别？

- **“:”的含义** 是指要再当前的进程前面附加上当前的包名，这是一种简写，它完整的名为“cn.isif.reviewandroid:remote”。而"cn.isif.reviewandroid.remote"是完整的命名方式，不会附加包名信息。

- 以”:“开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中，而不以”:“开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。

  ```xml
  android:sharedUserId="string" <!-- API 级别 29 中已弃用此常量。 -->
  ```

> 我们知道Android系统会为每个应用分配一个唯一的UID，具有相同UID的应用才能共享数据。这里要说明的是，两个应用通过ShareUID跑在同一个进程中是有要求的，需要这两个应用有相同的ShareUID并且签名相同才可以。在这种情况下，它们可以互相访问对方的私有数据，比如data目录、组件信息等，不管它们是否跑在同一个进程中。当然如果它们跑在同一个进程中，那么除了能共享data目录、组件信息，还可以共享内存数据，或者说它们看起来就像是一个应用的两个部分。

关于shareUID和UID可以参考[这里](./Android ShareUID和UID.md)。

## Android多进程运行机制

我们写一个静态成员变量，在IPCFirstActivity中我们将sUserId的值设置为2，然后我们在IPCSecondActivity中打印其值。

```kotlin
object UserManager {
    var sUserId = 1
}
```

我们会发现如下结果

```kotlin
2020-03-30 16:36:39.661 15061-15061/? D/IPCFirstActivity: Ths UserManger'sUserId is {2}
2020-03-30 16:36:43.800 15092-15092/? D/IPCSecondActivity: Ths UserManger'sUserId is {1}
```

上述问题出现的原因是IPCFirstActivity和IPCSecondActivity各自运行在一个单独的进程中，我们知道Android为每一个应用分配了一个独立的虚拟机，或者说为每个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致在不同的虚拟机中访问同一个类的对象会产生多份副本。

一般来说，使用多进程会造成如下几个方面的问题：

- （1）静态成员和单例模式完全失效。
- （2）线程同步机制完全失效。
- （3）SharedPreferences的可靠性下降。
- （4）Application会多次创建。

第1个问题在上面已经进行了分析。第2个问题本质上和第一个问题是类似的，既然都不是一块内存了，那么不管是锁对象还是锁全局类都无法保证线程同步，因为不同进程锁的不是同一个对象。第3个问题是因为SharedPreferences不支持两个进程同时去执行写操作，否则会导致一定几率的数据丢失，这是因为SharedPreferences底层是通过读/写XML文件来实现的，并发写显然是可能出问题的，甚至并发读/写都有可能出问题。第4个问题也是显而易见的，当一个组件跑在一个新的进程中的时候，由于系统要在创建新的进程同时分配独立的虚拟机，所以这个过程其实就是启动一个应用的过程。因此，相当于系统又把这个应用重新启动了一遍，既然重新启动了，那么自然会创建新的Application。这个问题其实可以这么理解，运行在同一个进程中的组件是属于同一个虚拟机和同一个Application的，同理，**运行在不同进程中的组件是属于两个不同的虚拟机和Application的**。

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        val pid = android.os.Process.myPid()
        val processName:String = getProcessId(this,pid).toString()
        Log.d(TAG, "application is start: {$pid} - {$processName}")
    }

    companion object {
        private const val TAG = "MyApplication"
        fun getProcessId(context: Context, pid: Int):String?{
            val mActivityManager = context.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
            for (appProcess in mActivityManager.runningAppProcesses){
                if (pid == appProcess.pid){
                    return appProcess.processName
                }
            }
            return null
        }
    }
}
```

logcat打印信息：

```kotlin
MyApplication: application is start: {13346} - {cn.isif.reviewandroid:remote}
MyApplication: application is start: {13384} - {cn.isif.reviewandroid.remote}
```

我们也可以这么理解同一个应用间的多进程：**它就相当于两个不同的应用采用了SharedUID的模式**，这样能够更加直接地理解多进程模式的本质。

# IPC基础概念介绍

## Serializble

Serializble是Java所提供的一个序列化接口，是用来辅助序列化和反序列化过程的。

serialVersionUID用于其序列化和反序列化中判断对象是否发生改变的，防止其序列化失败。如果我们不去设置的话，Java序列化或反序列化时会根据对象的hashcode值来判断对象是否发生改变。

## Parcelable

Parcelable是Android提供的序列化接口，在其实现上稍微比Serializable复杂。

它们之间的区别，或者我们该如何选择呢？

Serializable使用起来非常简单但开销很大，这也是Android为什么搞出来一个Parcalable了。Parcalable是Android中序列化方式，因此也更适合Android，虽然麻烦但是效率很高。Parcalable主要用在内存上的序列化，而网络传输的话还是推荐使用Serializable。

# AIDL

## 定向Tag语法

### 定向Tag中in、out和inout的区别？

其主修饰函数的参数，在aidl中使用自定义类时，必须指定该tag，基本数据类型默认是in。我们可以简单的理解为这个语法是代表数据的流向（不是函数的返回值，而是函数的参数），即：

**in**：表示数据只能流向服务端，服务端对数据修改了不会将修改的结果回写给客户端；

**out**：表示数据可以从服务端流向客户端，但服务端不接受客户端数据。比如我们向服务端传递一个对象，服务端会在`onTransact()`方法中新建这个对象并初始化，但不会将这个对象的值写入，在方法调用完成后，会将这个对象回写到客户端。

**inout**：是综合了in和out方案。



