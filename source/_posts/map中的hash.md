---
title: map中的hash
date: 2019-06-22 09:56:18
tags:
	- go
---

### 问题引入

```go
type Key struct {
	a     int
	eface interface{}
}

func main() {
	m := map[Key]bool{}
	m[Key{a: 10, eface: 10}] = true // 运行通过
	m[Key{a: 10, eface: map[string]interface{}{}}] = true // panic
}
```

运行上面的代码，我们会得到一个panic：

```sh
panic: runtime error: hash of unhashable type map[string]interface {}
```

原因是因为，当使用map存储时，`key`要求是可以计算`hash`值的，而在`go`中，`map`、`slice`、`channel`和`func`类型都是不能计算hash值的，因此当使用这些类型作为`map`的`key`时会发生panic，而当key是复合类型，并且包含了这四种类型，也会发生panic，如上面的例子中，如果包含了接口类型，那么在运行时会动态检查其类型。

### 具体实现

接下来，我们来看一下go是如何给map的key计算hash值的。

首先，我们先看一下map中关于hash的计算逻辑：

```go
alg := t.key.alg // 这里的t是maptype
hash := alg.hash(key, uintptr(h.hash0)) 
```

接下来看一下`maptype`的结构定义：

```go
// maptype表示一个map的类型
type maptype struct {
	typ        _type
	key        *_type // key的类型
	elem       *_type // val的类型
	bucket     *_type // internal type representing a hash bucket
	keysize    uint8  // size of key slot
	valuesize  uint8  // size of value slot
	bucketsize uint16 // size of bucket
	flags      uint32
}

type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldalign uint8
	kind       uint8
	alg        *typeAlg 
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}

type typeAlg struct {
	// function for hashing objects of this type
	// (ptr to object, seed) -> hash
	hash func(unsafe.Pointer, uintptr) uintptr
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
}
```
go中所有的类型对应的**元类型**声明都是：
```go
type xxxtype struct {
	typ  _type
	...
}
```
而`_type`中的`alg`字段，声明了用于计算该类型的hash函数和用于比较的比较函数。

而在`runtime/alg.go`中，列出了一些预先定义的的`alg`：
```go
var algarray = [alg_max]typeAlg{
	alg_NOEQ:     {nil, nil}, // 表示没有hash方法和equal方法 
	alg_MEM0:     {memhash0, memequal0},
	alg_MEM8:     {memhash8, memequal8},
	alg_MEM16:    {memhash16, memequal16},
	alg_MEM32:    {memhash32, memequal32},
	alg_MEM64:    {memhash64, memequal64},
	alg_MEM128:   {memhash128, memequal128},
	alg_STRING:   {strhash, strequal},
	alg_INTER:    {interhash, interequal}, // iface的hash和equal
	alg_NILINTER: {nilinterhash, nilinterequal}, // eface的hash和equal
	alg_FLOAT32:  {f32hash, f32equal},
	alg_FLOAT64:  {f64hash, f64equal},
	alg_CPLX64:   {c64hash, c64equal},
	alg_CPLX128:  {c128hash, c128equal},
}
```

我们就其中几个例子来感受一下：

计算eface的hash：
```go
func nilinterhash(p unsafe.Pointer, h uintptr) uintptr {
	a := (*eface)(p) // p实际上就是一个空接口指针
	t := a._type 
	if t == nil { 
		return h
	}
	// 使用实际类型的hash方法来计算
	fn := t.alg.hash 
	// 如果没有hash方法，表明该类型不支持计算hash，比如slice、map或者func
	if fn == nil { 
        // 这里的panic不就跟上面例子中的panic一样嘛
		panic(errorString("hash of unhashable type " + t.string()))
	}

	// isDirectIface如果返回true，表示这个接口存的实际是一个指针值
	if isDirectIface(t) {
		// v:= &struct{...}
		// i:=(interface{})(v)
		// 这时候是这种情况，使用指针v的值计算hash
		return c1 * fn(unsafe.Pointer(&a.data), h^c0)
	} else {
		// v:= struct{...}
		// i:=(interface{})(v)
		// 这时候是这种情况，对结构体计算hash
		return c1 * fn(a.data, h^c0)
	}
}
```
eface的equal：
```go
func nilinterequal(p, q unsafe.Pointer) bool {
	x := *(*eface)(p)
	y := *(*eface)(q)
	// 首先比较他们的类型是否一致
	return x._type == y._type && efaceeq(x._type, x.data, y.data)
}

func efaceeq(t *_type, x, y unsafe.Pointer) bool {
	if t == nil {
		return true
	}
	// 然后调用实际类型的equal方法
	eq := t.alg.equal
	if eq == nil {
		panic(errorString("comparing uncomparable type " + t.string()))
	}
	
	if isDirectIface(t) {
		// Direct interface types are ptr, chan, map, func, and single-element structs/arrays thereof.
		// Maps and funcs are not comparable, so they can't reach here.
		// Ptrs, chans, and single-element items can be compared directly using ==.
		// 我们代码中得到的chan实际上就是一个ptr，chan的比较实际上就是比较地址
		return x == y
	}
	return eq(x, y)
}
```
字符串的hash
```go
func strhash(a unsafe.Pointer, h uintptr) uintptr {
	x := (*stringStruct)(a)
	// str指向底层字节数组，使用底层字节数组的内容计算hash
	return memhash(x.str, h, uintptr(x.len))
}
```

