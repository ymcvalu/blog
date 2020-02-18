---
title: go map分析
date: 2019-03-30 19:43:26
tags:
	- go
---

`map`其实就是一个`hash table`，今天我们来看一下`go`中`map`的实现，相关代码位于`runtime/map.go`中。

### 结构定义

我们首先来看一下`map`的相关结构定义

```go
type hmap struct {
	count     int // 当前map中存放的元素，我们可以通过内置函数`len`来获取
	flags     uint8
	B         uint8  // 当前map的backet数量为2^B，其中最大能够存放loadFactor * 2^B个元素，当超过这个阈值时就会进行扩容，loadFactor默认为13/2，这个值是全局定义的常量
	noverflow uint16 // 总的overflow的数量
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // bucket数组，长度为2^B，这里bucket实际是bmap
	oldbuckets unsafe.Pointer // 如果发生扩容，旧的buckets就会保存到oldbuckets，在后续的操作中会慢慢迁移到新的buckets中
	nevacuate  uintptr        // 扩容时需要从原来的buckets将数据迁移到新的buckets中，该字段表示小于该数值的buckets当前都已经迁移完成

	extra *mapextra // 如果map中保存的key和value都没有包含指针，那么gc时就不需要对buckets里面的内容进行扫描，但是每个bucket本质上是一个链表，buckets头部保存的是每个bucket链表的头节点，这时候会将每个链表的后续节点保存到该字段内，从而gc时可以对这些后续节点进行扫描，防止被回收
}
```
**`map`的key和value，如果都没有包含指针，那么会对其进行优化，`gc`的时候就不需要去扫描每个键值对了**

上面`extra`字段对应的`mapextra`类型：

```go
type mapextra struct {
	overflow    *[]*bmap // 对应buckets
	oldoverflow *[]*bmap // 对应oldbuckets

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap // 分配bmap时，可能会预先分配一些，当需要时可以直接从这里获取
}
```

