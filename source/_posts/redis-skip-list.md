---
title: skip list
date: 2020-03-23 21:09:36
tags:
    - redis - skip_list
---

最近业务上有一个任务分配的需求，具体的需求可以简化为：

需要有一个任务队列，队列中每个任务都需要分配给两个不同的用户，并且需要按照FIFO的顺序进行分配，并且只有当用户来请求任务时才分配任务，不能预先分配。

因为一个任务需要分配给两个不同的用户，因此当分配一次之后，不能立即将其从队列中移除，这样就导致了，当一个用户请求时，需要从队列头部进行扫描，直到找到第一个他没有处理过的任务。

为了加快任务的分配，我们可以在每次给用户分配之后，记录当前分配的任务id，下次直接从该id之后进行分配。这时候我就想到了使用redis的zset，因为其底层采用skiplist，我们使用任务id作为score，根据skiplist的特性，可以高效地找到第一个大于某个score的元素。

但是，因为实际业务更复杂一点，并且需要保证分配的原子性，直接使用redis实现的话，需要通过lua脚本，并且开发上也会更复杂一点，因此最终是决定在代码中自行实现skiplist。

redis中的skiplist，与传统的skiplist有一点区别。
传统的skiplist是使用一个专门的key来进行排序的，而redis中的skiplist，是由score和val共同进行排序的，因此当zset中所有的元素score都设置为0之后，就变成按照val的字典序进行排序了，该特性可以用来实现一些需要前缀查询的场景，比如联想输入，并且redis的skiplist还会记录rank，可以更快的查找第i个元素。

很明显，redis的skiplist会更复杂一点，但是功能更加强大，既然要实现skiplist，那么我们肯定是实现功能更强大的redis skiplist了。

> redis中zset实际上是由dict和skiplist两个数据结构一起实现的，因为zset本身是一个集合，需要通过dict来保证唯一性，而使用skiplist来保证有序性，两者结合就变成了有序集合。

### 实现redis版本的skiplist
接下来使用go语言实现一下，redis版本的skiplist，实现逻辑主要是参考redis的源代码，这里只实现几个接口，其他操作，比如range查询，实际上都是大同小异的。

我们先来看skiplist的声明：
```go
type znode struct {
	key      []byte
	score    float64
	backward *znode // 执行前驱节点
	span     []int  // span[i]记录level[i]跨越的节点数量，用于快速查询第i个元素 
	level    []*znode // 记录每层的下一个节点
}

type SkipList struct {
	header *znode 
	tail   *znode 
	length int // 当前skiplist元素个数
	level  int // 记录当前最高level
}

func createNode(level int, score float64, key []byte) *znode {
	return &znode{
		key:   key,
		score: score,
		level: make([]*znode, level),
		span:  make([]int, level),
	}
}

func New() *SkipList {
	return &SkipList{
		header: createNode(ZSKIP_LIST_MAXLEVEL, 0, nil),
		level:  1,
	}
}
```
`znode`代表skiplist中的一个节点，`level`这个切片就是用于记录每一层的下一个节点。

