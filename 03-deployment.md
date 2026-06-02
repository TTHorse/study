# 工程化与项目部署

> 作为前端开发，你习惯了 `npm run dev` 然后在浏览器里看到页面。但那个 `localhost:5173` 只有你自己能访问。这一阶段的目标是**把代码变成互联网上任何人都能访问的服务**——从一个 Go 写的 Todo API 开始，逐步推进到 Docker 容器化 + 自动部署。

---

## 示例项目

本文全程使用同一个项目作为例子——Phase 2 中写的 **Todo API**：

```
myapi/
├── go.mod
├── go.sum
├── main.go           # Gin 框架，提供 /api/todos CRUD
├── config/
├── handler/
├── model/
├── middleware/
├── repository/
└── migrations/
```

- 后端：Go + Gin，监听 `:8080`
- 数据库：SQLite（文件 `data.db`，后续升级为 PostgreSQL）
- 前端（后续加入）：Vite + React，监听 `:5173`

---

## 目录

- [1. 部署全景图](#1-部署全景图)
  - [1.1 代码从本地到线上经历了什么](#11-代码从本地到线上经历了什么)
  - [1.2 部署方式的演进路径](#12-部署方式的演进路径)
- [2. PaaS 平台部署——最快让服务上线](#2-paas-平台部署最快让服务上线)
  - [2.1 Railway 部署 Go 项目](#21-railway-部署-go-项目)
  - [2.2 环境变量管理](#22-环境变量管理)
  - [2.3 看到你的服务有了公网 URL](#23-看到你的服务有了公网-url)
- [3. 云服务器部署——从买机器到上线](#3-云服务器部署从买机器到上线)
  - [3.1 买一台云服务器](#31-买一台云服务器)
  - [3.2 SSH 远程登录](#32-ssh-远程登录)
  - [3.3 Linux 基础命令速查](#33-linux-基础命令速查)
  - [3.4 手动部署 Todo API](#34-手动部署-todo-api)
  - [3.5 进程守护——systemd 保活](#35-进程守护systemd-保活)
  - [3.6 Nginx 反向代理](#36-nginx-反向代理)
  - [3.7 域名 + HTTPS](#37-域名--https)
- [4. Docker 容器化](#4-docker-容器化)
  - [4.1 镜像 vs 容器](#41-镜像-vs-容器)
  - [4.2 写 Dockerfile 打包 Todo API](#42-写-dockerfile-打包-todo-api)
  - [4.3 多阶段构建——让镜像更小](#43-多阶段构建让镜像更小)
  - [4.4 Docker Compose 编排多服务](#44-docker-compose-编排多服务)
  - [4.5 端口映射、数据卷、网络](#45-端口映射数据卷网络)
- [5. CI/CD——代码 push 自动部署](#5-cicd代码-push-自动部署)
  - [5.1 GitHub Actions 是什么](#51-github-actions-是什么)
  - [5.2 自动测试 + 构建](#52-自动测试--构建)
  - [5.3 push 后自动部署到服务器](#53-push-后自动部署到服务器)
  - [5.4 环境区分](#54-环境区分)
- [6. 监控与排错](#6-监控与排错)
  - [6.1 日志管理](#61-日志管理)
  - [6.2 常见故障排查](#62-常见故障排查)
- [附录：从零到上线完整操作清单](#附录从零到上线完整操作清单)
- [检验清单](#检验清单)

---

## 1. 部署全景图

### 1.1 代码从本地到线上经历了什么

在我开始之前，先建立整体认知。以我们的 Todo API 为例：

```
你的电脑（macOS）                   互联网                         用户
┌─────────────────┐              ┌──────────┐              ┌─────────┐
│  go build        │              │  GitHub  │              │  浏览器  │
│  ↓               │   git push   │  Actions │              │         │
│  二进制文件       │ ──────────→  │  ↓       │              │  GET    │
│  (myapi)         │              │  构建测试 │              │  /api/  │
│                  │              │  ↓       │              │  todos  │
│  Docker build    │              │  部署    │              │         │
│  ↓               │              │  ↓       │              └────┬────┘
│  镜像            │              │  服务器   │ ←──────────────┘
└─────────────────┘              │  运行中   │    https://api.todo.com
                                  └──────────┘
```

**完整链路：**

```
1. 本地开发    →  写代码、go run、curl 测试
2. git push    →  代码推到 GitHub
3. CI 检查     →  GitHub Actions 自动跑 go test、go vet
4. 构建产物    →  go build 生成二进制 / docker build 生成镜像
5. 部署        →  把产物推到服务器并启动
6. 域名解析    →  DNS 把 api.todo.com 指向服务器 IP
7. Nginx 反代  →  接收 HTTPS 请求，转发给 :8080
8. 用户访问    →  浏览器访问 https://api.todo.com/api/todos
```

> **生活化类比**：部署就像**开一家实体餐厅**。本地开发是在家试菜（只有家人能吃到）。PaaS 部署就像入驻**美食广场**——商场（平台）给你提供场地、水电、卫生许可，你只负责把菜品（代码）放上去。云服务器部署就像**自己租店面**——装修、水电、消防、营业执照全部自己搞定，但完全自主可控。Docker 就像**把每道菜的操作规程标准化成手册**——不管在哪个厨房、谁来做，出品一致。CI/CD 就是**中央厨房自动配送系统**——总店研发新菜（push 代码），各分店自动更新菜单（自动部署）。

### 1.2 部署方式的演进路径

| 层级 | 方案 | 类比 | 适合场景 | 先学这个 |
|------|------|------|---------|---------|
| PaaS 平台 | Railway / Fly.io | 美食广场摊位 | 个人项目、快速验证 | ✅ 从这里开始 |
| 云服务器 | 阿里云 ECS + Nginx | 自己租店面 | 有定制需求的项目 | 第二步 |
| 容器化 | Docker + Docker Compose | 标准化操作手册 | 多服务编排、团队协作 | 第三步 |
| 编排 | Kubernetes | 连锁餐厅总部调度 | 大规模微服务 | 进阶了解 |

> **原则**：每种方案我们都用同一个 Todo API 来演示，让你看到同一个项目从简单到复杂的完整演进过程。

---

## 2. PaaS 平台部署——最快让服务上线

### 2.1 Railway 部署 Go 项目

Railway 对 Go 项目有原生支持，检测到 `go.mod` 就自动识别。**零配置**部署。

**第一步：安装 Railway CLI**

```bash
# macOS
brew install railway
```

**第二步：在项目目录初始化**

```bash
cd ~/study/myapi
railway login     # 浏览器跳转，GitHub 账号登录
railway init      # 初始化，创建一个新项目
```

**第三步：部署**

```bash
railway up        # 把当前目录代码推上去，自动构建并部署
```

Railway 会自动：
1. 检测到 `go.mod` → 识别为 Go 项目
2. 执行 `go build` → 编译
3. 运行生成的二进制文件
4. 分配一个公网域名，如 `myapi-production-xxxx.up.railway.app`

**第四步：测试**

```bash
curl https://myapi-production-xxxx.up.railway.app/api/todos
```

就这么简单。**你的 Todo API 已经在公网上了。**

但是有个问题——我们的 SQLite 数据文件每次部署都会丢失（Railway 的容器是无状态的，重启就重置）。这就是为什么后续要用 PostgreSQL（外部数据库）和 Docker Volume（持久化存储）。

> **生活化类比**：Railway 就像**共享厨房**——你只需要带食材（代码）过去，锅碗瓢盆煤气水电（运行环境）厨房全包。前台还给你挂个招牌（公网域名）。但每天打烊后厨工会把灶台清干净（容器重启数据丢失），所以珍贵食材（数据库）不能放厨房，要存到专门的冷库（外部数据库服务）。

### 2.2 环境变量管理

敏感信息（数据库密码、JWT 密钥）不能写进代码。PaaS 平台通过环境变量注入：

```go
// config/config.go
package config

import "os"

type Config struct {
    Port      string
    DBPath    string
    JWTSecret string
}

func Load() *Config {
    return &Config{
        Port:      getEnv("PORT", "8080"),           // PaaS 通常会设置 PORT
        DBPath:    getEnv("DB_PATH", "./data.db"),
        JWTSecret: getEnv("JWT_SECRET", "dev-secret"),
    }
}

func getEnv(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}
```

在 Railway 控制台 → Variables 里设置：

```
JWT_SECRET = your-real-secret-key
DB_PATH    = /data/todos.db
```

> **生活化类比**：环境变量就像**不给员工直接发保险柜密码，而是每人一个独立密码信封**。密码不贴在保险柜上（不在代码里），每个人拿到信封才知道自己的密码。换密码时只需要换信封里的纸条（改环境变量），不需要重新印刷所有文件（重新部署代码）。

### 2.3 看到你的服务有了公网 URL

部署成功后你会看到类似这样的域名：`myapi-production-xxxx.up.railway.app`。

这意味着：
- 任何人都能通过这个 URL 访问你的 API
- Railway 自动处理了 HTTPS 证书
- 自动处理了负载均衡（访问量大时自动扩容）

你可以把这个 URL 发给朋友，让他们用 Postman 或 curl 测试你的 API。

---

## 3. 云服务器部署——从买机器到上线

PaaS 很方便，但：
- 有些定制需求做不了（比如装特定版本的 Nginx 模块）
- 价格随流量增长，不如买服务器划算
- 你想真正理解"服务器"是怎么运作的

所以下一步：买一台自己的云服务器，从头搭一个生产环境。

### 3.1 买一台云服务器

**推荐配置（学习够用了）：**

| 云厂商 | 产品 | 配置 | 参考价格 |
|--------|------|------|---------|
| 阿里云 | ECS / 轻量应用服务器 | 2核 2G 内存 | ~60元/年（新用户） |
| 腾讯云 | 轻量应用服务器 | 2核 2G 内存 | ~50元/年（新用户） |
| AWS | EC2 t3.micro | 1核 1G 内存 | 免费套餐 12 个月 |
| 搬瓦工 | VPS | 1核 1G 内存 | ~$50/年 |

**购买时注意：**
- 操作系统选 **Ubuntu 22.04 LTS**（文档多、社区大、包管理方便）
- 记住 **公网 IP**（如 `47.96.xx.xx`），这是你服务器的门牌号
- 设置 root 密码或创建 SSH 密钥对

> **生活化类比**：买云服务器就是**在机房租了个机柜格位**。机房给你分配了一个编号（公网 IP），提供电源和网线，接下来每个零件都得自己组装。买轻量应用服务器（如腾讯云轻量）就像**租了个精装办公室**——除了主机，还送基础防火墙、快照备份、监控面板。你可以拎包入住。

### 3.2 SSH 远程登录

服务器在云端，你只能通过 SSH（Secure Shell）远程登录。

```bash
# 密码登录
ssh root@47.96.xx.xx
# 输入密码

# 密钥登录（更安全，推荐）
# 1. 本地生成密钥对（如果没有）
ssh-keygen -t ed25519 -C "my-server"

# 2. 把公钥复制到服务器
ssh-copy-id root@47.96.xx.xx

# 3. 之后登录就不需要密码了
ssh root@47.96.xx.xx
```

**首次登录后必做的安全配置：**

```bash
# 1. 更新系统
apt update && apt upgrade -y

# 2. 创建普通用户（不要一直用 root）
adduser myapp
usermod -aG sudo myapp

# 3. 配置 SSH 密钥给普通用户
rsync --archive --chown=myapp:myapp ~/.ssh /home/myapp

# 4. 禁止 root SSH 登录 & 禁止密码登录（仅允许密钥）
sudo vim /etc/ssh/sshd_config
# 修改：
#   PermitRootLogin no
#   PasswordAuthentication no
#   PubkeyAuthentication yes

sudo systemctl restart sshd
```

> **生活化类比**：SSH 就像**远程操控一台千里之外的电脑**。密码登录就像用门禁密码——谁输入对谁就能进，但密码可能被偷看。SSH 密钥就像**指纹锁**——你把自己的指纹（公钥）录入了门锁，只有你的手指（私钥）能开。root 用户就像**万能管理员账号**——所有门都能进，所以平时不要用，给自己建个普通账号（myapp），需要的时候再提权（sudo）。

### 3.3 Linux 基础命令速查

作为前端开发者，你可能没用过 Linux 命令行。以下是最常用的：

```bash
# 文件操作
ls -la              # 列出文件（含隐藏文件）
cd /opt/myapi       # 切换目录
pwd                 # 当前在哪
mkdir -p /opt/myapi # 创建目录
cp file1 file2      # 复制
mv old new          # 移动/重命名
rm file             # 删除文件
rm -rf dir          # 删除目录（危险！确认再确认）

# 查看文件
cat app.log         # 看全部
tail -f app.log     # 看末尾，持续刷新
less app.log        # 分页查看
head -20 app.log    # 看前 20 行

# 进程管理
ps aux | grep myapi     # 查进程
kill -9 12345           # 杀进程（按 PID）
top                     # 实时看 CPU/内存
htop                    # top 的美化版（需安装）

# 磁盘/网络
df -h                   # 磁盘剩余空间
du -sh ./uploads        # 这个目录占了多大
netstat -tlnp           # 哪些端口在监听
ss -tlnp                # 同上（新版）

# 权限
chmod +x myapi          # 加执行权限
chown myapp:myapp myapi # 改文件归属

# systemd（管理服务）
sudo systemctl start myapi    # 启动
sudo systemctl stop myapi     # 停止
sudo systemctl restart myapi  # 重启
sudo systemctl status myapi   # 看状态
sudo systemctl enable myapi   # 开机自启
sudo journalctl -u myapi -f   # 看服务日志
```

> **生活化类比**：Linux 命令对比你熟悉的 GUI 操作——`cd` 就是双击打开文件夹，`ls` 就是看一眼文件夹里有啥，`ps aux | grep` 就是打开任务管理器搜进程名，`tail -f` 就是打开一个日志窗口自动滚屏，`systemctl` 就是服务（程序）的开关按钮。学会了这 15 个命令，就能搞定 80% 的服务器日常操作。

### 3.4 手动部署 Todo API

现在把我们的 Todo API 部署到这台服务器。

**第一步：在本地编译（交叉编译）**

你的 Mac 是 ARM 芯片（M1/M2/M3），但云服务器大多是 x86_64（Intel/AMD）。需要交叉编译：

```bash
# 在本地 Mac 上，编译出能在 Linux x86_64 上运行的可执行文件
GOOS=linux GOARCH=amd64 go build -o myapi .

# 验证编译产物
file ./myapi
# 输出：myapi: ELF 64-bit LSB executable, x86-64, ...

# 把二进制文件和 SQLite 数据库文件一起传上去
scp ./myapi root@47.96.xx.xx:/opt/myapi/
scp ./data.db root@47.96.xx.xx:/opt/myapi/
```

如果不需要跨平台（比如你用 Linux 开发），直接在服务器上编译也行：

```bash
# 在服务器上
# 先装 Go
sudo snap install go --classic

# clone 代码
cd /opt
git clone https://github.com/yourname/myapi.git
cd myapi

# 编译
go build -o myapi .
```

**第二步：在服务器上运行**

```bash
ssh root@47.96.xx.xx

# 第一次手动启动，看看有没有报错
cd /opt/myapi
./myapi

# 输出：
# 服务启动在 http://localhost:8080
```

**第三步：测试**

```bash
# 在服务器上另开一个终端
curl http://localhost:8080/api/todos
```

但这台服务器外网还访问不了——8080 端口可能被防火墙挡住了。

**第四步：开防火墙**

```bash
# 云服务商控制台通常有安全组（Security Group），确保 80 和 443 端口对外开放
# 服务器上的防火墙也要配：
sudo ufw allow 22     # SSH，不能关
sudo ufw allow 80     # HTTP
sudo ufw allow 443    # HTTPS
sudo ufw enable

# 查看防火墙状态
sudo ufw status
```

> **注意**：Go 程序直接监听 `0.0.0.0:8080`，在生产环境不安全（没有 HTTPS、没有静态文件服务、端口太随意）。正确的做法是用 Nginx 做反向代理，这也是下一节要做的。

### 3.5 进程守护——systemd 保活

现在你的 `./myapi` 是手动启动的，SSH 断开后进程就没了。需要 systemd 来守护：

```bash
sudo vim /etc/systemd/system/myapi.service
```

```ini
[Unit]
Description=Todo API Service
After=network.target

[Service]
Type=simple
User=myapp
WorkingDirectory=/opt/myapi
ExecStart=/opt/myapi/myapi
Restart=always           # 挂了自动重启
RestartSec=5             # 挂后等 5 秒再重启
Environment="PORT=8080"
Environment="DB_PATH=/opt/myapi/data.db"
Environment="JWT_SECRET=your-production-secret"

# 日志
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapi

# 安全加固
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/opt/myapi

[Install]
WantedBy=multi-user.target
```

```bash
# 启动服务
sudo systemctl daemon-reload    # 重载配置
sudo systemctl enable myapi     # 开机自启
sudo systemctl start myapi      # 启动

# 查看状态
sudo systemctl status myapi

# 查看日志
sudo journalctl -u myapi -f     # -f 持续输出
sudo journalctl -u myapi --since "10 minutes ago"
```

现在你的 Todo API：
- 挂了自动重启
- 服务器重启后自动启动
- 日志统一管理

> **生活化类比**：手动 `./myapi` 就像**你在店里亲自看着，累了也不能离开**——你走了店就关门。systemd 就像**雇了个店长**——你交代好营业时间、经营范围（service 配置），店长自己开门、关门、处理突发事件（挂了重启），你只需要定期看营业报表（journalctl 日志）。店长的工资是免费的（systemd 是操作系统自带的）。

### 3.6 Nginx 反向代理

现在 Todo API 监听 `:8080`，但用户访问的是 `https://api.todo.com`（端口 443）。需要 Nginx 在中间转发。

```
用户浏览器
  │  https://api.todo.com/api/todos
  ▼
Nginx (监听 443)
  │  匹配 /api/ 路径
  │  转发到 http://localhost:8080
  ▼
Todo API (监听 8080)
  │  处理请求
  │  返回 JSON
  ▼
Nginx → HTTPS → 用户浏览器
```

**第一步：安装 Nginx**

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx

# 访问 http://47.96.xx.xx，应该看到 Nginx 欢迎页
```

**第二步：配置反向代理**

```bash
sudo vim /etc/nginx/sites-available/myapi
```

```nginx
server {
    listen 80;
    server_name api.todo.com;    # 换成你的域名

    # 日志
    access_log /var/log/nginx/myapi_access.log;
    error_log  /var/log/nginx/myapi_error.log;

    # 客户端请求体大小限制
    client_max_body_size 10M;

    # 反向代理到 Go 服务
    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;

        # 重要：把客户端的真实 IP 和协议传给后端
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时配置
        proxy_connect_timeout 30s;
        proxy_read_timeout    60s;
        proxy_send_timeout    60s;
    }

    # 如果前端是 SPA，配静态文件服务
    # location / {
    #     root /opt/myapp/frontend/dist;
    #     try_files $uri $uri/ /index.html;
    # }
}
```

```bash
# 启用配置
sudo ln -s /etc/nginx/sites-available/myapi /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default   # 去掉默认配置

# 检查配置语法
sudo nginx -t

# 重新加载配置（平滑重启，不会断开现有连接）
sudo nginx -s reload
```

现在 `http://47.96.xx.xx/api/todos` 就能访问了。

> **生活化类比**：Nginx 反向代理就像**餐厅的前台服务员**——顾客直接跟服务员点单，服务员把单子传给后厨（Go 服务），菜做好后再端给顾客。顾客不需要知道厨房在哪（不知道 8080 端口），甚至连厨房换了厨师都不知道（后端 IP 变了只需要改 Nginx 配置）。同时服务员还可以帮你多做好几件事——检查菜单上的菜今天有没有（静态文件服务）、限制每个人点太多（速率限制）、帮你把点单记录存档（访问日志）。

### 3.7 域名 + HTTPS

买域名并配置好 HTTPS，从 `http://47.96.xx.xx` 变成 `https://api.todo.com`。

**买域名：**

去任意域名注册商（阿里云万网 / 腾讯云 DNSPod / Namecheap / Cloudflare）买一个。学习用途推荐 Namecheap 或 Cloudflare，不需要国内实名认证。

假设你买了 `todoproject.com`。

**配 DNS：**

在域名控制台添加 A 记录：

| 类型 | 主机记录 | 记录值 |
|------|---------|--------|
| A | `api` | `47.96.xx.xx` |

还可以加一条：

| 类型 | 主机记录 | 记录值 |
|------|---------|--------|
| A | `@` | `47.96.xx.xx` |

**等 DNS 生效（几分钟到几十分钟）：**

```bash
# 检查 DNS 是否解析成功
dig api.todoproject.com
# 或用
nslookup api.todoproject.com
```

**申请 HTTPS 证书（Let's Encrypt + certbot）：**

```bash
# 安装 certbot
sudo apt install certbot python3-certbot-nginx -y

# 一键申请证书（certbot 自动修改 Nginx 配置）
sudo certbot --nginx -d api.todoproject.com

# 按提示输入邮箱，同意条款
# 成功后访问 https://api.todoproject.com/api/todos
```

**证书自动续期：**

```bash
# certbot 自带定时任务，可以测试一下续期是否正常
sudo certbot renew --dry-run

# certbot 会自动在系统 crontab 或 systemd timer 里设置续期任务
sudo systemctl status certbot.timer
```

> **生活化类比**：域名就是**给 IP 这个难记的"经纬度坐标"起了个名字**。"47.96.xx.xx"是"北纬 xx 度，东经 xx 度"，"api.todoproject.com"是"北京市朝阳区xxx路 1 号"。HTTPS 证书就是**政府颁发的营业执照**——你开一家店（网站），办了营业执照（SSL 证书），顾客看到门口挂着执照（浏览器小锁）才敢进来消费。Let's Encrypt 就是**免费工商注册**——不花钱，但每 90 天要年审一次（自动续期），证明这家店还在营业。

---

## 4. Docker 容器化

手动部署的痛点：
- 每台服务器都要装 Go、配环境、手动管理依赖
- 开发环境（macOS）和生产环境（Linux）不同，经常出现"本地没问题，服务器上挂了"
- 数据库、缓存、后端服务分散管理，启动顺序靠人工保证

Docker 解决的就是**"在我电脑上能跑"问题**。

### 4.1 镜像 vs 容器

这两个词是 Docker 的核心概念，必须分清楚：

| 概念 | 说人话 | 类比 |
|------|--------|------|
| **镜像 (Image)** | 一个只读的模板，包含运行应用所需的全部环境 | 安装包（`.dmg`/`.exe`） |
| **容器 (Container)** | 镜像的运行实例，有自己的文件系统、网络、进程空间 | 安装后跑起来的程序 |
| **Dockerfile** | 制作镜像的配方文件 | 安装说明文档 |
| **Docker Compose** | 编排多个容器的配置文件 | 乐谱（指挥多个乐器同时演奏） |

**关键理解：** 一个镜像可以启动多个容器，就像你用同一个 Word 安装包在公司和家里的电脑上各装了一份，各自运行互不影响。

> **生活化类比**：镜像是**菜谱**——写上用料、步骤、火候。容器是**正在炒的一盘菜**——根据菜谱做出来的。同一份菜谱可以同时炒出很多盘菜。Dockerfile 就是写菜谱的过程——"先放油、再放菜、加盐、翻炒 3 分钟"。Docker Compose 就是**宴会菜单**——前菜、主菜、甜点的制作顺序和搭配关系。

### 4.2 写 Dockerfile 打包 Todo API

在项目根目录创建 `Dockerfile`（无后缀名）：

```dockerfile
# ---- 基础镜像 ----
FROM golang:1.22-alpine AS builder

# 设置工作目录
WORKDIR /app

# 先复制依赖文件（利用 Docker 缓存层，依赖不变就不会重新下载）
COPY go.mod go.sum ./
RUN go mod download

# 复制源代码
COPY . .

# 编译：生成静态链接的二进制文件
RUN CGO_ENABLED=0 GOOS=linux go build -o /myapi .

# ---- 运行阶段 ----
FROM alpine:3.19

# 安装 ca-certificates（HTTPS 请求需要）
# 创建非 root 用户
RUN apk add --no-cache ca-certificates tzdata && \
    adduser -D -g '' appuser

WORKDIR /app

# 从构建阶段复制二进制文件
COPY --from=builder /myapi .

# 切换到非 root 用户
USER appuser

# 暴露端口（文档作用，实际端口映射在 docker run / compose 中指定）
EXPOSE 8080

# 启动命令
CMD ["./myapi"]
```

**`.dockerignore`（排除不需要打包的文件）：**

```
.git
.env
*.log
data.db
uploads/
node_modules/
```

**构建并运行：**

```bash
# 在项目根目录（有 Dockerfile 的地方）
docker build -t myapi:latest .

# 查看镜像
docker images | grep myapi

# 运行容器
docker run -d \
  --name myapi \
  -p 8080:8080 \
  -e JWT_SECRET=production-secret \
  -v $(pwd)/data:/app/data \
  myapi:latest

# 查看运行状态
docker ps
docker logs myapi -f
```

> **生活化类比**：`docker build` 就是**按照菜谱做了一份预制菜**（镜像），真空包装好。`docker run` 就是把预制菜拿出来**微波加热**（启动容器），马上就能吃。`-p 8080:8080` 就是"把你的厨房 8080 号窗口对外营业"，外面的人从这个窗口取餐。`-v` 就是把**冰箱插上电**——容器删了冰箱还在，数据不会丢。

### 4.3 多阶段构建——让镜像更小

上面的 Dockerfile 用了**多阶段构建（Multi-stage Build）**。对比一下：

| 方案 | 镜像大小 | 说明 |
|------|---------|------|
| 直接用 `golang:1.22` 跑 | ~800MB | 包含了 Go 编译器、源码、所有工具链 |
| 多阶段构建（如上） | ~15MB | 只保留编译好的二进制 + alpine 基础系统 |

**`FROM golang:1.22-alpine AS builder`** 只在构建阶段用（有编译器），最后 `FROM alpine:3.19` 是全新的轻量系统，只从构建阶段拷走编译产物。

> **生活化类比**：多阶段构建就是**工厂只把成品发给超市**——工厂（builder 阶段）里有全套设备（编译器）、原材料（源码）、半成品，占地面积大。但发给超市（最终镜像）的只有包装好的成品（二进制文件），不带走任何生产设备。

### 4.4 Docker Compose 编排多服务

一个完整的 Todo 应用需要好几个服务配合：

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Nginx      │ ←→ │  Todo API    │ ←→ │  PostgreSQL  │
│  (前端+反代)  │    │  (Go :8080)  │    │  (:5432)     │
└──────────────┘    └──────────────┘    └──────────────┘
```

**项目根目录创建 `docker-compose.yml`：**

```yaml
version: "3.8"

services:
  # PostgreSQL 数据库
  db:
    image: postgres:16-alpine
    container_name: myapi-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: ${DB_PASSWORD:-devpassword}
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data   # 数据持久化
    ports:
      - "5432:5432"                        # 开发时方便直连，生产可注释掉
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Go Todo API
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: myapi-api
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy         # 等数据库就绪再启动
    environment:
      PORT: "8080"
      DB_HOST: db                          # 容器间通过服务名通信
      DB_PORT: "5432"
      DB_USER: myapp
      DB_PASSWORD: ${DB_PASSWORD:-devpassword}
      DB_NAME: myapp
      JWT_SECRET: ${JWT_SECRET:-dev-secret}
    ports:
      - "8080:8080"
    volumes:
      - ./uploads:/app/uploads

  # Nginx 反向代理
  nginx:
    image: nginx:alpine
    container_name: myapi-nginx
    restart: unless-stopped
    depends_on:
      - api
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro   # 只读挂载配置
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro

volumes:
  pgdata:   # 命名卷，数据不会丢
```

**一键启动全部服务：**

```bash
# 启动（-d 后台运行）
docker compose up -d

# 查看所有服务状态
docker compose ps

# 查看日志
docker compose logs -f api    # 只看 api 服务的日志
docker compose logs -f        # 看所有服务日志

# 停止
docker compose down

# 停止并删除数据卷（重置数据库）
docker compose down -v
```

> **生活化类比**：Docker Compose 就是**管弦乐队的指挥**——小提琴什么时候进、大提琴什么时候停、鼓点跟着谁的节奏，全部在乐谱（docker-compose.yml）里写好了。`depends_on` 就是"大提琴必须等小提琴调好音才能开始"（API 等数据库就绪）。一个手势（`docker compose up`），全员开始演奏。散场时一个手势（`docker compose down`），全员收工。

### 4.5 端口映射、数据卷、网络

这三个概念最容易踩坑：

**端口映射：**

```yaml
ports:
  - "8080:8080"   # 左边是宿主机端口，右边是容器内端口
  - "80:80"
```

访问 `http://服务器IP:8080` → Docker 转发到容器的 `8080` 端口。两个端口可以不一样，比如 `"9000:8080"` 表示外部 9000 映射到内部 8080。

**数据卷：**

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data    # 命名卷（推荐）：Docker 管理，不用担心路径
  - ./uploads:/app/uploads             # 绑定挂载：直接把宿主机目录挂进去
```

容器删除后：
- 命名卷的数据还在（`docker compose down` 不会删，`-v` 才会）
- 绑定挂载的数据在宿主机目录里，一直在

**容器间网络：**

Docker Compose 自动创建一个网络，容器之间通过**服务名**通信：

```go
// config/config.go — 数据库连接地址直接写服务名
dbHost := os.Getenv("DB_HOST")  // "db"，不是 "localhost"
dsn := fmt.Sprintf("host=%s user=%s password=%s dbname=%s sslmode=disable",
    dbHost, dbUser, dbPassword, dbName)
```

`db:5432` 在 API 容器内部就能访问到，不需要知道 PostgreSQL 容器的 IP。

> **生活化类比**：端口映射就像**分机总机系统**——外部拨打总机号码 80（"接前台"），总机转接到内部分机 80（Nginx 容器）。数据卷就像**外接硬盘**——你的笔记本电脑（容器）随时可能坏，但数据在硬盘（卷）里，换个电脑插上就能用。容器间网络就像**办公室内线电话**——拨个分机号（服务名 `db`）就能找到同事，不需要知道他今天坐在哪个工位（IP 地址可能变化）。

---

## 5. CI/CD——代码 push 自动部署

### 5.1 GitHub Actions 是什么

GitHub Actions 是 GitHub 内置的 CI/CD 工具。你在仓库里创建一个 `.github/workflows/xxx.yml` 文件，GitHub 就会在你 push 代码时自动执行你定义的任务。

**基本概念：**

```
GitHub Actions
├── Workflow     — 一个自动化流程（文件级别）
│   ├── Trigger  — 什么时候触发（push / PR / 定时）
│   ├── Job      — 一个运行环境（可以并行多个 Job）
│   │   ├── Step — 每个 Step 是一组命令或一个 Action
│   │   │   ├── actions/checkout   — 拉代码
│   │   │   ├── actions/setup-go   — 装 Go 环境
│   │   │   ├── run: go test ./...  — 执行测试
│   │   │   └── run: go build       — 构建
```

> **生活化类比**：GitHub Actions 就像**自动档汽车的流水线质检机器人**——车身焊接完成（代码 push），机器人自动检查焊点（跑测试），不合格就打回去（CI 标红），合格的就自动喷漆并送到停车场（部署到服务器）。不用人工盯着，全部自动化。

### 5.2 自动测试 + 构建

**`.github/workflows/ci.yml`：**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Test & Build
    runs-on: ubuntu-latest

    services:
      # 测试需要 PostgreSQL
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: myapp_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: 拉取代码
        uses: actions/checkout@v4

      - name: 安装 Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.22"

      - name: 下载依赖
        run: go mod download

      - name: 代码规范检查
        run: go vet ./...

      - name: 运行测试
        run: go test ./... -v -race -coverprofile=coverage.out
        env:
          DB_HOST: localhost
          DB_PORT: "5432"
          DB_USER: test
          DB_PASSWORD: test
          DB_NAME: myapp_test
          JWT_SECRET: test-secret

      - name: 构建检查（确保能编译通过）
        run: go build -o /dev/null .
```

**这个 workflow 做了：**

1. 每次 push 到 main 或提 PR 时自动触发
2. 启动一个临时 PostgreSQL 容器给测试用
3. `go vet` 检查代码规范问题
4. `go test -race` 运行测试并检查并发竞争
5. 最后确保 `go build` 能通过

### 5.3 push 后自动部署到服务器

CI 通过后，自动部署到服务器。

**方案选择：**

| 方案 | 原理 | 适合 |
|------|------|------|
| SSH + rsync | GitHub Actions 通过 SSH 登录服务器，传文件重启 | 简单直接 |
| Docker + 自托管 Registry | push 镜像 → 服务器 pull 镜像 → 重新运行 | Docker 化部署 |
| Webhook 触发 | CI 发一个 HTTP 请求，服务器上监听然后拉代码部署 | 灵活自定义 |

这里用**最直接的方式——SSH 部署 Docker Compose**：

**`.github/workflows/deploy.yml`：**

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch: # 允许手动触发

jobs:
  deploy:
    name: Deploy to Server
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'  # 只有 main 分支才部署

    steps:
      - name: 拉取代码
        uses: actions/checkout@v4

      - name: 部署到服务器
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/myapi
            git pull origin main

            # 重新构建并启动（--build 确保用最新代码构建镜像）
            docker compose up -d --build

            # 清理旧镜像
            docker image prune -f

      - name: 健康检查
        run: |
          sleep 5
          curl -f https://${{ secrets.DOMAIN }}/api/todos || exit 1
```

**配置 GitHub Secrets（Settings → Secrets and variables → Actions）：**

| Secret | 值 |
|--------|-----|
| `SERVER_HOST` | `47.96.xx.xx` |
| `SERVER_USER` | `myapp` |
| `SERVER_SSH_KEY` | `-----BEGIN OPENSSH PRIVATE KEY-----...`（~/.ssh/id_ed25519 的内容） |
| `DOMAIN` | `api.todoproject.com` |

**完整 CI/CD 流程：**

```
本地 git push origin main
  ↓
GitHub Actions: CI workflow 触发
  ├─ go vet     → 通过 ✓
  ├─ go test    → 通过 ✓
  └─ go build   → 通过 ✓
  ↓
GitHub Actions: Deploy workflow 触发
  ├─ SSH 到服务器
  ├─ git pull
  ├─ docker compose up -d --build
  └─ curl 健康检查 → HTTP 200 ✓
  ↓
https://api.todoproject.com 已是最新版本
```

> **生活化类比**：CI/CD 就是**全自动装配线**——你提交设计图纸（git push），产线机器人自动检查图纸有没有问题（CI），没问题就自动生产、质检、打包、发货（CD）。你唯一要做的就是画好图纸并提交。如果图纸有问题，机器人会立刻告诉你"图纸第 3 页有冲突"（CI 标红），你修好再提交就行。

### 5.4 环境区分

真实项目不会只有一套环境：

| 环境 | 用途 | 数据库 | 部署触发 |
|------|------|--------|---------|
| **dev** | 本地开发 | 本地 SQLite / Docker PostgreSQL | 手动 |
| **staging** | 上线前验证 | 独立的测试库，数据接近生产 | PR 合并到 main 时自动 |
| **production** | 正式服务 | 生产数据库，有备份 | tag 推送（`v1.0.0`）时手动/自动 |

**环境变量分离：**

```bash
# .env.dev（本地开发）
DB_HOST=localhost
DB_PASSWORD=devpassword
JWT_SECRET=dev-secret

# .env.production（生产 - 这个文件不能进 git）
DB_HOST=db
DB_PASSWORD=super-secret-real-password
JWT_SECRET=production-128bit-random-secret
```

```yaml
# docker-compose.yml — 通过 --env-file 切换环境
# 开发环境
# docker compose --env-file .env.dev up -d

# 生产环境
# docker compose --env-file .env.production up -d
```

**生产环境的安全底线：**

- [ ] `.env.production` **绝对不能进 git**（加入 `.gitignore`）
- [ ] 数据库密码、JWT 密钥等用足够长的随机字符串，不同环境完全不同
- [ ] 服务器上只开放 80、443、22 端口（通过安全组/防火墙限制）
- [ ] 使用非 root 用户运行服务
- [ ] 定期 `apt update && apt upgrade` 给系统打补丁

---

## 6. 监控与排错

上线后的服务不是丢上去就不管了。你需要知道它是否活着、是否健康、出错时怎么排查。

### 6.1 日志管理

**Go 服务日志：**

```go
// 用标准库 log，输出到 stdout（Docker/systemd 自动收集）
import "log"

func main() {
    log.SetFlags(log.LstdFlags | log.Lshortfile)
    log.Println("服务启动在 :8080")
}
```

**查日志：**

```bash
# systemd 部署方式
sudo journalctl -u myapi -f
sudo journalctl -u myapi --since "1 hour ago" | grep ERROR

# Docker 部署方式
docker compose logs -f api
docker compose logs --tail=100 api

# Nginx 日志
sudo tail -f /var/log/nginx/myapi_access.log
sudo tail -f /var/log/nginx/myapi_error.log
```

### 6.2 常见故障排查

**问题：网站访问不了**

```bash
# 1. 服务器还活着吗？
ssh root@47.96.xx.xx uptime

# 2. 服务在跑吗？
sudo systemctl status myapi      # systemd
docker compose ps                 # Docker

# 3. 端口在监听吗？
ss -tlnp | grep 8080

# 4. Nginx 跑着吗？
sudo systemctl status nginx
sudo nginx -t    # 配置有没有语法错误

# 5. 防火墙开了吗？
sudo ufw status

# 6. 云厂商安全组放行了吗？
# 登录云控制台 → 安全组 → 确认 80/443 端口入方向开放
```

**问题：服务 502 Bad Gateway**

```
502 = Nginx 收到了请求，但转发给后端时后端挂了或超时

解决：
1. 确认 Go 服务在运行：systemctl status myapi
2. 查看 Go 服务日志：journalctl -u myapi -f
3. 可能原因：端口配错了、Go 程序 panic 了、数据库连不上
```

**问题：服务 500 Internal Server Error**

```
500 = Go 后端崩了

解决：
1. 立刻看日志：journalctl -u myapi --since "5 minutes ago"
2. 最常见的几种：
   - 数据库连接失败（检查 DB_HOST/DB_PASSWORD 环境变量）
   - nil 指针 dereference（看日志里的 panic stack trace）
   - 文件权限问题（uploads 目录不可写）
```

> **生活化类比**：排查线上故障就像**医生看急诊**——病人只说"不舒服"（网站挂了），你要按系统检查：先测体温脉搏（server alive），再抽血化验（日志），拍个 CT（Nginx access log 看请求有没有进来），最终确诊开药。不要跳过检查步骤直接开刀——先确定问题在哪一层，再动手修。

---

## 附录：从零到上线完整操作清单

以我们的 Todo API 为例，从零起步的每一步操作：

### 第一阶段：PaaS 快速验证（1-2 小时）

```
[ ] 在 Railway 注册账号，用 GitHub 登录
[ ] railway init → railway up
[ ] 拿到公网 URL（如 xxx.up.railway.app）
[ ] curl 测试所有 API 端点
[ ] 设置环境变量（JWT_SECRET 等）
```

### 第二阶段：云服务器部署（半天）

```
[ ] 买一台云服务器（阿里云/腾讯云轻量，选 Ubuntu 22.04）
[ ] 拿到公网 IP，配置 SSH 密钥登录
[ ] ssh 登录，apt update && apt upgrade
[ ] 创建 myapp 用户，禁用 root SSH 登录
[ ] scp 上传编译好的 myapi 二进制文件
[ ] 配置 systemd 服务，设置环境变量
[ ] 配置 ufw 防火墙（开放 22, 80, 443）
[ ] 安装 Nginx，配置反向代理
[ ] 买域名，配置 A 记录指向服务器 IP
[ ] 安装 certbot，申请 HTTPS 证书
[ ] https://api.todoproject.com/api/todos 测试通过
```

### 第三阶段：Docker 化（半天）

```
[ ] 写 Dockerfile（多阶段构建）
[ ] 写 docker-compose.yml（api + db + nginx）
[ ] 写 nginx.conf 模板
[ ] docker compose up -d 本地验证
[ ] 把 docker-compose.yml 推到服务器，同样启动验证
[ ] 确认数据库数据持久化（docker compose down 再 up，数据还在）
```

### 第四阶段：CI/CD（2-3 小时）

```
[ ] 创建 .github/workflows/ci.yml（test + build）
[ ] 创建 .github/workflows/deploy.yml（SSH 自动部署）
[ ] 在 GitHub 仓库设置 Secrets（SERVER_HOST 等）
[ ] push 代码到 main，观察 Actions 是否自动执行
[ ] 确认 push 后 https://api.todoproject.com 自动更新
```

---

## 检验清单

在不看笔记的情况下能解释清楚并动手实现：

- [ ] 本地代码 → 服务器运行，中间经过哪些步骤？能画出完整链路图吗？
- [ ] Railway / Fly.io 部署一个 Go 项目需要几步？环境变量怎么配？
- [ ] `ssh-copy-id` 做了什么？为什么用密钥比密码安全？
- [ ] `ls` / `ps` / `top` / `tail` / `systemctl` / `journalctl` 各自查什么？
- [ ] systemd service 文件怎么写？`Restart=always` 和 `WantedBy=multi-user.target` 分别是什么意思？
- [ ] Nginx 反向代理的 `proxy_pass` 和 `proxy_set_header` 各做什么？
- [ ] 从买域名到 `https://xxx.com` 可访问的完整步骤？
- [ ] 镜像和容器的区别？`docker build` 和 `docker run` 做了什么？
- [ ] 多阶段构建解决了什么？为什么最终镜像比直接 golang 镜像小那么多？
- [ ] docker-compose.yml 里 `depends_on` / `volumes` / `ports` / `environment` 各起什么作用？
- [ ] 容器间通过服务名通信的原理是什么？
- [ ] GitHub Actions 的 workflow / job / step 是什么意思？触发条件怎么配？
- [ ] CI 和 CD 各做什么？为什么分开两个 workflow 文件？
- [ ] 生产环境至少需要做哪些安全措施（最小 5 条）？
- [ ] 线上出 502 怎么排查？500 怎么排查？从哪里开始看？
- [ ] 从零到上线，把 Todo API 部署到 `https://api.yourdomain.com`，全过程跑通

---

> **下一步**：服务上线后，进入 [Phase 4：Agent 开发基础](../README.md#phase-4agent-开发基础) —— 理解 LLM 和 Agent 的核心概念，从裸调 API 到手写 Agent 循环。