---
title: Java和Kotlin下的单例实现
keywords: Java和Kotlin下的单例实现
date: 2020-05-06 19:31:30
categories: 设计模式
tags:
	- 设计模式
	- Java
	- Kotlin
---

## 饿汉式

饿汉式的单例在类加载时就完成了初始化，所以类的加载较慢，但获取对象的速度快。这种方式直接就避免多线程的问题。

```
//Java实现
public class Singleton{

    private static Singleton instance = new Singleton();

    private Singleton(){}

    public static Singleton getInstance(){
        return instance;
    }
}

//Kotlin实现
object Singleton
```

## 懒汉式

在第一次调用的时候进行初始化，实现了懒加载。
分别有线程安全和线程不安全的两种写法，Java可通过synchronized同步锁来解决。
Kotlin直接使用by lazy代理即可实现，线程安全问题通过参数可控制，具体可参考我的另一篇文章[Kotlin延迟加载的线程安全探究](https://samiu.top/2020/04/10/kotlin-yan-chi-shu-xing-de-xian-cheng-an-quan-tan-jiu/)

```
//Java实现（线程不安全）
public class Singleton{

    private static Singleton instance;

    private Singleton(){}

    public static Singleton getInstance(){
        if(instance == null){
            instance = new Singleton();
        }
        return instance;
    }
}

//Java实现（线程安全）
public class Singleton{

    private static Singleton instance;

    private Singleton(){}

    public static synchronized Singleton getInstance(){
        if(instance==null)
            instance = new Singleton();
        return instance;
    }
}

//Kotlin实现
class Singleton private constructor() {

    companion object {
        val singleton: Singleton by lazy {
            Singleton()
        }
    }
}
```

## 双重检查模式

Java写法在获取实例的时候进行了两次判控，第一次判空是为了不必要的同步，只有第二次在实例为空的时候才创建实例，很好的解决了线程安全的问题。
Kotlin写法与上面的懒汉式类似。

```
//Java实现
public class Singleton{

    private volatile static Singleton instance;

    private Singleton(){}

    public static Singleton getInstance(){
        if(instance == null)
            synchronized(Singleton.class){
				if(instance == null)
                    instance = new Singleton();
			}
        return instance;
    }
}
```

## 静态内部类模式

```
//Java实现
public class Singleton{

    private Singleton(){}

    private static class SingletonHolder{
        private static final Singleton instance = new Singleton();
    }

    public static Singleton getInstance(){
        return SingletonHolder.instance;
    }
}

//Kotlin实现
class Singleton private constructor(){

    companion object{
        val instance = SingletonHolder.instance
    }

    private object SingletonHolder{
        val instance = Singleton()
    }
}
```
