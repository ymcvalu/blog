---
title: go的for range
date: 2020-02-20 20:09:03
tags: go
---
# go中的for-range

`for range`是我们在平常写代码中经常用到的，今天就从汇编的角度来看下其实现，以及背后存在的一些坑。

### 切片

```go
func sum(slice []int) int {
	sum := 0
	for _, k := range slice {
		sum += k
	}
	return sum
}
```

执行`go tool compile -S main.go `查看对应汇编：

```assembly
"".sum STEXT nosplit size=37 args=0x20 locals=0x0
	0x0000 00000 (range.go:10)	TEXT	"".sum(SB), NOSPLIT|ABIInternal, $0-32
	0x0000 00000 (range.go:12)	MOVQ	"".slice+8(SP), AX # 将slice的底层数组指针存储到AX
	0x0005 00005 (range.go:12)	MOVQ	"".slice+16(SP), CX # slice的len存储到CX
	0x000a 00010 (range.go:12)	XORL	DX, DX # i:= 0
	0x000c 00012 (range.go:12)	XORL	BX, BX # sum:= 0
	0x000e 00014 (range.go:12)	JMP	26
	0x0010 00016 (range.go:12)	MOVQ	(AX)(DX*8), SI # 读取第i个元素的值到SI
	0x0014 00020 (range.go:12)	INCQ	DX  # i++
	0x0017 00023 (range.go:13)	ADDQ	SI, BX # sum+= SI
	0x001a 00026 (range.go:12)	CMPQ	DX, CX # i < len(slice)
	0x001d 00029 (range.go:12)	JLT	16
	0x001f 00031 (range.go:15)	MOVQ	BX, "".~r1+32(SP) # 设置返回值
	0x0024 00036 (range.go:15)	RET
```

可以看到，在循环之前，就已经将切片的长度，和底层数组的指针备份下来了，因此就算我们在遍历过程中不断append，循环也会结束；而且我们可以看到，在这个例子中，我们在代码中用到了变量`k`，但是在生成的汇编中这个变量已经被优化掉了。



接下来看一下，一个在`for-range`的时候很容易遇到的坑：

```go
func toPtrSlice(slice []int) []*int {
	ns := []*int{} 
	for _, k := range slice {
		ns = append(ns, &k)
	}
	return ns
}
```

在前一个例子中，我们代码中的`k`实际上在生成的汇编中并没有出现，我们现在来看一下现在这个例子：

