---
title: "Golang初始化顺序"
date: 2018-11-05T16:03:25+08:00
draft: false
---

### 一个例子
```go
package main

import (
    "fmt"
)

func init() {
    a := "xxx"
    fmt.Printf("hello, %s, %s", "world!", a)
}

func main() {
    fmt.Println("main...")
}
```
编译之后执行：`go tool objdump -s "main.init" main`。可以看到
```
TEXT main.init.0(SB) /Users/chenkang/learn/go/go_under_the_hook/03/main1.go
...
  main1.go:9        0x108f25a       e801caf7ff          CALL runtime.convT2Estring(SB)
...
  main1.go:9        0x108f2f2       e89981ffff          CALL fmt.Printf(SB)
...
  main1.go:7        0x108f30e       e83dc3fbff          CALL runtime.morestack_noctxt(SB)
  main1.go:7        0x108f313       e998feffff          JMP main.init.0(SB)
...
TEXT main.init(SB) <autogenerated>
...
  <autogenerated>:1 0x108f3fe       e89d56f9ff      CALL runtime.throwinit(SB)
...
  <autogenerated>:1 0x108f40c       e84ffbffff      CALL fmt.init(SB)
...
  <autogenerated>:1 0x108f411       e89afdffff      CALL main.init.0(SB)
...
  <autogenerated>:1 0x108f426       e825c2fbff      CALL runtime.morestack_noctxt(SB)
  <autogenerated>:1 0x108f42b       eb93            JMP main.init(SB)
...
```
执行顺序：<br />
1. import中的包init <br />
2. func init()  <br />
3. func main()  <br />
