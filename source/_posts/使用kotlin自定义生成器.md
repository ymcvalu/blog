---
title: 使用kotlin自定义生成器
date: 2017-12-02 22:35:17
tags: 
	- kotlin
	- coroutine
	- generator
---

使用kotlin的coroutine机制，可以很容易实现一个generator

```kotlin
import java.util.concurrent.atomic.AtomicReference
import kotlin.coroutines.experimental.*


class Generater<T : Any> private constructor() {
    private var mContinuation: AtomicReference<Continuation<Unit>?> = AtomicReference(null)
    private val values: ThreadLocal<T?> = ThreadLocal()
    /**
     * -1:结束
     *  0:未开始
     *  1:开始
     */
    @Volatile
    private var status: Int = 0

    companion object {
        fun <T : Any> build(block: suspend Generater<T>.() -> Unit): Generater<T> {
            val g = Generater<T>()
            var c = object : Continuation<Unit> {
                override val context: CoroutineContext
                    get() = EmptyCoroutineContext

                override fun resume(value: Unit) {
                    g.status = -1
                }

                override fun resumeWithException(exception: Throwable) {
                    g.status = -1
                    throw exception
                }
            }
            g.mContinuation.compareAndSet(null, block.createCoroutine(g, c))
            g.status = 1
            return g
        }
    }

    suspend fun yield(t: T?) {
        suspendCoroutine<Unit> {
            values.set(t)
            mContinuation.compareAndSet(null, it)
            //Thread.sleep(100)
        }
    }

    fun next(): T? {
        while (true) {
            if (status == -1) {
                values.set(null)
                break
            }
          	//可以提到循环外面
            if (status == 0) {
                throw IllegalStateException("生成器未启动")
            }
            val c = mContinuation.getAndSet(null)
            c ?: continue

            synchronized(this) {
                c.resume(Unit)
            }
            break
        }
        return values.get()
    }

}
```

使用

```kotlin

fun main(vararg args: String) {
	//声明生成器
    var g = Generater.build {
        yield(0L)
        var i = 0L
        var j = 1L
        while (true) {
            yield(j)
            var next = i + j
            i = j
            j = next
        }
    }
	//多线程访问
    Thread {
        for (i in 0..2)
            println(Thread.currentThread().name + ":" + g.next())
    }.start()

    Thread {
        for (i in 0..2)
            println(Thread.currentThread().name + ":" + g.next())
    }.start()

    Thread {
        for (i in 0..2)
            println(Thread.currentThread().name + ":" + g.next())
    }.start()

    Thread {
        for (i in 0..2)
            println(Thread.currentThread().name + ":" + g.next())
    }.start()

    Thread {
        for (i in 0..2)
            println(Thread.currentThread().name + ":" + g.next())
    }.start()

    Thread {
        for (i in 0..2)
            println(Thread.currentThread().name + ":" + g.next())
    }.start()

    Thread {
        for (i in 0..2)
            println(Thread.currentThread().name + ":" + g.next())
    }.start()

    Thread {
        for (i in 0..2)
            println(Thread.currentThread().name + ":" + g.next())
    }.start()
}
```

