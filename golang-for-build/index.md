# Golang 如何交叉编译？在一个平台上生成另一个平台的可执行程序


<!--more-->
#### 介绍
 Golang 支持交叉编译，在一个平台上生成另一个平台的可执行程序，最近使用了一下，非常好用，这里备忘一下。
[https://blog.csdn.net/hbzhou2009/article/details/77677160 ](https://blog.csdn.net/hbzhou2009/article/details/77677160)

#### CGO_ENABLED 是否启用 CGO
```
CGO_ENABLED=1
```


#### Mac 下编译 Linux 和 Windows 64位可执行程序
> 如果启用了 cgo 需要安装 gcc , 如果是跨平台编译还需要通过 CC 参数指定所需平台的 gcc 组件
> 如下命令所示：
```
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
```

- mac to windows
```
CGO_ENABLED=1 GOOS=windows GOARCH=amd64 CC=/usr/local/bin/x86_64-w64-mingw32-gcc CXX=/usr/local/bin/x86_64-w64-mingw32-g+ go build -a -installsuffix cgo -ldflags "-w -s -X main.Version=v1.0" -o ./cmd/main.exe
```

- mac to linux
```
CGO_ENABLED=1 GOOS=linux  GOARCH=amd64 CC=/usr/local/gcc-4.8.1-for-linux64/bin/x86_64-pc-linux-gcc go build -a -installsuffix cgo -ldflags "-w -s -X main.Version=v1.0" -o ./cmd/main_lin
```



#### Linux 下编译 Mac 和 Windows 64位可执行程序

```
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```



#### Windows 下编译 Mac 和 Linux 64位可执行程序

```
SET CGO_ENABLED=0
SET GOOS=darwin
SET GOARCH=amd64
go build main.go

SET CGO_ENABLED=0
SET GOOS=linux
SET GOARCH=amd64
go build main.go
```


GOOS：目标平台的操作系统（darwin、freebsd、linux、windows）
GOARCH：目标平台的体系架构（386、amd64、arm）

