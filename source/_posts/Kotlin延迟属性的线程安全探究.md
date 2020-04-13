---
title: Kotlin延迟加载的线程安全探究
keywords: Kotlin延迟加载的线程安全探究
date: 2020-04-10 19:31:30
categories: Kotlin
tags:
	- Kotlin
	- 多线程
---

## 由单例模式引发的思考

众所周知，在Java语言中，创建一个对象我们可以在声明的时候就进行实例化，也可以把对象的声明和实例化分开进行。如

饿汉模式的单例

```java
public class Singleton{
    
	private static Singleton instance = new Singleton();
	
	private Singleton(){};
	
	public static Singleton getInstance(){
		return instance;
	}
}
```

懒汉模式的单例

```java
public class Singleton{
    
	private static Singleton instance;
    
	private Singleton(){};
    
	public static Singleton getInstance(){
		if(instance == null)
			instance = new Singleton();
		return instance;
	}
}
```

涉及到懒加载我们就不得不考虑到线程安全的问题，于是有了DCL的单例写法

```java
public class Singleton {

    private volatile static Singleton instance;

    private Singleton() { }

    public static Singleton getInstance() {
        if (instance == null)
            synchronized (Singleton.class) {
                if (instance == null)
                    instance = new Singleton();
            }
        return instance;
    }
}
```

## Kotlin的对象声明方式

相对于Java对象声明和实例化的过程，Kotlin对象的声明方式略显复杂，主要有以下几种：

1. 声明并赋值

   ```kotlin
   class Boy {
   	var name : String = "Samiu"
   	var height : Int = "169"
   }
   ```

2. 赋初始值为null

   ```kotlin
   class Boy {
   	var name : String? = null
   	var height : Int? = null
   }
   ```

3. 强制设为null

   ```kotlin
   class Boy {
   	var name : String = null!!
   	var height : Int = null!!
   }
   ```

当然除了以上之外还有一些其他的声明方式，与本篇文章关系不大，这里就不一一列举了。

这里我们就可以发现一个问题，看起来好像这个Kotlin对象的声明和初始化必须同时进行哇，声明的时候就必须给它赋一个确定的值，要么就算是null也必须显式的写出来赋值。

那么有没有办法达到Java的那种声明和初始化分开写，甚至是懒加载呢？

答案肯定的可以的。

## Kotlin的延迟加载

### lateinit和Delegates

```kotlin
class Boy {

    private lateinit var name: String
    private var height by Delegates.notNull<Int>()

    private fun client() {
        name = "Samiu"
        height = 169
    }
}
```

__lateinit var__可以让我们声明一个变量并且不用马上初始化，在我们需要的时候进行手动初始化即可。

__Delegates__是Kotlin给我们提供的一个委托类，这个类让我们可以通过委托的方式来声明一个变量，并且在需要的时候进行初始化。这个类总共提供了三个可用的方法

```kotlin
public object Delegates {
    
    public fun <T : Any> notNull(): ReadWriteProperty<Any?, T> = NotNullVar()

    public inline fun <T> observable(initialValue: T, crossinline onChange: (property: KProperty<*>, oldValue: T, newValue: T) -> Unit):
            ReadWriteProperty<Any?, T> =
        object : ObservableProperty<T>(initialValue) {
            override fun afterChange(property: KProperty<*>, oldValue: T, newValue: T) = onChange(property, oldValue, newValue)
        }

    public inline fun <T> vetoable(initialValue: T, crossinline onChange: (property: KProperty<*>, oldValue: T, newValue: T) -> Boolean):
            ReadWriteProperty<Any?, T> =
        object : ObservableProperty<T>(initialValue) {
            override fun beforeChange(property: KProperty<*>, oldValue: T, newValue: T): Boolean = onChange(property, oldValue, newValue)
        }
}

private class NotNullVar<T : Any>() : ReadWriteProperty<Any?, T> {
    private var value: T? = null

    public override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return value ?: throw IllegalStateException("Property ${property.name} should be initialized before get.")
    }

    public override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        this.value = value
    }
}
```

- notNull方法我们可以看到就是说这个对象不能为null，否则就会抛出异常。
- observable方法主要用于监控属性值发生变更，类似于一个观察者。当属性值被修改后会往外部抛出一个变更的回调。
- vetoable方法跟observable类似，都是用于监控属性值发生变更，当属性值被修改后会往外部抛出一个变更的回调。与observable不同的是这个回调会返回一个Boolean值，来决定此次属性值是否执行修改。

### lazy延迟属性

```kotlin
class Client{

    private val boy by lazy { Boy() }

    fun print(){
        println(boy.name)
        println(boy.height)
    }
}

class Boy {
    private val name = "Samiu"
    private val height = 169
}
```

这里我们使用lazy对boy这个对象进行了延迟加载，也就是说当我第一次调用boy对象的时候，才去对它进行一个初始化，也就是我们上面提到的懒汉模式。

那么既然是懒加载，就必然涉及到线程安全的问题，我们看看lazy是怎么解决的

```kotlin
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)

public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }
    
public actual fun <T> lazy(lock: Any?, initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer, lock)    
```

可以看到这里是根据LazyThreadSafetyMode的字段来分别进行不同的操作。

LazyThreadSafetyMode是一个枚举类

```kotlin
public enum class LazyThreadSafetyMode {

    SYNCHRONIZED,

    PUBLICATION,

    NONE,
}
```

- SYNCHRONIZED通过加锁来确保只有一个线程可以初始化Lazy实例，是线程安全的
- PUBLICATION表示不加锁，可以并发访问多次调用，但是我之接收第一个返回的值作为Lazy的实例，其他后面返回的是啥玩意儿我不管。这也是线程安全的
- NONE不加锁，是线程不安全的

从上面可以看到，当我们在使用lazy的时候，如果不加mode参数的话，它会选择执行SynchronizedLazyImpl这个方法，也就是默认线程安全的。

当然我们也可以通过添加mode参数来指定我们想要的懒加载效果。

## 参考资料

__1. [Kotlin官方文档](https://www.kotlincn.net/docs/reference/)__

__2. 《揭密Kotlin编程原理》__