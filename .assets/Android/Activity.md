# Activity生命周期

# **Activity**正常的生命周期

正常的生命周期是指用户操作，如启动、跳转、关闭等操作时Activity的生命周期。

![00004](.assets\00004.jpg)

**备注：**

- onStart时期Activity已经可见了，但是没有出现在前台我们还看不到，也无法响应用户操作
- onResume表示Activity已经可见了并在前台，能响应用户操作了
- 从生命周期来看，onCreate和onDestroy是配对的，标志着activity的创建和销毁，并且生命周期中只会出现一次；从可见性来看onStart和onStop是配对的，标志着activity是否处于可见状态，根据操作可能出现多次；从Activity是否处于前台来看，onResume和onPause是配对，根据用户操作可能被多次调用。

**问题：**假设当前Activity为A，如果这时用户打开一个新的Activity B，那么B的onResume和A的onPause哪个先执行呢？

结论：A先执行onPause方法，然后B执行onCreate→onStart→onResume，等B显示出来最后才是A的onStop方法调用。**在onPause和onStop方法中我们不应该做耗时操作**。

# 异常的生命周期

异常的生命周期是指Activity配置发生改变、系统资源不足导致Activity被回收等导致的生命周期。

## 情况1：资源配置相关的系统配置发生改变导致Activity被杀并重新创建

比如：一般情况下activity旋转、系统字体改变等资源配置改变之后（android:screenOrientation值为landscape或portrait），Activity会被杀掉并重建，生命流程如下：

![Untitled](.assets\Untitled.png)

onSaveInstanceState方法在onStop方法之前被执行，它与onPause没有既定的时许关系；onRestoreInstanceState方法在onStart方法之后被执行。

疑惑：view时如何onRestoreInstanceState？

## 资源内存不足导致低优先级的Activity被杀掉

Activity按优先级可以分为三种级别:

- 前台Activity——正在和用户产生交互的activity，优先级最高
- 可见但非前台Activity——比如activity上覆盖了一个dialog，导致此Activity可见但无法与用户产生交互
- 后台Activity——已经被暂停的Activity，比如执行了onStop，优先级最低

**备注：如果一个进程中没有四大组件在执行，那么这个进程将很快被系统杀死。**

**当配置发生变化时，我们如何才能不让Activity重启呢？**

我们可以在Mainfest配置文件的Activity标签里修改如下属性。

```
android:configChanges="orientation|screenSize"
```

下图展示了该属性各个参数所表示的含义：

![Inked110335_NdId_217294_LI](.assets\Inked110335_NdId_217294_LI.jpg)

当我们在Manifest文件中配置了上面参数后，我们旋转手机activity将不会在重新创建，但onConfigurationChanged方法将被调用。

# Activity启动模式

## 四种启动模式

- standard：**标准模式**，一个任务栈中可以有**多个实例**，**在这种模式下，谁启动这个Activity，那么这个Activity就运行在启动它的哪个Activity所在的栈中**。

  如果我们使用ApplicationContext去启动standard模式的Activity的时候会报错，错误如下：

  ```java
  android.util.AndroidRuntimeException: Calling startActivity() from outside of an Activity  context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?
  ```

  这是由于非Activity类型的Context并没有所谓的任务栈，解决办法就是指定标记位：FLAG_ACTIVITY_NEW_TASK，这个时候启动的Activity实际上是以singleTask模式启动。

- singleTop：**栈顶复用**，如果新Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时它的onNewIntent方法会被调用，通过此方法我们可以取出当前请求的信息，此时activity的onCreate、onStart不会被系统调用，因为它并没有改变。如果新Activity存在但不位于栈顶那么将新建Activity，此时同standard一样。

- singleTask：**栈内复用**，这是一种**单例模式**，在这种情况下，只要activity在一个栈中存在，那么多次启动此Activity都不会创建新的实例，同singleTop一样，系统会调用onNewIntent方法。具体一点的就是当启动一个该模式的activity时，先判断是否存中该activitiy想要的任务栈，如果不存在那么先创建想要的任务栈，再创建该activity；如果存在想要的任务栈，这时根据任务栈是否存中实例又分为两种情况，不存在则创建实例，存在的话则不创建直接将activity置于栈顶，并调用activity的onNewIntent方法。

  ![Untitleds](.assets\Untitleds.png)

- singleInstance：单实例模式，这是一种加强的singleTask模式，此模式的Activity只能单独地位于一个任务栈中，也就是说启动该模式的activity，系统会为它创建一个新的任务栈，然后它独自在这个新的任务栈中，由于栈内复用，后续的请求均不会创建新的activity，除非该任务栈被系统销毁了。

## 如何指定Activity的启动模式

