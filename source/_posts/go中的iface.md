---
title: go的接口值
date: 2019-05-11 00:03:35
tags:
	- go
	- interface
---
# iface

### 结构

```go
type iface struct {
	tab  *itab
	data unsafe.Pointer // 这里的data不是直接指向接口背后的实际值，而是指向实际值的拷贝
}

// layout of Itab known to compilers
// allocated in non-garbage-collected memory
// Needs to be in sync with
// ../cmd/compile/internal/gc/reflect.go:/^func.dumptypestructs.
type itab struct {
	inter *interfacetype // 接口类型
	_type *_type // 实际类型
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte // 4字节填充，与上面的4字节hash凑成8字节，与n内存对齐相关
    // itab末尾是实现方法的引用，如果多余1个，则其余方法引用紧跟itab内存之后分配
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}

type interfacetype struct {
	typ     _type // 接口类型信息
	pkgpath name	
	mhdr    []imethod // 接口声明的方法
}


// Needs to be in sync with ../cmd/link/internal/ld/decodesym.go:/^func.commonsize,
// ../cmd/compile/internal/gc/reflect.go:/^func.dcommontype and
// ../reflect/type.go:/^type.rtype.
type _type struct {
	size       uintptr
	ptrdata    uintptr  // size of memory prefix holding all pointers
	hash       uint32   // 类型hash值
	tflag      tflag    // 类型相关一些flag，可以在反射包中使用
	align      uint8    // 内存对齐
	fieldalign uint8    // 字段对齐
	kind       uint8    // 类型kind
	alg        *typeAlg // 类型的hash和equal方法
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff   // offset of name
	ptrToThis typeOff   
}
```

这里的`_type`是最基本的类型信息，而实际我们声明的类型还有包含字段、方法等信息，查看下面代码，可以看到`_type`只是实际类型结构的一部分

```go
type uncommontype struct {
	pkgpath nameOff
	mcount  uint16 // number of methods
	xcount  uint16 // number of exported methods
	moff    uint32 // offset from this uncommontype to [mcount]method
	_       uint32 // unused
}

func (t *_type) uncommon() *uncommontype {
	if t.tflag&tflagUncommon == 0 {
		return nil
	}
	switch t.kind & kindMask {
	case kindStruct:
		type u struct {
			structtype
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	case kindPtr:
		type u struct {
			ptrtype
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	case kindFunc:
		type u struct {
			functype
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	case kindSlice:
		type u struct {
			slicetype
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	case kindArray:
		type u struct {
			arraytype
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	case kindChan:
		type u struct {
			chantype
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	case kindMap:
		type u struct {
			maptype
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	case kindInterface:
		type u struct {
			interfacetype
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	default:
		type u struct {
			_type
			u uncommontype
		}
		return &(*u)(unsafe.Pointer(t)).u
	}
}
```



### 通过汇编看iface

```go
func main() {
	var r io.Reader = Arr{}
	r.Read(nil)
}

type Arr []byte

func (Arr) Read(n []byte) (int, error) {
	return 0, nil
}
```

```sh
$ go tool compile -S main.go > main.s
```

```assembly
	0x0000 00000 (test.go:7)	TEXT	"".main(SB), $88-0
	...
	0x0024 00036 (test.go:8)	LEAQ	type.[0]uint8(SB), AX  // newobject方法参数
	0x002b 00043 (test.go:8)	MOVQ	AX, (SP)
	0x002f 00047 (test.go:8)	CALL	runtime.newobject(SB)
	0x0034 00052 (test.go:8)	MOVQ	8(SP), AX // 这里把返回的指针保存到AX
	0x0039 00057 (test.go:8)	MOVQ	AX, ""..autotmp_1+56(SP) // Arr对象
	0x003e 00062 (test.go:8)	XORPS	X0, X0
	0x0041 00065 (test.go:8)	MOVUPS	X0, ""..autotmp_1+64(SP)
	0x0046 00070 (test.go:8)	LEAQ	go.itab."".Arr,io.Reader(SB), AX // itab
	0x004d 00077 (test.go:8)	MOVQ	AX, (SP)
	0x0051 00081 (test.go:8)	LEAQ	""..autotmp_1+56(SP), AX // Arr对象指针
	0x0056 00086 (test.go:8)	MOVQ	AX, 8(SP)
	0x005b 00091 (test.go:8)	CALL	runtime.convT2Islice(SB) // 生成iface
	0x0060 00096 (test.go:8)	MOVQ	24(SP), AX // 接口的data字段
	0x0065 00101 (test.go:8)	MOVQ	16(SP), CX // 接口的tab字段，即itab表
	0x006a 00106 (test.go:9)	MOVQ	24(CX), CX // itab的24~32为实际方法Read地址
	// 110~127构造一个 Arr{0 0 data}结构，来调用方法Read
	// 实际上方法的接收者就是方法第一个参数
	0x006e 00110 (test.go:9)	MOVQ	$0, 8(SP) // 0
	0x0077 00119 (test.go:9)	XORPS	X0, X0 
	0x007a 00122 (test.go:9)	MOVUPS	X0, 16(SP) // 0
	0x007f 00127 (test.go:9)	MOVQ	AX, (SP) // data
	0x0083 00131 (test.go:9)	CALL	CX // 调用Read方法
	0x008e 00142 (test.go:10)	RET

// 有些itab在编译期间可以自动生成
// 可能在a文件声明了Arr，b文件中声明了接口Reader，在C包文件c和D包文件d都用到了Arr初始化接口Reader，
// 则会在c文件和d文件都生成这个itab表，因此声明为dupok，由链接器任意选择一个
go.itab."".Arr,io.Reader SRODATA dupok size=32
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 d6 ed 1d 4e 00 00 00 00 00 00 00 00 00 00 00 00  ...N............
	rel 0+8 t=1 type.io.Reader+0
	rel 8+8 t=1 type."".Arr+0
	rel 24+8 t=1 "".(*Arr).Read+0 // 这里rel告诉链接器要将24~32的符号引用替换成方法的逻辑地址
```

