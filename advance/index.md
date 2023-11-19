# 学习篇-进阶


<!--more-->
### 注册 `windows` 服务

文章源码地址：[https://www.github.com/snowlyg/learns](https://www.github.com/snowlyg/learns)

将前面的程序注册成 `windows` 服务有多种方式：
- 使用 [nssm.exe](https://nssm.cc/download) 执行 `nssm.exe install myservice` 
- 使用 [https://github.com/kardianos](https://github.com/kardianos) 项目,如何使用参考 [example](https://github.com/kardianos/service/tree/master/example) 
- 还有 srvany.exe 等等

其中 [https://github.com/kardianos](https://github.com/kardianos) 比较适合实现自动更新项目的需求，这个项目是兼容 linux 、mac windows 等多个系统环境的。但是我们只需要 windows 系统环境，如果是其他环境下更新一个程序其实并不需要这么麻烦。

查看源码能看到 `windwos` 系统使用的是这个包 [golang.org/x/sys/windows](https://golang.org/x/sys/windows) , 用这个包简单的写一个新的 package。

- 项目目录下新建 `windows` 文件夹，并新建7个文件:
-  service.go 
-  beep.go
-  install.go   
-  start.go     
-  stop.go
-  status.go
-  uninstall.go

### service.go 文件

```go
package windows

import (
	"fmt"
	"os"
	"os/signal"
	"time"

	"golang.org/x/sys/windows/svc"
	"golang.org/x/sys/windows/svc/debug"
	"golang.org/x/sys/windows/svc/eventlog"
)

var (
	elog        debug.Log
	interactive = false
)

// init 初始化
// IsAnInteractiveSession 判断程序是否运行在交互模式
// 此方法说是要被废弃了，后期要使用 isWindowsService 代替
// 不过我使用次方法无法正常启动注册好的程序
func init() {
	var err error
	interactive, err = svc.IsAnInteractiveSession()
	if err != nil {
		panic(err)
	}
}

// Interface
type Interface interface {
	Start() error
	Stop() error
}

// WindowsService
type WindowsService struct {
	Name string 
	i    Interface
}

// NewWindowsService new一个服务对象
// name 服务名称
func NewWindowsService(i Interface, name string) (*WindowsService, error) {
	ws := &WindowsService{
		i:    i,
		Name: name,
	}
	return ws, nil
}

// Execute 服务执行方法，控制服务启动停止
func (ws *WindowsService) Execute(args []string, r <-chan svc.ChangeRequest, changes chan<- svc.Status) (bool, uint32) {
	const cmdsAccepted = svc.AcceptStop | svc.AcceptShutdown
	changes <- svc.Status{State: svc.StartPending}

	if err := ws.i.Start(); err != nil {
		elog.Info(1, fmt.Sprintf("%s service start failed: %v", ws.Name, err))
		return true, 1
	}

	changes <- svc.Status{State: svc.Running, Accepts: cmdsAccepted}
loop:
	for {
		c := <-r
		switch c.Cmd {
		case svc.Interrogate:
			changes <- c.CurrentStatus
		case svc.Stop:
			changes <- svc.Status{State: svc.StopPending}
			if err := ws.i.Stop(); err != nil {
				elog.Info(1, fmt.Sprintf("%s service stop failed: %v", ws.Name, err))
				return true, 2
			}
			break loop
		case svc.Shutdown:
			changes <- svc.Status{State: svc.StopPending}
			err := ws.i.Stop()
			if err != nil {
				elog.Info(1, fmt.Sprintf("%s service shutdown failed: %v", ws.Name, err))
				return true, 2
			}
			break loop
		default:
			continue loop
		}
	}

	return false, 0
}

func (ws *WindowsService) Run(isDebug bool) error {
	var err error
	if !interactive {
		if isDebug {
			elog = debug.New(ws.Name)
		} else {
			elog, err = eventlog.Open(ws.Name)
			if err != nil {
				return err
			}
		}
		defer elog.Close()

		elog.Info(1, fmt.Sprintf("starting %s service", ws.Name))
		run := svc.Run
		if isDebug {
			run = debug.Run
		}
		err = run(ws.Name, ws)
		if err != nil {
			elog.Error(1, fmt.Sprintf("%s service failed: %v", ws.Name, err))
			return err
		}
		elog.Info(1, fmt.Sprintf("%s service stopped", ws.Name))
	}

	err = ws.i.Start()
	if err != nil {
		return err
	}

	sigChan := make(chan os.Signal)

	signal.Notify(sigChan, os.Interrupt)

	<-sigChan

	return ws.i.Stop()
}
```
### beep.go 文件
  
```go
package windows

import (
	"syscall"
)

// BUG(brainman): MessageBeep Windows api is broken on Windows 7,
// so this example does not beep when runs as service on Windows 7.

var (
	beepFunc = syscall.MustLoadDLL("user32.dll").MustFindProc("MessageBeep")
)

func beep() {
	beepFunc.Call(0xffffffff)
}

```
###  install.go 文件
 
```go
package windows

import (
	"fmt"

	"golang.org/x/sys/windows/svc/eventlog"
	"golang.org/x/sys/windows/svc/mgr"
)

func ServiceInstall(svcName, execPath, dispalyName string, args ...string) error {
	m, err := mgr.Connect()
	if err != nil {
		return err
	}
	defer m.Disconnect()
	s, err := m.OpenService(svcName)
	if err == nil {
		s.Close()
		return fmt.Errorf("service %s already exists", svcName)
	}
	config := mgr.Config{
		DisplayName: dispalyName,
		StartType:   mgr.StartAutomatic,
	}
	if len(args) >= 2 {
		config.ServiceStartName = args[0]
		config.Password = args[1]
	}
	s, err = m.CreateService(svcName, execPath, config)
	if err != nil {
		return fmt.Errorf("CreateService() failed: %s", err)
	}
	defer s.Close()
	err = eventlog.InstallAsEventCreate(svcName, eventlog.Error|eventlog.Warning|eventlog.Info)
	if err != nil {
		s.Delete()
		return fmt.Errorf("InstallAsEventCreate() failed: %s", err)
	}
	return nil
}

``` 

###  start.go 文件
 
```go
package windows

import (
	"golang.org/x/sys/windows/svc/mgr"
)

func ServiceStart(srcName string) error {
	m, err := mgr.Connect()
	if err != nil {
		return err
	}
	defer m.Disconnect()

	s, err := m.OpenService(srcName)
	if err != nil {
		return err
	}
	defer s.Close()
	return s.Start()
}

``` 

###  status.go 文件
 
```go
package windows

import (
	"errors"
	"fmt"

	"golang.org/x/sys/windows/svc"
	"golang.org/x/sys/windows/svc/mgr"
)

type Status byte

const (
	StatusUnknown Status = iota // Status is unable to be determined due to an error or it was not installed.
	StatusRunning
	StatusStopped
	StatusUninstall
)

var (
	// ErrNameFieldRequired is returned when Config.Name is empty.
	ErrNameFieldRequired = errors.New("Config.Name field is required.")
	// ErrNoServiceSystemDetected is returned when no system was detected.
	ErrNoServiceSystemDetected = errors.New("No service system detected.")
	// ErrNotInstalled is returned when the service is not installed
	ErrNotInstalled = errors.New("the service is not installed")
)

// status
func ServiceStatus(srcName string) (Status, error) {
	m, err := mgr.Connect()
	if err != nil {
		return StatusUnknown, err
	}
	defer m.Disconnect()

	s, err := m.OpenService(srcName)
	if err != nil {
		if err.Error() == "The specified service does not exist as an installed service." {
			return StatusUninstall, nil
		}
		return StatusUnknown, err
	}
	defer s.Close()

	status, err := s.Query()
	if err != nil {
		return StatusUnknown, err
	}

	switch status.State {
	case svc.StartPending:
		fallthrough
	case svc.Running:
		return StatusRunning, nil
	case svc.PausePending:
		fallthrough
	case svc.Paused:
		fallthrough
	case svc.ContinuePending:
		fallthrough
	case svc.StopPending:
		fallthrough
	case svc.Stopped:
		return StatusStopped, nil
	default:
		return StatusUnknown, fmt.Errorf("unknown status %v", status)
	}
}

``` 

###  stop.go 文件
 
```go
package windows

import (
	"strconv"
	"time"

	"golang.org/x/sys/windows/registry"
	"golang.org/x/sys/windows/svc"
	"golang.org/x/sys/windows/svc/mgr"
)

func ServiceStop(srcName string) error {
	m, err := mgr.Connect()
	if err != nil {
		return err
	}
	defer m.Disconnect()

	s, err := m.OpenService(srcName)
	if err != nil {
		return err
	}
	defer s.Close()

	return stopWait(s)
}

func stopWait(s *mgr.Service) error {
	// First stop the service. Then wait for the service to
	// actually stop before starting it.
	status, err := s.Control(svc.Stop)
	if err != nil {
		return err
	}

	timeDuration := time.Millisecond * 50

	timeout := time.After(getStopTimeout() + (timeDuration * 2))
	tick := time.NewTicker(timeDuration)
	defer tick.Stop()

	for status.State != svc.Stopped {
		select {
		case <-tick.C:
			status, err = s.Query()
			if err != nil {
				return err
			}
		case <-timeout:
			break
		}
	}
	return nil
}

// getStopTimeout fetches the time before windows will kill the service.
func getStopTimeout() time.Duration {
	// For default and paths see https://support.microsoft.com/en-us/kb/146092
	defaultTimeout := time.Millisecond * 20000
	key, err := registry.OpenKey(registry.LOCAL_MACHINE, `SYSTEM\CurrentControlSet\Control`, registry.READ)
	if err != nil {
		return defaultTimeout
	}
	sv, _, err := key.GetStringValue("WaitToKillServiceTimeout")
	if err != nil {
		return defaultTimeout
	}
	v, err := strconv.Atoi(sv)
	if err != nil {
		return defaultTimeout
	}
	return time.Millisecond * time.Duration(v)
}

``` 

###  uninstall.go 文件
 
```go
package windows

import (
	"fmt"

	"golang.org/x/sys/windows/svc/eventlog"
	"golang.org/x/sys/windows/svc/mgr"
)

func ServiceUninstall(srcName string) error {
	m, err := mgr.Connect()
	if err != nil {
		return err
	}
	defer m.Disconnect()
	s, err := m.OpenService(srcName)
	if err != nil {
		return fmt.Errorf("service %s is not installed", srcName)
	}
	defer s.Close()
	err = s.Delete()
	if err != nil {
		return err
	}
	err = eventlog.Remove(srcName)
	if err != nil {
		return fmt.Errorf("RemoveEventLogSource() failed: %s", err)
	}
	return nil
}

``` 

### 修改前面的 `main.go` 文件
```go
package main

import (
	"fmt"
	"os"
	"strings"
	"time"

	"github.com/snowlyg/learns/advance/windows"
)

func run() {
	// 每隔 5 秒打印当前时间
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()
	// for 循环阻塞程序主进程
	// ticker.C 通道每隔五秒会发送一个值
	for {
		<-ticker.C
		// 格式化时间
		now := time.Now().Format(time.RFC3339)
		fmt.Printf("当前时间：%s \n", now)
	}
}

func (p *program) Start() error {
	go run()
	return nil
}

func (p *program) Stop() error {
	//stop
	return nil
}

type program struct{}

// usage 获取终端输入参数
func usage(errmsg string) {
	fmt.Fprintf(os.Stderr,
		"%s\n\n"+
			"usage: %s <command> <servicename>\n"+
			"       where <command> is one of\n"+
			"       install, remove, start, stop, status .\n"+
			"       and use install : .\n"+
			"       install <service name> <exec path> <display name> <system name> <password>  \n",
		errmsg, os.Args[0])
	os.Exit(2)
}

func main() {
	// new 一个服务
	s, err := windows.NewWindowsService(&program{}, "myservice")
	if err != nil {
		fmt.Printf("new service get error %v \n", err)
	}
	if len(os.Args) >= 2 {

		srvName := strings.ToLower(os.Args[2])
		cmd := strings.ToLower(os.Args[1])
		switch cmd {
		case "start": // 启动
			err := windows.ServiceStart(srvName)
			if err != nil {
				fmt.Printf("%v \n", err)
			}
			println("start success")
			return
		case "install": //安装
			if len(os.Args) == 7 {
				err := windows.ServiceInstall(srvName, os.Args[3], os.Args[4], os.Args[5], os.Args[6])
				if err != nil {
					fmt.Printf("%v \n", err)
				}
			} else if len(os.Args) == 5 {
				err := windows.ServiceInstall(srvName, os.Args[3], os.Args[4])
				if err != nil {
					fmt.Printf("%v \n", err)
				}
			} else {
				usage("error command specified")
			}

			println("install success")
			return
		case "stop": // 停止
			err := windows.ServiceStop(srvName)
			if err != nil {
				fmt.Printf("%v \n", err)
			}
			println("stop success")
			return
		case "remove": // 卸载
			err := windows.ServiceUninstall(srvName)
			if err != nil {
				fmt.Printf("%v \n", err)
			}
			println("remove success")
			return
		case "status": // 查询服务状态
			status, _ := windows.ServiceStatus(srvName)
			if status == windows.StatusRunning {
				println("运行中")
			} else if status == windows.StatusStopped {
				println("已停止")
			} else if status == windows.StatusUninstall {
				println("未安装")
			} else if status == windows.StatusUninstall {
				println("未安装")
			}
			return
		default:
			println("invaild command")
		}
		switch cmd {
		case "start": // 启动
			err := windows.ServiceStart(srvName)
			if err != nil {
				fmt.Printf("%v \n", err)
			}
			println("start success")
			return
		case "install": //安装
			if len(os.Args) == 7 {
				err := windows.ServiceInstall(srvName, os.Args[3], os.Args[4], os.Args[5], os.Args[6])
				if err != nil {
					fmt.Printf("%v \n", err)
				}
			} else if len(os.Args) == 5 {
				err := windows.ServiceInstall(srvName, os.Args[3], os.Args[4])
				if err != nil {
					fmt.Printf("%v \n", err)
				}
			} else {
				usage("error command specified")
			}

			println("install success")
			return
		case "stop": // 停止
			err := windows.ServiceStop(srvName)
			if err != nil {
				fmt.Printf("%v \n", err)
			}
			println("stop success")
			return
		case "remove": // 卸载
			err := windows.ServiceUninstall(srvName)
			if err != nil {
				fmt.Printf("%v \n", err)
			}
			println("remove success")
			return
		case "status": // 查询服务状态
			status, _ := windows.ServiceStatus(srvName)
			if status == windows.StatusRunning {
				println("运行中")
			} else if status == windows.StatusStopped {
				println("已停止")
			} else if status == windows.StatusUninstall {
				println("未安装")
			} else if status == windows.StatusUnknown {
				println("未知状态")
			}
			return
		default:
			println("invaild command")
		}
	}
	s.Run(false)
}

```
### 打包文件

再次打包程序 `go build -o myservice.exe main.go` 生成 `myservice.exe` 文件。
可以通过如下命令行控制服务：
```shell

# 注册服务，有七个参数 服务名称、服务文件路径、服务显示名称、系统账号、系统密码
# 只有 服务名称、服务文件路径 为必选参数，其他为可选参数
# 有账号密码
.\cmd\advance.exe install myservice "C:\Users\Administrator\go\src\github.com\snowlyg\learns\cmd\advance.exe" myservice  ".\Administrator" "123456"
# 没账号密码
.\cmd\advance.exe install myservice "C:\Users\Administrator\go\src\github.com\snowlyg\learns\cmd\advance.exe" myservice  

# 启动服务
.\cmd\advance.exe start myservice 

# 停止服务
.\cmd\advance.exe stop myservice 

# 卸载服务
.\cmd\advance.exe remove myservice 

# 查询服务状态
.\cmd\advance.exe status myservice 

```
### 小坑
**这里有个小坑：当使用账号（administrator）、密码(123456)参数注册服务的时候，启动的时候会提示账号密码错误。这个问题我各种看源码、各种搜索都没有解决问题。最后无意中看到了这个文档[https://docs.microsoft.com/en-us/windows/win32/api/winsvc/nf-winsvc-createservicea](https://docs.microsoft.com/en-us/windows/win32/api/winsvc/nf-winsvc-createservicea)**

**这个文档我有找了几个小时才找到，毕竟已经过了好几天了。仔细看这个文档会发现这样描述：**
![docs.png](docs.png)

就是最后一行，注册服务获取系统授权需要使用 `.\Administrator` 。

最后吐槽一下：windows 的文档是真的难找。
