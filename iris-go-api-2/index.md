# iris + casbin 从陌生到学会使用的过程


<!--more-->

### 前言
前面写了一篇 [【iris-go 框架构建登陆 API 项目开发过程】](https://blog.snowlyg.com/iris-go-api-1/ "【iris-go 框架构建登陆 API 项目开发过程】") 的文章，因为篇幅的关系文章并没有介绍如何实现权限控制。接下来我将会介绍如何使用 casbin 在 iris 的项目中实现权限控制，项目采用 grom 做数据处理。
>  本文纯属个人简陋观点和经验，如果有错误的地方，欢迎大家指出。

这里继续讲一些废话，原本 [IrisAdminApi](https://github.com/snowlyg/IrisAdminApi "IrisAdminApi") 项目采用的最原始的 RBAC 的权限控制实现，分别创建 roles ， permissions 表，然后定义关联关系，写一些关联创建逻辑。在实现的过程发现不管是使用 gorm ，xorm，beego的orm，定义关联关系都挺麻烦的，特别是多对对关系，还有一些多态关系等等。可能 Laravel 使用习惯性了，发现框架功能太齐全，对自身的提升真的会有阻碍。

这时候我就想有没有别人实现好的权限控制轮子，结果一顿搜索，让我找到了 casbin。这就是我和 casbin 的初识了。

刚见到 casbin 其实有些不知道如何下手，毕竟没有见过类似的项目。相信很多新手也会有同样的感受。不过没关系，一回生两回熟，像我们这样经过九年义务教育的社会主义好青年，最不怕的就是学习新事物。

大致介绍下我学习 casbin 的思路：
- 首先：看文档，就像买了新东西看说明书一样，学习任何新事物必不可少的一步。不需要完全背下来，大致有个印象就好。不懂的地方可以多看几遍，不要怕麻烦。
- 接下来：在文档中找到我们需要的功能，我们需要的是一个与 gorm 工作的 RBAC 权限控制，正好我发现作者有写 gorm 相关的适配器，其实作者为不同语言写非常多的适配器，这样可以省去我们的很多时间。
- 打开 [gorm-adapter](https://github.com/casbin/gorm-adapter "gorm-adapter") 项目，按照说明执行安装适配器：
```shell
go get github.com/casbin/gorm-adapter
```
- 查看实例代码，很多项目多有实例代码，其实这些代码的价值比文档的价值更高，能让我们更好的理解项目。特别是注释，需要仔细看完。

```go
package main

import (
	"github.com/casbin/casbin/v2"
	gormadapter "github.com/casbin/gorm-adapter/v2"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	// 初始化一个 Gorm 适配器并且在一个 Casbin enforcer 中使用它:
	// 这个适配器会使用一个名为 "casbin" 的 MySQL 数据库。
	// 如果数据库不存在，适配器会自动创建它。 
	// 你同样也可以像这样 gormadapter.NewAdapterByDB(gormInstance) 使用一个已经存在的 gorm 实例。
	a := gormadapter.NewAdapter("mysql", "mysql_username:mysql_password@tcp(127.0.0.1:3306)/") //你的驱动和数据源
	e, _ := casbin.NewEnforcer("examples/rbac_model.conf", a)
	
	// 或者你可以像这样使用一个其他的数据库 "abc" :
	// 适配器会使用名为 "casbin_rule" 的数据表。
	// 如果数据表不存在，适配器会自动创建它。
	// a := gormadapter.NewAdapter("mysql", "mysql_username:mysql_password@tcp(127.0.0.1:3306)/abc", true)

	// 从数据库加载策略规则
	e.LoadPolicy()
	
	// 检查权限
	e.Enforce("alice", "data1", "read")
	
	// 更新策略
	// e.AddPolicy(...)
	// e.RemovePolicy(...)
	
	// 保存策略到数据库
	e.SavePolicy()
}
```
- 看完这个实例的代码和注释，相信你已经了解 casbin 怎么使用了。如果还不了解，那就多看几遍。
- 在看文档的过程中发现作者已经写好了 [RBAC](https://casbin.org/docs/en/rbac-api " RBAC") 相关的 API 。所以我们并不需要去重复的这些工作。你没有看到？这说明了仔细看文档的重要的性。

- 现在我们接着前面的项目，将 casbin 功能添加到项目中。编辑 models/base.go 文件：
```go
var Db *gorm.DB 
var err error  
var dirverName string  
var conn string
var Enforcer *casbin.Enforcer
func Register() {
    dirverName = "mysql"
    if isTestEnv() { //如果是测试使用测试数据库
        conn = "root:wemT5ZNuo074i4FNsTwl4KhfVSvOlBcF@(127.0.0.1:3306)/tiris"
    } else {
        conn = "root:wemT5ZNuo074i4FNsTwl4KhfVSvOlBcF@(127.0.0.1:3306)/iris"
    }
    //初始化数据库
    Db, err = gorm.Open(dirverName, conn+"?charset=utf8&parseTime=True&loc=Local")
    if err != nil {
        color.Red(fmt.Sprintf("gorm open 错误: %v", err))
    }
	
	a := gormadapter.NewAdapter("mysql",conn) 
	Enforcer, _ = casbin.NewEnforcer("examples/rbac_model.conf", a)
	Enforcer.LoadPolicy()
}
```
- 这里我们不需要生成新的 casbin 数据库，直接使用在 iris 数据库生成 casbin_rule 数据表。同时将 `rbac_model.conf` 文件复制到自己的项目的对应路径中。这样 casbin 就已经集成好了，重启项目就会生成对应的 casbin_rule 数据表了。
- 为什么没有使用 e.AddPolicy(...)，e.SavePolicy()，e.Enforce("alice", "data1", "read") 这些方法？ 如果你还有这样的疑问，说明你没有看实例的注释，要么就是没有仔细阅读。请多看几遍。
- 虽然我们已经集成了 casbin ，但是我们还没有实现权限控制的目的。如果实现？我们需要关联角色，权限等数据到用户，并且在用户访问相应资源的时候检查用户是否拥有权限。
- 到这里有经验的朋友就会发现，我们需要一个中间件去实现用户权限的检测。那么我们移驾到 [iris 的中间件文档](https://github.com/snowlyg/iris/wiki/Routing-middleware "iris 的中间件文档")，我们仔细的看以遍文档，你会发现已经有了 casbin 的中间件实现。并不需要我们自己编写。
- 接着来到 [middleware/tree/master/casbin](https://github.com/iris-contrib/middleware/tree/master/casbin "middleware/tree/master/casbin")

按照说明文档，先安装 casbin 还有 casbin-middleware，分别执行一下命令：
```shell
 go get github.com/casbin/casbin
 
 go get github.com/iris-contrib/middleware/casbin
```

- 然后按照实例中的代码将中间件添加到路由：
```go
package routers
import (
"IrisAdmin/controllers"
"IrisAdmin/middleware"
"IrisAdmin/models"
"github.com/kataras/iris/v12"
"github.com/casbin/casbin/v2"
cm "github.com/iris-contrib/middleware/casbin"
)
func Register(app *iris.Application) {
// 路由集使用跨域中间件 CrsAuth()
// 允许 Options 方法 AllowMethods(iris.MethodOptions)
main := app.Party("/", middleware.CrsAuth()).AllowMethods(iris.MethodOptions)
{
 v1 := main.Party("/v1")
 {
     v1.Post("/admin/login", controllers.UserLogin)
     v1.PartyFunc("/admin", func(admin iris.Party) {
		casbinMiddleware := cm.New(models.Enforcer)
        admin.Use(middleware.JwtHandler().Serve, middleware.AuthToken，casbinMiddleware.ServeHTTP) //登录验证
         admin.Get("/logout",  controllers.UserLogout).Name = "退出"
     })
 }
}
}
```

- 这里中间件有一个新的 `casbinmodel.conf`文件，需要把里面的内容替换到前面提到的 `rbac_model.conf` 文件中。
- 现在登陆项目访问任何接口都会出现 `403` 拒绝访问的状态返回，这说明中间件生效了。接下来我们需要使用 casbin 提供的 Api 将项目的用户，角色和权限相互关联起来。
- 打开 [rbac-api](https://casbin.org/docs/zh-CN/rbac-api "rbac-api") 页面，我们看到有一个 `AddRoleForUser()` 给用户添加角色的方法。我们将它添加到用户的创建和更新方法中，实现用户和角色的关联。

```go
func CreateUser() (user *User) {
  salt, _ := bcrypt.Salt(10)
  hash, _ := bcrypt.Hash("password", salt)
  user = &User{
      Username: "username",
      Password: hash,
      Name:     "name",
  }
  if err := Db.Create(user).Error; err != nil {
      color.Red(fmt.Sprintf("CreateUserErr:%s \n ", err))
  }
	
	userId := strconv.FormatUint(uint64(user.ID), 10)
	if _, err := database.GetEnforcer().AddRoleForUser(userId, "1"); err != nil {
		color.Red(fmt.Sprintf("CreateUserErr:%s \n ", err))
	}
	
  return
}
```
- 这里需要注意的是  `AddRoleForUser()` 函数接受的参数都是字符串，所以用户和角色的 Id 都需要转换一下格式。如果是多角色传入多个角色的 Id 循环添加即可。
- 接下来我们要关联角色和权限， 在 [rbac-api](https://casbin.org/docs/zh-CN/rbac-api "rbac-api") 页面，找了半天发现没有添加权限的方法。仔细看过文档多次之后，确定真的没有添加权限到角色的方法。那么，我们接下来该怎么办呢？我的解决方法是查看 casbin 的源码。因为我十分确定，添加权限的方法肯定是存在的，只是[rbac-api](https://casbin.org/docs/zh-CN/rbac-api "rbac-api") 页面上没有而已。
- 果然，我在文档找到了相关方法:
```
func (e *Enforcer) AddPolicy(params ...interface{}) (bool, error) {
	return e.AddNamedPolicy("p", params...)
}
```
是不是很眼熟。没错，就是在前面适配器实例中出现过的方法。它的说明是添加策略，当时没想到策略就是权限，真的是一叶障目，不见南山。众里寻他千百度.....

- 既然找到方法，那么将角色关联权限（策略）的逻辑添加到角色的创建和更新当中。
```go
roleId := strconv.FormatUint(uint64(role.ID), 10)
var perms []Permission
models.DB.Where("id in (?)", permIds).Find(&perms)
for _, perm := range perms {
	if _, err := models.Enforcer.AddPolicy(roleId, perm.Name, perm.Act); err != nil {
		color.Red(fmt.Sprintf("AddPolicy:%s \n", err))
	}
}
```
同样 `AddPolicy()` 函数参数都是字符串，角色 Id 也需要转换一下。

- 所有数据都关联好了之后，我们创建好相关的角色和权限数据。用户就可以正常访问登陆了。详细代码请查看项目 [IrisAdminApi](https://github.com/snowlyg/IrisAdminApi "IrisAdminApi") 。

### 后记
这里记录一下如何使用 iris 自动生成权限（策略）的思路。
- 首先，肯定要从路由入手，给路由都加上名称，方便区分权限。
```go
app.PartyFunc("/users", func(users iris.Party) {
	users.Get("/", controllers.GetAllUsers).Name = "用户列表"
	users.Get("/{id:uint}", controllers.GetUser).Name = "用户详情"
	users.Post("/", controllers.CreateUser).Name = "创建用户"
	users.Put("/{id:uint}", controllers.UpdateUser).Name = "编辑用户"
	users.Delete("/{id:uint}", controllers.DeleteUser).Name = "删除用户"
	users.Get("/profile", controllers.GetProfile).Name = "个人信息"
})
```
这样就可以在获取路由的时候获取到对应的名称了。
- 如何获取 iris 应用的路由信息？找了文档发现没有相关的方法。这时候我们只能取查看源码了。
在源码中我找到了如下源码：

```go
// GetRoutes 返回路由的信息,
// 这些信息有些可以被修改，有些不可以。
//
// 需要刷新路由器到方法或路径或处理程序更改才能进行。
func (api *APIBuilder) GetRoutes() []*Route {
	return api.routes.getAll()
}

// GetRoute 基于路由名称返回已注册路由， 否则返回nil。
// 一个提示： "路由名称" 区分大小写。
func (api *APIBuilder) GetRoute(routeName string) *Route {
	return api.routes.get(routeName)
}

// GetRouteByPath 基于模版路径 (`Route.Tmpl().Src`) 返回已注册路由。
func (api *APIBuilder) GetRouteByPath(tmplPath string) *Route {
	return api.routes.getByPath(tmplPath)
}

// GetRoutesReadOnly 返回只读授权的已注册路由，
// 你不能也不应该在请求状态下改变路由的任何属性，
// 如果想要改变可以使用 `GetRoutes()`函数替代。
//
// 它返回基于接口的切片而不是实际切片，
// 以便在上下文（请求状态）和构建的应用程序之间安全提取。
//
// 同时请看 `GetRouteReadOnly` 。
func (api *APIBuilder) GetRoutesReadOnly() []context.RouteReadOnly {
	routes := api.GetRoutes()
	readOnlyRoutes := make([]context.RouteReadOnly, len(routes))
	for i, r := range routes {
		readOnlyRoutes[i] = routeReadOnlyWrapper{r}
	}
	return readOnlyRoutes
}
```
从以上源码中可以看出， `GetRouteReadOnly` 和 `GetRoutes()` 都满足我们的使用需求。我们使用 `GetRoutesReadOnly()`实现。
```go
app = NewApp() // 初始化app
routes := app.GetRoutesReadOnly()

// 保存数据到数据库，详细代码查看 https://github.com/snowlyg/IrisAdminApi 
... 
```
- 这里会有个小问题，就是数据没有问题，但是访问类似编辑，更新等后面带有 id 会变化的接口，会出现 403 返回。可是明明已经都有权限数据了，而且其他接口也能正常访问，为什么会这样？
检查发现原因是：生成的（权限）策略数据是这样的 `/v1/admin/users/{id:uint}`

![image.png](mLTP2qruGn.png)


实际上 casbin 匹配的权限应该是这样的 `/dataset1/*`，具体内容查看 [casbinpolicy.csv](https://github.com/iris-contrib/middleware/blob/master/casbin/_examples/middleware/casbinpolicy.csv "casbinpolicy.csv")

- 这时候有两种解决思路：
	- 修改自动生成权限的逻辑，将 `{id:uint}`，替换成 `*`。
	- 修改 casbin 的匹配逻辑。
因为第一种比较简单，这里主要讨论第二种方式，既然要修改 casbin 的匹配逻辑。首先要了解它的原理，很不巧和很多刚接触 casbin 的朋友一样，我对这些也不是很了解。这时候我门只能通过查看文档，以及源码来更深入的了解它。
- 点开文档 [matchers](https://casbin.org/docs/zh-CN/syntax-for-models#matchers "matchers") 我查找到 matachers 部分适用于定于匹配规则的，但是这里没有更深层的解释要如何定义。
- 继续查看文档 [function](https://casbin.org/docs/zh-CN/function "function")，在这里我们能够到不同的匹配模式，甚至我们可以自定义匹配函数。而且我们发现其中有两种匹配方式可能会满足我们，`keyMatch3`，`keyMatch4`都能匹配 `{}` 类似的表达式 。

![image.png](BjkjrOeOYx.png)

那么我们来尝试修改 `examples/rbac_model.conf` 这个文件的内容，将 `keyMatch`改为`keyMatch3`，重启项目之后访问编辑相关接口，发现问题已经解决。（发现使用 `keyMatch4`，同样有效）

```go
[matchers]
m = g(r.sub, p.sub) && keyMatch3(r.obj, p.obj) && (r.act == p.act || p.act == "*")
```

- 还有另外一种方式：用来修改默认的匹配。
	- 首先修改 `examples/rbac_model.conf` 这个文件：去除中间的 `&& keyMatch3(r.obj, p.obj)` 让 casbin 使用默认的匹配规则（默认为 `keyMatch`）。
```go
[matchers]
m = g(r.sub, p.sub)  && (r.act == p.act || p.act == "*")
```
- 然后在  `models/base.go` 文件增加代码：
```go
import (
	defaultrolemanager "github.com/casbin/casbin/v2/rbac/default-role-manager"
	"github.com/casbin/casbin/v2/util"
)
......
Enforcer, _ = casbin.NewEnforcer("examples/rbac_model.conf", a)
// 修改默认匹配为 KeyMatch3
rm := defaultrolemanager.NewRoleManager(10).(*defaultrolemanager.RoleManager)
rm.AddMatchingFunc("KeyMatch3", util.KeyMatch3)
Enforcer.LoadPolicy()
......
```
此方法和上面直接修改 `examples/rbac_model.conf` 文件的方式更复杂一些，这里只是介绍一下提供另一种思路。


> 写完了，感觉有点乱。主要还是讲述我解决需求和问题的一个思路。希望对你有帮助。
> 还是那句话：能力有限，不喜勿喷。


