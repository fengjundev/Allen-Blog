---
title: Android事件分发机制源码解析
date: 2017-02-22 19:02:01
tags:
- android
- 事件传递
categories: 
- android
---

![](https://ww2.sinaimg.cn/large/006tNbRwgy1fczcvg0sbnj30dw099di1.jpg)

触摸事件传递机制是Android中一块比较重要的知识体系，了解并熟悉整套的传递机制有助于更好的分析各种滑动冲突、滑动失效问题，更好去扩展控件的事件功能和开发自定义控件。

<!-- more -->

> 出处： *[Allen's Zone](http://allenfeng.com/)*
> 作者： *Allen Feng*

## 预备知识
### MotionEvent
在Android设备中，触摸事件主要包括点按、长按、拖拽、滑动等，点按又包括单击和双击，另外还包括单指操作和多指操作等。一个最简单的用户触摸事件一般经过以下几个流程：
- 手指按下
- 手指滑动
- 手指抬起

Android把这些事件的每一步抽象为`MotionEvent`这一概念，MotionEvent包含了触摸的坐标位置，点按的数量(手指的数量)，时间点等信息，用于描述用户当前的具体动作，常见的MotionEvent有下面几种类型：
- `ACTION_DOWN`
- `ACTION_UP`
- `ACTION_MOVE`
- `ACTION_CANCEL`

其中，`ACTION_DOWN`、`ACTION_MOVE`、`ACTION_UP`就分别对应于上面的手指按下、手指滑动、手指抬起操作，即一个最简单的用户操作包含了一个`ACTION_DOWN`事件，若干个`ACTION_MOVE`事件和一个`ACTION_UP`事件。

### 几个方法
事件分发过程中，涉及的主要方法有以下几个：

- `dispatchTouchEvent`: 用于事件的分发，所有的事件都要通过此方法进行分发，决定是自己对事件进行消费还是交由子View处理
- `onTouchEvent`: 主要用于事件的处理，返回true表示消费当前事件
- `onInterceptTouchEvent`: 是`ViewGroup`中独有的方法，若返回`true`表示拦截当前事件，交由自己的`onTouchEvent()`进行处理，返回`false`表示不拦截

我们的源码分析也主要围绕这几个方法展开。

## 源码分析

### Activity

我们从Activity的`dispatchTouchEvent`方法作为入口进行分析：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

这个方法首先会判断当前触摸事件的类型，如果是`ACTION_DOWN`事件，会触发`onUserInteraction`方法。根据文档注释，当有任意一个按键、触屏或者轨迹球事件发生时，栈顶Activity的`onUserInteraction`会被触发。如果我们需要知道用户是不是正在和设备交互，可以在子类中重写这个方法，去获取通知（比如取消屏保这个场景）。

然后是调用Activity内部`mWindow`的`superDispatchTouchEvent`方法，`mWindow`其实是`PhoneWindow的`实例，我们看看这个方法做了什么：

```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {

    ...

	@Override
	public boolean superDispatchTouchEvent(MotionEvent event) {
	    return mDecor.superDispatchTouchEvent(event);
	}


	private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
	    
	    ...

	    public boolean superDispatchTouchEvent(MotionEvent event) {
	        return super.dispatchTouchEvent(event);
	    }

	    ...
	}

}
```

原来PhoneWindow内部调用了DecorView的同名方法，而DecorView其实是FrameLayout的子类，FrameLayout并没有重写dispatchTouchEvent方法，所以事件开始交由ViewGroup的dispatchTouchEvent开始分发了，这个方法将在下一节分析。

我们回到Activity的`dispatchTouchEvent`方法，注意当`getWindow().superDispatchTouchEvent(ev)`这一语句返回false时，即事件没有被任何子View消费时，最终会执行Activity的`onTouchEvent`：

```java
public boolean onTouchEvent(MotionEvent event) {
    if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }
    
    return false;
}
```

> 小结：
事件从Activity的dispatchTouchEvent开始，经由DecorView开始向下传递，交由子View处理，若事件未被任何Activity的子View处理，将由Activity自己处理。

### ViewGroup

由上节分析可知，事件来到DecorView后，经过层层调用，来到了ViewGroup的dispatchTouchEvent方法中:

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {

    ... 

    boolean handled = false;

    if (onFilterTouchEventForSecurity(ev)) {

        final int action = ev.getAction();
        ...

        // 先检验事件是否需要被ViewGroup拦截
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {

            // 校验是否给mGroupFlags设置了FLAG_DISALLOW_INTERCEPT标志位
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {

            	// 走onInterceptTouchEvent判断是否拦截事件
                intercepted = onInterceptTouchEvent(ev);
            } else {
                intercepted = false;
            }
        } else {
            intercepted = true;
        }

        ...

        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        if (!canceled && !intercepted) {

        	// 注意ACTION_DOWN等事件才会走遍历所有子View的流程
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

                ...

                // 开始遍历所有子View开始逐个分发事件
                final int childrenCount = mChildrenCount;
                if (childrenCount != 0) {

                    for (int i = childrenCount - 1; i >= 0; i--) {

                    	// 判断触摸点是否在这个View的内部
                        final View child = children[i];
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            continue;
                        }

                        ...

                        // 事件被子View消费，退出循环，不再继续分发给其他子View
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            
                            ...
                            // addTouchTarget内部将mFirstTouchTarget设置为child，即不为null
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                    }
                }
            }
        }

        // 事件未被任何子View消费，自己处理
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 将MotionEvent.ACTION_DOWN后续事件分发给mFirstTouchTarget指向的View
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;

            while (target != null) {

                final TouchTarget next = target.next;

                // 如果已经在上面的遍历过程中传递过事件，跳过本次传递
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                    || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    ...
                }
                predecessor = target;
                target = next;
            }
        }

        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }

    return handled;
}

