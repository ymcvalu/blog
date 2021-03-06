---
title: golang内存逃逸
date: 2019-03-31 21:42:24
tags:
	- go
---

`golang`中，编译器在编译时会通过内存逃逸分析确定变量分配在堆上还是栈上。

```go
type Dog struct {
}

func (d *Dog) Eat() {
}

type Animal interface {
	Eat()
}

func main() {
	dog1 := new(Dog)
	noneEscape(dog1)
	nop(dog1)
	dog2 := new(Dog)
	escape(dog2)
	dog3 := Dog{}
	fmt.Println(&dog3)
}

//go:noinline
func nop( a Animal){
}

//go:noinline
func noneEscape(d *Dog) {
	d.Eat()
}

//go:noinline
func escape(a Animal) {
	a.Eat()
}

```

可以通过`--gcflags="-m -m"`参数，在编译时打印出内存逃逸分析信息，`-m`最多可以指定四个，越多打印的信息越详细。上面代码中的`go:noinline`用于告诉编译器禁止对该函数进行内联优化。

运行：

```sh
$ go build -gcflags="-m -m" .
```

查看打印结果：

```
# just-for-fun/escape
.\main.go:8:6: can inline (*Dog).Eat as: method(*Dog) func() {  }
.\main.go:30:6: cannot inline noneEscape: marked go:noinline
.\main.go:31:7: inlining call to (*Dog).Eat method(*Dog) func() {  }
.\main.go:26:6: cannot inline nop: marked go:noinline
.\main.go:35:6: cannot inline escape: marked go:noinline
.\main.go:15:6: cannot inline main: function too complex: cost 355 exceeds budget 80
.\main.go:8:7: (*Dog).Eat d does not escape
.\main.go:30:17: noneEscape d does not escape
.\main.go:26:11: nop a does not escape
.\main.go:35:13: leaking param: a
.\main.go:35:13:        from a.Eat() (receiver in indirect call) at .\main.go:36:7
.\main.go:20:8: dog2 escapes to heap
.\main.go:20:8:         from dog2 (passed to call[argument escapes]) at .\main.go:20:8
.\main.go:19:13: new(Dog) escapes to heap
.\main.go:19:13:        from dog2 (assigned) at .\main.go:19:7
.\main.go:19:13:        from dog2 (interface-converted) at .\main.go:20:8
.\main.go:19:13:        from dog2 (passed to call[argument escapes]) at .\main.go:20:8
.\main.go:22:14: &dog3 escapes to heap
.\main.go:22:14:        from ... argument (arg to ...) at .\main.go:22:13
.\main.go:22:14:        from *(... argument) (indirection) at .\main.go:22:13
.\main.go:22:14:        from ... argument (passed to call[argument content escapes]) at .\main.go:22:13
.\main.go:22:14: &dog3 escapes to heap
.\main.go:22:14:        from &dog3 (interface-converted) at .\main.go:22:14
.\main.go:22:14:        from ... argument (arg to ...) at .\main.go:22:13
.\main.go:22:14:        from *(... argument) (indirection) at .\main.go:22:13
.\main.go:22:14:        from ... argument (passed to call[argument content escapes]) at .\main.go:22:13
.\main.go:21:2: moved to heap: dog3
.\main.go:16:13: main new(Dog) does not escape
.\main.go:18:5: main dog1 does not escape
.\main.go:22:13: main ... argument does not escape
<autogenerated>:1: leaking param: .this
<autogenerated>:1:      from .this.Eat() (receiver in indirect call) at <autogenerated>:1
```

从上面的信息中可以看到，`dog1`分配在栈上，而`dog2`和`dog3`都分配在堆上。

`dog1`在方法`noneEscape`中，调用了`Eat`方法，因为`Eat`方法并没有发生内存逃逸，因此`dog1`在`noneEscape`中没有内存逃逸。而`nop`方法内`dog1`没有执行任何操作，也不会发生内存逃逸。可见，即使是使用`new`分配的变量，也不一定是分配在堆上。

而`dog2`在方法`escape`中，调用了`Eat`方法，因为这时候`dog2`是`Animal`接口类型，`golang`中接口类型的方法是动态派发的，编译器并不知道具体调用的是哪个`Eat`方法，从而无法确定`dog2`在`Eat`是否有发生内存逃逸。在这种情况下，编译器会认为`dog2`发生了内存逃逸，并将其分配在堆上。如果编译器能够在编译时就对接口的实际类型进行分析，对`Eat`方法进行静态派发，就可以发现`dog2`并没有内存逃逸。



### fun thing

代码1：

```go
func main() {
   byts := []byte("")
   s1 := append(byts,'a')
   s2:= append(byts,'b')
   fmt.Println(string(s1),string(s2)) // b b
}
```

代码2：

```go
func main() {
   byts := []byte("")
   s1 := append(byts,'a')
   s2:= append(byts,'b')
   fmt.Println(byts)
   fmt.Println(string(s1),string(s2)) // a b
}
```

观察上面两个程序，只是添加了一行代码，两次执行结构却完全不同。导致结果完全不同的原因在于，`fmt.Println(byts)`导致`byts`逃脱到堆上。

查看两份代码编译后的`[]byte("")`对应的汇编代码片段：

代码1：

```asm
0x0036 00054 (main.go:6)	LEAQ	""..autotmp_6+120(SP), AX
0x003b 00059 (main.go:6)	PCDATA	$2, $0
0x003b 00059 (main.go:6)	MOVQ	AX, (SP)
0x003f 00063 (main.go:6)	XORPS	X0, X0
0x0042 00066 (main.go:6)	MOVUPS	X0, 8(SP)
0x0047 00071 (main.go:6)	CALL	runtime.stringtoslicebyte(SB)
```

代码2：

```asm
0x0036 00054 (main.go:6)	MOVQ	$0, (SP)
0x003e 00062 (main.go:6)	XORPS	X0, X0
0x0041 00065 (main.go:6)	MOVUPS	X0, 8(SP)
0x0046 00070 (main.go:6)	CALL	runtime.stringtoslicebyte(SB)
```

可以看到，两次调用`runtime.stringtoslicebyte`时传递的参数不同，第一次的第一个参数是非空的，而第二次的第一次参数的空的，查看该函数实现：

```go
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
    // 如果buf不为空，并且len(buf)小于len(s)，会直接使用buf作为底层数组，而buf的长度为32
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}

// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
	cap := roundupsize(uintptr(size))
	p := mallocgc(cap, nil, false)
	if cap != uintptr(size) {
		memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
	}

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
	return
}
```

通过上面的代码，我们可以得知，代码1中的`byts`，`cap`为`32`，而代码2中的`byts`，`cap`为0。

因为在代码1中，`cap`为`32`，则两次`append`都不会对底层数组发送重分配，而且都是修改第一个元素，因此第二次操作会覆盖第一次操作。而在代码2中，每次`append`都会重新分配一个底层数组。



