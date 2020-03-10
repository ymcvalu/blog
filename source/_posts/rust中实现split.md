---
title: rust中实现split
date: 2020-03-10 21:16:51
tags:
    - rust
---
rust使用所有权系统来管理内存：
- 一个变量同一时刻只能绑定一个名称
- 一个变量可以同时拥有多个immutable reference，或者同一时刻只能有一个有效的mutable reference。

因此，当我们想要实现split一个slice时：
```rust
fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();

    assert!(mid <= len);

    (&mut slice[..mid],
     &mut slice[mid..])
}
```
编译器会拒绝编译，并告诉你：
```
error[E0499]: cannot borrow `*slice` as mutable more than once at a time
 -->
  |
6 |     (&mut slice[..mid],
  |           ----- first mutable borrow occurs here
7 |      &mut slice[mid..])
  |           ^^^^^ second mutable borrow occurs here
8 | }
  | - first borrow ends here
```
虽然我们很清楚这个操作是安全的，因为两个子slice并没有重叠，但是编译器还没有那么智能（如果是一个struct，现在我们可以同时获取两个不同的字段的mutable reference）。

`rust`的编译器是比较保守的，宁愿拒绝一些安全的操作，也不能让一些可能不安全的操作通过，因此当一些操作无法确认是否安全的时候，编译器就会拒绝编译。

这时候，我们就可以使用`unsafe`，因为我们确信这是安全的操作，也只有当我们可以确信操作是安全的时候，才使用`unsafe`。

在`rust`中，已经提供了`std::slice`这个crate，来帮助我们实现`split`：
```rust
use std::slice;

fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();

    assert!(mid <= len);

    unsafe {
        (
            slice::from_raw_parts_mut(ptr, mid),
            slice::from_raw_parts_mut(ptr.offset(mid as isize), len - mid),
        )
    }
}
```

当我们需要实现一个用于生产环境的`split`方法时，应该尽量使用成熟的代码，而不是重复造轮子，但是当以学习为主时，就应该尽量自己来实现，这样才能对一些细节有更深入的了解。

首先，我们要先知道，`rust`中的`slice`是什么。和普通的引用不同，`&[T]`是一个胖指针，它的内存布局是这样的：
```rust
#[derive(Copy)]
struct FatPtr<T> {
    data: *const T,
    len: usize,
}
```
它实际是有两个字段，一个是保存底层的数组的起始地址，另一个字段是slice的长度。当我们想要实现`split`的时候，首先就是要能拿到这两个字段的值。

那么问题来了，我们如何把一个`&[T]`转成一个`FatPtr`值呢？答案是使用`union`：
```rust
// union的字段需要实现 Copy trait
union Repr<T: Copy> {
    ptr: *mut [T],
    raw: FatPtr<T>,
}

impl<T> Clone for FatPtr<T> {
    fn clone(&self) -> Self {
        FatPtr { ..*self }
    }
}
```
`union`中的字段需要实现`Copy`这个`trait`，而要实现`Copy`就要实现`Clone`，因此我们要先为`FatPtr`实现`Clone`。我们定义了`Repr`这个`union`，他有两个字段，一个是`*mut[T]`类型的ptr，另一个是`FatPtr[T]`类型的`raw`。

我们可以借助`Repr`来获取`slice`的`data`和`len`这两个字段：
```rust
fn split_at_mut<T: Copy>(slice: &mut [T], mid: usize) -> (&mut [T], &mut [T]) {

    let r = Repr {
        ptr: slice as *mut [T],
    };

    // union的访问需要在unsafe block中
    let (ptr, len) = unsafe { (r.raw.data, r.raw.len) };

    // -snip--

}
```

在`union`中，所有的字段是共用存储的，因此使用`union`来进行相关类型的转换是一个比较常用的方法。

现在，我们就可以实现我们自己的`split`方法了，完整的代码如下：
```rust
fn split_at_mut<T: Copy>(slice: &mut [T], mid: usize) -> (&mut [T], &mut [T]) {

    let r = Repr {
        ptr: slice as *mut [T],
    };

    let (ptr, len) = unsafe { (r.raw.data, r.raw.len) };

    assert!(mid <= len);

    unsafe {
        (
            &mut *Repr {
                raw: FatPtr {
                    data: ptr,
                    len: mid as usize,
                },
            }.ptr,
            &mut *Repr {
                raw: FatPtr {
                    // `*mut T`实现了该方法
                    data: ptr.offset(mid as isize),
                    len: len - mid,
                },
            }.ptr,
        )
    }
}

union Repr<T: Copy> {
    ptr: *mut [T],
    raw: FatPtr<T>,
}

#[derive(Copy)]
struct FatPtr<T> {
    data: *const T,
    len: usize,
}

impl<T> Clone for FatPtr<T> {
    fn clone(&self) -> Self {
        FatPtr { ..*self }
    }
}
```

至此，我们已经实现了一个`&mut [T]`的`split`方法了。
