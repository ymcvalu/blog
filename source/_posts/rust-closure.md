---
title: rust的闭包其实是语法糖
date: 2020-03-09 22:32:06
tags:
    - rust
---
# closure

`rust`的闭包，实际上是语法糖，它本质上是一个实现了特定`trait`的匿名的`struct`。



### Fn trait

```rust
#[lang = "fn"]
#[must_use = "closures are lazy and do nothing unless called"]
pub trait Fn<Args>: FnMut<Args> {
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}
```

Instances of `Fn` can be called repeatedly without mutating state.

**`Fn` is implemented automatically by closures which only take immutable references to captured variables or don't capture anything at all**, as well as (safe) [function pointers](https://doc.rust-lang.org/std/primitive.fn.html) (with some caveats, see their documentation for more details). Additionally, **for any type `F` that implements `Fn`, `&F` implements `Fn`**, too. 

```rust
fn main() {
    let hello = String::from("hello, rust!");
    let l = || println!("{}", hello);
    call_fn(l);
    call_fnmut(l);
    l();
    (&l)();
}

fn call_fn<T>(t: T)
where
    T: Fn() + Copy,
{
    t();
}

fn call_fnmut<T>(mut t: T)
where
    T: FnMut() + Copy,
{
    t();
}

```

实际上，没有捕获外部变量或者只捕获了`immutable reference`的`closure`，会被编译成实现了`Fn`这个`trait`的匿名的`struct`。

并且这个`struct`还会实现`Copy`这个`trait`，因为这个匿名`struct`里面只有`ref`，是可以`copy`的，因此这个`closure`可以多次传输。

**闭包本质上就是一个重载了`call`运算符的匿名结构体，不同的闭包，其结构体是不一样的，因此是DST，因此当需要使用闭包作为参数时，需要使用泛型或者trait object，当需要返回闭包时，可以使用`impl Fn(T) -> U`或者使用 trait object，比如 Box<dyn Fn(T)->U>**



### FnMut trait

```rust
#[lang = "fn_mut"]
#[must_use = "closures are lazy and do nothing unless called"]
pub trait FnMut<Args>: FnOnce<Args> {
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}
```

当一个闭包捕获了外部的`mutable reference`时，生成的匿名结构体会实现`FnMut`这个`trait`，并且**不会实现`Copy`这个`trait`**，而且，从方法`call_mut`的签名可以看到参数是`&mut self`：

```rust
fn main() {
    let mut hello = String::from("hello, closure!");
    let mut l = || {
        hello.push_str("!");
        println!("{}", hello);
    };
    call_fnmut(l);
    		   - value moved here
    l();
    ^ value borrowed here after move
}

fn call_fnmut<T>(mut t: T) // 这里`t`需要指定为`mut`
where
    T: FnMut(),
{
    t();
}
```

可以看到，作为参数传输的时候，会`move`，我们可以通过传递引用来解决：

```rust

fn main() {
    let mut hello = String::from("hello, rust!");
    let mut l = || {
        hello.push_str("!");
        println!("{}", hello);
    };
    call_fnmut(&mut l); // 传递 mutable ref
    l(); // l需要是mut的，因为参数是`&mut self`
    (&mut l)();
	
    let l = || {
        hello.push_str("!");
        println!("{}", hello);
    };
    call_fnmut(l);
}
```

