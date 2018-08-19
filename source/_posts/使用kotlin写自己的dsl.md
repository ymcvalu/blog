---
title: 使用kotlin写自己的dsl
date: 2017-04-25 00:29:05
tags:
	- kotlin
	- dsl
---



相比于java，kotlin对**FP**更加友好，支持扩展函数和操作符重载，这就为定义dsl提供了支持。
什么是dsl呢？就是一种面向特定问题的语言。gradle就是是一种用groovy定义的dsl。而kotlin一样对定义dsl提供了友好的支持。
本篇文章就来定义一个简单用于配置hibernate框架的dsl，写起来就像：

```kotlin
var conf= buildConfiguration{
                connection {
                    username = "***"
                    password = "******"
                    url = "jdbc:mysql://localhost:3306/******"
                    driver = Driver::class.java
                }
                c3p0 {
                    max_size = 30
                    min_size = 10
                    timeout=5000
                    max_statements=100
                    idle_test_period=300
                    acquire_increment=2
                    validate=true
                }
                entity {
                    mapping = Client::class.java
                    mapping = Financial_account::class.java
                    mapping = FundHolding::class.java
                    mapping=Fund::class.java
                }
                dialect="org.hibernate.dialect.MySQL5InnoDBDialect"
            }
```

上面是一个对hibernate的简单配置，最后获取一个configuration实例。通过使用dsl，可以避免在运行时解析xml文件，同时又比使用java代码配置简洁，兼具xml的结构化和java的高效。
那么，这样一个dsl是如何实现的呢？

> 先介绍一下预备知识：
> 扩展函数：
```kotlin
fun Type.foo():Unit{
  ...
}
```
这样就为`Type`对象创建了一个扩展函数，假如`t`是`Type`的一个实例，就可以：
```t.foo()```
`Type`称作`reciver`，而在`foo`函数体内，可以使用`this`访问其`public`成员，甚至可以省略`this`，仿佛`foo`函数是定义在`class Type`内
而声明一个函数的参数是扩展函数，一般的语法：
`fun funName(block:Type.(params...)->ReturnType):ReturnType{...}`
关于扩展函数更多的用法，可以参考官方的文档

首先先声明一个方法：
```kotlin
fun buildConfiguration(block:ConfigurationBuilder.()->Unit):Configuration{
    var cfg=ConfigurationBuilder()
    cfg.block()
    return cfg.cfg
} 
```
这个方法接收一个有receiver的lambda表达式，因为这样在block的内部就可以直接访问receiver的公共成员了，这一点很重要
紧接着对这个Configuration这个类进行定义
```kotlin
class ConfigurationBuilder{
     val TAG="hibernate"
     val cfg=Configuration()
     var dialect:String? get() = null
        set(value){
            cfg.setProperty("$TAG.dialect",value!!)
        }
     inline fun connection(block:ConnectionBuilder.()->Unit)=ConnectionBuilder(cfg).block()
     inline fun c3p0(block:C3p0Builder.()->Unit)=C3p0Builder(cfg).block()
     inline fun entity(block:Entity.()->Unit)=Entity(cfg).block()
}
```

在里面我定义了三个成员函数，分别对应前面示例中的
```kotlin
var conf= buildConfiguration{
                connection {
                    ...
                }
                c3p0 {
                    ...
                }
                entity {
                    ...
                }
               ...
            }
```

在这个lambda里面，我就直接调用了buildConfiguration的成员函数，那么对象引用呢？还记得我前面说过的吗？buildConfiguration这个方法的参数是一个有receiver的lambda，而在buildConfiguration中声明了一个ConfigurationBuilder对象并通过这个对象调用了这个lambda。那么这个lambda就会在这个对象的上下文中，我们可以直接访问它的公共成员，甚至可以使用this引用这个对象。
后续的步骤都差不多,我这里为了省事直接就声明了一个Configuration对象，并传到了其他对象里面
后面的源码
```kotlin
class ConnectionBuilder(val cfg:Configuration){//直接接受了一个configuration对象
     val TAG="hibernate.connection"
     var username:String? get() = null  //重写了setter和getter，防止属性有field
        set(name){
            cfg.setProperty("$TAG.username",name!!)  //直接硬编码设置属性
        }
     var password:String?  get() = null
        set(password) {
            cfg.setProperty("$TAG.password", password!!)
        }
      var url:String? get() = null
        set(url){
            cfg.setProperty("$TAG.url",url!!)
        }
      var driver:Class<*>? get() = null
        set(driver){
            cfg.setProperty("$TAG.driver_class",driver!!.name)
        }
      var pool_size:Int? get() = null
        set(size){
            cfg.setProperty("$TAG.pool_size",size!!.toString())
        }
}
//后面的都差不多。。。
 class C3p0Builder(val cfg:Configuration){
     val TAG="hibernate.c3p0"
     var max_size:Int? get() = null
        set(max_size){
            cfg.setProperty("$TAG.max_size",max_size!!.toString())
        }
     var min_size:Int? get() = null
        set(min_size){
            cfg.setProperty("$TAG.min_size",min_size!!.toString())
        }
     var timeout:Int? get() = null
        set(timeout){
            cfg.setProperty("$TAG.timeout",timeout!!.toString())
        }
     var max_statements:Int? get() = null
        set(max_stmt){
            cfg.setProperty("$TAG.max_statements",max_stmt!!.toString())
        }
     var idle_test_period:Int? get() = null
        set(idle_test_period){
            cfg.setProperty("$TAG.idle_test_period",idle_test_period!!.toString())
        }
     var acquire_increment:Int? get() = null
        set(acquire){
            cfg.setProperty("$TAG.acquire_increment",acquire!!.toString())
        }
     var validate:Boolean? get() = null
        set(validate){
            cfg.setProperty("$TAG.validate",validate!!.toString())
        }
}
 class Entity(val cfg:Configuration){
     var mapping:Class<*>?
        get()=null
        set(clazz){
            cfg.addAnnotatedClass(clazz!!)
        }
}
```

至此，一个简单的dsl就完成了
总体来说，定义一个dsl的过程基本是一个递归下去的过程，每个步骤都很类似