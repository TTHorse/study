# 后端入门与前后端打通（Go 语言）

> 作为前端开发，你已经能写出漂亮的界面了。但数据从哪来？登录态怎么维持？文件上传到哪？这一阶段的目标是**用 Go 从零搭建一个能处理 HTTP 请求、读写数据库、管理用户认证的后端服务**——并且和前端的知识无缝对接。

---

## 目录

- [1. 为什么选 Go](#1-为什么选-go)
  - [1.1 前端转 Go 的视角切换](#11-前端转-go-的视角切换)
  - [1.2 Go 是什么](#12-go-是什么)
  - [1.3 环境搭建](#13-环境搭建)
- [2. Go 语法速通](#2-go-语法速通)
  - [2.1 类型系统](#21-类型系统)
  - [2.2 函数与方法](#22-函数与方法)
  - [2.3 struct 与 interface](#23-struct-与-interface)
  - [2.4 错误处理](#24-错误处理)
  - [2.5 并发：goroutine 与 channel](#25-并发goroutine-与-channel)
  - [2.6 包管理与项目结构](#26-包管理与项目结构)
- [3. 从零写 RESTful API](#3-从零写-restful-api)
  - [3.1 标准库 net/http](#31-标准库-nethttp)
  - [3.2 路由与中间件（Gin 框架）](#32-路由与中间件gin-框架)
  - [3.3 请求校验与响应格式](#33-请求校验与响应格式)
  - [3.4 错误处理中间件](#34-错误处理中间件)
- [4. 数据库入门](#4-数据库入门)
  - [4.1 SQLite 快速上手](#41-sqlite-快速上手)
  - [4.2 PostgreSQL 生产实践](#42-postgresql-生产实践)
  - [4.3 GORM 入门](#43-gorm-入门)
  - [4.4 Migration 管理](#44-migration-管理)
- [5. 认证与授权](#5-认证与授权)
  - [5.1 密码哈希](#51-密码哈希)
  - [5.2 JWT 的签发与验证](#52-jwt-的签发与验证)
  - [5.3 中间件鉴权](#53-中间件鉴权)
  - [5.4 Session vs Token vs OAuth2.0](#54-session-vs-token-vs-oauth20)
- [6. 文件上传](#6-文件上传)
  - [6.1 本地文件接收](#61-本地文件接收)
  - [6.2 对象存储与 Presigned URL](#62-对象存储与-presigned-url)
- [7. 前后端联调关键概念](#7-前后端联调关键概念)
  - [7.1 CORS 深入](#71-cors-深入)
  - [7.2 Cookie 跨域](#72-cookie-跨域)
  - [7.3 API 设计规范](#73-api-设计规范)
  - [7.4 接口文档](#74-接口文档)
- [附录：一个完整的 Todo API](#附录一个完整的-todo-api)
- [检验清单](#检验清单)

---

## 1. 为什么选 Go

### 1.1 前端转 Go 的视角切换

作为前端开发者，你已经习惯了 JavaScript 的灵活性——动态类型、一等公民函数、原型链、事件循环。切换到 Go 需要做几个思维转变：

| JavaScript 思维 | Go 思维 |
|----------------|---------|
| `const x = 1` — 类型运行时确定 | `var x int = 1` — 类型编译时确定 |
| `async/await` 单线程异步 | goroutine 多线程并发，channel 通信 |
| `try/catch` 抛出异常 | 函数返回 `(result, error)`，显式检查 |
| `class` 和继承 | `struct` 和组合（没有继承） |
| `npm install` 装依赖 | `go get` 装依赖，`go.mod` 管理版本 |
| 前端 build 完跑在浏览器 | 后端编译成二进制文件，直接运行 |

> **生活化类比**：JavaScript 像**乐高积木**——随手拼、随意改、错了拔下来重来。Go 像**宜家家具**——所有零件在装配前就确定了尺寸和规格（编译时检查），装好后结构稳固、不会散架（静态类型 + 编译成二进制）。你不需要螺丝刀也能玩乐高，但宜家家具一旦装好，十年不坏。

### 1.2 Go 是什么

Go 是 Google 在 2009 年发布的开源语言，设计哲学是**简洁、高效、并发优先**。

**为什么后端用 Go 很合适：**

- **编译成单一二进制文件**：没有运行时依赖，丢到服务器上就能跑。不像 Node.js 需要装 node_modules，Python 需要配虚拟环境
- **天生高并发**：`go` 关键字就能启动一个 goroutine（轻量级协程），一台机器跑几十万并发是常态
- **静态类型 + 编译检查**：类型错误、未使用的变量、未使用的 import——编译阶段就报错，不会等上线才发现
- **标准库强大**：`net/http` 写 HTTP 服务、`database/sql` 操作数据库、`encoding/json` 处理 JSON、`crypto/*` 处理加密——标准库就能覆盖大部分后端需求
- **部署简单**：`go build` 生成一个可执行文件，copy 到服务器就部署完了

> **生活化类比**：Go 编译出来的二进制就像**集装箱**——自包含、不需要额外环境、吊到任何一台服务器上就能运行。Node.js 就像**外卖**——你不仅需要食物本身，还需要外卖盒、筷子、塑料袋（node_modules），少一样都不行。

### 1.3 环境搭建

```bash
# macOS
brew install go

# 验证安装
go version  # 应输出 go1.22+ 

# 配置 GOPROXY（国内加速）
go env -w GOPROXY=https://goproxy.cn,direct
```

**第一个 Go 程序：**

```go
// main.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, 后端世界!")
}
```

```bash
go run main.go   # 直接运行
go build         # 编译成可执行文件
./study          # 运行编译后的二进制
```

> **生活化类比**：`go run` 就像**前台点餐即食**——一次性执行完。`go build` 就像**打包外卖**——打包好带走，到哪都能吃。

---

## 2. Go 语法速通

这一节的目标不是让你成为 Go 专家，而是**能看懂和改写后端代码**。每个概念都配前端对照，帮你建立映射。

### 2.1 类型系统

```go
// 变量声明
var name string = "sam"      // 完整写法
var age = 25                  // 类型推断
email := "sam@example.com"   // 短声明（最常用，仅函数内可用）

// 基本类型
var (
    isAdmin  bool    = true
    count    int     = 42          // int 的大小取决于平台（64位系统是 int64）
    score    int64   = 9999999999
    price    float64 = 19.99
    nickname string  = "sam"
)

// 数组 vs 切片
var arr [3]string = [3]string{"a", "b", "c"}  // 数组：固定长度，类型的一部分
names := []string{"sam", "alice"}             // 切片：动态长度，最常用
names = append(names, "bob")                  // 追加元素

// map
users := map[string]int{
    "sam":   25,
    "alice": 30,
}
users["bob"] = 28           // 写入
age, ok := users["unknown"] // ok=false 表示 key 不存在
delete(users, "bob")        // 删除

// 指针（和 JS 的引用不同）
x := 42
p := &x    // p 是指向 x 的指针
*p = 100   // 通过指针修改 x
fmt.Println(x)  // 100
```

| JS 概念 | Go 对应 |
|---------|---------|
| `const arr = [1,2,3]` | `arr := []int{1, 2, 3}` |
| `arr.push(4)` | `arr = append(arr, 4)` |
| `const obj = {a: 1}` | `obj := map[string]int{"a": 1}` |
| `obj.b` | `obj["b"]` |
| `typeof x === "undefined"` | `_, ok := m["key"]; !ok` |

> **生活化类比**：Go 的类型系统就像**机场安检**——你行李里（变量里）装的什么东西，在进去之前（编译时）就检查清楚了。JS 的动态类型就像**地铁进站**——先让你进去，出问题再说（运行时报错）。指针就像**快递取件码**——你不需要搬着包裹走，拿着取件码就能找到包裹在哪、修改它的状态。

### 2.2 函数与方法

```go
// 普通函数
func add(a, b int) int {
    return a + b
}

// 多返回值（Go 的特色）
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("除数不能为零")
    }
    return a / b, nil
}

// 调用多返回值
result, err := divide(10, 3)
if err != nil {
    // 处理错误
}

// 方法：给类型绑定函数
type User struct {
    Name string
    Age  int
}

func (u User) Greet() string {
    return "你好，我是" + u.Name
}

func (u *User) Birthday() {
    u.Age++  // 指针接收者：可以修改原值
}
```

**关键区别：**
- `func (u User)` — 值接收者，操作的是副本，原值不变
- `func (u *User)` — 指针接收者，操作的是原值

> **生活化类比**：值接收者就像**复印了一份文件然后在复印件上批注**——原件不变。指针接收者就像**直接在原件上用红笔批注**——原件变了，以后谁看都是改过的版本。

### 2.3 struct 与 interface

Go 没有 class，也没有继承，而是用 **struct（结构体）** 和 **interface（接口）** 的组合。

```go
// struct 定义
type Todo struct {
    ID        int64     `json:"id"`
    Title     string    `json:"title"`
    Completed bool      `json:"completed"`
    CreatedAt time.Time `json:"created_at"`
}

// interface 定义：只要实现了这些方法，就自动满足这个接口
type TodoRepository interface {
    FindAll() ([]Todo, error)
    FindByID(id int64) (*Todo, error)
    Create(todo *Todo) error
    Update(todo *Todo) error
    Delete(id int64) error
}

// 实现接口（隐式满足，不需要显式声明）
type SQLiteTodoRepo struct {
    db *sql.DB
}

func (r *SQLiteTodoRepo) FindAll() ([]Todo, error) {
    // ... 从 SQLite 查询
}
```

**Go 的 interface 是隐式满足的**——不需要写 `implements` 关键字。只要 `SQLiteTodoRepo` 实现了 `TodoRepository` 要求的所有方法，它就自动满足这个接口。这让你可以轻松替换实现：写测试时用一个内存版 repo，生产环境用 PostgreSQL 版 repo。

> **生活化类比**：Go 的 interface 就像**餐饮行业标准**——任何餐厅，只要它提供"点菜、上菜、结账"这三个服务，就是"餐厅"。你不需要在门口挂个牌子说"本店实现了餐厅接口"。Go 的 struct 组合（没有继承）就像**团队协作**——你不会说"设计师继承了员工"，而是说"设计师有绘图能力 + 员工有打卡能力"，把不同能力组合在一起。

### 2.4 错误处理

Go 没有 `try/catch`，函数通过返回 `error` 来传递错误。这是 Go 最让新手不习惯但也是最有争议的设计之一。

```go
// Go 风格：每个可能出错的操作后都检查 err
func GetUserByID(db *sql.DB, id int64) (*User, error) {
    row := db.QueryRow("SELECT id, name, email FROM users WHERE id = ?", id)
    
    var user User
    err := row.Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        return nil, fmt.Errorf("查询用户 %d 失败: %w", id, err)
    }
    
    return &user, nil
}

// 调用方
user, err := GetUserByID(db, 123)
if err != nil {
    // 处理错误：记录日志、返回 HTTP 500 等
    log.Printf("获取用户失败: %v", err)
    return
}
// 使用 user
```

> **生活化类比**：JS 的 `try/catch` 就像**买保险**——你正常生活，出事了保险公司兜底。Go 的显式错误处理就像**每次出门前检查天气预报**——你知道今天可能要下雨，顺手带把伞。虽然每次都要检查很繁琐，但你不会在暴雨中被淋成落汤鸡时才后悔。

### 2.5 并发：goroutine 与 channel

这是 Go 的杀手级特性。前端同学可以把 goroutine 理解为**浏览器里的 Web Worker**，但轻量得多。

```go
// 启动一个 goroutine：在函数调用前加 go
go func() {
    fmt.Println("我在另一个 goroutine 里运行")
}()

// channel：goroutine 之间通信的管道
ch := make(chan string)

// 发送方
go func() {
    ch <- "任务完成"  // 把数据发送到管道
}()

// 接收方
result := <-ch  // 从管道接收数据
fmt.Println(result)

// 实际场景：并发处理多个请求
func fetchAll(urls []string) []string {
    ch := make(chan string, len(urls))
    
    for _, url := range urls {
        go func(u string) {
            resp, _ := http.Get(u)
            // 处理...
            ch <- u + " 已获取"
        }(url)
    }
    
    var results []string
    for i := 0; i < len(urls); i++ {
        results = append(results, <-ch)
    }
    return results
}
```

> **生活化类比**：goroutine 就像**餐厅服务员**——一个餐厅不会只雇一个服务员（单线程），而是雇多个服务员同时服务多个桌。服务员之间通过传菜口（channel）交接工作——"3 号桌的菜好了！" Go 的调度器就像一个**经验丰富的餐厅经理**，自动协调服务员的工作，让每个人都不闲着。

### 2.6 包管理与项目结构

```
myapi/
├── go.mod              # 模块定义 + 依赖版本
├── go.sum              # 依赖校验和（自动生成）
├── main.go             # 入口文件
├── config/
│   └── config.go       # 配置管理
├── handler/
│   ├── todo.go         # HTTP 处理器（controller 层）
│   └── user.go
├── model/
│   ├── todo.go         # 数据模型
│   └── user.go
├── repository/
│   ├── todo.go         # 数据库操作（数据访问层）
│   └── user.go
├── service/
│   ├── todo.go         # 业务逻辑层
│   └── user.go
└── middleware/
    └── auth.go         # 鉴权中间件
```

```bash
# 初始化项目
go mod init github.com/yourname/myapi

# 安装依赖
go get github.com/gin-gonic/gin
go get gorm.io/gorm

# 整理依赖
go mod tidy
```

---

## 3. 从零写 RESTful API

### 3.1 标准库 net/http

Go 标准库已经能写 HTTP 服务，不需要框架也能干活。先理解标准库，再上框架。

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Todo struct {
    ID    int    `json:"id"`
    Title string `json:"title"`
}

// 处理器函数
func handleTodos(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        todos := []Todo{
            {ID: 1, Title: "学 Go"},
            {ID: 2, Title: "写 API"},
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(todos)

    case http.MethodPost:
        var todo Todo
        json.NewDecoder(r.Body).Decode(&todo)
        todo.ID = 3
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(todo)

    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
    }
}

func main() {
    http.HandleFunc("/api/todos", handleTodos)
    http.ListenAndServe(":8080", nil)
}
```

**标准库的不足：**
- 路由不支持路径参数（`/api/todos/:id`），需要自己解析
- 没有中间件机制，需要自己包装
- 请求参数提取和校验需要手写大量代码

这就是为什么我们需要一个轻量框架。**Gin** 是目前 Go 生态最流行的 HTTP 框架，性能极高，API 简洁。

### 3.2 路由与中间件（Gin 框架）

```bash
go get github.com/gin-gonic/gin
```

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()  // 带 Logger 和 Recovery 中间件

    // 路由组
    api := r.Group("/api")
    {
        api.GET("/todos", listTodos)
        api.GET("/todos/:id", getTodo)
        api.POST("/todos", createTodo)
        api.PUT("/todos/:id", updateTodo)
        api.DELETE("/todos/:id", deleteTodo)
    }

    r.Run(":8080")  // 默认监听 0.0.0.0:8080
}

func listTodos(c *gin.Context) {
    todos := []Todo{
        {ID: 1, Title: "学 Go", Completed: false},
    }
    c.JSON(http.StatusOK, todos)
}

func getTodo(c *gin.Context) {
    id := c.Param("id")  // 获取路径参数
    // ... 查询数据库
    c.JSON(http.StatusOK, gin.H{"id": id, "title": "学 Go"})
}

func createTodo(c *gin.Context) {
    var todo Todo
    if err := c.ShouldBindJSON(&todo); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // ... 写入数据库
    c.JSON(http.StatusCreated, todo)
}
```

**路由参数：**

```go
// /api/todos/123  →  c.Param("id") = "123"
r.GET("/api/todos/:id", getTodo)

// /api/todos/123/comments/456  →  c.Param("id")="123", c.Param("commentId")="456"
r.GET("/api/todos/:id/comments/:commentId", getComment)
```

### 3.3 请求校验与响应格式

```go
// 请求结构体 + 校验标签
type CreateTodoRequest struct {
    Title     string `json:"title"     binding:"required,min=1,max=200"`
    Completed *bool  `json:"completed"` // 用指针才能区分"没传"和"传了 false"
}

type UpdateTodoRequest struct {
    Title     *string `json:"title"     binding:"omitempty,min=1,max=200"`
    Completed *bool   `json:"completed"`
}

// 统一响应格式
type Response struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

func Success(c *gin.Context, data interface{}) {
    c.JSON(http.StatusOK, Response{
        Code:    0,
        Message: "ok",
        Data:    data,
    })
}

func Error(c *gin.Context, httpStatus int, msg string) {
    c.JSON(httpStatus, Response{
        Code:    httpStatus,
        Message: msg,
    })
}

// 使用
func createTodo(c *gin.Context) {
    var req CreateTodoRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        Error(c, http.StatusBadRequest, "参数校验失败: "+err.Error())
        return
    }
    
    todo := Todo{
        Title:     req.Title,
        Completed: false,
    }
    // ... 保存到数据库
    
    Success(c, todo)
}
```

> **生活化类比**：统一响应格式就像**快递公司的标准面单**——不管包裹里是手机、衣服还是书，面单格式都一样（单号、收件人、地址）。前端拿到每个响应，都知道 `code` 表示状态、`data` 里是数据、`message` 是描述。不用每个接口猜结构。

### 3.4 错误处理中间件

```go
// 全局错误处理中间件
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()  // 先执行后续处理器

        // 如果有错误
        if len(c.Errors) > 0 {
            err := c.Errors.Last()
            // 区分业务错误和系统错误
            if appErr, ok := err.Err.(*AppError); ok {
                Error(c, appErr.Code, appErr.Message)
            } else {
                Error(c, http.StatusInternalServerError, "服务器内部错误")
            }
        }
    }
}

// 自定义错误类型
type AppError struct {
    Code    int
    Message string
}

func (e *AppError) Error() string {
    return e.Message
}

// 在处理器中使用
func getTodo(c *gin.Context) {
    todo, err := findTodoByID(c.Param("id"))
    if err != nil {
        _ = c.Error(&AppError{Code: 404, Message: "Todo 不存在"})
        return
    }
    Success(c, todo)
}
```

---

## 4. 数据库入门

### 4.1 SQLite 快速上手

SQLite 是**零配置**的嵌入式数据库，数据存一个文件，不需要安装服务端。非常适合学习阶段和原型开发。

```bash
go get github.com/mattn/go-sqlite3
```

```go
import (
    "database/sql"
    _ "github.com/mattn/go-sqlite3"
)

func main() {
    // 打开数据库（文件不存在会自动创建）
    db, err := sql.Open("sqlite3", "./data.db")
    if err != nil {
        panic(err)
    }
    defer db.Close()

    // 创建表
    db.Exec(`CREATE TABLE IF NOT EXISTS todos (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        completed INTEGER DEFAULT 0,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )`)

    // 插入
    result, _ := db.Exec("INSERT INTO todos (title) VALUES (?)", "学 Go")
    id, _ := result.LastInsertId()

    // 查询单条
    var todo Todo
    db.QueryRow("SELECT id, title, completed FROM todos WHERE id = ?", id).
        Scan(&todo.ID, &todo.Title, &todo.Completed)

    // 查询多条
    rows, _ := db.Query("SELECT id, title, completed FROM todos ORDER BY created_at DESC")
    defer rows.Close()
    
    var todos []Todo
    for rows.Next() {
        var t Todo
        rows.Scan(&t.ID, &t.Title, &t.Completed)
        todos = append(todos, t)
    }

    // 更新
    db.Exec("UPDATE todos SET completed = ? WHERE id = ?", true, id)

    // 删除
    db.Exec("DELETE FROM todos WHERE id = ?", id)
}
```

> **生活化类比**：SQLite 就像**随身笔记本**——不用去图书馆（数据库服务器），随时随地掏出来就能记。存一个文件，拷贝到哪都能打开。PostgreSQL 就像**图书馆管理系统**——需要专门的服务器、管理员、权限控制，但能同时服务成千上万人、支持复杂查询、数据不会丢。

### 4.2 PostgreSQL 生产实践

```bash
# macOS 安装
brew install postgresql@16
brew services start postgresql@16

# 创建数据库
createdb myapp
```

```go
import (
    "database/sql"
    _ "github.com/lib/pq"
    "time"
)

// 连接配置
func NewDB() (*sql.DB, error) {
    dsn := "host=localhost port=5432 user=postgres password=postgres dbname=myapp sslmode=disable"
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }

    // 连接池配置
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)

    // 验证连接
    if err := db.Ping(); err != nil {
        return nil, err
    }

    return db, nil
}
```

**SQL 速查（前端同学优先掌握这些）：**

```sql
-- 建表
CREATE TABLE todos (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    completed BOOLEAN DEFAULT FALSE,
    user_id INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- 增
INSERT INTO todos (title, user_id) VALUES ('学 Go', 1);

-- 查
SELECT * FROM todos WHERE user_id = 1 ORDER BY created_at DESC LIMIT 10;
SELECT * FROM todos WHERE id = 1;

-- 改
UPDATE todos SET completed = TRUE, updated_at = NOW() WHERE id = 1;

-- 删
DELETE FROM todos WHERE id = 1;

-- 联表查询
SELECT t.*, u.name AS user_name
FROM todos t
JOIN users u ON t.user_id = u.id
WHERE t.user_id = 1;

-- 索引（加速查询）
CREATE INDEX idx_todos_user_id ON todos(user_id);
```

### 4.3 GORM 入门

写 SQL 很灵活，但 CRUD 重复代码太多。GORM 是 Go 最流行的 ORM。

```bash
go get gorm.io/gorm
go get gorm.io/driver/sqlite
go get gorm.io/driver/postgres
```

```go
import (
    "gorm.io/gorm"
    "gorm.io/driver/sqlite"
)

// 模型定义
type Todo struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Title     string         `gorm:"size:200;not null" json:"title"`
    Completed bool           `gorm:"default:false" json:"completed"`
    UserID    uint           `gorm:"index" json:"user_id"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
}

func main() {
    db, _ := gorm.Open(sqlite.Open("data.db"), &gorm.Config{})
    db.AutoMigrate(&Todo{})  // 自动建表

    // CRUD
    db.Create(&Todo{Title: "学 Go", UserID: 1})           // 增
    var todo Todo
    db.First(&todo, 1)                                     // 查（按主键）
    db.Where("user_id = ?", 1).Find(&todos)                // 多条件查
    db.Model(&todo).Update("Completed", true)              // 改
    db.Delete(&todo)                                       // 删（软删除）
}
```

> **生活化类比**：手写 SQL 就像**自己做饭**——灵活、精确控制每一种调料，但麻烦。GORM 就像**料理包**——拆开微波炉加热就能吃，能应付大部分场景。但遇到复杂查询（联表、聚合、性能优化），你还是得自己写 SQL——GORM 也支持执行原生 SQL。

### 4.4 Migration 管理

数据库表结构会随着项目迭代而改变，需要一种方式来管理这些变更。

```bash
go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

```bash
# 创建 migration 文件
migrate create -ext sql -dir migrations -seq create_todos_table

# 这会生成两个文件：
# migrations/000001_create_todos_table.up.sql   (升级)
# migrations/000001_create_todos_table.down.sql (回滚)
```

**`000001_create_todos_table.up.sql`：**
```sql
CREATE TABLE todos (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

**`000001_create_todos_table.down.sql`：**
```sql
DROP TABLE IF EXISTS todos;
```

```go
// 在代码中执行 migration
import "github.com/golang-migrate/migrate/v4"

func runMigrations(db *sql.DB) error {
    m, err := migrate.NewWithDatabaseInstance(
        "file://migrations",
        "postgres",
        db,
    )
    if err != nil {
        return err
    }
    return m.Up()  // 执行所有未执行的 migration
}
```

> **生活化类比**：Migration 就像**手机的软件更新日志**——iOS 每次更新有版本号，知道当前哪个版本、下一个版本改了什么。你也可以回滚到上一个版本。没有 Migration 的话，你就像每次改数据库都"直接改线上"，连自己改了什么都不知道。

---

## 5. 认证与授权

### 5.1 密码哈希

**绝对不要明文存密码。** 用 bcrypt 哈希。

```bash
go get golang.org/x/crypto/bcrypt
```

```go
import "golang.org/x/crypto/bcrypt"

// 注册时：哈希密码
func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

// 登录时：验证密码
func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// 使用
hashed, _ := HashPassword("mysecret123")
// 存入数据库的是 $2a$10$... 这样的哈希值，不是明文

// 验证
if CheckPassword("mysecret123", hashed) {
    fmt.Println("密码正确")
}
```

> **生活化类比**：明文存密码就像**把家门钥匙放在门垫下面**——你觉得藏得好，但别人一翻就找到。bcrypt 哈希就像**粉碎机**——你把钥匙图案画在纸上，用粉碎机打成碎片。每次有人来，你让他们把钥匙图案画出来，用同样的粉碎机粉碎，然后对比碎片的排列是否一致。你永远不知道原来的图案长什么样（不可逆），但能验证对方画的对不对。

### 5.2 JWT 的签发与验证

```bash
go get github.com/golang-jwt/jwt/v5
```

```go
import (
    "time"
    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("your-secret-key-change-in-production")

// 自定义 Claims
type Claims struct {
    UserID   uint   `json:"user_id"`
    Username string `json:"username"`
    jwt.RegisteredClaims
}

// 签发 Token
func GenerateToken(userID uint, username string) (string, error) {
    claims := Claims{
        UserID:   userID,
        Username: username,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)), // 24 小时过期
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

// 验证 Token
func ParseToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return jwtSecret, nil
    })
    if err != nil {
        return nil, err
    }
    
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("无效的 token")
    }
    
    return claims, nil
}
```

**JWT 的结构：**

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxfQ.xxx
│                    │                │
Header              Payload          Signature
(Base64编码)        (Base64编码)     (防篡改签名)
```

> **注意**：Header 和 Payload 只是 Base64 编码，**不是加密**。任何人拿到 Token 都能解码看到里面的内容。所以**不要把敏感信息（密码、身份证号）放 JWT 的 Payload 里**。

> **生活化类比**：JWT 就像**演唱会门票**——票上印了你的座位号、场次、有效期（Payload），上面盖了主办方的防伪章（Signature）。你不需要每次去服务台验身份，只要亮出门票，工作人员扫一眼防伪章就知道是真的。门票过期了（ExpiresAt）自然作废。缺点是票一旦发出就不能远程作废——你没法"远程注销"一张已经发出去的纸票（这就是 JWT 的无状态特性）。

### 5.3 中间件鉴权

```go
// Gin 中间件：从请求头提取并验证 JWT
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 从 Header 获取 token
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "未提供认证信息"})
            c.Abort()
            return
        }

        // 格式：Bearer <token>
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "认证格式错误"})
            c.Abort()
            return
        }

        // 解析 token
        claims, err := ParseToken(parts[1])
        if err != nil {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "token 无效或已过期"})
            c.Abort()
            return
        }

        // 把用户信息存入上下文，后续处理器可以取出
        c.Set("userID", claims.UserID)
        c.Set("username", claims.Username)
        c.Next()
    }
}

// 使用
func main() {
    r := gin.Default()

    // 公开路由（不需要登录）
    r.POST("/api/register", register)
    r.POST("/api/login", login)

    // 需要认证的路由
    auth := r.Group("/api")
    auth.Use(AuthMiddleware())
    {
        auth.GET("/todos", listTodos)
        auth.POST("/todos", createTodo)
        auth.PUT("/todos/:id", updateTodo)
        auth.DELETE("/todos/:id", deleteTodo)
    }

    r.Run(":8080")
}

// 在处理器中获取当前用户
func createTodo(c *gin.Context) {
    userID := c.GetUint("userID")  // 从中间件注入的上下文获取
    // ...
}
```

### 5.4 Session vs Token vs OAuth2.0

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **Session** | 服务器存会话，客户端只存 session_id（Cookie） | 服务端可控，随时踢人下线 | 服务端有状态，水平扩展麻烦 |
| **JWT Token** | 服务端签发签名，客户端存储 token，每次请求带上 | 无状态，适合分布式 | 签发后无法撤销（除非加黑名单） |
| **OAuth2.0** | 第三方授权，用户在授权方（微信/Google）登录后回调 | 不需要自己管密码，用户方便 | 流程复杂，调试困难 |

> **生活化类比**：Session 就像**健身房存包柜**——你拿手牌（session_id），衣服存在柜子里（服务端）。Token 就像**驾照**——交警不需要查"你是不是有驾照"，只要驾照是真的、在有效期内，就可以开车。OAuth2.0 就像**用微信登录第三方网站**——你不需要在每一个网站注册账号，只要点"微信登录"，微信确认你是本人就放行了。

---

## 6. 文件上传

### 6.1 本地文件接收

```go
// Gin 接收文件上传
func uploadFile(c *gin.Context) {
    // 限制文件大小（32MB）
    c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, 32<<20)

    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "文件获取失败"})
        return
    }

    // 校验文件类型
    ext := strings.ToLower(filepath.Ext(file.Filename))
    allowed := map[string]bool{".jpg": true, ".jpeg": true, ".png": true, ".pdf": true}
    if !allowed[ext] {
        c.JSON(http.StatusBadRequest, gin.H{"error": "不支持的文件类型"})
        return
    }

    // 生成唯一文件名，防止覆盖和路径穿越
    filename := fmt.Sprintf("%d_%s%s", time.Now().UnixNano(), 
        uuid.New().String()[:8], ext)
    savePath := filepath.Join("./uploads", filename)

    if err := c.SaveUploadedFile(file, savePath); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "文件保存失败"})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "url":  "/uploads/" + filename,
        "name": file.Filename,
        "size": file.Size,
    })
}
```

### 6.2 对象存储与 Presigned URL

生产环境一般不会把文件存服务器本地，而是用对象存储（AWS S3 / 阿里云 OSS / MinIO）。

```go
// 使用 MinIO（开源的 S3 兼容对象存储）
import "github.com/minio/minio-go/v7"

func GeneratePresignedUploadURL(client *minio.Client, bucket, objectName string) (string, error) {
    return client.PresignedPutObject(context.Background(), bucket, objectName, 15*time.Minute)
}
```

**Presigned URL 的工作流程：**

```
1. 前端请求后端："我要上传一个文件"
2. 后端生成一个 Presigned URL（有时效的一次性上传链接）返回给前端
3. 前端直接 PUT 文件到 Presigned URL（文件直传，不经过后端）
4. 上传完成后前端通知后端："文件已上传，对象名为 xxx"
```

> **生活化类比**：本地文件存储就像**把文件放在自己办公桌抽屉里**——方便但空间有限，同事实名拿不到。对象存储就像**把文件放在公司档案室**——空间大、有专人管理、有权限控制。Presigned URL 就像**档案室的一次性临时门禁卡**——你给访客一张 15 分钟有效的门禁卡，他可以直接进档案室放文件，时间一到卡就失效。文件不需要经过你转手，你只管发卡。

---

## 7. 前后端联调关键概念

### 7.1 CORS 深入

当你的前端跑在 `localhost:5173`（Vite），后端跑在 `localhost:8080`，浏览器会因为**同源策略**（协议+域名+端口三要素）阻止跨域请求。

```go
// Gin CORS 中间件
import "github.com/gin-contrib/cors"

func main() {
    r := gin.Default()

    r.Use(cors.New(cors.Config{
        AllowOrigins:     []string{"http://localhost:5173", "https://myapp.com"},
        AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
        AllowCredentials: true,  // 允许带 Cookie
        MaxAge:           12 * time.Hour,  // 预检请求缓存时间
    }))

    r.Run(":8080")
}
```

**什么是预检请求（Preflight）？**

当请求是"非简单请求"（如 `Content-Type: application/json`、带 `Authorization` 头、`PUT`/`DELETE` 方法），浏览器会先发一个 `OPTIONS` 请求，询问服务器"你允不允许这个跨域请求？"。服务器返回 `Access-Control-Allow-*` 头后，浏览器才会发真正的请求。

> **生活化类比**：预检请求就像**出国前办签证**——你不会直接买机票飞到目的地，而是先去大使馆申请签证（OPTIONS），领事馆确认"这个国家允许你入境"（Access-Control-Allow-Origin），你才买机票出发（真正的 POST 请求）。简单请求（GET 不带自定义头）就像**免签国家**——直接去就行。

### 7.2 Cookie 跨域

Cookie 默认只在同源（Same Origin）下发送。要让跨域请求也带 Cookie：

**前端需要：**
```javascript
// fetch
fetch('https://api.example.com/data', {
  credentials: 'include',  // 关键：告诉浏览器带上 Cookie
})

// axios
axios.defaults.withCredentials = true
```

**后端需要：**
```go
// CORS 中间件中
AllowCredentials: true,  // 允许携带凭证
AllowOrigins: []string{"https://myapp.com"},  // 必须指定具体域名，不能是 *
```

**Cookie 安全属性：**

| 属性 | 作用 |
|------|------|
| `HttpOnly` | JS 无法读取，防止 XSS 窃取 |
| `Secure` | 仅 HTTPS 发送 |
| `SameSite` | `Strict`（同站才发）、`Lax`（允许从外部链接跳转时发）、`None`（跨站也发，但必须配合 `Secure`） |

### 7.3 API 设计规范

**RESTful 命名约定：**

```
GET    /api/todos          → 获取 Todo 列表
GET    /api/todos/123      → 获取 id=123 的 Todo
POST   /api/todos          → 创建 Todo
PUT    /api/todos/123      → 完整替换 id=123 的 Todo
PATCH  /api/todos/123      → 部分更新 id=123 的 Todo
DELETE /api/todos/123      → 删除 id=123 的 Todo
```

**分页：**

```go
// 请求
// GET /api/todos?page=1&page_size=20

type Pagination struct {
    Page     int `form:"page"      binding:"min=1"`
    PageSize int `form:"page_size" binding:"min=1,max=100"`
}

// 响应
type PaginatedResponse struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data"`
    Meta    PageMeta    `json:"meta"`
}

type PageMeta struct {
    Page       int   `json:"page"`
    PageSize   int   `json:"page_size"`
    Total      int64 `json:"total"`
    TotalPages int   `json:"total_pages"`
}
```

**错误码体系：**

```go
// 统一错误码
const (
    ErrCodeInvalidParam  = 40001  // 参数校验失败
    ErrCodeUnauthorized  = 40100  // 未登录
    ErrCodeTokenExpired  = 40101  // token 过期
    ErrCodeForbidden     = 40300  // 无权限
    ErrCodeNotFound      = 40400  // 资源不存在
    ErrCodeConflict      = 40900  // 资源冲突（如重复注册）
    ErrCodeInternalError = 50000  // 服务器内部错误
)
```

> **生活化类比**：好的 API 设计就像**好的路标**——你不需要问路，看到路标就知道左转去哪、右转去哪。一致的命名规范（`/api/todos` 而不是 `/api/get_todos`）、统一的响应格式、清晰的错误码，让前后端对接时不需要反复沟通"这个接口返回什么、那个错误码是什么意思"。

### 7.4 接口文档

Go 生态最流行的方案是用 `swaggo/swag` 通过注释自动生成 Swagger 文档。

```go
// @Summary      获取 Todo 列表
// @Description  返回当前用户的所有 Todo，支持分页
// @Tags         todos
// @Accept       json
// @Produce      json
// @Param        page       query     int     false  "页码"        default(1)
// @Param        page_size  query     int     false  "每页数量"    default(20)
// @Success      200  {object}  Response
// @Router       /api/todos [get]
// @Security     BearerAuth
func listTodos(c *gin.Context) {
    // ...
}
```

```bash
# 安装 swag
go install github.com/swaggo/swag/cmd/swag@latest

# 生成文档
swag init

# 访问 http://localhost:8080/swagger/index.html
```

---

## 附录：一个完整的 Todo API

把以上所有知识串起来，一个可直接运行的 Todo API 项目结构：

```
myapi/
├── go.mod
├── go.sum
├── main.go
├── config/
│   └── config.go
├── model/
│   ├── todo.go
│   └── user.go
├── handler/
│   ├── todo.go
│   └── user.go
├── middleware/
│   └── auth.go
├── repository/
│   └── todo.go
└── migrations/
    ├── 000001_create_users.up.sql
    ├── 000001_create_users.down.sql
    ├── 000002_create_todos.up.sql
    └── 000002_create_todos.down.sql
```

**`main.go` — 完整入口：**

```go
package main

import (
    "log"
    "github.com/gin-gonic/gin"
    "github.com/gin-contrib/cors"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

func main() {
    // 1. 初始化数据库
    db, err := gorm.Open(sqlite.Open("data.db"), &gorm.Config{})
    if err != nil {
        log.Fatal("数据库连接失败:", err)
    }
    db.AutoMigrate(&model.User{}, &model.Todo{})

    // 2. 初始化仓库、处理器
    todoRepo := repository.NewTodoRepo(db)
    userRepo := repository.NewUserRepo(db)
    todoHandler := handler.NewTodoHandler(todoRepo)
    userHandler := handler.NewUserHandler(userRepo)

    // 3. 配置路由
    r := gin.Default()
    r.Use(cors.New(cors.Config{
        AllowOrigins:     []string{"http://localhost:5173"},
        AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
        AllowCredentials: true,
    }))

    // 公开路由
    r.POST("/api/register", userHandler.Register)
    r.POST("/api/login", userHandler.Login)

    // 需要认证的路由
    auth := r.Group("/api")
    auth.Use(middleware.AuthMiddleware())
    {
        auth.GET("/todos", todoHandler.List)
        auth.POST("/todos", todoHandler.Create)
        auth.GET("/todos/:id", todoHandler.Get)
        auth.PUT("/todos/:id", todoHandler.Update)
        auth.DELETE("/todos/:id", todoHandler.Delete)
    }

    log.Println("服务启动在 http://localhost:8080")
    r.Run(":8080")
}
```

---

## 检验清单

在不看笔记的情况下能解释清楚并动手实现：

- [ ] Go 的值类型和引用类型有哪些？`:=` 和 `var` 的区别？
- [ ] 值接收者和指针接收者的区别？什么时候用哪个？
- [ ] Go 怎么处理错误？和 `try/catch` 的本质区别是什么？
- [ ] goroutine 和 channel 分别解决什么问题？
- [ ] 用 Gin 从零写出一个 CRUD API（路由、参数绑定、响应格式）
- [ ] 中间件的工作原理？写一个 JWT 鉴权中间件
- [ ] SQL 的 CRUD 怎么写？联表查询的语法？
- [ ] GORM 的 AutoMigrate 做了什么？和手写 migration 的区别？
- [ ] bcrypt 哈希为什么不可逆？JWT 的 Payload 为什么不能放敏感信息？
- [ ] JWT 的签发和验证流程能自己写出来吗？
- [ ] CORS 什么时候会触发预检请求？`AllowCredentials: true` 时为什么不能配 `*`？
- [ ] 文件上传的三个安全要点：大小限制、类型校验、文件名防篡改
- [ ] Presigned URL 的工作流程？比直接上传到后端好在哪？
- [ ] 能画出用户注册 → 登录 → 带 Token 请求 → 后端验证 → 返回数据的完整时序图吗？
- [ ] 从零搭一个 Todo API，用 Postman 测通 CRUD + 登录

---

> **下一步**：完成后端 API 后，进入 [Phase 3：工程化与项目部署](../README.md#phase-3工程化与项目部署) —— 把写好的代码部署到公网，让任何人都能访问。