上面说的，每个`bucket`实际上是一个`bmap`链表，而`hmap`中的`buckets`是一个`bmap`链表数组，这实际上就是[开散列](https://en.wikipedia.org/wiki/Hash_table#Open_addressing)。和普通的开散列不同的是，**一个`bmap`中保存了8个键值对**，下面来看一下`bmap`的定义：

```go
// A bucket for a Go map.
// bucket实际上是一个bmap
type bmap struct {
	// 这里bucketCnt是一个全局声明的常量，大小为8，也就是限制每个bmap中保存8个键值对
    // 当判断一个key是否在当前bmap中时，会先获取这个key的hash的高8位，然后在tophash中查找是否有匹配的索引，如果有再进一步比较key是否相同，如果当前bmap中没有找到，则到下一个bmap中查找
	tophash [bucketCnt]uint8
	// tophash后面紧跟着保存在当前bmap中的8个key和8个value
    // 因为不同类型的key和value内存大小是不同的，这里需要在运行时根据指针运算来访问
    // 在bmap中是按照 `k1,k2,..,k8,v1,v2,...,v8` 这样来排列的，而不是按照直观上的`k1,v1,k2,v2,...,k8,v8`这样来排列，主要是为了减少内存对齐时额外的内存开销
    // 8个键值对之后还有一个overflow指针，用来链接下一个bmap，正如上面说的，每个bucket实际上是一个bmap链表，这里通过overflow链接的这些bmap被称为`overflow bucket`
}
```
> 因为bmap链接下一个bmap的overflow指针在末尾，而不同类型的key和value的内存大小又不同，因此无法直接获取到下一个bmap的地址，前面说过，当key和value不包含指针的时候，gc时不需要扫描这些键值对；但是又需要扫描这些bmap，因此这种情况下会把这些bmap存到mapextra字段中的`overflow和oldoverflow中，这样gc时直接遍历这两个切片就好了。
那为什么不把overflow放到bmap头部呢？个人觉得可能是为了让内存访问更友好吧，连续的内存访问肯定更加高效。


如上面`bmap`所见，键值对并没有显示声明出来，而是需要在运行时根据指针运算来访问，这里来看一下一个全局声明的常量`dataOffset`，这个常量在后续会经常看到：

```go
const(
    ...
	dataOffset = unsafe.Offsetof(struct {
		b bmap
		v int64
	}{}.v)
    ...
)
```

因为`bmap`实际上只声明了一个`uint8`类型的数组，因此这里`v`字段的偏移量就是实际`bmap`中第一个`key`值的偏移量

这里假设`bmap`的地址是`bptr`，`key`的类型大小是`ksize`，`val`的类型大小是`vsize`，那么访问第`i`个`key`和`val`的地址可以通过下面公式计算（`i`从`0`开始计算）：

```
ith key: bptr + dataOffset + i * ksize
ith val: bptr + dataOffset + bucketCnt * ksize + i * vsize
```

至此，我们对`map`的结构有了一个大体的了解，其键值对的存储结构大致可以用下图来描述：

```
    bucket数组         bmap中的overflow指向下一个bmap 
  实际上是bmap数组    这些由overflow引用的bucket被称作overflow bucket
|-----------------|  |-----------------|
|k1,..,k8,v1,..,v8|->|k1,..,k8,v1,..,v8|-> ...		  
|-----------------|  |-----------------|
|k1,..,k8,v1,..,v8|-> ...
|-----------------|
|k1,..,k8,v1,..,v8|-> ...
|-----------------|
|k1,..,k8,v1,..,v8|-> ...
|-----------------|
```



> 从上面可以看到，一个`bucket`实际上是一个`bmap`，而且`bmap`可以通过`overflow`指针来形成链表，这些通过`overflow`引用的`bmap`被称作`overflow bucket`，而`buckets`中`bmap`的称为`bucket`。`bucket`实际上也可以认为是整条`bmap`链表，比如扩容时的`bucket`迁移，实际上就是迁移整条`bmap`链表。


### map创建

当我们使用`make`创建`map`时，可以指定一个`size`参数，当我们没有指定`size`时，或者`size`在编译时已知是一个不大于`8`的数值时（也就是size是一个常量），会通过`makemap_small`来创建一个`map`：

```go
// makehmap_small implements Go map creation for make(map[k]v) and
// make(map[k]v, hint) when hint is known to be at most bucketCnt
// at compile time and the map needs to be allocated on the heap.
func makemap_small() *hmap {
	h := new(hmap)
	h.hash0 = fastrand() // 初始化hash seed
	return h
}
```

可以看到，通过`makemap_small`创建的`map`，此时的`B`是0，`buckets`是`nil`，在第一次写入的时候才会去创建具体的`buckets`

对应的，当我们指定了一个合适的`size`时，会通过另一个函数来创建`map`：

```go
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
    // 如果小于0或者过大，则设为0
	if hint < 0 || hint > int(maxSliceCap(t.bucket.size)) {
		hint = 0
	}

	// initialize Hmap
    // 编译器可能会进行优化，在栈上创建map，因此h可能不为空
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// find size parameter which will hold the requested # of elements
	B := uint8(0)
    // 根据传入的size来计算B的大小，这里的B是buckets的数量，前文说过，map最多可以容纳 0.65*2^B 个键值对，当超过这个阈值时，会执行扩容
	for overLoadFactor(hint, B) { 
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
        // 创建bucket数组，这里可能会预先分配几个bmap，后续可以直接使用而不需要执行内存分配
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```

当我们创建一个`map`时，如果`map`中要存储的键值对数据可以估计的话，最好在创建的时候指定好`size`，这样可以预先分配好`bucket`的数量。



### map的访问

通常我们对`map`的访问有`get`、`set`、`delete`、和`for range`操作，在分析具体的实现下时，我们要先来了解一下如果定位一个`key`的位置，这里先不涉及在执行扩容时的场景。

我们再回顾一下图：

```
    bucket数组         bmap中的overflow指向下一个bmap 
  实际上是bmap数组    这些由overflow引用的bucket被称作overflow bucket
|-----------------|  |-----------------|
|k1,..,k8,v1,..,v8|->|k1,..,k8,v1,..,v8|-> ...		  
|-----------------|  |-----------------|
|k1,..,k8,v1,..,v8|-> ...
|-----------------|
|k1,..,k8,v1,..,v8|-> ...
|-----------------|
|k1,..,k8,v1,..,v8|-> ...
|-----------------|
```

要定位一个`key`，我们需要先确定这个`key`是落在哪个`bucket`中，也就是哪一条`bmap`链表中，通过计算`key`的哈希值，然后通过`hash%bucket数量`的来定位 ，具体的逻辑如下：

```go
alg := t.key.alg // 这里的t是map的类型描述
hash := alg.hash(key, uintptr(h.hash0)) // 计算hash值
m := bucketMask(h.B) // 这里的m为 1<<B-1
// 这里的hash&m等价于求模操作
b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize))) 
```

当定位到具体的`bucket`后，只需要从头开始遍历该`bmap`链表，因为`bmap`中有个`tophash`数组，保存了每个`key`对应的哈希值的高八位，因此我们不需要挨个对每个`key`进行比较，只需要先比较哈希值的高八位是否相同就行了，如果相同才接着比较`key`是否相等，具体代码如下：

```go
top := tophash(hash) // 取key的hash值的高八位
	// 这里b是一个bmap指针，如果当前bmap没有，则查找下一个bmap
	for ; b != nil; b = b.overflow(t) { 
        // 遍历当前bmap，这里bucketCnt是常量8，限制一个bmap中最多只有8个键值对
		for i := uintptr(0); i < bucketCnt; i++ {
            // 先比较哈希值高八位，如果不相等则continue
			if b.tophash[i] != top {
				continue
			}
            // 计算当前的key的位置
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            // 是否间接存储（存储的是实际key的地址）
			if t.indirectkey {
				k = *((*unsafe.Pointer)(k))
			}
            // 比较key是否相等
			if alg.equal(key, k) {
				......
			}
		}
	}
