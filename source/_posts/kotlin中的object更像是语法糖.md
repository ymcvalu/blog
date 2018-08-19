---
title: kotlin中的object更像是语法糖
date: 2017-06-10 19:59:44
tags:
	- kotlin
---



### 单例声明



kotlin中，声明一个单例的语法很简单：
```
object obj
```
我们使用`object`关键字替代`class`关键字就可以声明一个单例对象
`object`一样可以继承其他类，或者实现其他接口：
```kotlin
interface IObj
abstract class AbstractObj
object obj : AbstractObj(),IObj  
```
在这里，我们让`obj`这个单例继承了`AbstractObj`，并且实现了`IObj`接口
声明一个单例对象，和声明一个`class`很类似
但是，`object`声明的单例对象**不能声明构造函数**，因为单例对象只有一个实例，无需我们手动将它创建出来，因此自然不需要构造函数。
> 如果需要对单例对象做初始化操作，可以在`init`初始化块内进行

那么`object`是什么时候被创建的呢？
>官方文档的解释是，`object`是`lazy-init`，即在第一次使用时被创造出来的

`object`单例基本的使用就像上面这样了，基本的用法参照官方的文档说明就好了

### 实现

在java中，我们要使用一个单例模式时，一般使用双重检查锁：

```java
public class Obj {
      private Obj(){}
      private static volatile Obj INSTANCE;		//volatile防止指令重排
      public static Obj getObj(){
          if(INSTANCE==null){
              synchronized(Obj.class){
                  if (INSTANCE==null){
                      INSTANCE=new Obj();
                  }
              }
          }
          return INSTANCE;
      }
}
```
而相同的功能，在kotlin只要`object Obj`就搞定了，这样的黑魔法是怎么实现的呢？
为了探究一二，我们先来看看编译之后的字节码:

```kotlin
//源代码:
object Obj{
    init{
        println("object init...")
    }
}
```
```java
//对应的字节码：
public final class Obj {   //可以看到生成了一个class，而类名就是object name
  // access flags 0x2
  private <init>()V     //注意看，<init>的可见性是`private`
   L0
    LINENUMBER 8 L0
    ALOAD 0  //将局部变量表slot 0处的引用入栈，即this引用
    INVOKESPECIAL java/lang/Object.<init> ()V   //调用父类的<init>
    ALOAD 0  //和上面一样，将局部变量表slot 0处的引用入栈，即this引用
    CHECKCAST Obj    //类型检查
    PUTSTATIC Obj.INSTANCE : LObj;     //保存this引用到`INSTANCE`这个静态域
   L1
    LINENUMBER 10 L1
    LDC "object init..."    //从常量池将字符串引用送至栈顶
    ASTORE 1    //将栈顶的引用保存到局部遍历表第一个slot处
   L2
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;    //获取out实例
    ALOAD 1    //从局部变量表第一个slot处加载引用到栈顶
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/Object;)V   //输出
   L3
   L4
   L5
    RETURN  //返回
   L6
    LOCALVARIABLE this LObj; L0 L6 0
    MAXSTACK = 2    //操作数栈深度为2
    MAXLOCALS = 2    //局部变量表为2个slot
  // access flags 0x19
  public final static LObj; INSTANCE    //**静态域**，类型为Obj
  // access flags 0x8
  static <clinit>()V    //静态初始化块，类初始化时执行
   L0
    LINENUMBER 8 L0
    NEW Obj     //创建一个Obj实例，引用保存在栈顶
    INVOKESPECIAL Obj.<init> ()V   //调用其<init>，初始化对象，此时会把栈顶引用作为this引用传入
    RETURN
    MAXSTACK = 1
    MAXLOCALS = 0
}
```
从上面的字节码中，我们可以看到，声明一个`object`，实际上就是创建了一个`class`，在类静态初始化时，会创建一个该类的实例，保存到其静态域`INSTANCE`中
进而可以猜想，源码中对单例的引用会被编译器替换为对`INSTANCE`这个静态域的引用
为了验证我们的分析，现在来看看使用单例时，对应的字节码
```kotlin
源码：
fun main(args:Array<String>){
    Obj is Any
}
```
```
对应的字节码：
public final static main([Ljava/lang/String;)V
    @Lorg/jetbrains/annotations/NotNull;() // invisible, parameter 0
   L0
    ALOAD 0
    LDC "args"
    INVOKESTATIC kotlin/jvm/internal/Intrinsics.checkParameterIsNotNull (Ljava/lang/Object;Ljava/lang/String;)V
   L1
    LINENUMBER 5 L1
    GETSTATIC Obj.INSTANCE : LObj;    //获取Obj的静态域`INSTANCE`
    INSTANCEOF java/lang/Object        //判断是否是`Object`类型
    POP
   L2
    LINENUMBER 6 L2
    RETURN
   L3
    LOCALVARIABLE args [Ljava/lang/String; L0 L3 0
    MAXSTACK = 2
    MAXLOCALS = 1
```

可以看到，我们在源码中直接使用`object name`访问单例对象，而编译器帮我们做了翻译，实际使用的是内部的静态域`INSTANCE`
而且，从对上面`Obj`这个生成的类的分析，我们可以发现，`object`在java中的对应的实现是类型这样的：

```java
public class  Obj {
    private Obj(){}
    private static Obj INSTANCE=null;
    static {
        INSTANCE=new Obj();
    }
}
```
这是最简单的单例实现方法，在类加载器加载class后执行静态初始化时就创建单例对象。

而`object`单例初始化的时机，准确来说，应该是这个类被加载时，静态初始化的时候。
做个小实验：

```kotlin
object Obj{
    init{
        println("object init...")
    }
}
```
```kotlin
fun main(args:Array<String>){
         Class.forName("Obj")
    }
```
>控制台输出：object init...

可见，当我们加载这个类的时候，单例就被创建了。
而单例名就是类名。

那么，`object`真的就是单例吗？
一般情况下是的，因为字节码中`<init>`方法被声明为`private`，虽然不太准确但是我们可以认为对应了类的一个`private`的无参构造函数，这就使得我们无法创建出一个新的对象出来。
但是，我们完全可以使用反射机制，从一个`private`的构造函数中创建一个对象出来：
```kotlin
fun main(args:Array<String>){
    println(Obj)
    var clazz=Class.forName("Obj")
    var constrouctor=clazz.getDeclaredConstructor()
    constrouctor.setAccessible(true)
    var instance=constrouctor.newInstance()
    constrouctor.setAccessible(false)
    println(instance)
}
输出：
object init...
Obj@511d50c0
object init...
Obj@60e53b93
```
可见，两次输出的对象引用是不一样的。

那么，这就说明kotlin的单例是不安全的吗？这到未必
我们在原先的基础上，加上几个属性声明：
```
object Obj{
    var name="name"
    var age="10"
    init{
        println("object init...")
    }
}
```
观察对应的字节码：
```
public final class Obj {
  private static Ljava/lang/String; name
   ...
  private static Ljava/lang/String; age
  ....
  public final static LObj; INSTANCE
  ...  
}
```
可以看到，这些属性的`field`都被声明为`static`了，尽管可以通过反射手段创建多个`object`的实例，但是它们的状态都是共享的
总结：

- `object`实际上还是生成一个`class`，但是这个`class`在`kotlin`中是透明的，无法直接访问，比如`Obj.INSTANCE`在kotlin中是不允许的，只能通过`Obj`来引用这个单例
- `object name`本质上是类名，只是编译器在编译时自动将`object name`换成了 `object`的`INSTANCE`
- `object`更像是语法糖