接下来，我们看一下文章开头例子反编译后的汇编代码，我们主要是要看一下`Key`类型的`hash`方法

首先，看一下`Key`的类型元信息：

```go
type."".Key SRODATA size=144
	0x0000 18 00 00 00 00 00 00 00 18 00 00 00 00 00 00 00  ................
	0x0010 18 cb 58 54 07 08 08 19 00 00 00 00 00 00 00 00  ..XT............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 02 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 00 00 00 00 40 00 00 00 00 00 00 00  ........@.......
	0x0060 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0070 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0080 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	rel 24+8 t=1 type..alg."".Key+0 // 24+8偏移量对应的就是_type.alg
	rel 32+8 t=1 runtime.gcbits.04+0
	rel 40+4 t=5 type..namedata.*main.Key.+0
	rel 44+4 t=5 type.*"".Key+0
	rel 48+8 t=1 type..importpath."".+0
	rel 56+8 t=1 type."".Key+96
	rel 80+4 t=5 type..importpath."".+0
	rel 96+8 t=1 type..namedata.a-+0
	rel 104+8 t=1 type.int+0
	rel 120+8 t=1 type..namedata.Iface.+0
	rel 128+8 t=1 type.interface {}+0

// key的alg包含了两个字段，hashfunc和eqfunc
type..alg."".Key SRODATA dupok size=16
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	rel 0+8 t=1 type..hashfunc."".Key+0 
	rel 8+8 t=1 type..eqfunc."".Key+0

type..hashfunc."".Key SRODATA dupok size=8
	0x0000 00 00 00 00 00 00 00 00                          ........
	rel 0+8 t=1 type..hash."".Key+0
```

接着来看一下汇编生成的`type..hash."".Key`方法，这里只保留主要逻辑

我们首先回顾一下该struct的声明：
```go
type Key struct {
	a     int
	eface interface{}
}
```
对应hash方法的代码：
```assembly
// 首先该函数有两个参数p和h，p表示要hash的对象指针，h表示hash seek，返回一个hash值
TEXT	type..hash."".Key(SB), DUPOK|ABIInternal, $40-24
	MOVQ	"".p+48(SP), AX // 调用hash时传入的指针p，*Key类型
	MOVQ	AX, (SP) // memhash第一个参数，实际上是 Key.a 的地址
	MOVQ	"".h+56(SP), CX
	MOVQ	CX, 8(SP) // memhash第二个参数，调用hash时传入的seed
	MOVQ	$8, 16(SP) // memhash第三个参数，因为字段a是int类型，在64位平台上占用8个字节
	CALL	runtime.memhash(SB) // 调用memhash计算hash值
	MOVQ	24(SP), AX // 将返回值保存到ax中
	MOVQ	"".p+48(SP), CX
	ADDQ	$8, CX // 这里指针计算，p+8对应的就是Key.eface
	MOVQ	CX, (SP) // nilinterhash第一个参数
	MOVQ	AX, 8(SP) // 将前面对 a 计算的hash作为seed传入
	CALL	runtime.nilinterhash(SB) // 调用nilinterhash
	MOVQ	16(SP), AX // 返回值mov到ax
	MOVQ	AX, "".~r2+64(SP) // 设置返回值
	RET
```

直接看上面的逻辑，当计算`Key`的`hash`时，首先对`key.a`进行hash，然后将结果作为`nilinterhash`的第二个参数`seed`，计算`eface`的hash值，得到最终的hash值。

那么开头我们的例子中，运行的结果是`panic`，因此说明在`nilinterhash`中，因此对应的type没有hash方法而导致panic，为了验证我们的结论，我们看下`map[string]interface{}{}`对应的`_type`中的`alg`

```go
func main() {
	m := map[string]interface{}{}
	m["a"] = nil
}
```

查看编译后的汇编代码：

```go
type.map[string]interface {} SRODATA dupok size=80
	0x0000 08 00 00 00 00 00 00 00 08 00 00 00 00 00 00 00  ................
	0x0010 86 62 71 0e 02 08 08 35 00 00 00 00 00 00 00 00  .bq....5........
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 00 00 00 00 00 00 00 00 10 10 10 01 0c 00 00 00  ................
	// 看这里，runtime.algarray对应就是runtime.algarray[0]
	// 这个数组上面提到过，algarray[0]对应的hash和eq都是nil
	rel 24+8 t=1 runtime.algarray+0
	rel 32+8 t=1 runtime.gcbits.01+0
	rel 40+4 t=5 type..namedata.*map[string]interface {}-+0
	rel 44+4 t=6 type.*map[string]interface {}+0
	rel 48+8 t=1 type.string+0
	rel 56+8 t=1 type.interface {}+0
	rel 64+8 t=1 type.noalg.map.bucket[string]interface {}+0
```

