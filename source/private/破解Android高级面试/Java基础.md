## Java的char是两个字节，是怎么存储UTF-8字符的？

- Java char不存UTF-8的字节，而是存UTF-16的
- Unicode通用字符集占两个字节，例如“中”
- Unicode扩展字符集需要用一对char来表示，例如emoji表情
- Unicode是字符集，不是编码，作用类似于ASCII码
- Java String的length不是字符数，而是char的个数

## Java的String有多长？

String两个不同的存储位置

```java
//存放在栈内
String longString = "aaa...aaa";	

//存放在堆
byte[] bytes = loadFromFile(new File("xxx.txt"));
String superLongString = new String(bytes);
```

- 受字节码限制，字符串最终的MUTF-8字节数不超过65535个

- Latin字符，受Javac代码限制，最多65534个

- 非Latin字符最多字节数是65535个

- 如果运行时方法区设置较小，也会收到方法区大小的限制

  

- 受虚拟机指令限制，堆存放的字符数理论上线为Integer.MAX_VALUE，实际情况上线可能小于Integer.MAX_VALUE

- 如果堆内存较小，也会收到堆内存的限制

## Java的匿名内部类有哪些限制？

匿名内部类的名字为外部类加$n，n是匿名内部类的名字，如com.samiu.doctor.Home$1

匿名内部类的构造方法：

- 编译器生成
- 参数列表包括
  - 外部对象
  - 父类的外部对象
  - 父类的构造方法参数
  - 外部捕获的变量（final）
  - 只有单一方法的接口可以用Lambda转换

## Java的方法分派

- 静态分派——方法重载分派
  - 编译期确定
  - 依据调用者的声明类型和方法参数类型
- 动态分派——方法重写分派
  - 运行时确定
  - 一句调用者的实际类型分派

## Java泛型的实现机制是怎样的？

泛型擦除：在编译的时候会进行泛型擦除，也就是说编译的时候所有的泛型都会变成Object

1. 基本类型无法所谓泛型实参

   ```java
   List<int> list;		//编译出错
   List<Integer> list;	//编译通过	
   ```

2. 泛型类型无法用作方法重载的参数

   ```java
   public void printList(List<Integer> list){}	//编译出错
   ```

3. 泛型类型无法当作真实类型使用（因为编译时擦除了）

   ```java
   private <T> void fun(T t){
   	T newInstance = new T();
   	T[] array = new T[0];
   	Class c = T.class;
       List<T> list = new ArrayList<T>();	//只有这一行可以编译，其他都不行
   	if(list instanceof List<String>){}
   }
   ```

   这里也就能解释为什么Gson的fromJson方法需要传一个classOfT参数了，因为不传的话它就不晓得你到底要啥类型的，虽然你有T，但是编译会进行泛型擦除，T都变成Object了。

## onActivityResult使用起来非常麻烦，为什么不设计成回调？

1. __Activity的销毁和恢复机制不允许匿名内部类出现：__

   假设A Activity使用回调启动了B activity，假设在使用B的过程中因为内存不足A被回收了，这个时候再从B回到A，Activity的恢复机制会恢复一个A Activity，但是这里的A已经不是最初的那个A了，是两个对象。

   这个时候如果在回调里面做操作，比如说更新UI，这里操作的对象是最初那个A，已经不是当前的A了，所以就会表现出来这个操作没有作用。

2. __非要这么做怎么实现？__

   基于注解处理器和Fragment的回调



## 面试技巧

- 抓住细节，有技巧的回避知识盲区
- 把我节奏，不要等面试官追问
- 主动深入，让面试官了解你的知识体系
- 触类旁通，让面试官眼前一亮