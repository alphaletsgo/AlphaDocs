# View的事件分发机制

## 事件的产生

当屏幕被触摸，Linux内核会将硬件产生的触摸事件包装为Event存到`/dev/input/event[x]`目录下。对，你没看错，事件被搞成文件保存了下来！

系统服务InputManagerService（InputManager本地实现的包装）启动时会创建InputReaderThread线程和InputDispatherThread线程，其中InputReaderThread线程会轮询（loop）通过EventHub的getEvent从`/dev/input/`文件夹下读取事件，并由InputReader将其进行解释变成对应的事件InputMapper。然后通知InputDispatherThread并把相应的事件交其进行分发处理，

而**InputDispatch**又会把事件分发到需要的地方，比如**ViewRootImpl**的**WindowInputEventReceiver**中。

**InputManager** 是 InputReader 和 InputDispatcher 线程的创建者，它只有一个职责，就是被 **WindowManagerService** 使用，从 Native 层获取按键事件。

**WindowManagerService** 则负责与窗口对接，分发按键消息。

这部分是通过C++实现。

## 触摸事件的类型

对应的触摸事件类是MotionEvent，事件的类型主要有：

- ACTION_DOWN：用户手指按下操作，也标志着一次触摸事件的开始。
- ACTION_MOVE：手指在离开屏幕之前发生移动且移动的距离超过阈值就会触发ACTION_MOVE事件。
- ACTION_UP：手指离开屏幕，也标志着触摸事件结束。



