# Docker 全面入门指南
---
## 一、什么是 Docker？
**官方定义：** Docker 是一个开源的**容器化平台**，用于开发、交付和运行应用程序。
### 🗣️ 大白话解释
> 想象你搬家。你有两种方式：
	**方式一（传统部署）：** 把家具拆了，一件件搬到新家，再重新组装。电视要重新调频道，洗衣机要重新接水管……折腾得够呛。
	**方式二（Docker）：** 把整个房间连同家具、电器、水电接口，打包成一个"集装箱"📦，直接搬到新家，打开就能用。
	**Docker 就是方式二。** 它把你的应用程序 + 运行环境 + 依赖库 + 配置，全部打包成一个标准化的"集装箱"（容器），在任何地方都能一模一样地运行。
---
## 二、Docker 解决什么问题？
### 问题 1：「在我电脑上能跑啊！」😩
```plain text
开发：我本地跑得好好的！
运维：在服务器上就是跑不起来！
原因：Python 版本不同 / 少了个系统库 / 配置文件路径不对……
```
**Docker 解决方案：** 环境跟着应用走，开发环境 = 测试环境 = 生产环境。
### 问题 2：环境配置地狱 🔥
```plain text
新同事入职，配环境配了三天：
- 装 Python 3.9（不能是 3.10）
- 装 MySQL 5.7（不能是 8.0）
- 装 Redis 6.x
- 改 20 个配置文件
- 还有 3 个神秘的环境变量……
```
**Docker 解决方案：** 一条命令搞定 → `docker compose up`，完事。
### 问题 3：应用之间互相干扰 💥
```plain text
项目 A 需要 Java 8
项目 B 需要 Java 17
项目 C 需要 Node 14
项目 D 需要 Node 18
全装在一台机器上？版本冲突到崩溃。
```
**Docker 解决方案：** 每个应用在自己的容器里运行，互不影响。
### 问题 4：资源浪费 💸
```plain text
传统虚拟机：每个应用一个 VM，每个 VM 都要装完整的操作系统
→ 一台服务器只能跑 3-5 个 VM
```
**Docker 解决方案：** 容器共享宿主机内核，超轻量级。一台服务器能跑几十上百个容器。
---
## 三、核心概念——用类比搞懂
| 概念 | 类比 | 解释 |
| --- | --- | --- |
| **镜像 (Image)** | 安装光盘/模板 | 一个只读的模板，包含了运行应用所需的一切 |
| **容器 (Container)** | 用光盘装好的电脑 | 镜像的运行实例，是真正跑起来的东西 |
| **Dockerfile** | 安装说明书 | 告诉 Docker 如何一步步构建镜像 |
| **仓库 (Registry)** | 应用商店 | 存放和分享镜像的地方（如 Docker Hub） |
| **数据卷 (Volume)** | U盘/外接硬盘 | 容器中需要持久化的数据 |
### 🗣️ 用一个完整的类比串起来
```plain text
Dockerfile（菜谱）  →  docker build（做菜）  →  Image（一道菜的标准样品）
                                                       ↓
                                              docker run（上菜）
                                                       ↓
                                              Container（客人桌上的那盘菜）

Docker Hub（美食平台）← docker push（上传菜谱）/ docker pull（下载菜谱）
```
---
## 四、Docker vs 虚拟机
```plain text
┌─────────────────────────┐    ┌─────────────────────────┐
│      虚拟机 (VM)         │    │      Docker 容器         │
├─────────────────────────┤    ├─────────────────────────┤
│  App A  │  App B │ App C│    │  App A │  App B │ App C │
│  Libs   │  Libs  │ Libs │    │  Libs  │  Libs  │ Libs  │
│  Guest  │  Guest │Guest │    ├────────┴────────┴───────┤
│   OS    │   OS   │  OS  │    │      Docker Engine       │
├─────────┴────────┴──────┤    ├─────────────────────────┤
│       Hypervisor        │    │        宿主机 OS          │
├─────────────────────────┤    ├─────────────────────────┤
│        宿主机硬件        │    │        宿主机硬件         │
└─────────────────────────┘    └─────────────────────────┘

VM：每个应用带一个完整OS        容器：共享宿主机OS内核
启动：分钟级                    启动：秒级
体积：GB级                      体积：MB级
```
> **大白话：** 虚拟机是搬了一整栋楼，Docker 只是搬了一个房间。
---
## 五、如何使用 Docker（详细版）
### 1. Docker 的生命周期
```plain text
                    docker pull
  Docker Hub ──────────────────────► 本地镜像
                                        │
                                        │ docker run
                                        ▼
                                      容器(运行中)
                                     ╱    │    ╲
                           docker stop   │   docker exec
                                ↓        │        ↓
                          容器(已停止)    │    进入容器内部
                                ↓        │
                          docker start   │
                            (重新启动)    │
                                        │
                                   docker rm
                                   (彻底删除)
```
> **大白话：** 镜像是"冷冻食品"，`docker run` 是"解冻加热"变成可以吃的"容器"。吃完可以扔掉（rm），也可以放回冰箱下次再热（stop/start）。
### 2. 安装 Docker
```bash
# macOS / Windows：下载 Docker Desktop
# https://www.docker.com/products/docker-desktop

# Linux（以 Ubuntu 为例）：
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # 让当前用户可以用 docker 命令
```
### 3. `docker run` 详解——最核心的命令
```bash
docker run [选项] 镜像名[:标签] [命令]
```
#### 🔹 基础运行方式
```bash
# (a) 前台运行（会占住终端，Ctrl+C 退出）
docker run nginx

# (b) 后台运行（推荐，不占终端）
docker run -d nginx
# -d = detach（分离），容器在后台默默运行

# (c) 交互式运行（进入容器内部，像 SSH 一样）
docker run -it ubuntu bash
# -i = interactive（保持标准输入开启）
# -t = tty（分配一个终端）
# bash = 进入后执行 bash
# 你会看到提示符变成 root@容器ID，说明你"进入"了容器内部
```
> **大白话区分：**
	- `d`：开了个店，你不用站在柜台前，店自己营业
	- `it`：你亲自走进店里，四处看看摸摸
