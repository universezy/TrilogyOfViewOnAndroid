# Android中View的事件分发

## 一、 摘要

介绍Android中View的事件分发流程，以及对事件的消费和拦截。

---
## 二、 三个核心方法

### 1. 概述
View的事件分发流程涉及到三个核心方法：dispatchTouchEvent()、onTouchEvent()和onInterceptTouchEvent()，这三个方法穿插于Activity、Window、DecorView、ViewGroup、View之间。

通过前两章（《[Android中View的绘制流程](https://github.com/universezy/TrilogyOfViewOnAndroid/blob/master/RenderProcess.md)》、《[Android中View的异步消息](https://github.com/universezy/TrilogyOfViewOnAndroid/blob/master/AsyncMessage.md)》）的学习，我们知道了Window、ViewRootImpl、DecorView、ViewGroup、View之间的关系，在源码中，Window只有唯一实现类——PhoneWindow：
```java
/**
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
public abstract class Window {...}
```

根据这些信息，我们能够更快地理解事件分发流程。

接下来我们先介绍这三个方法。
- dispatchTouchEvent()存在于Activity、ViewGroup、View中；
- onTouchEvent()也存在于Activity、ViewGroup、View中；
- onInterceptTouchEvent()仅存在于ViewGroup中。

即：
|组件 \ 方法|dispatchTouchEvent|onTouchEvent|onInterceptTouchEvent|
| :- | :-: | :-: | :-: |
|Activity|YES|YES| |
|View|YES|YES| |
|ViewGroup|YES|YES|YES|

从字面意思上看，dispatchTouchEvent和分发触摸事件有关，onTouchEvent和执行触摸事件有关，onInterceptTouchEvent和拦截触摸事件有关。

---
### 2. Activity中

#### 2.1. dispatchTouchEvent()
```java
/**
 * Called to process touch screen events.  You can override this to
 * intercept all touch screen events before they are dispatched to the
 * window.  Be sure to call this implementation for touch screen events
 * that should be handled normally.
 *
 * @param ev The touch screen event.
 *
 * @return boolean Return true if this event was consumed.
 */
public boolean dispatchTouchEvent(MotionEvent ev) {...}
```

- 用于处理触屏事件。
- 你可以重写这个方法来拦截所有的触屏事件，不让他们分发到Window。
- 通常情况下你应该调用这里的具体实现去处理触屏事件。
- 如果事件被消费了，返回true。

---
#### 2.2. onTouchEvent()
```java
/**
 * Called when a touch screen event was not handled by any of the views
 * under it.  This is most useful to process touch events that happen
 * outside of your window bounds, where there is no view to receive it.
 *
 * @param event The touch screen event being processed.
 *
 * @return Return true if you have consumed the event, false if you haven't.
 * The default implementation always returns false.
 */
public boolean onTouchEvent(MotionEvent event) {...}
```

- 当一个触屏事件没有被这个Activity下的任何View处理时，调用该方法。
- 该方法最大的用处是去处理Window边界以外，没有View去响应的触屏事件。
- 如果事件被消费了，返回true。默认返回false。

---
### 3. View中

#### 3.1. dispatchTouchEvent()
```java
/**
 * Pass the touch screen motion event down to the target view, or this
 * view if it is the target.
 *
 * @param event The motion event to be dispatched.
 * @return True if the event was handled by the view, false otherwise.
 */
public boolean dispatchTouchEvent(MotionEvent event) {...}
```

- 将触屏事件向下传递到目标视图，如果当前视图就是目标，则传递到当前视图。
- 如果事件被处理了，返回true。

---
#### 3.2. onTouchEvent()
```java
/**
 * Implement this method to handle touch screen motion events.
 * <p>
 * If this method is used to detect click actions, it is recommended that
 * the actions be performed by implementing and calling
 * {@link #performClick()}. This will ensure consistent system behavior,
 * including:
 * <ul>
 * <li>obeying click sound preferences
 * <li>dispatching OnClickListener calls
 * <li>handling {@link AccessibilityNodeInfo#ACTION_CLICK ACTION_CLICK} when
 * accessibility features are enabled
 * </ul>
 *
 * @param event The motion event.
 * @return True if the event was handled, false otherwise.
 */
public boolean onTouchEvent(MotionEvent event) {...}
```

- 该方法用于处理触屏事件。
- 如果想用这个方法来监听点击动作，建议实现并调用performClick()。这将确保系统行为一致性，包括：遵守点击声音首选项、分发OnClickListener、处理ACTION_CLICK。
- 如果事件被处理了，返回true。

---
### 4. ViewGroup中

#### 4.1. dispatchTouchEvent()

重写了View中的实现。

---
#### 4.2. onTouchEvent()

继承View中的方法。

---
#### 4.3. onInterceptTouchEvent()
```java
/**
 * Implement this method to intercept all touch screen motion events.  This
 * allows you to watch events as they are dispatched to your children, and
 * take ownership of the current gesture at any point.
 *
 * <p>Using this function takes some care, as it has a fairly complicated
 * interaction with {@link View#onTouchEvent(MotionEvent)
 * View.onTouchEvent(MotionEvent)}, and using it requires implementing
 * that method as well as this one in the correct way.  Events will be
 * received in the following order:
 *
 * <ol>
 * <li> You will receive the down event here.
 * <li> The down event will be handled either by a child of this view
 * group, or given to your own onTouchEvent() method to handle; this means
 * you should implement onTouchEvent() to return true, so you will
 * continue to see the rest of the gesture (instead of looking for
 * a parent view to handle it).  Also, by returning true from
 * onTouchEvent(), you will not receive any following
 * events in onInterceptTouchEvent() and all touch processing must
 * happen in onTouchEvent() like normal.
 * <li> For as long as you return false from this function, each following
 * event (up to and including the final up) will be delivered first here
 * and then to the target's onTouchEvent().
 * <li> If you return true from here, you will not receive any
 * following events: the target view will receive the same event but
 * with the action {@link MotionEvent#ACTION_CANCEL}, and all further
 * events will be delivered to your onTouchEvent() method and no longer
 * appear here.
 * </ol>
 *
 * @param ev The motion event being dispatched down the hierarchy.
 * @return Return true to steal motion events from the children and have
 * them dispatched to this ViewGroup through onTouchEvent().
 * The current target will receive an ACTION_CANCEL event, and no further
 * messages will be delivered here.
 */
public boolean onInterceptTouchEvent(MotionEvent ev) {...}
```

- 用于拦截所有的触屏事件。
- 这使你可以在事件被分发到子视图时进行监听，并且在任何时候把控当前手势。
- 使用时需要当心，因为它和onTouchEvent()的交互相当复杂，你得以正确的方式来实现这个方法。
- 按这个顺序接收事件：（1）接收down事件。（2）既可以用这个ViewGroup的子视图，也可以用onTouchEvent()这个方法来处理down事件，这意味着你应该实现onTouchEvent()这个方法并返回true，这样你将看到手势的剩余部分（不要用父视图来处理它）。同样，通过在onTouchEvent()中返回true，你就不会在onInterceptTouchEvent()中收到任何后续事件，并且所有的触摸处理只能在onTouchEvent()中正常进行。（3）只要你从这返回false，接下来的每个事件(直到并包括最终的up事件)都将首先传递到这里，然后传递给目标的onTouchEvent()。（4）如果你从这返回true，你将接收不到以下事件：目标视图会接收相同的事件但是带着ACTION_CANCEL动作，所有其他事件将被传递到onTouchEvent()方法，不再出现在这里。
- 返回true来拦截来自子视图的事件，通过onTouchEvent()来分发它们。当前目标将接收一个ACTION_CANCEL事件，并且此处不会再传递更多的消息。

---
## 三、 事件分发流程

看完了三个核心方法的注释，被一大堆布尔返回值搞懵了，

---
## 四、 消费和拦截

### 1. 什么是消费


---
### 2. 什么是拦截


---
### 3. 基于消费和拦截的分发流程


---
## 五、 流程图


---
## 六、 参考文献

- [Android事件分发机制详解：史上最全面、最易懂](https://www.jianshu.com/p/38015afcdb58)