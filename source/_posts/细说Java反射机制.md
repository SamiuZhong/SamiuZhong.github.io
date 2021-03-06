---
title: 细说Java反射机制
keywords: 细Java反射机制
date: 2020-02-27 19:31:30
categories: Java
tags:
	- Java
---

## 反射到底是个啥？

> 能够分析类能力的程序被称为反射（reflective），通过反射可以在运行时（而不是在编译时）动态地分析软件组件并描述组件的功能。

首先我们要明确一点，反射并不是什么很难很高深的东西。正相反，反射其实就是Java系统提供给我们的一系列API，具体来说就是由java.lang.reflect包提供的。就跟调用其他API一个道理，我们只要了解了这个API用法，那就掌握了反射。

所谓存在即合理，既然Java给我们提供了这个功能，那么它一定是有用的。那它具体有啥用？通过反射，我们可以

- 分析类的能力
- 获得类的对象，即使你不知道这个类到底是个啥玩意儿

试想这样一个场景，我们有一个产品类

```java
public class Product {
    
    private String mName;
    private int mPrice;

}
```

然后我们现在要弄一个工厂类来生产这个产品。

有朋友说这有何难？看我的

```java
public class Factory {

    public Product create(){
        return new Product();
    }
}
```

然后调用这个工厂来生产即可

```java
public class Client {

    public static void main(String[] args) {
        Factory mFactory = new Factory();
        Product product = mFactory.create();
    }
}
```



乍一看好像没毛病，工厂生产的时候new一个Product不就完事了吗。

那好，我现在要求这个工厂要生产两个不同的产品又咋整？

朋友说，那也简单，我再加一个create方法不就行了

```java
class Factory {

    public ProductA createA(){
        return new ProductA();
    }
    
    public ProductB createB(){
        return new ProductB();
    }
}
```



那现在问题就来了：

1. 假如说我要生产一万个产品，难道我需要定义一万个方法？
2. 假如我甚至都不知道要生产的东西具体是个啥，又怎么办？

有杠精可能会说：你是不是在存心刁难我？我自己写的程序我还不晓得它要生产个啥？

你还别说，还真会有这种情况。试想假如你现在是在开发一个框架给别人用，这个Product类是交由这个框架的使用者来定义的，那你怎么能知道别人到底是定义了一个牛还是一个马？那我现在要生产这个东西，但是我又不知道到底是要生产一个啥玩意儿，那咋整？

嗯。。。好像确实不行。

那能不能有这样一种牛逼的技术，一个方法就可以生产一万个不同的产品，甚至我不知道它具体是个啥东西我都能生产出来？

巧了，还真有，这个牛逼的技术就是我们今天所说的反射。

## 整一个反射出来看看

我们对工厂类进行改写

```java
public class Factory {

    public <T> T create(Class clazz){
        Object object = null;
        try {
            object = Class.forName(clazz.getName()).newInstance();
        } catch (IllegalAccessException | InstantiationException | ClassNotFoundException e) {
            e.printStackTrace();
        }
        return (T)object;
    }
}
```

然后生产

```Java
public class Client {

    public static void main(String[] args) {
        Factory mFactory = new Factory();
        Product product = mFactory.create(Product.class);
    }
}
```

好了，这样我们就使用反射创建了一个产品的实例。要创建多个不同的产品，只需要传入不同的参数即可。

现在来解释一下这个create方法。

首先我们定义返回类型为泛型T，方法的形参为Class类型表明传入的是一个类。

然后Class.forName方法是通过类加载器来加载这个类，加载的实现遵循双亲委派机制。关于双亲委派机制的原理后面会写专栏介绍。

最后调用newInstance方法来返回这个类的实例。

这样我们就通过反射获得了一个类的对象，并且在这个过程中我们并不关心这个加载的类到底是个啥东西。

## 反射还能做啥？

除了获取类的对象之外，反射还可以分析类的能力。

### 获取构造函数的信息

- __getConstructors()方法__

我们先给产品类整几个构造函数

```java
public class Product {
    private String mName;
    private int mPrice;

    public Product() { }

    public Product(String name) {
        mName = name;
    }

    public Product(int price) {
        mPrice = price;
    }

    public Product(String name, int price) {
        mName = name;
        mPrice = price;
    }
}
```