**`FnMut` is implemented automatically by closures which take mutable references to captured variables**, as well as all types that implement [`Fn`](https://doc.rust-lang.org/core/ops/trait.Fn.html), e.g., (safe) [function pointers](https://doc.rust-lang.org/std/primitive.fn.html) (since `FnMut` is a supertrait of [`Fn`](https://doc.rust-lang.org/core/ops/trait.Fn.html)). Additionally, **for any type `F` that implements `FnMut`, `&mut F` implements `FnMut`, too.**

`FnMut`是`Fn`的`supertrait`，也就是实现`Fn`的前提是实现`FnMut`。



除了`borrow`外部变量的`reference`，有时候我们需要`closure`获得外部变量的`ownership`，比如我们的`closure`需要传到子线程执行，这时候它的`lifetime`需要是`'static`的，我们可以使用`move`关键字，来表明我们需要让`closure`获取所有它捕获的所有外部变量的`ownership`：

```rust
fn main() {
    let mut hello = String::from("hello, closure!");
    let mut l = move || {
        hello.push_str("!");
        println!("{}", hello);
    };
    call_fnmut(&mut l);
    l();
    l();
    // println!("{}", hello); // hello已经被move到closure内部了，这里无法使用了
}

fn call_fnmut<T>(t: &mut T)
where
    T: FnMut(),
{
    t();
}
```

**当我们在声明`closure`的时候，指定了`move`关键字，那么它捕获的所有外部变量都会`move`到匿名结构体中。**

对于获取了外部变量的`ownership`，但是没有`consume`外部变量的`closure`，也同样实现`FnMut`这个`trait`，就像上面的例子中的一样。



### FnOnce trait

当一个`closure`获取了外部变量的`ownership`，并且可能`consume`这个变量，那么这个`closure`只会实现`FnOnce`这个`trait`，并且只能被调用一次：

```rust
fn main() {
    let mut hello = String::from("hello, closure!");
    let l = move || { // hello被move到closure内
        hello.push_str("!");
        println!("{}", hello);
        drop(hello); // 这里消费了hello
    };
    l();
    // l(); // 只能被调用一次
}
```

我们来看一下`FnOnce`的声明：

```rust
#[lang = "fn_once"]
#[must_use = "closures are lazy and do nothing unless called"]
pub trait FnOnce<Args> {
    type Output;
    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}
```

可以看到，**`call_once`的参数是`self`**。

**`FnOnce` is implemented automatically by closure that might consume captured variables**, as well as all types that implement [`FnMut`](https://doc.rust-lang.org/core/ops/trait.FnMut.html), e.g., (safe) [function pointers](https://doc.rust-lang.org/std/primitive.fn.html) (since `FnOnce` is a supertrait of [`FnMut`](https://doc.rust-lang.org/core/ops/trait.FnMut.html)). 

**这里的`consume`指的是，在执行过程中，把move到匿名结构体中的变量又`move`到别的地方去了**

这个时候，原来的变量已经被`move`掉了，如果允许再次被执行，就和所有权系统相违背了。



这三个`trait`，`FnOnce`是`FnMut`的`supertrait`，而`FnMut`是`Fn`的`supertrait`





### 部分获取ownership

前面说到，**使用`move`，会把闭包捕获的所有变量的`ownership`都`move`到闭包内**

但是，有时候，可能我们只需要`move`部分变量的`ownership`到闭包内，这时候，我们可以：

```rust
fn main() {
    let mut hello = String::from("hello, closure!");
    let s = String::from("this is a string");
    let l = || {
        hello = capture(hello);
        hello.push_str("!");
        println!("{}", hello);
        println!("{}", s);
    };

    l();
    // println!("{}", hello); // hello被move到闭包内了
    println!("{}", s);   // s并没有被move到闭包内
}

fn capture<T>(t: T) -> T { // 该方法会让编译器分析需要获取ownership
    t
}
```

在上面的例子中，编译器会自动分析，需要捕获外部变量的`ownership`，因此`hello`被`move`到闭包内了，但是`s`只是获取了它的引用。

但是，这个闭包只实现了`FnOnce`，因为消费了`hello`，即使我们又把`hello`给move回去了，编译器还是认为它被消费了，不知道这是不是一个bug???



### closure基本使用

首先来看一个比较容易迷惑人的细节：

首先，我们的代码如下：

```rust
fn main() {
    let mut hello = String::from("hello, closure!");
    let mut l = || hello.push_str("!");
    println!("{}", hello);
    l();
}
```

但是运行的时候，编译器会报错：

```rust
    let mut l = || hello.push_str("!");
    			-- ----- first borrow occurs due to use of `hello` in closure
                |
                mutable borrow occurs here
    println!("{}", hello);
                   ^^^^^ immutable borrow occurs here
    l();
    - mutable borrow later used here
```

可能我们会想，我们不是还没有调用闭包吗，怎么就拿了`hello`的`mutable ref`了呢？？？

**那是因为，前面说到，闭包的本质是一个实现了特定trait的匿名结构体，因此我们在声明一个闭包的时候，就是创建了一个匿名结构体，而这个匿名结构体，拿了它捕获的外部变量`hello`的`mutable ref`**

一旦我们知道了闭包实现的原理，一切就变得明晰了。



闭包和函数一样，可以有参数和返回值，对于参数和返回值的类型，可以由编译器根据上下文自动推导，我们也可以显示指定：

```rust
fn main() {
    // 参数在`||`里面
    let l1 = |x| x + 1; // 如果只有一个表达式，可以省略`{}`
    let l2 = |x, y| x + y;
    let l3 = || 1;
    let l4 = |x: i32, y: i32| x + y;
    let l5 = |x, y| -> i32 { x + y };
    let l6 = |x: i32, y: i32| -> i32 { x + y }; // 如果显示指定了返回值，需要`{}`
    let l7 = |x: i32, y: i32| {
        println!("{}+{}", x, y);
        x + y
    };

    println!("{}", l1(1));
    println!("{}", l2(1, 2));
    println!("{}", l3());
    println!("{}", l4(1, 2));
    println!("{}", l5(1, 2));
    println!("{}", l6(1, 2));
    println!("{}", l7(1, 2));
}
```

目前，还不支持声明泛型`closure`，如果需要泛型还是使用函数，或者使用泛型函数返回一个闭包。





### FnXxx和fn的区别

`FnXxx`是`trait`，因此它们是`DST`，**而`fn`是类型，函数指针**，它的size的编译时已知的。

`fn`也实现了这些`trait`，因此声明参数的时候，我们可以统一使用泛型加`FnXxx`，这样调用的时候，无论传`closure`还是传函数指针都是可以。

不过，比如与`c`语言进行交互，就需要用函数指针了。