```assembly
"".toPtrSlice STEXT size=288 args=0x30 locals=0x60
	0x0000 00000 (range.go:10)	TEXT	"".sum(SB), ABIInternal, $96-48
	0x0028 00040 (range.go:12)	LEAQ	type.int(SB), AX
	0x002f 00047 (range.go:12)	MOVQ	AX, (SP)
	0x0033 00051 (range.go:12)	CALL	runtime.newobject(SB) # new(int)
	0x0038 00056 (range.go:12)	MOVQ	8(SP), AX # 返回值保存在AX，实际是*int指针
	0x003d 00061 (range.go:12)	MOVQ	AX, "".&k+80(SP) # k是创建在堆上面的，因为逃逸了
	0x0042 00066 (range.go:12)	MOVQ	"".slice+112(SP), CX # slice的底层数组地址
	0x0047 00071 (range.go:12)	MOVQ	"".slice+104(SP), DX # slice的长度
	0x004c 00076 (range.go:12)	XORL	BX, BX # 循环i
	0x004e 00078 (range.go:11)	LEAQ	runtime.zerobase(SB), SI # ns的底层数组SI，一开始cap为0，底层数组初始化为runtime.zerobase
	0x0055 00085 (range.go:11)	XORL	DI, DI # DI用于存储当前要写到ns的第几个元素
	0x0057 00087 (range.go:11)	XORL	R8, R8 # R8存储ns的cap，一开始ns的cap是0
	0x005a 00090 (range.go:12)	JMP	98 # 首先跳转到98行
	0x005c 00092 (range.go:12)	INCQ	BX # i++
	0x005f 00095 (range.go:15)	MOVQ	R9, DI # R9为ns当前的len，也就是下一次append的位置
								# 这里是98行，也是循环开始的地方
	0x0062 00098 (range.go:12)	CMPQ	BX, CX # i<len(slice)
	0x0065 00101 (range.go:12)	JGE	244        # 如果false，跳到244
	0x006b 00107 (range.go:12)	MOVQ	(DX)(BX*8), R9 # 把切片第i个元素存到R9
	0x006f 00111 (range.go:12)	MOVQ	R9, (AX) # 等价于：k=slice[i]，这里的AX实际存的是地址，因此使用(AX)
	0x0072 00114 (range.go:13)	LEAQ	1(DI), R9 # R9存储DI+1，也就是append该元素后ns的len
	0x0076 00118 (range.go:13)	CMPQ	R9, R8  # 判断ns是否有足够cap可以append
	0x0079 00121 (range.go:13)	JHI	152 # 不足，跳转过去执行扩容操作
	0x007b 00123 (range.go:13)	LEAQ	(SI)(DI*8), R10 # 这个R10没有用到，不知道这行有什么用
	0x0088 00136 (range.go:13)	MOVQ	AX, (SI)(DI*8) # 把AX存到ns[i]，这里AX实际上就是k的地址
	0x008c 00140 (range.go:13)	JMP	92 # 跳转到92
 	0x0098 00152 (range.go:12)	MOVQ	BX, ""..autotmp_14+72(SP)
	0x009d 00157 (range.go:15)	MOVQ	DI, "".ns.len+64(SP) # DI为ns当前的len
	0x00a2 00162 (range.go:13)	LEAQ	type.*int(SB), AX
	0x00a9 00169 (range.go:13)	MOVQ	AX, (SP) # *_type
	0x00ad 00173 (range.go:13)	MOVQ	SI, 8(SP) # old slice data ptr
	0x00b2 00178 (range.go:13)	MOVQ	DI, 16(SP) # old slice len
	0x00b7 00183 (range.go:13)	MOVQ	R8, 24(SP) # old slice cap
	0x00bc 00188 (range.go:13)	MOVQ	R9, 32(SP) # new cap，实际上就是len+1
	0x00c1 00193 (range.go:13)	CALL	runtime.growslice(SB) # 扩容
	0x00c6 00198 (range.go:13)	MOVQ	40(SP), SI # 扩容后的 data ptr
	0x00cb 00203 (range.go:13)	MOVQ	48(SP), AX # 扩容后的len
	0x00d0 00208 (range.go:13)	MOVQ	56(SP), R8 # 扩容后的 cap
									   # 下面这些指令是，扩容后把一些变量再存到寄存器中
	0x00d5 00213 (range.go:13)	LEAQ	1(AX), R9  # R9 = len+1
	0x00d9 00217 (range.go:13)	MOVQ	"".&k+80(SP), AX # 把k的地址存到AX
	0x00de 00222 (range.go:12)	MOVQ	"".slice+112(SP), CX
	0x00e3 00227 (range.go:12)	MOVQ	"".slice+104(SP), DX
	0x00e8 00232 (range.go:12)	MOVQ	""..autotmp_14+72(SP), BX
	0x00ed 00237 (range.go:13)	MOVQ	"".ns.len+64(SP), DI 
	0x00f2 00242 (range.go:13)	JMP	123
	0x00f4 00244 (range.go:15)	MOVQ	SI, "".~r1+128(SP)
	0x00fc 00252 (range.go:15)	MOVQ	DI, "".~r1+136(SP)
	0x0104 00260 (range.go:15)	MOVQ	R8, "".~r1+144(SP)
	0x010c 00268 (range.go:15)	MOVQ	88(SP), BP
	0x0111 00273 (range.go:15)	ADDQ	$96, SP
	0x0115 00277 (range.go:15)	RET
```

可以看到生成的汇编很长，主要还涉及到了切片的扩容，只保留了主要代码。

因为变量`k`会逃逸，因此函数一开始，先在堆上面给他分配了空间，并把地址保存到了`AX`上面。

后面每次循环，都是把`AX`的内容`append`到`ns`切片中，也就是最后整个切片实际上都是同一个元素。

也就是说`for _, k :=range xxx {}`中，这个`k`在整个循环过程中实际上都是同一个变量，有很多方法可以解决该问题，其中之一就是：

```go
for _, k:= range slice {
    k := k // 在当前作用域内重新定义变量k，覆盖上层作用域的变量k
    ...
}
```



### 数组

```go
func sum(arr [5]int) int {
	sum := 0
	for _, k := range arr {
		sum += k
	}
	return sum
}
```

