---
title: 使用kotlin协程机制撸一个简易的异步执行库
date: 2017-11-10 21:43:24
tags:
	- kotlin
	- coroutine
---



> 由于android限制了只能在UI线程更新视图，而在UI线程中做耗时任务又会导致ANR，因此在平时的开发中，需要将耗时的数据请求工作放到子线程中执行，而视图更新工作放到UI线程中，使用传统的handler或者asyncTask，需要将逻辑分到多个函数内

**使用`kotlin`的协程机制，可以用同步的方式实现异步**
kotlin的协程机制是基于状态机模型和`C-P-S`风格实现的。
一个协程通过`resume`启动，当协程内部调用`supended`函数时，协程会被暂停，通过调用 `resume`可以再次启动协程。每次暂停都会修改协程的状态，再次启动协程时，会从新的状态处开始执行。

现在通过kotlin的基础api实现一个简单的异步调用接口，最后的效果如下：

 ```
 btn.setOnClickListener {
            runOnUI {  
                //执行在主线程，可以做一些初始化操作                         
                Log.e("log", Thread.currentThread().name)
                var used = async {               //从工作线程直接返回数据到主线程
                   //切换到工作线程执行，而且lambda可以直接访问外部变量，构成闭包
                    Log.e("log", Thread.currentThread().name)
                    var start = System.currentTimeMillis()
                    Thread.sleep(3000)
                    System.currentTimeMillis() - start
                }
                //继续执行在主线程
                Log.e("log", Thread.currentThread().name)
                Toast.makeText(this@MainActivity, "后台线程用时${used}ms", Toast.LENGTH_SHORT).show()
            }
        }
 ```
>在后续的内容中，我将在实现的过程中逐步分析kotlin协程机制的基本原理

首先声明一个创建协程的函数：
```
//该函数接收一个 suspend类型的lambda
inline fun runOnUI(noinline block: suspend () -> Unit) {
    var continuation = object : Continuation<Unit> {
      //ThreadSwitcher是ContinuationInterceptor的子类，用于在协程resume时切换到主线程执行
        override val context: CoroutineContext
            get() = ThreadSwitcher()  

        override inline fun resume(value: Unit) = Unit

        override inline fun resumeWithException(exception: Throwable) = Unit
    }
        //使用suspend类型的lambda创建一个协程并启动
        block.createCoroutine(continuation).resume(Unit)
}
```
`createCoroutine`是官方提供的一个基础api，该函数如下：
```
public fun <T> (suspend () -> T).createCoroutine(
        completion: Continuation<T>
): Continuation<Unit> = SafeContinuation(createCoroutineUnchecked(completion), COROUTINE_SUSPENDED)
```
可以看到调用了`createCoroutineUnchecked`创建一个`Coroutine`，继续查看该方法：
```
@SinceKotlin("1.1")
@kotlin.jvm.JvmVersion
public fun <T> (suspend () -> T).createCoroutineUnchecked(
        completion: Continuation<T>
): Continuation<Unit> =
//这里的this是执行createCoroutine函数的block
        if (this !is kotlin.coroutines.experimental.jvm.internal.CoroutineImpl)
            buildContinuationByInvokeCall(completion) {
                @Suppress("UNCHECKED_CAST")
                (this as Function1<Continuation<T>, Any?>).invoke(completion)
            }
        else
//编译时，block会被编译成一个CoroutineImpl的子类，所以走这个分支
            (this.create(completion) as kotlin.coroutines.experimental.jvm.internal.CoroutineImpl).facade
```
查看编译之后生成的`block`：
```
//查看在Activity#onCreate调用runOnUI处传入的lambda的编译类
final class ymc/demo/com/asyncframe/MainActivity$onCreate$1$1 
          extends kotlin/coroutines/experimental/jvm/internal/CoroutineImpl   
          implements kotlin/jvm/functions/Function1  {      //lambda编译类都实现FunctionN函数
  ...
}
```
可以看到传入`runOnUI`的`lambda`确实被编译成了一个`CoroutineImpl`，这是因为编译器推断出了这个`lambda`是`suspend`类型的。