然后改写工厂类

```java
public class Factory {

    public void create(Class clazz){
        //获得构造函数
        Constructor[] constructors = clazz.getConstructors();
        for (Constructor constructor : constructors) {
            System.out.println("" + constructor);
        }
    }
}
```

跑一下看看结果

```java
public com.samiu.archi.reflect.Product(java.lang.String,int)
public com.samiu.archi.reflect.Product(int)
public com.samiu.archi.reflect.Product(java.lang.String)
public com.samiu.archi.reflect.Product()
```

OK，这不就把构造函数打印粗来了。

### 获得类的变量

- __getFields方法__
- __getDeclaredFields方法__

我们对产品类进行改写，多整几个变量

```java
public class Product {

    //私有变量
    private String mName;
    private int mPrice;

    //公共变量
    public float width;
    public float height;
    
}
```

然后改写工厂类

```java
public class Factory {

    public void create(Class clazz){
        //获得fields
        Field[] fields = clazz.getFields();
        System.out.println("fields：");
        for (Field field : fields) {
            System.out.println("" + field);
        }
        //获得declaredFields
        Field[] declaredFields = clazz.getDeclaredFields();
        System.out.println("declaredFields：");
        for (Field field : declaredFields) {
            System.out.println("" + field);
        }
    }
}
```

打印一下结果

```java
fields：
public float com.samiu.archi.reflect.Product.width
public float com.samiu.archi.reflect.Product.height

declaredFields：
private java.lang.String com.samiu.archi.reflect.Product.mName
private int com.samiu.archi.reflect.Product.mPrice
public float com.samiu.archi.reflect.Product.width
public float com.samiu.archi.reflect.Product.height
```

可以看到，getFields方法获取了类的公共变量，而getDeclaredFields方法获取了类的所有变量，包括私有变量。

### 获得类的方法

- __getMethods方法__
- __getDeclaredMethods方法__

我们先改写产品类，给它多整几个方法

```java
public class Product {
    private String mName;
    private int mPrice;

    public String getName() {
        return mName;
    }

    private void setName(String name) {
        mName = name;
    }

    public int getPrice() {
        return mPrice;
    }

    private void setPrice(int price) {
        mPrice = price;
    }
}
```

然后改写工厂类

```Java
public class Factory {

    public void create(Class clazz){
        //获得methods
        Method[] methods = clazz.getMethods();
        System.out.println("methods：");
        for (Method method:methods){
            System.out.println(""+method);
        }
        System.out.println("");
        //获得declaredMethods
        Method[] declaredMethods = clazz.getDeclaredMethods();
        System.out.println("declaredMethods：");
        for (Method method:declaredMethods){
            System.out.println(""+method);
        }
    }
}
```

打印一下看看

```java
methods：
public java.lang.String com.samiu.archi.reflect.Product.getName()
public int com.samiu.archi.reflect.Product.getPrice()
public final void java.lang.Object.wait() throws java.lang.InterruptedException
public final void java.lang.Object.wait(long,int) throws java.lang.InterruptedException
public final native void java.lang.Object.wait(long) throws java.lang.InterruptedException
public boolean java.lang.Object.equals(java.lang.Object)
public java.lang.String java.lang.Object.toString()
public native int java.lang.Object.hashCode()
public final native java.lang.Class java.lang.Object.getClass()
public final native void java.lang.Object.notify()
public final native void java.lang.Object.notifyAll()

declaredMethods：
public java.lang.String com.samiu.archi.reflect.Product.getName()
private void com.samiu.archi.reflect.Product.setName(java.lang.String)
private void com.samiu.archi.reflect.Product.setPrice(int)
public int com.samiu.archi.reflect.Product.getPrice()
```

可以看到，getMethods获取了类的所有公共方法，包括定义在Object类里面的公共方法，而getDeclaredMethods则获取了类自己定义的所有方法，包括私有的方法。

除了这些以外，反射还有很多实用的功能，这里就不展开细讲了，感兴趣的朋友可以自己去查阅。

## 参考资料

__1. 《Java核心技术 第八版》__

__1. 《Java完全参考手册 第八版》__