private void resetTouchState() {
    clearTouchTargets();
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
}

private void clearTouchTargets() {
    TouchTarget target = mFirstTouchTarget;
    if (target != null) {
        do {
            TouchTarget next = target.next;
            target.recycle();
            target = next;
        } while (target != null);
        mFirstTouchTarget = null;
    }
}

private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}

private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        ...
        
        // 注意传参child为null时，调用的是自己的dispatchTouchEvent
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(transformedEvent);
        }

        return handled;
}

public boolean onInterceptTouchEvent(MotionEvent ev) {
    // 默认不拦截事件
    return false;
}
```

这个方法比较长，只要把握住主要脉络，修枝剪叶后还是非常清晰的：

**(1) 判断事件是够需要被ViewGroup拦截**

首先会根据`mGroupFlags`判断是否可以执行`onInterceptTouchEvent`方法，它的值可以通过`requestDisallowInterceptTouchEvent`方法设置：

```java
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

    if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
        // We're already in this state, assume our ancestors are too
        return;
    }

    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }

    // Pass it up to our parent
    if (mParent != null) {
        // 层层向上传递，告知所有父View不拦截事件
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}
```

所以我们在处理某些滑动冲突场景时，可以从子View中调用父View的`requestDisallowInterceptTouchEvent`方法，阻止父View拦截事件。

如果view没有设置`FLAG_DISALLOW_INTERCEPT`，就可以进入onInterceptTouchEvent方法，判断是否应该被自己拦截，
ViewGroup的onInterceptTouchEvent直接返回了false，即默认是不拦截事件的，ViewGroup的子类可以重写这个方法，内部判断拦截逻辑。

**注意：**只有当事件类型是`ACTION_DOWN`或者mFirstTouchTarget不为空时，才会走*是否需要拦截事件*这一判断，如果事件是`ACTION_DOWN`的后续事件（如`ACTION_MOVE`、`ACTION_UP`等），且在传递`ACTION_DOWN`事件过程中没有找到目标子View时，事件将会直接被拦截，交给ViewGroup自己处理。mFirstTouchTarget的赋值会在下一节提到。

**(2) 遍历所有子View，逐个分发事件：**

执行遍历分发的条件是：当前事件是`ACTION_DOWN`、`ACTION_POINTER_DOWN`或者`ACTION_HOVER_MOVE`三种类型中的一个（后两种用的比较少，暂且忽略）。所以，如果事件是`ACTION_DOWN`的后续事件，如`ACTION_UP`事件，将不会进入遍历流程！

进入遍历流程后，拿到一个子View，首先会判断触摸点是不是在子View范围内，如果不是直接跳过该子View；
否则通过`dispatchTransformedTouchEvent`方法，间接调用`child.dispatchTouchEvent`达到传递的目的；

如果`dispatchTransformedTouchEvent`返回true，即事件被子View消费，就会把mFirstTouchTarget设置为child，即不为null，并将alreadyDispatchedToNewTouchTarget设置为true，然后跳出循环，事件不再继续传递给其他子View。

可以理解为，这一步的主要作用是，在事件的开始，即传递`ACTION_DOWN`事件过程中，找到一个需要消费事件的子View，我们可以称之为`目标子View`，执行第一次事件传递，并把mFirstTouchTarget设置为这个目标子View

**(3) 将事件交给ViewGroup自己或者目标子View处理**

经过上面一步后，如果mFirstTouchTarget仍然为空，说明没有任何一个子View消费事件，将同样会调用dispatchTransformedTouchEvent，但此时这个方法的`View child`参数为null，所以调用的其实是`super.dispatchTouchEvent(event)`，即事件交给ViewGroup自己处理。ViewGroup是View的子View，所以事件将会使用View的dispatchTouchEvent(event)方法判断是否消费事件。

反之，如果mFirstTouchTarget不为null，说明上一次事件传递时，找到了需要处理事件的目标子View，此时，`ACTION_DOWN`的后续事件，如`ACTION_UP`等事件，都会传递至mFirstTouchTarget中保存的目标子View中。这里面还有一个小细节，如果在上一节遍历过程中已经把本次事件传递给子View，alreadyDispatchedToNewTouchTarget的值会被设置为true，代码会判断alreadyDispatchedToNewTouchTarget的值，避免做重复分发。

> 小结：
>dispatchTouchEvent方法首先判断事件是否需要被拦截，如果需要拦截会调用`onInterceptTouchEvent`，若该方法返回true，事件由ViewGroup自己处理，不在继续传递。
>若事件未被拦截，将先遍历找出一个目标子View，后续事件也将交由目标子View处理。
>若没有目标子View，事件由ViewGroup自己处理。
>
>此外，如果一个子View没有消费`ACTION_DOWN`类型的事件，那么事件将会被另一个子View或者ViewGroup自己消费，之后的事件都只会传递给目标子View（mFirstTouchTarget）或者ViewGroup自身。简单来说，就是如果一个View没有消费`ACTION_DOWN`事件，后续事件也不会传递进来。

### View
现在回头看上一节的第2、3步，不管是对子View分发事件，还是将事件分发给ViewGroup自身，最后都殊途同归，调用到了View的`dispatchTouchEvent`，这就是我们这一节分析的目标。

```java
public boolean dispatchTouchEvent(MotionEvent event) {

        ...

        if (onFilterTouchEventForSecurity(event)) {

        	// 判断事件是否先交给ouTouch方法处理
            if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
                    mOnTouchListener.onTouch(this, event)) {
                return true;
            }

            // onTouch未消费事件，传给onTouchEvent
            if (onTouchEvent(event)) {
                return true;
            }
        }

        ...

        return false;
    }