```assembly
"".sum STEXT nosplit size=80 args=0x30 locals=0x30
	0x0000 00000 (range.go:10)	TEXT	"".sum(SB), NOSPLIT|ABIInternal, $48-48
	0x0000 00000 (range.go:10)	SUBQ	$48, SP
	0x0004 00004 (range.go:10)	MOVQ	BP, 40(SP)
	0x0009 00009 (range.go:10)	LEAQ	40(SP), BP
	# 会先将整个数组拷贝到一个栈上的临时数组上
	0x000e 00014 (range.go:12)	MOVQ	"".arr+56(SP), AX # 拷贝第一个元素
	0x0013 00019 (range.go:12)	MOVQ	AX, ""..autotmp_4(SP)
	0x0017 00023 (range.go:12)	MOVUPS	"".arr+64(SP), X0  # 拷贝第2，3个元素，一次拷贝16字节
	0x001c 00028 (range.go:12)	MOVUPS	X0, ""..autotmp_4+8(SP)
	0x0021 00033 (range.go:12)	MOVUPS	"".arr+80(SP), X0 # 拷贝第4，5个元素
	0x0026 00038 (range.go:12)	MOVUPS	X0, ""..autotmp_4+24(SP)
	0x002b 00043 (range.go:12)	XORL	AX, AX # i:=0
	0x002d 00045 (range.go:12)	XORL	CX, CX # 代码中的sum
	0x002f 00047 (range.go:12)	JMP	59 # 跳转到执行CMPQ
	0x0031 00049 (range.go:12)	MOVQ	""..autotmp_4(SP)(AX*8), DX # 读取第i个数
	0x0035 00053 (range.go:12)	INCQ	AX # i++
	0x0038 00056 (range.go:13)	ADDQ	DX, CX # sum累加
	0x003b 00059 (range.go:12)	CMPQ	AX, $5 # 5是数组长度
	0x003f 00063 (range.go:12)	JLT	49
	0x0041 00065 (range.go:15)	MOVQ	CX, "".~r1+96(SP)
	0x0046 00070 (range.go:15)	MOVQ	40(SP), BP
	0x004b 00075 (range.go:15)	ADDQ	$48, SP
	0x004f 00079 (range.go:15)	RET
```

可以看到生成的汇编中，会先把整个数组拷贝一份，因此如果数组很长，应该使用切片操作，将其转变成切片：

```go
for _, k := range arr[:] {}
```

或者：

```go
for _, k := range &arr {} // go支持对数组的指针遍历，但是切片不行
```

```assembly
"".sum STEXT nosplit size=29 args=0x30 locals=0x0
	0x0000 00000 (range.go:10)	TEXT	"".sum(SB), NOSPLIT|ABIInternal, $0-48
	0x0000 00000 (range.go:10)	XORL	AX, AX # 循环i
	0x0002 00002 (range.go:10)	XORL	CX, CX # sum
	0x0004 00004 (range.go:12)	JMP	17
	0x0006 00006 (range.go:12)	MOVQ	"".arr+8(SP)(AX*8), DX # movQ arr[i], DX
	0x000b 00011 (range.go:12)	INCQ	AX # i++
	0x000e 00014 (range.go:13)	ADDQ	DX, CX # sum+=DX
	0x0011 00017 (range.go:12)	CMPQ	AX, $5 # i<5
	0x0015 00021 (range.go:12)	JLT	6
	0x0017 00023 (range.go:15)	MOVQ	CX, "".~r1+48(SP)
	0x001c 00028 (range.go:15)	RET
```



### map

```go
func sum(m map[string]int) int {
	sum := 0
	for _, v := range m {
		sum += v
	}
	return sum
}
```

对应汇编：

