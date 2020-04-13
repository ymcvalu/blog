---
title: go代码测试覆盖率
date: 2020-04-13 23:38:37
tags:
    - go
    - test cover
---
go 工具链自带代码测试覆盖率：
```sh
$ go test -coverprofile=cover.out ./...
ok      github.com/ymcvalu/leakcheck    2.377s  coverage: 87.8% of statements
```
会输出代码测试覆盖率，并且生成 cover.out 文件。

> `...`表示路径通配， `./...`表示当前目录以及所有子目录，子目录的子目录......。可以通过`go help packages`查看


`go test`关于代码测试覆盖率相关的`flag`：
- `-cover`：启用代码覆盖率分析。如果指定了`covermode`, `coverpkg`或`coverprofile`，默认设置`cover`
- `-covermode`：如果设置了`-race`，默认值为`atomic`，否则默认为`set`
    - `set`： does this statement run?
    - `count`：how many times does this statement run?
    - `atomic`：count, but correct in multithreaded tests
- `-coverpkg pattern1,pattern2,pattern3`： Apply coverage analysis in each test to packages matching the patterns. The default is for each test to analyze only the package being tested.
- `-coverprofile`：Write a coverage profile to the file after all tests have passed.

可以使用`go tool cover`来分析`cover profile`。

查看函数的覆盖率：
```sh
$ go tool cover -func=cover.out
github.com/ymcvalu/leakcheck/leakcheck.go:28:   RegisterGoroutineToIgnore       100.0%
github.com/ymcvalu/leakcheck/leakcheck.go:37:   stack                           100.0%
github.com/ymcvalu/leakcheck/leakcheck.go:43:   interestingGoroutines           100.0%
github.com/ymcvalu/leakcheck/leakcheck.go:57:   parseAndFilterGoroutine         76.5%
github.com/ymcvalu/leakcheck/leakcheck.go:90:   bytesToString                   100.0%
github.com/ymcvalu/leakcheck/leakcheck.go:98:   Check                           100.0%
github.com/ymcvalu/leakcheck/leakcheck.go:102:  check                           90.9%
total:                                          (statements)                    87.8%
```
生成html文件，查看具体的代码测试覆盖情况：
```sh
$ go tool cover -html=cover.out  -o cover.html
```