- 通过AndroidManifest>Activity指定

  ```xml
  <activity android:name="cn.isif.viewandroid.LaunchModeActivity" android:configChanges="screenLayout" android:launchMode="singleTask" android:label="@string/app_name"/>```
  ```

- 通过在Intent中设置标志位

  ```java
  Intent intent = new Intent(); intent.setClass(MainActivity.this, LaunchModeActivity.class); intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK); startActivity(intent);
  ```

区别：

- 1 优先级上，第二钟要高于第一种
- 2 限定范围有所不同，第一种无法直接给Activity设定FLAG_ACTIVITY_CLEAR_TOP标识；第二种无法为Activity指定singleInstance模式。

## TaskAffinity

TaskAffinity可以翻译为任务相关性，这个参数标识了一个Activity所需要的任务栈的名字，默认情况下Activity的任务栈的名字为包名。TaskAffinity属性主要和singleTask启动模式或者allowTaskReparenting属性配对使用，在其他情况下没有意义。

**备注：任务栈分为前台任务栈和后台任务栈，后台任务栈中的activity处于暂停状态，用户可以通过切换将后台任务栈再次调到前台。**

- taskAffinity配合singleTask使用：启动的activity会在名字和TaskAffinity相同人的任务栈中。

- taskAffinity配合allowTaskReparenting使用：这种情况比较复杂，下面用图来分析：

  ![Untitled](.assets\Untitled-1586920719744.png)

查看Activity栈：`adb shell dumpsys activity`

## Activity 常用的Flags

- FLAG_ACTIVITY_NEW_TASK：为activity指定”singleTask“启动模式
- FLAG_ACTIVITY_SINGLE_TOP：为activity指定”singleTop“启动模式
- FLAG_ACTIVITY_CLEAR_TOP：具有此标记为的activity，启动时，在同一个任务栈中所有位于它上面的activity都有出栈，这模式一般需要配合FLAG_ACTIVITY_NEW_TASK使用：被启动activity实例如果已经存在，那么系统就会调用它的onNewIntent。如果被启动的activity实例采用standard模式，那么它连同它之上的activity都有出栈，系统会创建新的activity实例并放入栈顶。
- FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS：具有这个标记的activity不会出现在历史activity的列表中，它等同于在xml中指定activity的属性`android:excludeFromRecent="true"`。当某些情况下我们不希望用户通过历史列表回到我们的activity的时候我们可以使用这个标记

# IntentFilter的匹配规则

启动Activity有两种类型即：显示和隐式。这里主要介绍隐式启动Intent。

```xml
<activty android:name="ShareActivity">
	<intent-filter>
		<action android:name="cn.isif.action_a"/>
		<action android:name="cn.isif.action_b"/>
		<category android:name="cn.isif.category_a"/>
		<category android:name="cn.isif.category_b"/>
		<category android:name="android.intent.category.DEFAULT"/>
		<data android:mimeType="text/plain"/>
	</intent-filter>
</activity>
```

以上是一个Activity的intent-filter，它遵循以下：

- 一个Activity可以有多个intent-filter
- 一个intent-filter可以有多个action、category、data
- 多个intent-filter，只要匹配其中任意一个就可以
- 一个Intent只有同时匹配了action、category、data才算完全匹配

## 匹配规则

### action

是一个字符串，Intent中可以有多个action，只要其中有一个action与intent-filter匹配即可。

### category

是一个字符串，为了能接受隐式调用，category必须包含“android.intent.category.DEFAULT”。Intent中可以没有category，这时匹配的时default，但是如果有的话要求每一个都必须与intent-filter中定义的匹配。

### data

date的语法如下：

```xml
<data android:scheme=""
		android:host=""
		android:port=""
		android:path=""
		android:pathPattern=""
		android:pathPrefix=""
		android:mimeType=""/>
```

data由两部分组成，mimeType和URI。mimeType指媒体类型：image/jpeg、audio/mpeg4-generic、video/*等。URI结构如：`://:/[||]`，例如：`content://com.example.project:200/folder/subfolder/etc`。

**匹配规则：**与action类似，也要求Intent必须含有data数据，并且data数据能够完全匹配过滤规则中的某一个。

data注意事项：

- mimeType可以不用指定，它由默认值：content或者file。
- 如果要为Intent指定完整的data，必须要调用setDataAndTaype，不能用setData再setType，因为这个两个方法会清楚对方的值。

例子：匹配ShareActivity

```java
Intent intent = new Intent("cn.isif.action_a");
intent.addCategory("cn.isif.category_a");
intent.setDataAndType(Uri.Parse("file://abd"),"text/plain");
startActivity(intent);
```

## 建议

使用隐式Intent的时候我们可以使用PackegeManager的resolveActivity方法或者Intent的resolveActivity方法进行判断是否由匹配的Activity。另外PackageManager还提供了queryIntentActivities可以用来查询匹配intent的activity列表。

```java
public abstract List<ResolveInfo> queryIntentActivities(Intent intent, int flags);
public abstract ResolveInfo resolveActivity(Intent intent,int flags);
```

第二个参数我们需要使用MATCH_DEFAULT_ONLY，含义：仅仅匹配那些再intent-filter中声明了"android.intent.category.DEFAULT"category的activity。[不含DEFAULT的activity无法隐式启动]