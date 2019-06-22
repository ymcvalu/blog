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
	Iface interface{}
}

func main() {
	m := map[Key]bool{}
	m[Key{a: 10, Iface: 10}] = true // 运行通过
	m[Key{a: 10, Iface: map[string]interface{}{}}] = true // panic
}
```

运行上面的代码，我们会得到一个panic：

```sh
panic: runtime error: hash of unhashable type map[string]interface {}
```

原因是因为，当使用map存储时，`key`要求是可以计算`hash`值的，而在`go`中，`map`、`slice`、`channel`和`func`类型都是不能计算hash值的，因此当使用这些类型作为`map`的`key`时会发生panic，而当key是复合类型，并且包含了这四种类型，也会发生panic，如上面的例子中，如果包含了接口类型，那么在运行时会动态检查其类型。

### 具体实现

接下来，我们来看一下其背后的实现逻辑。

首先，我们先看一下map中关于hash的计算逻辑：

```go
alg := t.key.alg // 这里的key是maptype
hash := alg.hash(key, uintptr(h.hash0)) 
```

接下来看一下`maptype`的结构定义：

```go
type maptype struct {
	typ        _type
	key        *_type
	elem       *_type
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

实际上，每个类型都会包含`_type`，而`_type`中的`alg`字段则定义了该类型对象的`hash`和`equal`方法

而在`runtime`包中，已经声明了一些基本的`alg`：

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
	alg_INTER:    {interhash, interequal}, // 用于计算iface类型
	alg_NILINTER: {nilinterhash, nilinterequal}, // 用于计算eface类型
	alg_FLOAT32:  {f32hash, f32equal},
	alg_FLOAT64:  {f64hash, f64equal},
	alg_CPLX64:   {c64hash, c64equal},
	alg_CPLX128:  {c128hash, c128equal},
}
```

我们就看其中的`nilinterhash`的实现：

```go
func nilinterhash(p unsafe.Pointer, h uintptr) uintptr {
	a := (*eface)(p) // p实际上就是一个空接口指针
	t := a._type 
	if t == nil { 
		return h
	}
	fn := t.alg.hash 
	if fn == nil { // 如果没有指定hash方法，表明该类型不支持计算hash
        // 这里的panic不就跟上面例子中的panic一样嘛
		panic(errorString("hash of unhashable type " + t.string()))
	}
	if isDirectIface(t) {
		return c1 * fn(unsafe.Pointer(&a.data), h^c0)
	} else {
		return c1 * fn(a.data, h^c0)
	}
}
```

接下来，我们看一下文章开头例子反编译后的汇编代码，我们主要是要看一下`Key`类型的`hash`方法

首先，看一下`Key`类型的`type`：

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

接着来看一下汇编生成的`type..hash."".Key`方法，只保留主要逻辑

```assembly
// 首先该函数有两个参数p和h，p表示要hash的对象指针，h表示hash seek，返回一个hash值
TEXT	type..hash."".Key(SB), DUPOK|ABIInternal, $40-24
	MOVQ	"".p+48(SP), AX // 将要hash的镀锡指针mov到ax
	MOVQ	AX, (SP) // memhash第一个参数
	MOVQ	"".h+56(SP), CX
	MOVQ	CX, 8(SP) // memhash第二个参数
	MOVQ	$8, 16(SP) // memhash第三个参数，这里8表示只对前8个字节进行hash计算
	CALL	runtime.memhash(SB) // 调用memhash计算hash值
	MOVQ	24(SP), AX // 将返回值保存到ax中
	MOVQ	"".p+48(SP), CX
	ADDQ	$8, CX // 这里指针计算，p+8对应的就是Key.Iface
	MOVQ	CX, (SP) // nilinterhash第一个参数
	MOVQ	AX, 8(SP) // 将memhash的第一个参数作为nilinterhash第二个参数
	CALL	runtime.nilinterhash(SB) // 调用nilinterhash
	MOVQ	16(SP), AX // 返回值mov到ax
	MOVQ	AX, "".~r2+64(SP) // 设置返回值
	RET
```

直接看上面的逻辑，当计算`Key`的`hash`时，会先计算前8个字节的hash值，然后在调用`nilinterhash`来计算`Iface`的hash值。

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

