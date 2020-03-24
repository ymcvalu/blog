---
title: kotlin入门之访问数据库的奇淫巧技
date: 2017-06-11 23:02:54
tags:
    - kotlin
---

如何在业务层最便捷的调用dao层来实现数据的增删改查呢

如果我说插入一条记录只要写`+ fund`，你信吗？

得益于kotlin的操作符重载功能和，我们可以很容易的实现该功能。

首先，我们要在dao层定义好访问数据库的通用接口并实现一个通用的dao层类，这里面的所有方法都要求是泛型方法。

下面是我自己封装的一个简单的通用dao
```kotlin
open class BaseDBAccessorImpl<T>: IBaseDBAccessor<T>{

    //dao层使用了hirbernate，但是没有使用ioc框架，需要直接获取session
    //SessionFac是我自己封装的一个工具类，里面有个counter计数器对同个线程的getSession和close调用进行数，
    //第一次get开启事务，最后一次close关闭事务
    
    override val session: Session=SessionFac.getSession()

    override fun close(){
        SessionFac.closeSession(session)
    }

    override fun insert(t: T) {
        session.saveOrUpdate(t)
    }

    override fun delete(id: Int,clazz:Class<T>) {
        var obj=session.get(clazz,id)
        session.delete(obj)
    }

    override fun delete(t: T) {
        session.delete(t)
    }

    override fun getCount(clazz:Class<T>): Int
            =(session.createQuery("select count(*) from ${clazz.simpleName}")
            .uniqueResult() as Long).toInt()

    override fun getListByPage(clazz:Class<T>, pageNo: Int, rows: Int)
            = session.createQuery("from ${clazz.simpleName}")
            .setFirstResult((pageNo-1)*rows)
            .setMaxResults(rows)
            .list() as List<T>

    override fun getObj(clazz:Class<T>,id: Int): T 
            = session.get(clazz,id) as T
    
    }
```

现在已经有dao层了，如果没有使用ioc框架，那么每次使用dao层都要先实例化一个dao层的对象，然后调用dao层的方法实现我们的业务需求。
这无疑是繁琐而又无趣的，好在kotlin支持top-level函数，我们可以直接声明一系列函数把对数据库的访问封装起来，比如插入一条记录：
```kotlin
inline fun <T> T.insert(){
    val acc= NewDatabase<T>()
    acc.insert(this)
    acc.close()
} 
```

在这里我们声明一个泛型的**extension函数**，**而且receiver就是泛型参数T**
这个函数的效果就是：
对任意的entity实体对象obj,使用：

>`obj.insert()`

就是将obj本身作为一条记录插入数据库。

因为是一个泛型函数，所以无论是什么类型的entity都可以直接插入数据库，那么问题来了，如果调用者不是一个entity呢？那么dao层内部肯定会直接抛出异常。会抛异常的代码肯定不是好代码，而使用了泛型，所有的对象都可以调用这个函数。因此，我们需要在调用dao层之前先做安全检查。

我们重新定义一个函数方便重用：
```kotlin
inline fun <T> T.checkobj():Boolean{
    // 判断是否是一个entity
    if((this as Any).javaClass.getAnnotation(Entity::class.java)==null)
       	return false
    return true
}
```
接着对原函数修改：
```kotlin
inline fun <T> T.insert(){
    if(!checkobj())
        return
    val acc= NewDatabase<T>()
    acc.insert(this)
    acc.close()
}
```
现在，如果是非@entity对象调用insert将直接返回。

我们还可以进一步简化：
```kotlin
inline operator fun <T> T.unaryPlus(){
    insert()
}
```
在这里我们声明了一个**重载了一元运算符`+`**的方法，而这个函数同上面的`insert`一样是一个泛型的`extension`函数，在里面我们直接重用上面的insert函数来实现功能逻辑。

那么现在，我们就可以随意使用
```kotlin
+ entity_obj
```
来插入一条数据到数据库中了。而且这个调用完全符合我们的思维：插入数据不就是`+`一条数据吗。

在一个复杂的业务流程中需要对许多entity进行操作，而每个类型的entity都需要提供一个dao对象，尽管它们之间的差别可能只是指定了不同的泛型参数。

然而现在，插入一条entity_a，直接`+ entity_a`，entity_b呢？一样的`+entity_b`
没有任何的不同，也不需要额外声明两个dao类对象，**简洁而又优雅**

到此，我们就完成了对数据插入操作的封装。而对于其他的操作，比如删除可以直接`- fund`或者`Fund::class - id`，查找指定id的记录可以像`Fund::class[id]`，跟上面的差别无非就是重载的运算符和函数的泛型参数个数有些许差别


上面所有的函数里面，dao层的对象都是通过NewDatabase()来获取的，那么这个NewDataBase又是什么呢？
```kotlin
inline fun <T> NewDatabase():IBaseDBAccessor<T>{
    	return BaseDBAccessorImpl<T>()
}
```

可以看到NewDatabase是一个函数而不是一个类，这里返回值声明的是一个接口，面向接口编程从而降低了代码间的耦合。可以看出这个函数实际上就是起到了一个静态工厂类的作用。

以往在java中我们需要定义的一个个工厂类在kotlin中完全可以直接使用top-level函数代替，实际上编译成jvm字节码后，这些top-level函数在java中就是一个个类的静态方法

上面所有的函数都被定义成了**inline**，实际运行和直接实例化一个dao层对象来访问数据库是一样的，并没有太多额外的运行开销。