接着我们来看一下插入逻辑：
```go
// caller需要确保key是唯一的
func (sl *SkipList) Insert(score float64, key []byte) {
	if math.IsNaN(score) {
		panic("unsupported: the score is NaN")
	}

	update := make([]*znode, ZSKIP_LIST_MAXLEVEL)
	rank := make([]int, ZSKIP_LIST_MAXLEVEL) // 记录每一层的span
	x := sl.header

    // TODO: 这块代码提取出来做公共逻辑
    // 首先要定位插入的位置
	for i := sl.level - 1; i >= 0; i-- {
		if i < sl.level-1 {
			rank[i] = rank[i+1]
		}

		for x.level[i] != nil &&
			// 首先比较分数
			(x.level[i].score < score ||
				// 分数相同则按照key的字节序排序
				(x.level[i].score == score && bytes.Compare(x.level[i].key, key) < 0)) {
			rank[i] += x.span[i]
			x = x.level[i]
		}
		update[i] = x
	}

    level := randomLevel() // 随机生成level
    // 更新skiplist当前最大的level
	if level > sl.level {
		for i := sl.level; i < level; i++ {
			rank[i] = 0
			update[i] = sl.header
			update[i].span[i] = sl.length
		}
		sl.level = level
	}

    x = createNode(level, score, key)

    // 维护level指针和span
	for i := 0; i < level; i++ {
		x.level[i] = update[i].level[i]
		update[i].level[i] = x
		// rank[0]-rank[i]表示从update[i]到当前节点的span
		x.span[i] = update[i].span[i] - (rank[0] - rank[i])
		update[i].span[i] = (rank[0] - rank[i]) + 1
	}

	for i := level; i < sl.level; i++ {
		update[i].span[i]++
	}

    // 维护backward指针和skiplist的tail指针
	if update[0] != sl.header {
		x.backward = update[0]
	}

	if x.level[0] != nil {
		x.level[0].backward = x
	} else {
		sl.tail = x
    }
    
	sl.length++
}

func randomLevel() int {
	h := 1

	for h < ZSKIP_LIST_MAXLEVEL && rand.Int()&0xFFFF < 0xFFFF>>2 {
		h++
	}

	return h
}
```
跟传统的skiplist实现的差别，主要在于对于`span`切片的维护和`backward`指针的维护了。首先需要定位到要插入的位置，然后执行插入逻辑。

接下来，看一下删除逻辑：
```go
func (sl *SkipList) Delete(score float64, key []byte) bool {
	updates := make([]*znode, ZSKIP_LIST_MAXLEVEL)

	x := sl.header
    // 定位要删除的元素
	for i := sl.level - 1; i >= 0; i-- {
		for x.level[i] != nil &&
			(x.level[i].score < score ||
				(x.level[i].score == score && bytes.Compare(x.level[i].key, key) < 0)) {
			x = x.level[i]
		}
		updates[i] = x
	}

    x = x.level[0]
    // 判断元素是否存在
	if x.score != score || bytes.Compare(x.key, key) != 0 {
		return false
	}

    // 删除节点
	sl.delNode(x, updates)

	return true
}

func (sl *SkipList) delNode(x *znode, updates []*znode) {
    // 首先，更新level和span
	for i := 0; i < sl.level; i++ {
		if updates[i].level[i] == x {
			updates[i].span[i] += x.span[i] - 1
			updates[i].level[i] = x.level[i]
		} else {
			updates[i].span[i]--
		}
	}

    // 更新backward和tail指针
	if x.level[0] != nil {
		x.level[0].backward = x.backward
	} else {
		sl.tail = x.backward
	}

	for sl.level > 1 && sl.header.level[sl.level-1] == nil {
		sl.level--
	}

	sl.length--
}
```
删除的逻辑，一样也是需要先定位到目标元素，可以看到比插入方法还简单。

接着，我们看一下如果借助`span`切片快速获取第i个元素：
```go
//  0 获取第一个元素
// -1 获取最后一个元素
func (sl *SkipList) GetByRank(r int) (float64, []byte, bool) {
    // 如果小于0，表示获取倒数个元素，将其转换为正向获取
	if r < 0 {
		r = int(sl.length) + r
	}

	if r < 0 || r >= sl.length {
		return 0, nil, false
	}

	r += 1 // 0代表第1个元素，1代表第2个元素

	x := sl.header
	for i := sl.level - 1; i >= 0; i-- {
		for x.level[i] != nil && r-x.span[i] >= 0 {
			r -= x.span[i]
			x = x.level[i]
		}
	}

	if r == 0 {
		return x.score, x.key, true
	}

	return 0.0, nil, false
}
```

可以看到，我们实现的skiplist，可以比较快速的查询第i个元素，甚至是快速查找一个`range`。