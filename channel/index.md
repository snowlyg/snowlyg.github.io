# 学习篇-并发通道


<!--more-->
#### 控制 `windows` 服务

文章源码地址：[https://www.github.com/snowlyg/learns](https://www.github.com/snowlyg/learns)

这一篇将介绍如何远程控制服务，通过一个心跳机制，每隔一段时间检测一个服务是否需要更新，停止，启动等操作。

然后通过得到的信息和指令来操控服务。

大概的思路就是通过并发和通道特性来实现。并发和通道都是 `go` 很重要的特性，学会如何运用它们很有必要。

#### 启动一个简单的协程

用 `go` 关键字很简单就可以启动一个协程。
这个协程每隔 5 秒打印一次当前时间

```go
package main

import (
	"time"
)

func hearbeat() {
	for {
		println(time.Now().Format(time.RFC3339))
		time.Sleep(time.Second * 5)
	}
}

func main() {
	go hearbeat()
	// 防止主进程退出
	select {}
}

```

#### 新建一个容量为1的通道，每隔 5 秒发送一个操作指令到通道
```go
package main

import (
	"time"
)
// actionChan 容量为 1 的通道，在 hearbeat , do 两个协程中传递指令信息
// 容量为 1 可以控制并发的数量，只有等发送的指令被处理后才能发送另一个指令
var actionChan = make(chan string, 1)

func hearbeat() {
	for {
		// TODO： 这里需要做处理写死了 update，正常这里需要请求某个接口返回得到一个操作指令
		action := "update"
		actionChan <- action
		// 休眠 5 秒
		println(time.Now().Format(time.RFC3339))
		time.Sleep(time.Second * 5)
	}
}

func main() {
	go hearbeat()
	// 防止主进程退出
	select {}
}

```

通道容量为 1 ,发送一个指令后如果指令没有处理。
没那么将无法再向通道发送指令，`hearbeat` 会被阻塞直到指令被处理通道才能从新接受新的指令。
新建一个新的协程接受指令，并执行相应的操作。

```go
package main

import (
	"fmt"
	"time"

	"github.com/snowlyg/learns/advance/windows"
)

// actionChan 容量为 1 的通道，在 hearbeat , do 两个协程中传递指令信息
// 容量为 1 可以控制并发的数量，只有等发送的指令被处理后才能发送另一个指令
var actionChan = make(chan string, 1)

// hearbeat 每隔 5 秒返回一个操作
func hearbeat() {
	for {
		// TODO： 这里需要做处理写死了 update，正常这里需要请求某个接口返回得到一个操作指令
		action := "update"
		actionChan <- action
		// 休眠 5 秒
		println(time.Now().Format(time.RFC3339))
		time.Sleep(time.Second * 5)
	}
}

// 执行操作，从通道 actionChan 获取对应指令并执行对应操作
func do() {
	for {
		switch <-actionChan {
		case "start":
			err := windows.ServiceStart("myservice")
			if err != nil {
				fmt.Printf("start myservice %v", err)
			}
			println("start")
		case "stop":
			err := windows.ServiceStop("myservice")
			if err != nil {
				fmt.Printf("stop myservice %v", err)
			}
			println("stop")
		case "install":
			err := windows.ServiceInstall("myservice", "../cmd/myservice.exe", "myservice")
			if err != nil {
				fmt.Printf("install myservice %v", err)
			}
			println("install")
		case "uninstall":
			err := windows.ServiceUninstall("myservice")
			if err != nil {
				fmt.Printf("uninstall myservice %v", err)
			}
			println("uninstall")
		case "update":
			err := windows.ServiceStop("myservice")
			if err != nil {
				fmt.Printf("stop myservice %v", err)
			}
			err = windows.ServiceUninstall("myservice")
			if err != nil {
				fmt.Printf("uninstall myservice %v", err)
			}
			err = windows.ServiceInstall("myservice", "myservice.exe", "myservice")
			if err != nil {
				fmt.Printf("install myservice %v", err)
			}
			println("update")
		default:
			println("unknow action")
		}
	}
}

func main() {
	// 启动一个心跳协程，每隔5秒发送指令
	go hearbeat()
	// 启动一个执行命令协程，并等待心跳协程发送指令
	go do()

	// 防止主进程退出
	select {}
}
```

新的 `do` 协程从通道中取出指令，并执行对应的指令。
这个程序中 `hearbeat` 和 `do` 两个方法不会安装原有的执行顺序方式执行。
增加了 `go` 关键字的修饰后，它们同时都在执行中，一个向通道 `actionChan` 中发送数据，
一个从通道 `actionChan` 中取出数据。

通过这个程序，我们就能远程控制某个或者多个服务了。
这个在我们需要远程更新或者控制多个服务器上的服务程序时非常有用。
只需要在本地执行操作，而不用一个个登录远程服务器操作。