```assembly
"".sum STEXT size=215 args=0x10 locals=0x90
	0x0000 00000 (range.go:9)	TEXT	"".sum(SB), ABIInternal, $144-16
	0x0036 00054 (range.go:11)	LEAQ	""..autotmp_5+40(SP), DI
	0x003b 00059 (range.go:11)	XORPS	X0, X0
	0x003e 00062 (range.go:11)	LEAQ	-32(DI), DI
	0x0042 00066 (range.go:11)	DUFFZERO	$273
	0x0055 00085 (range.go:11)	LEAQ	type.map[string]int(SB), AX
	0x005c 00092 (range.go:11)	MOVQ	AX, (SP)
	0x0060 00096 (range.go:11)	MOVQ	"".m+152(SP), AX
	0x0068 00104 (range.go:11)	MOVQ	AX, 8(SP)
	0x006d 00109 (range.go:11)	LEAQ	""..autotmp_5+40(SP), AX
	0x0072 00114 (range.go:11)	MOVQ	AX, 16(SP)
								# 创建迭代器
	0x0077 00119 (range.go:11)	CALL	runtime.mapiterinit(SB)
	0x007c 00124 (range.go:11)	XORL	AX, AX
	0x007e 00126 (range.go:11)	JMP	173
	0x0080 00128 (range.go:14)	MOVQ	AX, "".sum+32(SP)
	0x0085 00133 (range.go:11)	MOVQ	""..autotmp_5+48(SP), AX
	0x008a 00138 (range.go:11)	MOVQ	(AX), AX
	0x008d 00141 (range.go:11)	MOVQ	AX, "".v+24(SP)
	0x0092 00146 (range.go:11)	LEAQ	""..autotmp_5+40(SP), CX
	0x0097 00151 (range.go:11)	MOVQ	CX, (SP)
								# 获取下一个
	0x009b 00155 (range.go:11)	CALL	runtime.mapiternext(SB)
	0x00a0 00160 (range.go:12)	MOVQ	"".v+24(SP), AX
	0x00a5 00165 (range.go:12)	MOVQ	"".sum+32(SP), CX
	0x00aa 00170 (range.go:12)	ADDQ	CX, AX
	0x00ad 00173 (range.go:11)	CMPQ	""..autotmp_5+40(SP), $0
	0x00b3 00179 (range.go:11)	JNE	128
	0x00b5 00181 (range.go:14)	MOVQ	AX, "".~r1+160(SP)
	0x00bd 00189 (range.go:14)	MOVQ	136(SP), BP
	0x00c5 00197 (range.go:14)	ADDQ	$144, SP
	0x00cc 00204 (range.go:14)	RET
```

可以看到，对应`map`的`for range`，实际是通过其迭代器完成的。如果对map的实现了解的话，应该知道，在遍历map过程中，新加入的元素可能会遍历到，也可能不会。



### channel 

```go
func sum(ch chan int) int {
	sum := 0
	for v := range ch {
		sum += v
	}
	return sum
}
```

对应汇编：

```assembly
"".sum STEXT size=126 args=0x10 locals=0x30
	0x0000 00000 (range.go:9)	TEXT	"".sum(SB), ABIInternal, $48-16
	0x0024 00036 (range.go:9)	XORL	AX, AX
	0x0026 00038 (range.go:11)	JMP	63 # 跳到63行
	0x0028 00040 (range.go:11)	MOVQ	""..autotmp_5+32(SP), CX
	0x002d 00045 (range.go:11)	MOVQ	$0, ""..autotmp_5+32(SP)
	0x0036 00054 (range.go:12)	MOVQ	"".sum+24(SP), DX
	0x003b 00059 (range.go:12)	LEAQ	(DX)(CX*1), AX
	0x003f 00063 (range.go:14)	MOVQ	AX, "".sum+24(SP) 
	0x0044 00068 (range.go:11)	MOVQ	"".ch+56(SP), CX 
	0x0049 00073 (range.go:11)	MOVQ	CX, (SP) 
	0x004d 00077 (range.go:11)	LEAQ	""..autotmp_5+32(SP), DX
	0x0052 00082 (range.go:11)	MOVQ	DX, 8(SP)
					# func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool)
	0x0057 00087 (range.go:11)	CALL	runtime.chanrecv2(SB)
	0x005c 00092 (range.go:11)	CMPB	16(SP), $0 # 如果返回false，说明已经close了
	0x0061 00097 (range.go:11)	JNE	40
	0x0063 00099 (range.go:14)	MOVQ	"".sum+24(SP), AX
	0x0068 00104 (range.go:14)	MOVQ	AX, "".~r1+64(SP)
	0x006d 00109 (range.go:14)	MOVQ	40(SP), BP
	0x0072 00114 (range.go:14)	ADDQ	$48, SP
	0x0076 00118 (range.go:14)	RET
```

就是循环从channel读取，如果channel关闭了就跳出循环。