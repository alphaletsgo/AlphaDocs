# Android内存泄漏与优化

## 什么是内存泄露？

当一个对象已经不需要再使用了，本该被回收时，而有另外一个正在使用的对象持有它的引用从而就导致对象不能被回收。这种导致了本该被回收的对象不能被回收而停留在堆内存中，就产生了内存泄漏。简单讲就是内存不在GC掌控之内了。

了解java的GC内存回收机制：某对象不再有任何的引用的时候才会进行回收。

## 内存分配的几种策略：

- 1.静态的 静态的存储区：内存在程序编译的时候就已经分配好，这块的内存在程序整个运行期间都一直存在。 它主要存放静态数据、全局的static数据和一些常量。
- 2.栈式的 在执行函数(方法)时，函数一些内部变量的存储都可以放在栈上面创建，函数执行结束的时候这些存储单元就会自动被释放掉。 栈内存包括分配的运算速度很快，因为内置在处理器的里面的。当然容量有限。
- 3.堆式的 也叫做动态内存分配。有时候可以用malloc或者new来申请分配一个内存。在C/C++可能需要自己负责释放（java里面直接依赖GC机制）。 在C/C++这里是可以自己掌控内存的，需要有很高的素养来解决内存的问题。java在这一块貌似程序员没有很好的方法自己去解决垃圾内存，需要的是编程的时候就要注意自己良好的编程习惯。

**区别：**

- 堆是不连续的内存区域，堆空间比较灵活也特别大。
- 栈式一块连续的内存区域，大小是有操作系统觉决定的。
- 堆管理很麻烦，频繁地new/remove会造成大量的内存碎片，这样就会慢慢导致效率低下。对于栈的话，他先进后出，进出完全不会产生碎片，运行效率高且稳定。

## 内存分析

```java
public class Main{
	int a = 1;
	Student s = new Student();
	public void XXX(){
		int b = 1;//栈里面
		Student s2 = new Student();
	}
}
```

- 1.成员变量全部存储在堆中(包括基本数据类型，引用与引用的对象实体)---因为他们属于类，类对象最终还是要被new出来的。
- 2.局部变量的基本数据类型和引用存储于栈当中，引用的对象实体存储在堆中。-----因为他们属于方法当中的变量，生命周期会随着方法一起结束。

**我们所讨论内存泄露，主要讨论堆内存，他存放的就是引用指向的对象实体。**

有时候确实会有一种情况：当需要的时候可以访问，当不需要的时候可以被回收也可以被暂时保存以备重复使用。比如：ListView或者GridView、REcyclerView加载大量数据或者图片的时候，图片非常占用内存，一定要管理好内存，不然很容易内存溢出。滑出去的图片就回收，节省内存。看ListView的源码----回收对象，还会重用ConvertView。如果用户反复滑动或者下面还有同样的图片，就会造成多次重复IO（很耗时），那么需要缓存---平衡好内存大小和IO，算法和一些特殊的java类。

**解决方案：**

- 算法：lrucache(最近最少使用先回收)
- 特殊的java类：利于回收，StrongReference，SoftReference，WeakReference，PhatomReference

![image-20200415113035814](.assets\image-20200415113035814.png)

开发时，为了防止内存溢出，处理一些比较占用内存大并且生命周期长的对象的时候，可以尽量使用软引用和弱引用。软引用比LRU算法更加任性，回收量是比较大的，你无法控制回收哪些对象。

比如使用场景：默认头像、默认图标。 

ListView或者GridView、REcyclerView要使用内存缓存+外部缓存(SD卡)

## 如何判断一个app是否存中内存泄漏？

可以利用AndroidStudio》AndroidMonitor》System Information》Memory Usage 查看Objects里面的views和activity的数量是否为0（如果不为零则存中内存泄漏）。

## 常见内存泄漏案例

### 1、静态变量引起的内存泄漏

如下单例工具类是非常常见的一种写法

```java
public class CommUtil {
	private static CommUtil instance;
	private Context context;
	private CommUtil(Context context){
		this.context = context;
	}

	public static CommUtil getInstance(Context mcontext){
		if(instance == null){
			instance = new CommUtil(mcontext);
		}
//        else{
//            instance.setContext(mcontext);
//        }
	return instance;
	}
}
```

在Activity中使用

```java
CommUtil util = CommUtil.getInstance(this);
```

使用该工具类时应避免直接传入activity，原则：能用application.getContext()绝不使用activity。

### 2、非静态内部类引起的内存泄漏

错误用法：

```java
public void loadData(){//隐士持有MainActivity实例。MainActivity.this.a
	new Thread(new Runnable() {
		@Override
		public void run() {
			while(true){
			    try {
				//int b=a;
				Thread.sleep(1000);
			    } catch (InterruptedException e) {
				e.printStackTrace();
			    }
			}
		}
	}).start();
}
```

解决方案：将非静态内部类修改为静态内部类（静态内部类不会隐士持有外部类）。

### 3、不需要用的监听未移除会发生内存泄露

例子1：

```java
//tv.setOnClickListener();//监听执行完回收对象
//add监听，放到集合里面
tv.getViewTreeObserver().addOnWindowFocusChangeListener(new ViewTreeObserver.OnWindowFocusChangeListener() {
    @Override
    public void onWindowFocusChanged(boolean b) {
        //监听view的加载，view加载出来的时候，计算他的宽高等。
        //计算完后，一定要移除这个监听
        tv.getViewTreeObserver().removeOnWindowFocusChangeListener(this);
    }
});
```

例子2：

```java
SensorManager sensorManager = getSystemService(SENSOR_SERVICE);
Sensor sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ALL);
sensorManager.registerListener(this,sensor,SensorManager.SENSOR_DELAY_FASTEST);
//不需要用的时候记得移除监听
sensorManager.unregisterListener(listener);
```

### 4、资源未关闭引起的内存泄露情况

比如：BroadCastReceiver、Cursor、Bitmap、IO流、自定义属性attribute `attr.recycle()`回收。 当不需要使用的时候，要记得及时释放资源。否则就会内存泄露。

### 5、无限循环动画

没有在onDestroy中停止动画，否则Activity就会变成泄露对象。 比如：轮播图效果。

## 工具篇

分析内存泄漏工具

- Android monitors —— 新的Androidstudio 已经使用更好的profiler工具了

- MAT

- leakcanary —— [github](https://github.com/square/leakcanary)

  ```
    //该工具使用了android.os.Debug 中的`void dumpHprofData(String fileName) throws IOException;`api
    Application
    	install()
    LeakCanary
    	androidWatcher()
    RefWatcher
    	new AndroidWatcherExecutor() //--->dumpHeap()/analyze()(--->runAnalysis())--->Hprof文件分析
    	new AndroidHeapDumper()
    	new ServiceHeapDumpListener
  ```

- Allaction Tracking — 追踪内存分配 —— profiler工具

- Lint

## 扩展

[缓存淘汰算法—LRU算法](https://zhuanlan.zhihu.com/p/34989978)