#### 🔹 端口映射（-p）
容器默认是"与世隔绝"的，要让外面访问，需要映射端口：
```bash
docker run -d -p 8080:80 nginx
#              ↑      ↑
#           主机端口  容器端口
```
```plain text
你的电脑                    Docker 容器
┌──────────┐               ┌──────────┐
│          │   8080 → 80   │          │
│ 浏览器 ──┼───────────────┼→ Nginx   │
│          │               │          │
└──────────┘               └──────────┘

访问 localhost:8080 → 实际上访问的是容器里的 80 端口
```
```bash
# 多个端口映射
docker run -d -p 8080:80 -p 8443:443 nginx

# 随机分配主机端口
docker run -d -p 80 nginx
docker port <容器名>     # 查看分配了哪个端口

# 只允许本机访问（安全场景）
docker run -d -p 127.0.0.1:8080:80 nginx
```
#### 🔹 环境变量（-e）
```bash
docker run -d \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  -e MYSQL_DATABASE=myapp \
  -e MYSQL_USER=admin \
  mysql:8.0
```
> **大白话：** `-e` 就像给应用"填写配置表"，告诉它用什么密码、连哪个数据库。
#### 🔹 数据持久化（-v / --mount）
**核心问题：** 容器删了，里面的数据就没了！数据库的数据怎么办？
```bash
# 方式一：挂载目录（Bind Mount）—— 把主机目录"接入"容器
docker run -d \
  -v /home/admin/mysql-data:/var/lib/mysql \
  mysql:8.0
#    ↑ 主机目录                ↑ 容器内目录

# 方式二：命名卷（Named Volume）—— 让 Docker 管理存储（推荐）
docker run -d \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0
```
```plain text
方式一（Bind Mount）：
  你的电脑                    容器
  /home/admin/data/ ←──────→ /var/lib/mysql/
  你能直接看到和编辑文件         容器也能看到和编辑

方式二（Named Volume）：
  Docker 管理的存储区 ←──────→ /var/lib/mysql/
  Docker 自动管理              容器正常读写
  docker volume ls 查看        删容器不影响数据
```
> **大白话：**
	- Bind Mount 像"把你家的一个抽屉插到容器里共用"
	- Named Volume 像"给容器配了一个保险箱，容器没了保险箱还在"
