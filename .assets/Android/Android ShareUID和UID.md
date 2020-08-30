# Android ShareUID和UID

# uid pid gid gids 的含义和作用

- uid: android中**uid用于标识一个应用程序**，**uid在应用安装时被分配**，并且在应用存在于手机上期间，都不会改变。一个应用程序只能有一个uid，多个应用可以使用sharedUserId 方式共享同一个uid，前提是这些应用的签名要相同。
- pid : 进程ID，可变的
- gid: 对应于linux中用户组的概念，android 中 gid 等于uid
- gids: 个GIDS相当于一个权限的集合，一个UID可以关联GIDS，表明该UID拥有多种权限

一个进程就是host应用程序的沙箱，里面一般有一个UID和多个GIDS，每个进程只能访问UID的权限范围内的文件和GIDS所允许访问的接口，构成了Android最基本的安全基础。

Android的应用的UID是从10000开始，可以在`Process.java`中查看到（`FIRST_APPLICATION_UID`和`LAST_APPLICATION_UID`），由于UID是应用安装时确认的。我们可以从源码看到UID的产生（`Settings.java`）。

```java
// uid 的分配
// Returns -1 if we could not find an available UserId to assign
private int newUserIdLPw(Object obj) {
    // Let's be stupidly inefficient for now...
    final int N = mUserIds.size();
    //从0开始，找到第一个未使用的ID，此处对应之前有应用被移除的情况，复用之前的ID
    for (int i = mFirstAvailableUid; i < N; i++) {
        if (mUserIds.get(i) == null) {
            mUserIds.set(i, obj);
            return Process.FIRST_APPLICATION_UID + i;
        }
    }

    //最多只能安装 9999 个应用
    // None left?
    if (N > (Process.LAST_APPLICATION_UID-Process.FIRST_APPLICATION_UID)) {
        return -1;
    }

    mUserIds.add(obj);
    // 可以解释为什么普通应用的UID 都是从 10000开始的
    return Process.FIRST_APPLICATION_UID + N;
}
```

# 查看应用UID的方式

## 方法1：

终端输入adb shell，然后再输入ps。也可以使用grep进行指定筛选：`ps | grep cn.isif.reviewandroid`。

![Untitled](.assets\Untitled-1586921219673.png)

u0_axxx代表着应用程序的用户，从上面内容我们知道UID是从10000开始的。所以UID等于u0_a后面的数字加上10000所得数。更多ps指令详情参考[这里](https://www.notion.so/uncle404/Android-ps-0cde122cf9e848dcb19537d7e2b75fbe)。

## 方法2：

通过程序获得所有安装应用的UID

```java
List<PackageInfo> packinfos = pManager.getInstalledPackages(PackageManager.GET_PERMISSIONS);
for(PackageInfo info : packinfos)
{
   Log.e(TAG,"app:"+info.applicationInfo.loadLabel(pManager).toString()+" uid:"+info.applicationInfo.uid);
}
```

## 方法3：

通过应用PID查看对应用户的UID：终端输入`adb shell`，然后输入 `cat /proc//status`。

## 方法4：

通过`packages.xml`，查看需要查询的应用的UID：终端中输入`adb shell`，然后输入`cat /data/system/packages.xml`。