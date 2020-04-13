---
title: go中timer的陷阱
date: 2020-04-13 22:53:57
tags:
    - go
---
timer 在 Stop 的时候，如果已经 fire 了，可能需要根据需求清除 timer.C
```go
func TestTimer(t *testing.T) {
    timer := time.NewTimer(time.Second)
    time.Sleep(time.Second * 2)
    if !timer.Stop() {
        // stop的时候已经fire了，但是没有清除timer.C
        //<-timer.C
    }
    // Reset不会重置timer.C
    timer.Reset(time.Second * 10)
    // 这里会立即返回之前fire的time
    t.Log(time.Now())
    t.Log(<-timer.C)
}

==output:
2020-04-09 16:17:56.7184971 +0800 CST m=+2.012515401
2020-04-09 16:17:55.7171205 +0800 CST m=+1.011138801
```
从上面的输出中，我们可以看到第二次打印的时间早于第一次打印的时间，说明`timer.Reset`并不会重置`timer.C`。

当我们 Stop 一个 timer ，并且在后续还需要对其进行重用的时候，应该：
```go
if !timer.Stop() {
    <- timer.C
}
```
注意，要确保是在同一个协程，因为如果是多协程的情况，因为存在竞态条件，会导致阻塞，从而导致协程泄露。