**怎么选？**
| 场景 | 推荐 |
| --- | --- |
| 开发时想实时同步代码 | Bind Mount（`-v ./src:/app/src`） |
| 数据库数据持久化 | Named Volume（`-v pgdata:/var/lib/postgresql/data`） |
| 配置文件挂入容器 | Bind Mount（`-v ./nginx.conf:/etc/nginx/nginx.conf:ro`） |
> `:ro` 表示只读（read-only），容器不能修改这个文件。
#### 🔹 容器命名和自动重启
```bash
docker run -d \
  --name my-web \                # 起个好记的名字
  --restart unless-stopped \     # 自动重启策略
  -p 8080:80 \
  nginx
```
重启策略选项：
```plain text
no              ── 默认，不自动重启
always          ── 总是重启（包括手动 stop 后 Docker 重启时）
unless-stopped  ── 除非手动 stop，否则总是重启（推荐）
on-failure:3    ── 失败时最多重试 3 次
```
#### 🔹 资源限制
```bash
docker run -d \
  --memory=512m \      # 最多用 512MB 内存
  --cpus=1.5 \         # 最多用 1.5 个 CPU 核
  nginx
```
> **大白话：** 就像给房客签合同，"你最多用 500 度电、3 吨水"，防止一个容器吃掉所有资源。
#### 🔹 网络相关
```bash
# 创建自定义网络（让容器之间用名字互相访问）
docker network create my-net

docker run -d --name db    --network my-net postgres:15
docker run -d --name web   --network my-net my-flask-app
# 在 web 容器里，可以直接用 "db" 作为主机名连接数据库
# 连接字符串：postgresql://user:pass@db:5432/mydb
#                                      ↑ 直接用容器名！
```
> **大白话：** 自定义网络就像给几个容器建了一个"微信群"，群里的人可以互相 @名字 找到对方。
### 4. 常用命令速查表
```bash
# ========== 镜像相关 ==========
docker pull nginx          # 下载镜像
docker images              # 查看本地所有镜像
docker rmi nginx           # 删除镜像

# ========== 容器相关 ==========
docker ps                  # 查看运行中的容器
docker ps -a               # 查看所有容器（包括已停止的）
docker start my-nginx      # 启动容器
docker stop my-nginx       # 停止容器
docker restart my-nginx    # 重启容器
docker rm my-nginx         # 删除容器
docker logs my-nginx       # 查看容器日志
docker logs -f my-nginx    # 实时跟踪日志（像 tail -f）
docker logs --tail 100 my-nginx   # 只看最后 100 行
docker logs --since 30m my-nginx  # 最近 30 分钟的日志
docker exec -it my-nginx bash     # 进入容器内部

# ========== 检查详情 ==========
docker inspect my-web      # 查看容器完整信息（JSON）
docker inspect --format '{{.NetworkSettings.IPAddress}}' my-web  # 只查 IP
docker stats               # 实时资源占用（CPU/内存/网络）

# ========== 清理相关 ==========
docker system prune        # 清理所有没用的东西（慎用）
```
### 5. Docker Compose（编排多个容器）
实际项目通常不止一个容器。比如：Web 应用 + 数据库 + 缓存。
```yaml
# docker-compose.yml
version: '3.8'

services:
  # Web 应用
  app:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/app                    # 代码实时同步
    environment:
      - DATABASE_URL=postgresql://dev:dev123@db:5432/myapp
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  # 数据库
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev123
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev"]
      interval: 5s
      retries: 5

  # 缓存
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```
```bash
# 一条命令启动所有服务
docker compose up -d

# 查看状态
docker compose ps

# 查看日志
docker compose logs -f app

# 停止并删除所有
docker compose down

# 关掉并删除数据卷（彻底清理）
docker compose down -v
```
> **大白话：** Docker Compose 就是一个"乐队指挥"🎵，告诉每个"乐手"（容器）什么时候上场、怎么配合。
---
## 六、如何制作自己的 Docker 镜像（重点）
### 1. 理解镜像的"层"结构
Docker 镜像是**一层一层叠上去**的，Dockerfile 中每条指令都会产生一层：
```plain text
┌─────────────────────────────┐
│  CMD ["python", "app.py"]   │  ← 第5层：启动命令
├─────────────────────────────┤
│  COPY . .                   │  ← 第4层：复制代码（经常变）
├─────────────────────────────┤
│  RUN pip install ...        │  ← 第3层：安装依赖
├─────────────────────────────┤
│  COPY requirements.txt .    │  ← 第2层：复制依赖清单
├─────────────────────────────┤
│  FROM python:3.11-slim      │  ← 第1层：基础镜像
└─────────────────────────────┘
```
> **大白话：** 想象镜像是"千层蛋糕"🍰，每一层烤好后就不会变了。如果你只改了代码（第 4 层），下面 1-3 层直接用缓存，不用重新来。**这就是为什么 Dockerfile 的写法顺序很重要。**
### 2. Dockerfile 指令详解
```docker
# ===== 最常用 =====

FROM python:3.11-slim
# 指定基础镜像，你的一切都建立在它之上
# 就像盖房子要先选一块地基
# 常见基础镜像：
#   python:3.11-slim  → Python 环境（slim 比默认版小很多）
#   node:18-alpine    → Node.js 环境（alpine 最小）
#   ubuntu:22.04      → 通用 Ubuntu
#   alpine:3.18       → 最小 Linux（约 5MB）

WORKDIR /app
# 设置工作目录，后面的命令都在这个目录下执行
# 如果目录不存在会自动创建
# 类似于 cd /app，但更规范

COPY requirements.txt .
COPY . .
# 把主机上的文件复制到镜像里
# COPY <主机路径> <镜像内路径>
# "." 表示当前 WORKDIR

RUN pip install -r requirements.txt
RUN apt-get update && apt-get install -y curl
# 在构建时执行命令（安装软件、编译代码等）
# 每个 RUN 产生一层，所以尽量合并：
# ✅ RUN apt-get update && apt-get install -y curl vim git
# ❌ RUN apt-get update
#    RUN apt-get install -y curl
#    RUN apt-get install -y vim

CMD ["python", "app.py"]
# 容器启动时默认执行的命令
# 只能有一个 CMD，写多个只有最后一个生效
# 用户 docker run 时可以覆盖它

EXPOSE 5000
# 声明容器监听的端口（只是文档作用，不会真的开端口）
# 真正开端口靠 docker run -p

ENV DATABASE_URL=sqlite:///db.sqlite
# 设置环境变量（构建时和运行时都有效）

# ===== 进阶 =====

ARG PYTHON_VERSION=3.11
FROM python:${PYTHON_VERSION}-slim
# 构建参数，只在构建时有效（运行时看不到）
# docker build --build-arg PYTHON_VERSION=3.12 .

ENTRYPOINT ["python"]
CMD ["app.py"]
# ENTRYPOINT 是"固定前缀"，CMD 是"默认参数"
# 最终执行：python app.py
# 用户运行 docker run my-app test.py → 执行 python test.py
# ENTRYPOINT 不会被覆盖，CMD 会被覆盖

VOLUME /data
# 声明匿名数据卷挂载点

USER appuser
# 切换为非 root 用户运行（安全最佳实践）

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:5000/health || exit 1
# 健康检查，Docker 会定期检查容器是否正常
```
#### 🗣️ ENTRYPOINT vs CMD 大白话解释
```plain text
把容器想象成一个"工具"：

ENTRYPOINT = 这个工具是什么（比如"计算器"）
CMD        = 默认算什么（比如"1+1"）

ENTRYPOINT ["python"]    → 这是一个 Python 执行器
CMD ["app.py"]           → 默认执行 app.py

用户可以改 CMD：
docker run my-app test.py   → 改成执行 test.py
但工具本身（ENTRYPOINT）不变，还是用 python 来执行
```
### 3. 缓存优化——为什么顺序很重要
```plain text
场景：你改了一行代码

❌ 错误顺序（全部重新来）：
   COPY . .                ← 代码变了，这层缓存失效
   RUN pip install ...     ← 缓存也跟着失效，重新安装所有依赖！（慢）

✅ 正确顺序（只重建变化的部分）：
   COPY requirements.txt . ← 依赖没变，缓存命中 ✓
   RUN pip install ...     ← 缓存命中 ✓（秒完成）
   COPY . .                ← 代码变了，只重建这一层（快！）
```
> **大白话：** 把"不常变的"放前面，"经常变的"放后面。就像装修时先做硬装（水电管道），后做软装（窗帘沙发），改软装不用砸墙。
### 4. 实战案例
#### 案例一：静态网站（最简单）
```docker
FROM nginx:alpine
COPY ./website /usr/share/nginx/html
```
```bash
docker build -t my-website .
docker run -d -p 80:80 my-website
```
#### 案例二：Python Web 应用（规范写法）
```docker
FROM python:3.11-slim

# 防止 Python 输出缓冲（让日志实时显示）
ENV PYTHONUNBUFFERED=1
# 不生成 .pyc 文件（容器里没必要）
ENV PYTHONDONTWRITEBYTECODE=1

WORKDIR /app

# 先复制依赖文件（利用缓存！）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 再复制代码（代码经常变，放后面不会破坏上面的缓存）
COPY . .

# 创建非 root 用户（安全最佳实践）
RUN adduser --disabled-password --no-create-home appuser
USER appuser

EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```
#### 案例三：Node.js 应用 + 多阶段构建（缩小镜像体积的杀手锏）
```docker
# ==================== 阶段1：构建 ====================
FROM node:18 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci                  # 安装全部依赖（包括 devDependencies）
COPY . .
RUN npm run build           # 编译 / 打包

# ==================== 阶段2：运行 ====================
FROM node:18-alpine AS runner

WORKDIR /app

# 只安装生产依赖
COPY package*.json ./
RUN npm ci --omit=dev

# 只从 builder 阶段拿构建产物
COPY --from=builder /app/dist ./dist

USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```
```plain text
镜像体积对比：
  单阶段构建：~1.2 GB  (node_modules + 源码 + 构建工具 + 产物)
  多阶段构建：~150 MB   (只有运行时需要的东西)
```
> **大白话：** 多阶段构建就像"做蛋糕"🎂：
	- 阶段 1（厨房）：面粉、鸡蛋、搅拌机、烤箱……乱七八糟
	- 阶段 2（上桌）：只拿做好的蛋糕，厨房的乱不用管
	客人（用户）拿到的是干净的蛋糕，不是整个厨房。