```

代码量不多，主要做了三件事：
1. 若View设置了OnTouchListener，且处于enable状态时，会先调用mOnTouchListener的onTouch方法
2. 若onTouch返回false，事件传递给`onTouchEvent`方法继续处理
3. 若最后onTouchEvent也没有消费这个事件，将返回false，告知上层parent将事件给其他兄弟View

这样，我们的分析转到了View的`onTouchEvent`方法：
```java
public boolean onTouchEvent(MotionEvent event) {
    final int viewFlags = mViewFlags;

    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PRESSED) != 0) {
            mPrivateFlags &= ~PRESSED;
            refreshDrawableState();
        }
        // 如果一个View处于DISABLED状态，但是CLICKABLE或者LONG_CLICKABLE的话，这个View仍然能消费事件
        return (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
    }
    
    ...

    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
        switch (event.getAction()) {

            case MotionEvent.ACTION_UP:

                boolean prepressed = (mPrivateFlags & PREPRESSED) != 0;
                if ((mPrivateFlags & PRESSED) != 0 || prepressed) {
                    

                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    if (prepressed) {
                        // The button is being released before we actually
                        // showed it as pressed.  Make it show the pressed
                        // state now (before scheduling the click) to ensure
                        // the user sees it.
                        mPrivateFlags |= PRESSED;
                        refreshDrawableState();
                   }

                    if (!mHasPerformedLongPress) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }

                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }

                    if (prepressed) {
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        // If the post failed, unpress right now
                        mUnsetPressedState.run();
                    }
                    removeTapCallback();
                }
                break;

            case MotionEvent.ACTION_DOWN:
                mHasPerformedLongPress = false;

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    mPrivateFlags |= PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // Not inside a scrolling container, so show the feedback right away
                    mPrivateFlags |= PRESSED;
                    refreshDrawableState();
                    checkForLongClick(0);
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                mPrivateFlags &= ~PRESSED;
                refreshDrawableState();
                removeTapCallback();
                break;

            case MotionEvent.ACTION_MOVE:
                final int x = (int) event.getX();
                final int y = (int) event.getY();

                // Be lenient about moving outside of buttons
                if (!pointInView(x, y, mTouchSlop)) {
                    // Outside button
                    removeTapCallback();
                    if ((mPrivateFlags & PRESSED) != 0) {
                        // Remove any future long press/tap checks
                        removeLongPressCallback();

                        // Need to switch from pressed to not pressed
                        mPrivateFlags &= ~PRESSED;
                        refreshDrawableState();
                    }
                }
                break;
        }
        return true;
    }

    return false;
}

