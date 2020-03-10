---
title: rust中的unit type
date: 2020-03-10 20:44:17
tags:
  - rust
---
`()` 是rust中的unit type，该类型是zero-size的，并且有一个唯一的值：`()`

> kotlin中也有一个类似的类型，就叫做`Unit`，并且它的唯一值就是`Unit`（一个Object）。

unit type 和 c 语言中的`void`有点类似。在 c 中，当一个函数不需要返回值时，可以将返回值声明为`void`。类似的，在rust中，如果我们的函数不需要返回值时，可以声明为返回`()`，实际上当我们没有声明返回值时，默认就是返回`()`：
```rust
fn no_return(){}
```
实际上等价于
```rust
fn no_return()->(){}
```
在kotlin中，当我们不需要返回值时，也是将其声明为返回`Unit`。

然而，rust中的`()`和c中的`rust`还是有一些区别的，`()`有一个唯一值，就是`()`，但是`void`并没有值。
> rust 中还有一个特殊的类型 `!`，该类型没有值，当一个类型的返回值声明为`!` 时，表明该函数不会返回。

在`rust`中，一切都可以认为是expression，当一个expression没有返回一个值时，实际上返回的是`()`，比如：
```rust
fn hello_rust() -> (){
    println!("hello, rust!");
}
```
在`rust`中以`;`结尾的是`statement`，其值是`()`。

`()`用在很多场合中，比如：
- 当一个`Option`或者`Result`并不关心返回值时，可以使用`Option<()>`或者`Result<(), Box<dyn Error>>`
- `HashSet<T>`实际上就是`HashTable<T, ()>`
- 比如在`trait object`的声明中：
  ```rust
    pub struct TraitObject {
        pub data: *mut (),
        pub vtable: *mut (),
    }
  ```
  当我们需要表示一个`raw pointer`，但是并不关心其实际的类型时，就可以使用`*mut ()`或者`*const ()`，类似于c中的`void*`和`const void*`。