#### 案例四：Java Spring Boot 应用
```docker
# 阶段1：构建
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline       # 先下载依赖（利用缓存）
COPY src ./src
RUN mvn package -DskipTests         # 打包

# 阶段2：运行
FROM eclipse-temurin:17-jre-alpine  # 只要 JRE，不要 JDK
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

RUN addgroup -S spring && adduser -S spring -G spring
USER spring

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```
### 5. `.dockerignore` 文件——别忘了它！
就像 `.gitignore`，告诉 Docker "构建时不要复制这些文件"：
```plain text
# .dockerignore
.git
.gitignore
node_modules
__pycache__
*.pyc
.env                  # 千万别把密码打包进镜像！
.venv
dist
*.md
docker-compose.yml
Dockerfile
.idea
.vscode
```
> **为什么重要？**
	1. 没有 `.dockerignore` → `COPY . .` 会把 `node_modules`（几百 MB）、`.git`（可能几 GB）都复制进去
	2. 构建变慢（Docker 需要先把所有文件发送给 Docker 守护进程）
	3. 镜像变大、可能泄漏敏感信息（`.env` 文件里的密码）
### 6. 构建、标签与推送
```bash
# ========== 构建 ==========
docker build -t my-app .                        # 基本构建
docker build -t my-app:v1.0 .                   # 带版本标签
docker build -t my-app -f deploy/Dockerfile .   # 指定 Dockerfile 路径
docker build --no-cache -t my-app .             # 不用缓存，全部重新构建
docker build --build-arg ENV=prod -t my-app .   # 传入构建参数

# ========== 标签 ==========
docker tag my-app:v1.0 registry.example.com/team/my-app:v1.0
docker tag my-app:v1.0 my-app:latest
# 一个镜像可以有多个标签，就像一个人可以有多个外号

# ========== 推送到仓库 ==========
docker login registry.example.com               # 先登录
docker push registry.example.com/team/my-app:v1.0
```
标签命名规范建议：
```plain text
my-app:latest       ← 最新版（不推荐用于生产，因为"最新"会变）
my-app:v1.2.3       ← 语义化版本（推荐）
my-app:20260924     ← 日期版本
my-app:abc1234      ← Git commit SHA（可追溯）
```
### 7. Dockerfile 最佳实践清单 ✅
```docker
# ✅ 1. 用具体版本，不要用 latest
FROM python:3.11-slim          # ✅ 确定版本
# FROM python:latest           # ❌ 今天和明天可能不一样

# ✅ 2. 合并 RUN 命令，减少层数
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git && \
    rm -rf /var/lib/apt/lists/*    # 清理缓存

# ✅ 3. 不常变的放前面，经常变的放后面
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# ✅ 4. 使用非 root 用户
RUN adduser --disabled-password appuser
USER appuser

# ✅ 5. 使用多阶段构建（见案例三）

# ✅ 6. 选择合适的基础镜像
# python:3.11        → 约 900MB（带完整 Debian）
# python:3.11-slim   → 约 120MB（精简 Debian）   ← 推荐
# python:3.11-alpine → 约  50MB（最小，但可能有兼容问题）

# ✅ 7. 一个容器只做一件事
# Web 应用一个容器，数据库一个容器，不要塞一起

# ✅ 8. 用 COPY 不用 ADD（除非需要自动解压 tar）
COPY . .       # ✅ 行为明确
# ADD . .      # ❌ 有隐式行为（自动解压、支持 URL）
```
### 8. 调试技巧——镜像构建出问题了怎么办
```bash
# 1. 构建时看到详细输出
docker build --progress=plain -t my-app .

# 2. 构建到某一步失败了，进入中间镜像检查
docker run -it <上一层的镜像ID> sh

# 3. 查看镜像每一层的大小（找出哪层太大）
docker history my-app

# 4. 用 dive 工具可视化分析镜像（推荐）
# https://github.com/wagoodman/dive
dive my-app
```
---
## 七、类似/相关概念
| 技术 | 和 Docker 的关系 | 一句话说清 |
| --- | --- | --- |
| **Podman** | Docker 的替代品 | 无守护进程、无 root，命令几乎兼容 Docker |
| **Kubernetes (K8s)** | Docker 的"升级版管理者" | 管理成百上千个容器的编排系统 |
| **虚拟机 (VM)** | Docker 的"前辈" | 更重、更隔离，但启动慢、资源占用大 |
| **LXC** | Docker 的底层技术之一 | Linux 容器，Docker 最初就基于它 |
| **containerd** | Docker 的运行时组件 | 真正负责运行容器的底层引擎 |
| **OCI** | 容器标准规范 | 保证不同容器工具之间的兼容性 |
| **Vagrant** | 开发环境管理 | 侧重管理虚拟机，Docker 侧重容器 |
### Docker 和 K8s 的关系（最常问）
```plain text
Docker：    把应用打包成集装箱 📦
Kubernetes：管理一个集装箱码头 🏗️（调度、扩缩容、自愈、负载均衡）

你开一个小店   → 用 Docker 就够了
你开一条商业街 → 需要 Kubernetes 来管理
```
---
## 八、进阶知识点推荐 🗺️
按学习顺序排列：
```plain text
Level 1 ── 入门（基础使用）
  ├── Docker 基本命令
  ├── Dockerfile 编写
  └── Docker Compose

Level 2 ── 进阶
  ├── 多阶段构建（Multi-stage Build）── 大幅缩小镜像体积
  ├── 网络模式（bridge / host / overlay）── 容器间怎么通信
  ├── 数据持久化（Volume / Bind Mount）── 数据不丢失
  └── 镜像优化 ── 让镜像从 1GB 缩小到 50MB

Level 3 ── 生产实战
  ├── Docker 安全最佳实践 ── 非 root 运行、镜像扫描
  ├── CI/CD 集成 ── GitHub Actions / Jenkins + Docker
  ├── 私有镜像仓库 ── Harbor / 阿里云 ACR
  └── 日志与监控 ── 容器的日志怎么收集

Level 4 ── 容器编排
  ├── Kubernetes 基础 ── Pod / Service / Deployment
  ├── Helm ── K8s 的包管理器
  └── Service Mesh（Istio）── 微服务治理
```
### 🔥 实用小技巧
```bash
# 1. 查看容器资源占用
docker stats

# 2. 复制文件进/出容器
docker cp local-file.txt my-container:/app/
docker cp my-container:/app/log.txt ./

# 3. 查看镜像构建历史
docker history my-flask-app

# 4. 给正在运行的容器打标签保存为新镜像
docker commit my-container my-new-image:v2

# 5. 多阶段构建减小前端镜像体积
# ---- Dockerfile ----
# FROM node:18 AS builder
# WORKDIR /app
# COPY . .
# RUN npm ci && npm run build
#
# FROM nginx:alpine
# COPY --from=builder /app/dist /usr/share/nginx/html
# 最终镜像只有 nginx + 静态文件，不到 30MB
```
---
## 总结
> **Docker = 标准化打包 + 随处运行。**
它解决的核心问题是：**消除「我这里能跑」和「你那里跑不了」之间的差异。**
快速上手路径：
1. 装 Docker Desktop
2. `docker run -it ubuntu bash`（进入一个 Ubuntu 容器感受一下）
3. 写一个 Dockerfile 把自己的项目容器化
4. 用 Docker Compose 编排多个服务
5. 学习多阶段构建优化镜像
6. 在 CI/CD 中集成 Docker