可以看到，实现接口的方法引用列表会保存在itab末尾，调用时，需要先计算具体调用函数的偏移获取实际方法引用

上面生成接口值的时候，调用了`runtime.convT2Islice`方法，其实在`runtime.iface.go`中声明了一系列`runtime.convT2IXXX`的方法，表示将`XXX`类型的值转换成一个接口值，因为这里的例子中，`Arr`的`Kind`是`slice`，因此调用的是`runtime.convT2Islice`这个方法。

下面我们来看一下`convT2Islice`和`convT2I`这两个方法的实现。

先来看`convT2Islice`：

```go
// 参数elem实际上是一个slice的指针
// 将一个实际类型的值转换成接口值这种情况，itab由编译器自动生成
func convT2Islice(tab *itab, elem unsafe.Pointer) (i iface) {
	t := tab._type //这里的_type就是该接口背后的真实数据类型
	if raceenabled {
		raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2Islice))
	}
	if msanenabled {
		msanread(elem, t.size)
	}
	var x unsafe.Pointer
    // 如果切片的底层数组是nil
	if v := *(*slice)(elem); uintptr(v.array) == 0 {
		x = unsafe.Pointer(&zeroVal[0])
	} else {
		x = mallocgc(t.size, t, true) // 分配一个slice
		*(*slice)(x) = *(*slice)(elem) // 赋值
	}
	i.tab = tab
	i.data = x // 返回的接口值中的data指向的内存是elem的拷贝
	return
}
```

然后是`convT2I`，这个是比较通用的转换方法：

```go
// 这里的elem是要转换成接口值的实际值的指针
// 由实际类型转换成接口类型的情况，itab由编译器自动生成
func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
	t := tab._type 
	if raceenabled {
		raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
	}
	if msanenabled {
		msanread(elem, t.size)
	}
	x := mallocgc(t.size, t, true) // 这里根据*elem的大小分配一块内存
	typedmemmove(t, x, elem) // 内存拷贝
	i.tab = tab
	i.data = x
	return
}
```

比如我们把一个指针值赋给一个接口变量，调用`convT2I`方法时，`elem`就是指针的指针了。**这里我们也可以看到，`iface.data`不是直接指向接口背后的实际值，而是指向其拷贝，因为这个原因，也就很好理解为什么方法接收者是指针的话，值类型就不会实现对应的接口类型了。**



### itab生成

有的接口赋值在编译时可以分析，因此可以直接生成itab表，但是有的是运行时动态赋值的（比如接口断言），因此需要运行时动态生成itab表

##### 接口类型转换

```go
// 接口转换
func convI2I(inter *interfacetype, i iface) (r iface) {
   tab := i.tab
   if tab == nil {
      return
   }
   if tab.inter == inter {
      r.tab = tab
      r.data = i.data
      return
   }
   r.tab = getitab(inter, tab._type, false)
   r.data = i.data
   return
}
```

**将接口值A强制转换成接口B时，需要满足：接口A的方法集包含或者等于接口B的方法集**

接口值之间的类型转换，不会考虑实际类型的方法集，而是简单的对接口的方法集进行判断

```go
var r io.ReadCloser = XXX{}
r.Read(nil)
_ = io.Reader(r) // ok
_ = io.ReadCloser(r) // ok
_ = io.Writer(r) // no
var rc io.Reader = XXX{}
_ = io.ReaderCloser(rc) // no
```



##### 接口类型断言

**接口断言：根据接口值的实际类型，判断是否实现了目标接口**

