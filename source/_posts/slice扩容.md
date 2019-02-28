---
title: slice扩容
date: 2019-02-26 14:00:47
tags:
	- go
---

### slice header 

go中的`slice`声明如下：

```go
type slice struct {
	array unsafe.Pointer // 指向底层数组
	len   int // 长度，当前存储的元素个数
	cap   int // 容量，底层数组的长度
}
```

### grow

`slice`中的`len`表示当前切片中存在的元素个数，而`cap`表示切边底层数组总共可以存放的元素个数，

当我们使用`append`函数为切片追加元素时，如果底层数组剩余容量`cap-len`不足以容纳新的元素，则会发生扩容，具体的扩容逻辑如下：

```go 
// @params et: slice元素类型
// @params old: 老的slice
// @params cap: 期待的最小cap值，这里的cap等于(old.len + append的元素个数)
// @return: 新的slice，并且拷贝老的数据到新的slice
func growslice(et *_type, old slice, cap int) slice {
    // 如果元素不需要存储空间，比如类型struct{}
	if et.size == 0 {
		if cap < old.cap {
			panic(errorString("growslice: cap out of range"))
		}
        // 直接创建一个新的slice，不需要内存分配
        // 这里zerobase是一个值为0的uintptr
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}
	
	newcap := old.cap
	doublecap := newcap + newcap
	// 如果x2不能满足，则使用期待值
    if cap > doublecap {
		newcap = cap
	} else {
        // 否则，如果元素个数小于1024，直接x2
		if old.len < 1024 {
			newcap = doublecap
		} else {
            // 持续1.25倍直到满足
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// 如果溢出了，直接使用期待值
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
    // 原来元素占用内存大小，现在元素占用内存大小，新的底层数组容量大小
	var lenmem, newlenmem, capmem uintptr
	// 计算上面声明的变量值，这里根据et.size进行优化
	switch {
	case et.size == 1: // 不需要乘除法
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap)) 
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize: // 会别优化成移位运算
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size): // 位运算
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem = roundupsize(uintptr(newcap) * et.size)
		overflow = uintptr(newcap) > maxSliceCap(et.size)
		newcap = int(capmem / et.size)
	}

	// The check of overflow (uintptr(newcap) > maxSliceCap(et.size))
	// in addition to capmem > _MaxMem is needed to prevent an overflow
	// which can be used to trigger a segfault on 32bit architectures
	// with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if cap < old.cap || overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
    // 切片元素内不包含指针
	if et.kind&kindNoPointers != 0 {
        // 分配新的底层数组，这里false指示不需要内存清零
		p = mallocgc(capmem, nil, false)
        // 不包含指针，内存拷贝
		memmove(p, old.array, lenmem)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
        // 新数组中未被使用的内存清零
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
        // 因为元素中包含指针，垃圾收集器需要跟踪指针，因此分配内存时需要在位图中标记指针的位置
		p = mallocgc(capmem, et, true)
		// 没有开启写屏障，直接拷贝内存
        if !writeBarrier.enabled {
			memmove(p, old.array, lenmem)
		} else { // gc中，开启了写屏障
			for i := uintptr(0); i < lenmem; i += et.size {
				typedmemmove(et, add(p, i), add(old.array, i))
			}
		}
	}
	// 返回新的slice
	return slice{p, old.len, newcap}
}

```



### turn string to []byte

当执行强制类型转换，将`string`类型转换成`[]byte`时，会执行`stringtoslicebyte`：

```go
// The constant is known to the compiler.
// There is no fundamental theory behind this number.
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte
// 这里的tmpBuf是一个长度为32的数组
func stringtoslicebyte(buf *tmpBuf, s string) []byte {
   var b []byte
   // 如果buf不为空并且字符串长度小于32，直接使用buf
   if buf != nil && len(s) <= len(buf) { 
      *buf = tmpBuf{} // 清零
      b = buf[:len(s)]
   } else {
      b = rawbyteslice(len(s))
   }
   copy(b, s)
   return b
}
```

根据上面的逻辑，当对长度小于32的小字符串进行强制类型转换时，会返回一个`cap`为32的`slice`