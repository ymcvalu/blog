---
title: go自定义类型的序列化过程
date: 2018-08-13 22:22:31
tags:
	- go
---



### 问题引入
当某个struct存在某个字段为string或者[]byte类型但是实际上保存的内容是json格式的数据时，对其进行json序列化，比如
```go
type Message struct {
	From string     `json:"from"`
	To   string     `json:"to"`
	Data string `json:"data"`
}

func main() {
	msg := Message{
		From: "XiaoMing",
		To:   "LiGang",
		Data: `{"title":"test","body":"something"}`,
	}
	jsonData, err := json.Marshal(msg)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(jsonData))
}
```
在上面的例子中，Data字段是string类型，但是保存的内容是json格式的数据，这个时候，程序输出：
```json
{"from":"XiaoMing","to":"LiGang","data":"{\"title\":\"test\",\"body\":\"something\"}"}
```
可以看到，序列化之后的data是一个字符串。
如果Message对应的是数据库中的一张表，而data字段在数据库中是json类型，当我们需要一个接口，查询Message表中的记录返回给客户端。如果直接执行序列化，那么客户端获取到的Data实际上是一个字符串，客户端还需要自行对这个字符串进行json反序列化。
>这时候我们就会想，有没有什么办法能够在服务端序列化Message时，将data字段序列化成json对象而不是字符串呢？

### 自定义序列化
因为data字段的值本身就是json类型，为什么不能在序列化时直接使用呢？
查看json包的官方文档，我们可以发现关于 [自定义序列化](https://godoc.org/encoding/json#ex-package--CustomMarshalJSON)的例子
当执行json序列化时，如果对应的类型实现了`Marshaler`接口：
```go
type Marshaler interface {
	MarshalJSON() ([]byte, error)
}
```
那么就会执行其`MarshalJSON`方法，并将返回的字节数组作为该值的序列化值。
那么回到上面的例子，我们就很容易实现目标：
```go
type JsonString string

func (j JsonString) MarshalJSON() ([]byte, error) {
	fmt.Println("marshal...")
	return []byte(j), nil
}

type Message struct {
	From string     `json:"from"`
	To   string     `json:"to"`
	Data JsonString `json:"data"`
}
```
在上面的代码中基于`string`类型声明了`JsonString`，代表json格式的字符串，并实现了Marshaler接口。因为JsonString代表的就是json字符串，直接将其转换成字节数组返回。
然后将Message中的Data字段换成JsonString类型。
再次执行程序，可以看到：
```json
{"from":"XiaoMing","to":"LiGang","data":{"title":"test","body":"something"}}
```
**Perfect!**