```go
// 接口断言
func assertI2I(inter *interfacetype, i iface) (r iface) {
   tab := i.tab
   if tab == nil {
      // explicit conversions require non-nil interface value.
      panic(&TypeAssertionError{nil, nil, &inter.typ, ""})
   }
    
   // 如果目标接口类型就是当前接口类型，直接返回
   if tab.inter == inter {
      r.tab = tab
      r.data = i.data
      return
   }
   r.tab = getitab(inter, tab._type, false) // 获取itab，如果失败直接panic
   r.data = i.data
   return
}

func assertI2I2(inter *interfacetype, i iface) (r iface, b bool) {
	tab := i.tab
	if tab == nil {
		return
	}
	if tab.inter != inter {
		tab = getitab(inter, tab._type, true) // true表示容忍失败
		if tab == nil { // 不符合，返回false
			return
		}
	}
	r.tab = tab
	r.data = i.data
	b = true
	return
}

func getitab(inter *interfacetype, typ *_type, canfail bool) *itab {
	if len(inter.mhdr) == 0 {
		throw("internal error - misuse of itab")
	}

	// easy case
	if typ.tflag&tflagUncommon == 0 {
		if canfail {
			return nil
		}
		name := inter.typ.nameOff(inter.mhdr[0].name)
		panic(&TypeAssertionError{nil, typ, &inter.typ, name.name()})
	}

	var m *itab

	// First, look in the existing table to see if we can find the itab we need.
	// This is by far the most common case, so do it without locks.
	// Use atomic to ensure we see any previous writes done by the thread
	// that updates the itabTable field (with atomic.Storep in itabAdd).
    // 先查表是否已经存在需要的itab
	t := (*itabTableType)(atomic.Loadp(unsafe.Pointer(&itabTable)))
	if m = t.find(inter, typ); m != nil {
		goto finish
	}

	// Not found.  Grab the lock and try again.
	lock(&itabLock)
    // 双重锁检查
	if m = itabTable.find(inter, typ); m != nil {
		unlock(&itabLock)
		goto finish
	}

	// Entry doesn't exist yet. Make a new entry & add it.
    // 分配itab内存，itab的内存分配在gc堆之外，不会被垃圾扫描、回收
	m = (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*sys.PtrSize, 0, &memstats.other_sys))
	m.inter = inter
	m._type = typ
	m.init() // 初始化
	itabAdd(m) // 添加到itabTable中，后续直接查表，不需要重新构造
	unlock(&itabLock)
finish:
	if m.fun[0] != 0 { // itab初始化成功
		return m
	}
	if canfail {
		return nil
	}
	// this can only happen if the conversion
	// was already done once using the , ok form
	// and we have a cached negative result.
	// The cached result doesn't record which
	// interface function was missing, so initialize
	// the itab again to get the missing function name.
	panic(&TypeAssertionError{concrete: typ, asserted: &inter.typ, missingMethod: m.init()})
}


// init fills in the m.fun array with all the code pointers for
// the m.inter/m._type pair. If the type does not implement the interface,
// it sets m.fun[0] to 0 and returns the name of an interface function that is missing.
// It is ok to call this multiple times on the same m, even concurrently.
func (m *itab) init() string {
	inter := m.inter
	typ := m._type
	x := typ.uncommon() 

	// both inter and typ have method sorted by name,
	// and interface names are unique,
	// so can iterate over both in lock step;
	// the loop is O(ni+nt) not O(ni*nt).
    // 接口和类型的方法列表是按照名字排序的，因此实际循环时间复杂度是O(ni+nt)
	ni := len(inter.mhdr) // 目标接口方法总数
	nt := int(x.mcount) // 实际类型方法总数
    // 计算实际类型的方法引用列表的偏移
	xmhdr := (*[1 << 16]method)(add(unsafe.Pointer(x), uintptr(x.moff)))[:nt:nt]
	j := 0
imethods:
	for k := 0; k < ni; k++ {
		i := &inter.mhdr[k]
		itype := inter.typ.typeOff(i.ityp) // 目标接口方法类型，与参数和返回值相关
		name := inter.typ.nameOff(i.name)  // 目标接口方法名
		iname := name.name()
		ipkg := name.pkgPath() // 接口的包名
		if ipkg == "" {
			ipkg = inter.pkgpath.name()
		}
		for ; j < nt; j++ {
			t := &xmhdr[j]
			tname := typ.nameOff(t.name) 
            // 如果实际方法类型和方法名与目标方法的一致
			if typ.typeOff(t.mtyp) == itype && tname.name() == iname {
				pkgPath := tname.pkgPath()
				if pkgPath == "" {
					pkgPath = typ.nameOff(x.pkgpath).name()
				}
                // 如果方法是导出的或者包名一致
                // 如果接口有未导出方法，只能在同一个包内被实现，可以用来限制其他包实现该接口
				if tname.isExported() || pkgPath == ipkg {
					if m != nil {
						ifn := typ.textOff(t.ifn) //实际函数入口PC
                        // 保存到itab的方法列表中
						*(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*sys.PtrSize)) = ifn
					}
					continue imethods
				}
			}
		}
		// didn't find method
		m.fun[0] = 0 // 没有找到方法，即目标类型没有实现该方法
		return iname
	}
	m.hash = typ.hash
	return ""
}
```

