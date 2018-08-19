---
title: kotlin函数初级入门
date: 2017-12-11 23:36:59
tags:
	- kotlin
---



### 普通函数声明
使用 `func` 关键字声明一个函数，像这样

```kotlin
fun add(a:Int,b:Int):Int{
    return a+b
}
```
**在`kotlin`中，所有函数的参数都是`val`的，即不可变参数**
如果函数体只有一行代码，可以简洁点：
```kotlin
fun add(a: Int, b: Int): Int = a + b
```
更简单点，返回值自动推断：
```kotlin
fun add(a: Int, b: Int) = a + b
```
###带默认值的函数
可以在声明函数参数的时候，直接指定默认值，如果调用时没有传入，将使用默认值，带有默认值的参数可以在任何位置
```kotlin
fun add(a: Int=1, b: Int) = a + b
```
调用的时候，可以使用`参数名=值`的形式给出参数
```kotlin
add(b=1)
```
换个位置声明默认值：
```kotlin
fun add(a: Int , b: Int=1) = a + b
fun adds (a:Int,b:Int=1,c:Int)= a+b+c
fun main(vararg args:String){
    add(1)    //自动匹配第一个参数
    adds (1,c=2)    //默认值之后的参数需要显示指出参数名
}
```
函数匹配优先级：
```kotlin
fun add(a:Int) = a*a
fun add(a: Int , b: Int=1) = a + b
fun main(vararg args:String){
   println(add(3))  //输出 9 
}
```
当有多个函数匹配时，带默认值参数个数少的优先级越高，不带默认值的优先级最高

### 可变参数
在`kotlin`中，使用 `vararg` 关键字来标识可变参数
和`java`一样，多个参数会被封装成数组赋值给a

```kotlin
fun add(vararg a: Int, b: Int):Int{
    var sum :Int = 0
    a.forEach { 
        sum+=it
    }
    return sum+b
}
```
和`java`不同的是，在java中我们可以直接将一个数组赋值给可变参数，这有时候会引起混淆，因此在kotlin中，我们需要显示使用运算符`*`将数组解构成元素
比如我们可以这样调用上面的方法：
```kotlin
fun main(vararg args: String) {
    var arr = IntArray(10) { it }
    add(*arr, b = 1)
}
```
注意，第二个参数我们需要明确指出他的参数名，否则他会被当作可变参数中的一个值

### 使用lambda
`lambda`是一种特殊的函数。和传统函数不同的是，他可以被存储，传递，并且可以捕获外部变量形成闭包。
#####`lambda`的类型
因为`lambda`本质上还是对象，因此他是有类型的。
 `lambda`的类型格式为：

```kotlin
(参数列表)->返回值类型
```
`lambda`的`body`结构为：
```kotlin
{形参列表->
  语句
  ...
  最后一个语句的值为返回值（如果需要返回值）
}
```
比如：
```kotlin
val max:(Int,Int)->Int = {a,b -> if (a>b) a else b}
```
`max`用于比较两个整数，并返回他们中的最大者。因为这边接收了两个参数，因此我们需要在`lambda`的`body`中显示的为这两个参数指定一个名字，就像我们需要为函数指定形参名。
上面的例子也可以这样写：
```kotlin
val max =  { a:Int,b:Int-> if (a>b) a else b}  as (Int,Int)->Unit
```
不过这是比较2b的写法了，使用`as`对`lambda`进行类型转换，然后`max`的类型自动推出，这里我只是想说明`lambda`本质上还是个对象，因此一样可以进行类型转换。

### 高阶函数
`lambda`可以作为函数的参数或者返回值，举个例子：

```kotlin
fun <T> forEach(list:List<T>,block:(t:T)->Unit){
    for (t in list){
        block(t)
    }
}

fun main(vararg args:String){
    var list = listOf(1,2,3,4,5)
    forEach(list){
        println(it)
    }
}
```
在上面的例子中，我们声明了一个`forEach`的函数，他的功能是遍历一个列表，并对列表中的每个元素调用指定的`lambda`，然后我们在`main`函数中调用它。
值得注意的是，这里当我们的`lambda`是函数的最后一个参数时，我们可以将其写在`()`外面，当函数参数只有一个`lambda`时，可以直接省略`()`；
还有第二个要注意的是，我们使用了匿名参数，当`lambda`只有一个参数时，我们可以不用显示的指定一个参数名，而是使用默认的`it`来引用。

### 扩展函数
在`kotlin`中，我们可以很方便的扩展某个类，为其增加方法：

```kotlin
fun Int.compareWith(a: Int): Int {
    return when {
        this > a -> 1
        this < a -> -1
        else -> 0
    }
}

fun main(vararg args: String) {
    println(10.compareWith(4))
}
```
如上所示，声明一个扩展函数的语法很简单，只需要在方法名前面加上`类名.`，在方法中我们可以使用`this`来引用他，但是只能访问`public`的成员。这个`类名`我们使用`receiver`来描述它。
扩展函数只是`kotlin`中众多语法糖中的一个，他并没有真正的扩展这个类，只是将方法的一个参数提到了方法名前面作为`receiver`：
```kotlin
fun Int.compareWith(a:Int):Int
===>
fun compareWith(this:Int,a:Int):Int  
```
所以调用扩展函数和普通的函数调用没有区别，函数的`receiver`本质上还是这个函数的参数，而不是这个方法的所有者，因此在调用时使用的是静态分派，并不支持多态。
而且当扩展函数与类的方法冲突时，默认使用的是类的方法。

##### 结合泛型的扩展函数

```kotlin
fun <T:Any> T?.toStr(){
    println(this?.toString()?:"this is a null ref")
}
fun main(vararg args:String){
    null.toStr()
}
```
正如上面的例子中所看到的，我们可以使用泛型参数作为函数的`receiver`，而且我们使用了`T?`，说明支持`null`

##### 函数参数使用扩展函数
对刚刚的`forEach`函数稍加改造：

```kotlin
fun <T> forEach(list:List<T>,block:T.()->Unit){
    for (t in list){
       t.block()
    }
}

fun main(vararg args:String){
    var list = listOf(1,2,3,4,5)
    forEach(list){
        println(this)
    }
}
```
注意在声明函数参数时，我们使用了`T.()->Unit`，也就是声明了一个具有`receiver`的`lambda`

 