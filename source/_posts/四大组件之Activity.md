---
title: 四大组件之Activity
keywords: 四大组件之Activity
date: 2020-03-13 19:31:30
categories: Android
tags:
	- Android
	- 四大组件
---

## Activity的生命周期

### 典型情况下

1. __onCreate：__ 表示Activity正在被创建
2. __onRestart：__ 表示Activity正在重新启动
3. __onStart：__ 表示Activity正在被启动（已经显示出来了，但是我们还看不到）
4. __onResume：__ 表示Activity已经可见了
5. __onPause：__ 表示Activity正在停止
6. __onStop：__ 表示Activity即将停止
7. __onDestroy：__ 表示Activity即将被销毁

举例说明

- 第一次启动：onCreate -> onStart -> onResume
- 回到桌面：onPause -> onStop
- 再次回到原Activity：onRestart -> onStart -> onResume
- back键返回：onPause -> onStop -> onDestroy
- A Activity到B Activity：A.onPause -> B.onCreate -> B.onStart -> B.onResume -> A.onStop

### 异常情况下

Activity异常终止会调用__onSaveInstanceState__来保存当前Activity的状态，然后重新创建这个Activity的时候会调用__onRestoreInstanceState__把前面保存的Bundle对象作为参数传过来，这样就可以取出之前保存的数据并恢复。

## Activity的启动模式

### LaunchMode

1. __standard：__ 每一个Activity都会创建一个新的实例
2. __singleTop：__ 栈顶复用
3. __singleTask：__ 栈内复用，会将上面的activity出栈
4. __singleInstance：__ 单例，这个Activity创建一个任务栈单独享用

### Flags

有很多，这里列举几个常用的

|               Flags                |                      描述                      |
| :--------------------------------: | :--------------------------------------------: |
|       FLAG_ACTIVITY_NEW_TASK       |             启动模式为"singleTask"             |
|      FLAG_ACTIVITY_SINGLE_TOP      |             启动模式为"singleTop"              |
|      FLAG_ACTIVITY_CLEAR_TOP       | 使栈上面的所有Activity出栈，一般与上面一个连用 |
| FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS |         不会出现在历史Activity的列表中         |

## Activity的启动流程
