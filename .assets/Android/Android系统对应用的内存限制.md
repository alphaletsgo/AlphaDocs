# Android系统对应用的内存限制

Android设备出厂以后，Java虚拟机对单个应用的内存分配就固定下来了，超出这个值就会OOM。这个属性值定义在 /system/build.prop中，build.prop中的部分内容如下：

```kotlin
dalvik.vm.heapgrowthlimit=192m // heapgrowthlimit参数表示单个应用最大可用内存
dalvik.vm.heapstartsize=16m // heapstartsize参数表示堆内存分配的初始大小
dalvik.vm.heapsize=512m // heapsize参数表示单个进程可用的最大内存
```

## heapgrowthlimit

这表示单个应用最大可用内存是192m，超出就会报OOM。这个内存溢出是针对dalvik堆而言，而不是native堆。

通过代码查看每个进程可用的最大内存，即heapgrowthlimit值：

```java
ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
int heapGrowthLimit = am.getMemoryClass(); // 192，以m为单位
```

## heapstartsize

堆内存分配的初始大小会影响整个系统对RAM的使用程度，和第一次使用应用的流畅速度。

它值越小，系统ram消耗越慢，但一些较大的应用一开始不够用，需要调用gc和堆调整策略，导致应用反应较慢。

它值越大，系统ram消耗越快，但是应用更流畅。

## heapsize

heapsize表示不受控情况下的极限堆，表示单个进程可用的最大内存。但如果存在heapgrowthsize参数，则以heapgrowthsize定义为最大内存。

在应用开发中，如果要使用大堆，可在manifest文件中指定android:largeHeap为true，这样dalvik的堆内存可以达到heapsize。

## 获取内存使用信息

通过ActivityManager获取相关信息，下面是一个例子代码：

```java
private void displayBriefMemory()  
　　{     
　　    final ActivityManager activityManager = (ActivityManager) getSystemService(ACTIVITY_SERVICE);     
　　    ActivityManager.MemoryInfo info = new ActivityManager.MemoryInfo();    
　　    activityManager.getMemoryInfo(info);     
　　    Log.i(tag,"系统剩余内存:"+(info.availMem >> 10)+"k");    
　　    Log.i(tag,"系统是否处于低内存运行："+info.lowMemory); 
　　    Log.i(tag,"当系统剩余内存低于"+info.threshold+"时就看成低内存运行"); 
　　}
```

一般地，厂家针对设备的配置情况都会适当的修改/system/build.prop文件来调高这个值。随着设备硬件性能的不断提升，从最早的16M限制（G1手机）到后来的24m,32m，64m等，都遵循Android框架对每个应用的最小内存大小限制。

android上的应用是带有独立虚拟机的，也就是每开一个应用就会打开一个独立的虚拟机。这样设计的优点就是在单个程序崩溃的情况下不会导致整个系统的崩溃。

## 参考

- [关于build.prop原始Dalvik虚拟机设定与调整](https://blog.csdn.net/liwei405499/article/details/43408989)