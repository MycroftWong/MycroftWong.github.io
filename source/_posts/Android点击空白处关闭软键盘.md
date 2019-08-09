---
title: Android点击空白处关闭软键盘
date: 2019-08-09 15:02:34
categories: Android
tags: android,EditText,软键盘
---

# Android点击空白处关闭软键盘

## 答案
直接说解决方法，在`Activity`中重写`dispatchTouchEvent(MotionEvent ev)`方法
```java
/**
 * 点击空白处，关闭软键盘
 *
 * @param event event
 * @return boolean Return true if this event was consumed.
 */
@Override
public boolean dispatchTouchEvent(MotionEvent event) {
    if (event.getAction() == MotionEvent.ACTION_DOWN) {
        View v = getCurrentFocus();
        if (v instanceof EditText) {
            Rect outRect = new Rect();
            v.getGlobalVisibleRect(outRect);
            if (!outRect.contains((int) event.getRawX(), (int) event.getRawY())) {
                v.setFocusable(false);
                v.setFocusableInTouchMode(true);

                InputMethodManager imm = (InputMethodManager) getSystemService(Context.INPUT_METHOD_SERVICE);
                imm.hideSoftInputFromWindow(v.getWindowToken(), 0);
            }
        }
    }
    return super.dispatchTouchEvent(event);
}
```

原理：在当前焦点在`EditText`的情况下，点击的不是该`EditText`所在区域，则关闭软键盘

## 另外的方法
监听整个根布局`MotionEvent`，当然效果没有上面这个方法好