```

##### get

与`get`相关的方法有多个：

```go
// val := m[key]
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
// val,has := m[key]
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) 
// for-range时，用于获取key和val
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer)
```

这三个方法大同小异，这里只分析第一个方法：

先看该函数的注释，如果`key`不存在时，返回的是对应的`val`类型的空值。

```go
// mapaccess1 returns a pointer to h[key].  Never returns nil, instead
// it will return a reference to the zero object for the value type if
// the key is not in the map.
// NOTE: The returned pointer may keep the whole map live, so don't
// hold onto it for very long.
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 如果我们访问的map是nil或者map中没有存储内容，直接返回零值
	if h == nil || h.count == 0 {
		return unsafe.Pointer(&zeroVal[0])
	}
    // map不是并发安全的，如果存在写操作则panic，因此我们需要在多个goroutine对map进行并发读写时，需要加锁保护
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	alg := t.key.alg
    // 计算key的哈希值
	hash := alg.hash(key, uintptr(h.hash0))
    // 获取buckets数组长度的掩码
	m := bucketMask(h.B)
    // 定位key所在的bucket
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    // 当前是否处于扩容操作，当发生扩容操作时，或重新分配buckets数组，并且需要将旧的buckets内的键值对迁移到新的buckets数组中，这个迁移过程不是一次性完成的而是分批完成的
	if c := h.oldbuckets; c != nil {
       	// 扩容操作有两种：第一种，如果是因为当前的键值对数量已经达到了阈值0.65*2^B，则会触发扩容，这时候新的buckets数组的数量为原来的2倍；第二种是当前键值对数量并没有达到阈值，但是当前存在太多的overflow bucket，可能是因为之前存储了太多的键值对，但是后面又被删除掉了，这时候有的bucket链表会比较长，但是实际上存储的键值对比较稀疏，这样会影响查找效率，这种情况会触发sameSizeGrow，即扩容时buckets数组长度与原来一致
        // 如果是前一种，则原先的buckets数量的掩码是m>>1
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
        // 计算该key落在的oldbuckets中的哪个bucket中
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        // 如果当前bucket还没有迁移，则在该bucket中查询，迁移操作是按照bucket为单位进行的
		if !evacuated(oldb) {
			b = oldb
		}
	}
    // 获取哈希值高八位
	top := tophash(hash)
    // 遍历bmap链表
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
            // 比较哈希值高八位
			if b.tophash[i] != top {
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey {
				k = *((*unsafe.Pointer)(k))
			}
            // 比较key是否相等
			if alg.equal(key, k) {
                // 获取key对应的val，这里的计算公式在前面结构定义一节中已经说明过
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                // 如果这里存储的是val的地址
				if t.indirectvalue {
					v = *((*unsafe.Pointer)(v))
				}
				return v
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}
```

##### set 

`set`操作对应的方法是`mapassign`，这个方法为指定的`key`分配一个用于存放`val`的槽，或者返回已经存在的槽，新的`val`直接写入这个槽中，即完成了`map`的`set`操作。

在写入时，因为可能该`key`已经存在，因此需要先遍历目标`bucket`的`bmap`链表，具体代码如下：

```go
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 不允许写入空map
    if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
    // 如果有并发写操作，直接panic
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	alg := t.key.alg
    // 计算key的哈希值
	hash := alg.hash(key, uintptr(h.hash0))

	// Set hashWriting after calling alg.hash, since alg.hash may panic,
	// in which case we have not actually done a write.
    // 设置写标志
	h.flags |= hashWriting
	// 如果buckets数组为空，则会创建一个长度为1的buckets数组
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
    // 计算目标bucket在buckets数组中的索引
	bucket := hash & bucketMask(h.B)
    // 如果当前正在执行扩容操作，则会执行迁移操作，前面说过扩容时的迁移操作是分批进行的，而迁移是按照bucket为单位进行的，而触发迁移的时机是执行写操作
	if h.growing() {
        // 该函数会执行迁移操作，迁移的目标是bucket%len_of_oldbuckets
		growWork(t, h, bucket) 
	}
    // 计算要写入的bucket
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
    // 计算哈希值高八位
	top := tophash(hash)
	// 在bmap中要插入的索引，如果前文所说，一个bmap最多可以有8个键值对
	var inserti *uint8
    // 要插入key的地址
	var insertk unsafe.Pointer
    // 对应的val的地址
	var val unsafe.Pointer
    // 因为前面已经执行过扩容操作，因此写入只需要对新的bucket进行操作
	for {
        // 遍历当前bmap
		for i := uintptr(0); i < bucketCnt; i++ {
            // 高八位哈希值不相同
			if b.tophash[i] != top {
                // 对第一个遇到的空槽，设置为待定槽，因为可能在后续遍历中发现当前key已经存在，则直接返回对应的val就好了
				if b.tophash[i] == empty && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				}
				continue
			}
            // 这里说明哈希值高八位一致，因此需要比较key是否相等
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey {
				k = *((*unsafe.Pointer)(k))
			}
            // 如果不相等，继续遍历下一个
			if !alg.equal(key, k) {
				continue
			}
			// 如果key已经存在，直接返回对应的val地址就好了
			if t.needkeyupdate {
				typedmemmove(t.key, k, key)
			}
			val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
            // goto大法好
			goto done
		}
        // 当前bmap没有找到对应的key，则继续遍历下一个bmap
		ovf := b.overflow(t)
		if ovf == nil { // 如果当前bucket已经遍历完成，则跳出
			break
		}
		b = ovf
	}
	
    // 首先判断是否需要进行扩容：如果当前没有扩容操作正在执行并且数据达到阈值或者存在太多的bmap，则会触发扩容操作
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        // 触发扩容操作，然后跳到开头重新执行上面流程
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
	// 如果不需要扩容操作，并且没有可以使用的空槽，则需要分配一个新的overflow bucket，这里是分配在链表尾部，当前的b指向的就是整条bmap链表的最后一个节点
	if inserti == nil {
		// all current buckets are full, allocate a new one.
		newb := h.newoverflow(t, b)
        // 插入第一个槽中
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		val = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// store new key/value at insert position
    // 设置key
    // 这里针对是否需要对key或者val进行间接存储的处理
	if t.indirectkey {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectvalue {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(val) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++	// map中元素数据加1

done:
    // 检查写操作标志位
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
    // 清除写操作
	h.flags &^= hashWriting
	if t.indirectvalue {
		val = *((*unsafe.Pointer)(val))
	}
	return val
}
```

##### delete

删除操作流程如下：

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
   if h == nil || h.count == 0 {
      return
   }
    // 设置写操作标志位
   if h.flags&hashWriting != 0 {
      throw("concurrent map writes")
   }

   alg := t.key.alg
   hash := alg.hash(key, uintptr(h.hash0))

   // Set hashWriting after calling alg.hash, since alg.hash may panic,
   // in which case we have not actually done a write (delete).
   h.flags |= hashWriting

   bucket := hash & bucketMask(h.B)
    // 如果正在扩容，先执行迁移操作
   if h.growing() {
      growWork(t, h, bucket)
   }
   b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
   top := tophash(hash)
search:
    // 查找要删除的可以，因为前面已经执行了迁移操作，因此这里只需要在新的bucket中查找即可
   for ; b != nil; b = b.overflow(t) {
      for i := uintptr(0); i < bucketCnt; i++ {
         if b.tophash[i] != top {
            continue
         }
         k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
         k2 := k
         if t.indirectkey {
            k2 = *((*unsafe.Pointer)(k2))
         }
         if !alg.equal(key, k2) {
            continue
         }
         // 执行清除操作
         // Only clear key if there are pointers in it.
         if t.indirectkey {
            *(*unsafe.Pointer)(k) = nil
         } else if t.key.kind&kindNoPointers == 0 {
            memclrHasPointers(k, t.key.size)
         }
         v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
         if t.indirectvalue {
            *(*unsafe.Pointer)(v) = nil
         } else if t.elem.kind&kindNoPointers == 0 {
            memclrHasPointers(v, t.elem.size)
         } else {
            memclrNoHeapPointers(v, t.elem.size)
         }
         b.tophash[i] = empty // 将tophash中的值设为empty，表示空槽
         h.count-- // 数量减一
         break search
      }
   }

   if h.flags&hashWriting == 0 {
      throw("concurrent map writes")
   }
   h.flags &^= hashWriting
}
```

##### for-range

执行`for-range`操作时，需要一个迭代器，我们先来看迭代器的声明：

```go
// A hash iteration structure.
type hiter struct {
	key         unsafe.Pointer // key值，nil表示迭代结束
	value       unsafe.Pointer // val值
	t           *maptype
	h           *hmap
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
	bptr        *bmap          // 当前正在迭代的buckete的指针
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // 记录这次迭代从哪个bucket开始
	offset      uint8          // 记录这次迭代从hmap中的哪个offset开始
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	B           uint8
	i           uint8
	bucket      uintptr
	checkBucket uintptr
}
```

`hiter`的创建：

```go
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	if h == nil || h.count == 0 {
		return
	}

	it.t = t // 记录map类型信息
	it.h = h // 引用map

	// grab snapshot of bucket state
	it.B = h.B // 记录当前的B
	it.buckets = h.buckets // 记录当前的buckets
    // 如果当前map中的键值对没有指针
	if t.bucket.kind&kindNoPointers != 0 {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
    // 这里随机选举一个开始迭代的位置，因此我们对map进行for-range操作，每次输出的序列都是不同的
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
    // 设置开始的bucket和offset
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// 当前迭代的bucket索引
	it.bucket = it.startBucket

	// 设置迭代标志位
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}
	// 执行next操作
	mapiternext(it)
}
```

`mapiternext`方法用于推进迭代器前进到下一个键值对，对应逻辑：

```go

func mapiternext(it *hiter) {
	h := it.h
	if h.flags&hashWriting != 0 {
		throw("concurrent map iteration and map write")
	}
	t := it.t
	bucket := it.bucket
	b := it.bptr // 当前正在遍历的bucket
	i := it.i
	checkBucket := it.checkBucket
	alg := t.key.alg

next:
	if b == nil { // 如果it.bptr还没有初始化，需要根据it.bucket进行初始化，第一次执行或者每次遍历完一个bucket都会清空bptr
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.value = nil
			return
		}
        // 如果正在执行扩容，并且迭代是在扩容之后开始的，这时候如果旧的bucket还没有迁移到新的bucket中，那么需要到旧的bucket中遍历
		if h.growing() && it.B == h.B {
            // 获取当前要迭代的bucket对应的oldbucket
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
            // 如果oldbucket中的元素还没有迁移到新的bucket中
			if !evacuated(b) {
				checkBucket = bucket // oldbucket中的键值对迁移到新的bucket中时，可能会迁移到两个bucket中，比如原来长度是4，hash%4等于3，限制长度是8，hash%8可能为4也可能为7，这里的checkBucket记录当前正在遍历h.buckets数组中的哪个bucket，用于后面的判断
			} else {
                // 已经迁移完成，直接在新的bucket中遍历
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else { // 当前没有扩容操作，直接在新的bucket中遍历
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++ // 下一次遍历的bucket的索引
        // 因为开始迭代的bucket位置是随机的，如果越界了，从第一个bucket开始
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
    // 遍历bmap中键值对
	for ; i < bucketCnt; i++ {
        // 计算开始遍历的偏移位置
		offi := (i + it.offset) & (bucketCnt - 1)
        // 如果空槽，则跳过
		if b.tophash[offi] == empty || b.tophash[offi] == evacuatedEmpty {
			continue
		}
        // 获取key
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey {
			k = *((*unsafe.Pointer)(k))
		}
        // 获取val
		v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.valuesize))
        // 如果正在执行扩容，并且正在遍历的bucket还没有迁移完成，并且扩容是由于键值对达到阈值触发的，这时候扩容后的buckets数组的长度为原来的两倍，原来的一个bucket中的键值对迁移时会迁移到新的两个bucket中，因为目标bucket是通过哈希取模计算的，而这时候bucket数组长度扩大了两倍，比如原理数组长度是4，取模后是3，现在数组长度是8，取模后可能为3也可能为7
		if checkBucket != noCheck && !h.sameSizeGrow() {
            // 如果key==key，正常都是走这个逻辑
			if t.reflexivekey || alg.equal(k, k) { 
				// 如果当前的key不是落在当前的bucket中的，比如现在正在遍历的bucket为3，但是key现在的hash是7，以后迁移时将迁移到索引为7的bucket中，因此这个时候应该跳过这个key
				hash := alg.hash(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else { // key != key的情况，比如math.NaN() != math.NaN()
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			} 
		}
        // 走到这里，可能有一种情况，在迭代开始之后发生了扩容（比如我们在for-range里面插入新的键值对，这时候触发了迁移），这时候迭代的是扩容之前的键值对，即只会迭代当前的oldbuckets里面键值对，这时候如果这些键值对还没有发生迁移或者说key是不可比较的，比如key为math.NAN()，因为key!=key，因此这个key不会被删除或者更新，可以直接返回
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey || alg.equal(k, k)) {
			it.key = k
			if t.indirectvalue {
				v = *((*unsafe.Pointer)(v))
			}
			it.value = v
		} else { // 走到这里说明发生了迁移，使用mapaccessK来获取
			rk, rv := mapaccessK(t, h, k)
			if rk == nil { // 已经被删除了
				continue // key has been deleted
			}
			it.key = rk
			it.value = rv
		}
        // 更新迭代器状态
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
    // 当前bmap遍历完成，遍历下一个bmap，如果已经是bmap链表的最后一个节点，则返回的b为nil，这时候会触发遍历下一个bucket
	b = b.overflow(t)
	i = 0
	goto next
}
```



### 扩容

当写入时，如果当前`bucket`已经满了，则会触发扩容检查，如果当前不处于扩容状态并且满足：

- 当前键值对个数已经达到 `0.65*2^B`
- 当前`overflow bucket`数量达到`1<<(B&15)`，这里的`B`如果大于15，按照15计算

下面看一下开始扩容的逻辑：

```go
func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
	bigger := uint8(1)
    // 如果键值对数量没有达到阈值，则说明是overflow bucket数量过多触发的
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0 // 这里将bigger设置成了0
		// 设置标志位，用于后续区分是哪种情况触发的扩容
		// 如果是overflow bucket数量过多触发的，实际上并不会增加bucket的数量
		// 因此称作sameSizeGrow
		h.flags |= sameSizeGrow
	}

	// 保存原来的buckets到oldbuckets
	oldbuckets := h.buckets
	// 分配新的buckets
	// 如果是键值对达到阈值触发的扩容，bucket数量为原来的2倍，也就是扩展buckets数组
	// 否则维持原来的数量
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
	// 设置更新迭代标志位
	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
    // 更新hmap字段
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}
```

再来看一下`bucket`的迁移操作：

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// 迁移正在使用的bucket对应的oldbucket的数据
	evacuate(t, h, bucket&h.oldbucketmask())

	// 继续对h.nevacuate对应的bucket进行迁移，让迁移尽快完成
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```

从上面可以看到，一次`growwork`最多可以迁移两个`bucket`，这样可以尽早完成扩容之后的迁移

接着看一下`evacuate`的逻辑：

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
    // 计算要进行迁移的bucket
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets() // 获取oldbuckets数组的长度
    // 如果还没有执行过迁移
	if !evacuated(b) {
		// 一个oldbucket中的key可能迁移到两个新的bucket中，这里使用x来表示低位目标bucket，y表示高位目标bucket
		var xy [2]evacDst
		x := &xy[0]
        // 低位目标bucket的索引和当前要迁移的oldbucket的索引一致
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.v = add(x.k, bucketCnt*uintptr(t.keysize))
		// 如果是sameSizeGrow，buckets长度没有变化，则不会有高位目标bucket
		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
            // 计算高位目标bucket
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.v = add(y.k, bucketCnt*uintptr(t.keysize))
		}
		// 遍历bucket对应的bmap链表
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			v := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
				top := b.tophash[i]
				if top == empty { // 空槽，没有键值对
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
                // 不是sameSizeGrow，说明可能会迁移到高位目标bucket
				if !h.sameSizeGrow() {
					hash := t.key.alg.hash(k2, uintptr(h.hash0))
                    // 存在迭代器，并且key!=key(NaNs)，一般不会走这个分支
					if h.flags&iterator != 0 && !t.reflexivekey && !t.key.alg.equal(k2, k2) {
						useY = top & 1
						top = tophash(hash)
					} else {
                        // 比如hash是7，oldbuckets长度是4，现在是8，原来bucket是3，现在应该迁移到7这个bucket，这种情况hash&newbit=newbit    
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}
                // 这里是对全局常量进行检查：evacuatedY=evacuatedX+1
                // evacuatedX标记迁移到低位目标bucket
                // evacuatedY标记迁移到高位目标bucket
				if evacuatedX+1 != evacuatedY {
					throw("bad evacuatedN")
				}
				// 对当前槽进行标记，表示已经完成迁移
				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
                // 根据是否迁移到高位bucket选择目标bucket
				dst := &xy[useY]                 // evacuation destination
				// 如果当前bmap满了，创建新的bmap
				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
                // 执行键值对迁移
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy value
				}
				if t.indirectvalue {
					*(*unsafe.Pointer)(dst.v) = *(*unsafe.Pointer)(v)
				} else {
					typedmemmove(t.elem, dst.v, v)
				}
              	// 更新迁移目标状态
				dst.i++
				// These updates might push these pointers past the end of the
				// key or value arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.v = add(dst.v, uintptr(t.valuesize))
			}
		}
		// Unlink the overflow buckets & clear key/value to help GC.
        // 如果不存在对oldbuckets的迭代器并且键值对中包含指针，在oldbuckets清除当前迁移的bucket，帮助尽快gc掉这些没有用的内存
		if h.flags&oldIterator == 0 && t.bucket.kind&kindNoPointers == 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}
	// 如果当前迁移的bucket等于h.nevacuate则更新h.nevacuate的值
    // 如果所有bucket都已经迁移完成，则消除扩容状态
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
```