继续上面的分析，创建协程所涉及到的两个方法中都出现了 `Continuation`这个类，那么这个类是干嘛的呢？
首先，先看看`completion`，这个是我们调用`createCoroutine`手动传入的，当协程结束时，他的`resume`会被调用，当协程异常结束时，他的`resumeWithException`会被调用。
再看看`createCoroutineUnchecked`，这个函数也返回了一个`Continuation`，那么这个又是什么呢？
```
 (this.create(completion) as kotlin.coroutines.experimental.jvm.internal.CoroutineImpl).facade
```
可以看到，返回的是`CoroutineImpl`的`facade`，那这个又是什么呢？
我们进入`CoroutineImpl`，可以看到
```
abstract class CoroutineImpl(
        arity: Int,
        @JvmField
        protected var completion: Continuation<Any?>?
) : Lambda(arity), Continuation<Any?> {     //Coroutine本身是一个Continuation

  override val context: CoroutineContext
          get() = _context!!

  private var _facade: Continuation<Any?>? = null
 
  val facade: Continuation<Any?> get() {
          if (_facade == null) _facade = interceptContinuationIfNeeded(_context!!, this)
          return _facade!!
      }
  ...
}
```
原来这是一个代理属性，接着查看`interceptContinuationIfNeeded`，
```
internal fun <T> interceptContinuationIfNeeded(
        context: CoroutineContext,
        continuation: Continuation<T>
) = context[ContinuationInterceptor]?.interceptContinuation(continuation) ?: continuation
```
这个函数从`Coroutine`的上下文中查找`ContinuationInterceptor`，如果有就调用他的`interceptContinuation`对传入的`continuation`进行包装，否则直接返回传入的`continuation`

**`Continuation`**是一个可**继续执行**体的抽象，每个`Coroutine`都是一个可继续执行体，`Continuation`是一个协程对外的接口，启动/恢复协程的`resume`就是在该接口中定义的。
协程可以是链式连接的，一个协程可以有子协程，子协程持有父协程的引用，当子协程执行时，父协程暂停，子协程结束时，内部通过调用父协程的`resume`返回父协程。

还记得我们前面用到的`ThreadSwitcher`吗，他就是一个`ContinuationInterceptor`
我们来看看来看`ThreadSwitcher`的实现：
```
/**
Interceptor用于用于拦截并包装Continuation，让我们有机会在协程resume前做一些额外的操作，比如线程切换
**/
class ThreadSwitcher : ContinuationInterceptor, AbstractCoroutineContextElement(ContinuationInterceptor.Key) {

    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
            = object : Continuation<T> by continuation {

        override fun resume(value: T) {
          //如果在主线程，直接执行
            if (Looper.getMainLooper() === Looper.myLooper()) {
                continuation.resume(value)
            } else {
            //否则，使用handler机制post到主线程执行
                postman.post {
                    resume(value)
                }
            }
        }

        override fun resumeWithException(exception: Throwable) {
            if (Looper.getMainLooper() === Looper.myLooper()) {
                continuation.resumeWithException(exception)
            } else {
                postman.post {
                    resumeWithException(exception)
                }
            }
        }
    }
}
```
从上面的分析中，我们可以想象，我们创建的协程会被`ThreadSwitcher`包装，
```
block.createCoroutine(continuation).resume(Unit)
```
`createCoroutine`返回的实际是`ThreadSwitcher`返回的`Continuation`，所以当我们执行`resume`启动协程时，会先切换到主线程执行。