public final boolean isFocusable() {
    return FOCUSABLE == (mViewFlags & FOCUSABLE_MASK);
}

public final boolean isFocusableInTouchMode() {
    return FOCUSABLE_IN_TOUCH_MODE == (mViewFlags & FOCUSABLE_IN_TOUCH_MODE);
}
```

`onTouchEvent`方法的主要流程如下：
1. 如果一个View处于DISABLED状态，但是CLICKABLE或者LONG_CLICKABLE的话，这个View仍然能消费事件，只是不会再走下面的流程;
2. 如果View是enable的且处于可点击状态，事件将被这个View消费：
在方法返回前，onTouchEvent会根据MotionEvent的不同类型做出不同响应，如调用refreshDrawableState()去设置View的按下效果和抬起效果等。
这里我们主要关注`ACTION_UP`分支，这个分支内部经过重重判断之后，会调用到performClick方法：

```java
public boolean performClick() {
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);

    if (mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        mOnClickListener.onClick(this);
        return true;
    }

    return false;
}
```

可以看到，如果设置了OnClickListener，就会回调我们的onClick方法，***最终消费事件***。

## 总结

通过上面的源码解析，我们可以总结出事件分发的整体流程：

![事件传递流程](http://ww1.sinaimg.cn/large/72f96cbagw1f9aljfns45j20nv15on04.jpg)

下面做一个总体概括：

事件由Activity的`dispatchTouchEvent()`开始，将事件传递给当前Activity的根ViewGroup：mDecorView，事件开始自上而下进行传递，直至被消费。

事件传递至`ViewGroup`时，调用`dispatchTouchEvent()`进行分发处理:
1. 检查送否应该对事件进行拦截:`onInterceptTouchEvent()`，若为true，跳过2步骤；
2. 将事件依次分发给子View，若事件被某个View消费了，将不再继续分发；
3. 如果2中没有子View对事件进行消费或者子View的数量为零，事件将由ViewGroup自己处理，处理流程和View的处理流程一致；

事件传递至`View`的`dispatchTouchEvent()`时， 首先会判断`OnTouchListener`是否存在，倘若存在，则执行`onTouch()`，若`onTouch()`未对事件进行消费，事件将继续交由`onTouchEvent`处理，根据上面分析可知，View的`onClick`事件是在`onTouchEvent`的`ACTION_UP`中触发的，因此，onTouch事件优先于`onClick`事件。

若事件在自上而下的传递过程中一直没有被消费，而且最底层的子View也没有对其进行消费，事件会**反向**向上传递，此时，父`ViewGroup`可以对事件进行消费，若仍然没有被消费的话，最后会回到Activity的`onTouchEvent`。

如果一个子View没有消费`ACTION_DOWN`类型的事件，那么事件将会被另一个子View或者ViewGroup自己消费，之后的事件都只会传递给目标子View（mFirstTouchTarget）或者ViewGroup自身。简单来说，就是如果一个View没有消费`ACTION_DOWN`事件，后续事件也不会传递进来。