紧接着，我们来实现`async`：
```
suspend inline fun <T> async(crossinline block: () -> T): T
        = suspendCoroutine {
//dispatcher是一个对线程池的封装，将任务分发到子线程中
    dispatcher.dispatch {
        it.resume(block())
    }
}
```
使用`suspend`修饰的方法只可以在协程内部调用，而`suspendCoroutine`方法是`kotlin`提供的一个基础api,用于实现暂停协程。
我们接着来分析`suspendCoroutine`，查看他的实现：
```
public inline suspend fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T =
        suspendCoroutineOrReturn { c: Continuation<T> ->
            val safe = SafeContinuation(c)
            block(safe)
            safe.getResult()
        }
```
可以看到这个方法接收的`block`是带`Continuation`参数的
真正实现功能的是`suspendCoroutineOrReturn`，当我们继续跟进时，发现：
```
public inline suspend fun <T> suspendCoroutineOrReturn(crossinline block: (Continuation<T>) -> Any?): T =
        throw NotImplementedError("Implementation is intrinsic")
```
what!直接抛出异常了???
这是因为这是一个特殊的函数，需要编译器特殊处理，他需要将当前协程内的`_facade`属性，包装成`SafeContinuation`，再作为我们传入的`block`的参数，而且这个`_facade`是经过`ContinuationInterceptor`处理过的，也就是说当我们调用`resume`恢复线程时，会先切换到主线程。
为了验证上面的分析，我们查看`async`编译之后的字节码：
```
//可以看到编译之后，我们的async多了一个Continuation类型的参数
 private final static async(Lkotlin/jvm/functions/Function0;Lkotlin/coroutines/experimental/Continuation;)Ljava/lang/Object;
   L0
    LINENUMBER 70 L0
    NOP
   L1
    LINENUMBER 77 L1
    ICONST_0
    INVOKESTATIC kotlin/jvm/internal/InlineMarker.mark (I)V
    ALOAD 1  //将第二个参数，也就是Continuation入栈
//调用CoroutineIntrinsics.normalizeContinuation 
    INVOKESTATIC kotlin/coroutines/experimental/jvm/internal/CoroutineIntrinsics.normalizeContinuation (Lkotlin/coroutines/experimental/Continuation;)Lkotlin/coroutines/experimental/Continuation;  
//将返回值存到slot3
    ASTORE 3
   L2
    LINENUMBER 78 L2
//new 一个SafeContinuation
    NEW kotlin/coroutines/experimental/SafeContinuation
    DUP  
  //将刚刚normalizeContinuation返回的continuation传入SafeContinuation的构造函数
    ALOAD 3
    INVOKESPECIAL kotlin/coroutines/experimental/SafeContinuation.<init> (Lkotlin/coroutines/experimental/Continuation;)V
    ASTORE 4
   L3
  ...
```
我们可以看到，编译之后的字节码已经没有了`suspendCoroutine`和`suspendCoroutineOrReturn`的身影，因为这两个函数都是`inline`函数。
我们接着来看`CoroutineIntrinsics.normalizeContinuation`的实现：
```
fun <T> normalizeContinuation(continuation: Continuation<T>): Continuation<T> =
        (continuation as? CoroutineImpl)?.facade ?: continuation
```
 还记得我们刚刚分析过`facade`这个属性吗？他是对`_facade`的代理，这个函数返回的是经过拦截器处理过的`Continuation`
根据刚刚的字节码，我们可以发现`suspend`类型的函数，都会**隐式**额外接受一个当前协程的引用，但是又不能在函数中直接访问。

最后，还有两个上文出现过的线程切换处理类，`postman`和`dispatcher`，使用的是单例模式：
```
object postman : Handler(Looper.getMainLooper()) {
    override fun handleMessage(msg: Message?) {
        msg?.callback?.run()
    }
}

object dispatcher {
    val mCachedThreads = Executors.newCachedThreadPool()
    inline fun dispatch(noinline block: () -> Unit) {
        mCachedThreads.execute(block)
    }
}
```
到此，我们实现了一个简易的异步调用库！