# Docker 学习资料

> 一份系统的 Docker 学习笔记：从核心概念到生产实践，覆盖常用命令、Dockerfile、数据卷、网络、Compose 与最佳实践。
> 编写时间：2026 年 9 月。截至编写时，Docker Engine 最新稳定版为 29.x（BuildKit 为默认构建器，containerd 为默认容器运行时与镜像存储）。安装与使用以官方文档为准。

---

## 目录

1. [Docker 是什么](#1-docker-是什么)
2. [核心概念](#2-核心概念)
3. [架构与工作原理](#3-架构与工作原理)
4. [安装与环境准备](#4-安装与环境准备)
5. [镜像操作](#5-镜像操作)
6. [容器操作](#6-容器操作)
7. [Dockerfile 编写](#7-dockerfile-编写)
8. [数据管理（卷与挂载）](#8-数据管理卷与挂载)
9. [网络](#9-网络)
10. [Docker Compose](#10-docker-compose)
11. [多阶段构建与镜像优化](#11-多阶段构建与镜像优化)
12. [安全与生产最佳实践](#12-安全与生产最佳实践)
13. [日志与监控](#13-日志与监控)
14. [故障排查](#14-故障排查)
15. [学习路线与练习建议](#15-学习路线与练习建议)

---

## 1. Docker 是什么

Docker 是一个开源的容器化平台，用于**构建、分发、运行**应用。它将应用及其依赖（代码、运行时、库、配置）打包进一个标准化的单元——**容器**，使应用在任何环境下都能以相同方式运行。

**核心价值：**

| 价值 | 说明 |
|---|---|
| 环境一致性 | "在我机器上能跑"的问题被消除，开发/测试/生产环境一致 |
| 轻量隔离 | 容器共享宿主机内核，比虚拟机更轻、启动更快（秒级） |
| 交付标准化 | 镜像即交付物，可推送到仓库、随处拉取运行 |
| 资源高效 | 相比虚拟机省去整套 Guest OS，单机可运行更多实例 |

**与虚拟机的区别：**

| 维度 | 虚拟机 | 容器 |
|---|---|---|
| 隔离级别 | 硬件级（Hypervisor） | 操作系统级（内核共享） |
| 内核 | 每台虚拟机有独立 Guest OS | 共享宿主机内核 |
| 启动时间 | 分钟级 | 秒级 |
| 镜像大小 | GB 级 | MB 级 |
| 性能 | 有虚拟化开销 | 接近原生 |

---

## 2. 核心概念

### 2.1 镜像（Image）

- 一个**只读的、分层的模板**，包含运行应用所需的一切。
- 由多层（Layer）叠加组成，每一层对应 Dockerfile 中的一条指令。
- 镜像不可变：构建后内容不会改变；运行容器时在其上增加一个可写层。

### 2.2 容器（Container）

- 镜像的**运行实例**，本质是"镜像 + 可写层 + 运行时配置"。
- 容器启动快、可停止/删除/重建；多个容器可以基于同一镜像运行。
- 容器内部进程相互隔离，但可通过网络与卷进行交互。

### 2.3 仓库（Registry）

- 存储和分发镜像的服务。
- 常用公共仓库：Docker Hub（默认）、GHCR、Quay、阿里云 ACR 等。
- 镜像命名格式：`仓库地址/命名空间/镜像名:标签`，如 `nginx:1.27-alpine`。

### 2.4 三者关系

```
Dockerfile（构建）→ Image（镜像）→ 容器（运行）
                     ↓
               Registry（分发）
```

---

## 3. 架构与工作原理

### 3.1 组件构成

| 组件 | 作用 |
|---|---|
| Docker CLI（客户端） | 用户操作的命令行工具 |
| dockerd（守护进程） | 核心服务，接收 API 请求并管理容器生命周期 |
| containerd | 容器运行时管理组件，负责镜像管理与容器执行 |
| runc | 符合 OCI 规范的低层运行时，真正创建/运行容器进程 |
| BuildKit | 新一代镜像构建引擎（自 v23+ 逐步成为默认，v28 起完全默认） |

### 3.2 底层技术

容器隔离依赖 Linux 内核能力：

| 技术 | 作用 |
|---|---|
| Namespace（命名空间） | 隔离视图：进程 PID、网络、文件系统、用户、主机名等 |
| Cgroups（控制组） | 限制资源：CPU、内存、IO、PID 数量 |
| UnionFS（联合文件系统） | 镜像分层与写时复制（overlay2） |
| Capabilities / Seccomp | 最小化权限与系统调用控制 |

> 理解要点：容器不是一个"小虚拟机"，而是宿主机上的一组受限进程。这也是为什么容器里跑 `top` 看到的是宿主机的进程列表（PID namespace 未隔离部分）。

---

## 4. 安装与环境准备

### 4.1 Linux（Ubuntu/Debian 为例）

官方推荐使用 apt 仓库安装：

```bash
# 1. 安装依赖
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# 2. 添加 Docker 官方 GPG 密钥与仓库
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. 安装
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 4. 验证
sudo docker run --rm hello-world
```

**免 sudo 使用**（把当前用户加入 docker 组，需重新登录生效）：

```bash
sudo usermod -aG docker $USER
```

> 注意：将用户加入 docker 组等同于授予 root 级权限（docker 组用户可控制守护进程）。仅适用于可信的开发环境；生产服务器建议使用 rootless 模式或严格管理。

### 4.2 配置镜像加速（国内环境）

编辑 `/etc/docker/daemon.json`：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.net"
  ]
}
```

修改后重启：`sudo systemctl restart docker`

> 镜像加速地址可能随时间变化，请以当前可用源为准；拉取失败时多试几个源。

### 4.3 验证安装

```bash
docker version        # 客户端与服务端版本
docker info           # 环境详细信息
docker run hello-world  # 端到端验证
```

---

## 5. 镜像操作

```bash
# 拉取镜像
docker pull nginx:1.27-alpine
docker pull nginx                      # 不写标签默认 latest
docker pull ghcr.io/owner/repo:tag     # 从其他仓库拉取

# 查看本地镜像
docker images
docker image ls

# 查看镜像详细信息
docker inspect nginx
docker history nginx                   # 查看分层历史

# 搜索镜像（Hub）
docker search redis

# 删除镜像
docker rmi nginx:1.27-alpine
docker image prune                     # 清理悬空镜像
docker image prune -a                  # 清理所有未被使用的镜像

# 保存与加载（离线迁移）
docker save -o nginx.tar nginx:1.27-alpine
docker load -i nginx.tar

# 打标签
docker tag nginx:1.27-alpine myrepo/nginx:v1

# 推送（需先登录）
docker login
docker push myrepo/nginx:v1
```

**镜像标签规范：** `latest` 不保证是最新，生产环境务必使用明确的版本标签（如 `1.27-alpine`）。

---

## 6. 容器操作

### 6.1 创建与运行

```bash
# 前台运行（占用终端，Ctrl+C 停止）
docker run nginx

# 后台运行
docker run -d --name web -p 8080:80 nginx:1.27-alpine

# 交互式运行（调试用）
docker run -it --rm ubuntu:24.04 bash

# 常用参数
docker run -d \                # 后台运行
  --name myapp \               # 容器名
  -p 8080:80 \                 # 端口映射 宿主机:容器
  -e ENV_VAR=value \           # 环境变量
  -v /host/path:/container/path \  # 卷挂载
  --network my-net \           # 指定网络
  --restart unless-stopped \   # 重启策略
  -m 512m --cpus 1 \           # 资源限制
  --rm \                       # 退出后自动删除
  nginx:1.27-alpine
```

### 6.2 查看与管理

```bash
docker ps                 # 运行中的容器
docker ps -a              # 所有容器（含已停止）
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"  # 自定义输出

docker start web          # 启动已存在的容器
docker stop web           # 停止（优雅，等待 SIGTERM）
docker restart web        # 重启
docker kill web           # 强制停止（SIGKILL）
docker rm web             # 删除容器
docker rm -f web          # 强制删除运行中的容器
docker rm $(docker ps -aq)  # 删除所有容器
docker container prune    # 清理所有已停止的容器
```

### 6.3 进入容器与执行命令

```bash
# 进入运行中容器（推荐，分配 TTY + 标准输入）
docker exec -it web bash

# 在容器内执行单条命令
docker exec web cat /etc/nginx/nginx.conf

# 查看进程
docker top web

# 查看资源占用
docker stats               # 实时监控所有容器 CPU/内存
docker stats --no-stream   # 单次快照
```

### 6.4 日志

```bash
docker logs web            # 查看全部日志
docker logs -f web         # 实时跟踪
docker logs --tail 100 web # 只看最后 100 行
docker logs --since 10m web  # 最近 10 分钟
```

### 6.5 文件拷贝

```bash
docker cp ./app.conf web:/etc/nginx/conf.d/
docker cp web:/var/log/nginx/access.log ./access.log
```

> 说明：`docker cp` 适合临时调试。持久化数据应使用数据卷（见第 8 章）。

### 6.6 重启策略（--restart）

| 值 | 行为 |
|---|---|
| `no` | 默认，不自动重启 |
| `on-failure[:max]` | 非零退出码时重启（可限次数） |
| `always` | 总是重启（含守护进程启动时） |
| `unless-stopped` | 总是重启，但手动停止后不再拉起（生产常用） |

---

## 7. Dockerfile 编写

### 7.1 指令速查

| 指令 | 作用 |
|---|---|
| `FROM` | 基础镜像，**必须是第一条指令** |
| `RUN` | 构建时执行命令（每个 RUN 产生一层） |
| `COPY` | 从构建上下文复制文件进镜像 |
| `ADD` | 类似 COPY，额外支持自动解压 tar 与远程 URL（不推荐用于远程 URL） |
| `WORKDIR` | 设置工作目录（影响后续 RUN/CMD/COPY 的相对路径） |
| `ENV` | 设置环境变量（运行时也生效） |
| `ARG` | 构建参数（仅构建时有效，可用 `--build-arg` 覆盖） |
| `EXPOSE` | 声明容器监听端口（仅文档作用，不真正映射） |
| `CMD` | 容器启动的默认命令，可被 `docker run` 后的参数覆盖 |
| `ENTRYPOINT` | 容器启动的固定入口命令，不易覆盖，常与 CMD 配合 |
| `USER` | 指定运行用户（安全实践：非 root） |
| `VOLUME` | 声明匿名卷挂载点 |
| `HEALTHCHECK` | 健康检查指令 |
| `LABEL` | 元数据 |
| `ONBUILD` | 留给后续子镜像构建时触发的指令（少用） |

### 7.2 示例：Node.js 应用

```dockerfile
# 语法：Dockerfile 采用 BuildKit 语法特性（可选）
# syntax=docker/dockerfile:1

# 多阶段构建 - 第一阶段：编译/安装依赖
FROM node:22-alpine AS build
WORKDIR /app
# 先复制 package 文件，利用层缓存
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# 第二阶段：精简运行镜像
FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
# 复制构建产物与生产依赖
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
# 非 root 用户运行
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### 7.3 CMD 与 ENTRYPOINT 的配合

- **ENTRYPOINT**：固定命令入口，如 `ENTRYPOINT ["nginx", "-g", "daemon off;"]`
- **CMD**：默认参数，可被覆盖。

两种写法对比：

```dockerfile
# 写法一：CMD 提供默认可覆盖命令
CMD ["node", "app.js"]
# docker run image node app2.js   → 执行 node app2.js

# 写法二：ENTRYPOINT + CMD 配合
ENTRYPOINT ["node"]
CMD ["app.js"]
# docker run image app2.js        → 执行 node app2.js（入口固定，参数可换）
```

### 7.4 构建命令

```bash
# 基本构建
docker build -t myapp:v1 .

# 指定 Dockerfile 与上下文
docker build -f docker/Dockerfile.prod -t myapp:v1 .

# 传构建参数
docker build --build-arg VERSION=1.0 -t myapp:v1 .

# 指定平台（跨架构构建）
docker build --platform linux/amd64,linux/arm64 -t myapp:v1 .

# 使用 BuildKit（当前版本默认，可直接启用特性）
DOCKER_BUILDKIT=1 docker build -t myapp:v1 .
```

### 7.5 .dockerignore

与 `.gitignore` 同理，排除不需要进入构建上下文的内容（**显著减小上下文、防止敏感文件入镜像**）：

```
node_modules
dist
.git
*.log
.env
.DS_Store
```

---

## 8. 数据管理（卷与挂载）

容器删除后，其可写层数据会丢失。持久化数据需要卷或挂载。

### 8.1 三种方式

| 方式 | 特点 | 适用 |
|---|---|---|
| **命名卷（Volume）** | 由 Docker 管理，位于 `/var/lib/docker/volumes/`，推荐 | 数据库、应用数据 |
| **绑定挂载（Bind Mount）** | 直接挂载宿主机目录，双向同步 | 开发调试、配置文件 |
| **tmpfs 挂载** | 仅存内存，容器停止即消失 | 敏感临时数据 |

### 8.2 使用示例

```bash
# 命名卷
docker volume create pgdata
docker run -d --name postgres -v pgdata:/var/lib/postgresql/data postgres:16

# 绑定挂载（开发场景：代码实时同步）
docker run -d --name dev -v $(pwd):/app -p 3000:3000 node:22-alpine

# 只读挂载
docker run -d --name nginx -v /etc/nginx:/etc/nginx:ro nginx

# 更规范的 --mount 语法
docker run -d \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  --mount type=bind,source=$(pwd)/config,target=/app/config,readonly \
  postgres:16

# 卷管理
docker volume ls
docker volume inspect pgdata
docker volume prune
```

### 8.3 容器间共享数据

```bash
# 数据卷容器模式（旧）已被 --volumes-from 继承
docker run -d --name data-share -v /shared ubuntu sleep infinity
docker run -d --name app1 --volumes-from data-share nginx
```

> 更推荐的现代做法：两个容器挂载**同一个命名卷**，实现共享。

---

## 9. 网络

### 9.1 网络类型

| 网络 | 说明 | 场景 |
|---|---|---|
| `bridge`（默认） | 容器间可通过 IP/容器名通信，经 NAT 访问外网 | 单机多容器 |
| `host` | 容器直接使用宿主机网络栈，无隔离 | 性能敏感、端口冲突少的场景 |
| `none` | 无网络 | 纯离线计算任务 |
| `overlay` | 跨宿主机网络（Swarm/K8s 场景） | 多机集群 |

### 9.2 桥接网络实践

```bash
# 创建自定义网络（推荐：自定义网络自带 DNS 解析，可用容器名互访）
docker network create my-net

# 用自定义网络运行容器
docker run -d --name web --network my-net -p 8080:80 nginx
docker run -d --name api --network my-net myapi:v1

# 容器内通过服务名互访
docker exec api curl http://web:80

# 常用网络命令
docker network ls
docker network inspect my-net
docker network connect my-net api      # 运行中容器加入网络
docker network disconnect my-net api
docker network rm my-net
```

> 重点：**默认 bridge 网络不支持容器名 DNS 解析**（旧版），自定义 bridge 网络支持。多容器应用务必使用自定义网络。

### 9.3 端口映射

```bash
docker run -d -p 8080:80 nginx          # 宿主机8080 → 容器80
docker run -d -p 127.0.0.1:8080:80 nginx  # 只绑定回环地址
docker run -d -p 80:80/tcp -p 53:53/udp  # 指定协议
docker run -d -P nginx                  # 随机映射所有 EXPOSE 端口
```

---

## 10. Docker Compose

Compose 用于**定义和运行多容器应用**，通过 YAML 描述服务、网络、卷。

### 10.1 示例：Web + Redis + 数据库

```yaml
# compose.yaml（新版官方推荐文件名；docker-compose.yaml 也兼容）
name: myapp

services:
  web:
    build: .
    ports:
      - "8080:3000"
    environment:
      - NODE_ENV=production
      - REDIS_HOST=redis
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - app-net

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 5
    networks:
      - app-net

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - app-net

networks:
  app-net:

volumes:
  redis-data:
  pg-data:
```

### 10.2 常用命令

```bash
docker compose up -d          # 启动（-d 后台）
docker compose up -d --build  # 重新构建后启动
docker compose ps             # 查看服务状态
docker compose logs -f web    # 查看某服务日志
docker compose exec web bash  # 进入某服务容器
docker compose down           # 停止并删除容器与网络
docker compose down -v        # 同时删除卷（数据会丢！慎用）
docker compose config         # 校验并展开配置
docker compose top            # 查看进程
docker compose restart web    # 重启单个服务
```

### 10.3 常用配置项速查

```yaml
services:
  app:
    image: nginx                      # 指定镜像
    build:                            # 或从 Dockerfile 构建
      context: .
      dockerfile: Dockerfile.prod
      args:
        VERSION: "1.0"
    container_name: myapp             # 固定容器名（慎用，影响扩展）
    restart: unless-stopped           # 重启策略
    ports: ["8080:80"]                # 端口映射
    expose: ["80"]                    # 仅内部暴露
    environment:                      # 环境变量
      KEY: value
    env_file: .env                    # 从文件读取
    volumes: ["data:/data"]
    networks: ["app-net"]
    depends_on:                       # 依赖与等待条件
      - redis
    healthcheck:                      # 健康检查
      test: ["CMD", "curl", "-f", "http://localhost:80"]
    deploy:
      replicas: 2                     # 副本数（Swarm 生效）
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
```

---

## 11. 多阶段构建与镜像优化

### 11.1 多阶段构建的价值

- 构建工具链（编译器、依赖安装器）不进最终镜像
- 镜像体积可缩小 5-10 倍
- 参考 7.2 节示例

### 11.2 体积优化清单

| 手段 | 效果 |
|---|---|
| 使用 Alpine / Distroless 基础镜像 | 从数百 MB 降至几十 MB |
| 多阶段构建 | 去除构建期依赖 |
| 合并 RUN 指令 | 减少层数 |
| `--no-cache` 之外的依赖锁定（`npm ci`/`pip install -r`） | 减少无用包 |
| `.dockerignore` | 减小上下文 |
| 清理 apt 缓存（`rm -rf /var/lib/apt/lists/*`） | 每层瘦身 |
| 注意顺序：**变更频率低的指令放前面** | 最大化层缓存命中 |

### 11.3 层缓存原理

Docker 按 Dockerfile 指令逐层构建；某一层内容未变时直接复用缓存。因此：

```
❌ 低效顺序（源码变更 → 依赖全部重装）
COPY . .
RUN npm install

✅ 高效顺序（package 文件不变 → 依赖层命中缓存）
COPY package*.json ./
RUN npm install
COPY . .
```

---

## 12. 安全与生产最佳实践

### 12.1 镜像安全

- 使用**明确版本标签**，不用 `latest`
- 优先官方镜像与 Alpine 精简版
- 定期更新基础镜像，修复 CVE
- 使用 `docker scout` 或第三方工具扫描漏洞：

```bash
docker scout cves nginx:1.27-alpine
```

### 12.2 运行安全

- **非 root 运行**：Dockerfile 中 `USER nobody` 或创建专用用户
- **最小权限**：避免使用 `--privileged`；按需添加 capabilities（`--cap-drop ALL --cap-add NET_BIND_SERVICE`）
- 容器内**不存密钥**：使用环境变量/Secret 管理（Docker Secrets、外部密钥管理）
- 限制资源（`--memory`、`--cpus`、`--pids-limit`），防资源耗尽
- 只读根文件系统：`--read-only`，必要时挂 tmpfs 到 `/tmp`

```bash
# 安全运行示例
docker run -d \
  --name app \
  --read-only \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  -m 512m --cpus 0.5 --pids-limit 100 \
  --tmpfs /tmp \
  myapp:v1
```

### 12.3 配置管理

- 开发/生产配置分离（`.env` + Compose 变量替换）
- 敏感信息不写进 Dockerfile 与镜像层
- 镜像推送前用 `.dockerignore` 排除 `.env`、密钥文件

### 12.4 守护进程安全

- 限制 Docker API 暴露面（默认 unix socket 仅本机）
- 如需远程访问，启用 TLS 认证
- 生产环境考虑 rootless Docker 或受管容器服务

---

## 13. 日志与监控

### 13.1 日志驱动

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

- 生产环境务必配置日志轮转，防止磁盘写满
- 常用驱动：`json-file`（默认）、`journald`、`fluentd`、`awslogs`、`loki`

### 13.2 资源监控

```bash
docker stats                    # 实时 CPU/内存/网络
docker stats --no-stream
docker system df                # 磁盘占用概览
docker system prune -a          # 清理未使用镜像/容器/网络/缓存
docker system prune -a --volumes  # 连卷一起清理（慎用）
```

---

## 14. 故障排查

### 14.1 容器无法启动

```bash
# 1. 查看状态与退出码
docker ps -a
# 2. 看完整日志
docker logs --tail 200 <container>
# 3. 查看详情（含 ExitCode、错误）
docker inspect <container> | grep -A5 ExitCode
# 4. 覆盖入口调试（不执行原 CMD，直接进 shell）
docker run -it --rm --entrypoint sh <image>
```

### 14.2 常见问题速查

| 现象 | 常见原因 | 处理 |
|---|---|---|
| 端口无法访问 | 未映射端口 / 服务监听 127.0.0.1 | `-p` 映射；容器内监听 `0.0.0.0` |
| 容器间无法互访 | 默认网络无 DNS / 未加入同一网络 | 使用自定义网络 + 容器名 |
| 磁盘占满 | 镜像/容器/日志堆积 | `docker system prune`，配置日志轮转 |
| 拉取超时 | 网络问题 | 配置镜像加速/换源 |
| exec 失败 | 容器内无 bash | 用 `sh` 或 `--entrypoint` |
| 数据丢失 | 未用卷，数据在可写层 | 改用命名卷持久化 |
| 时区不对 | 基础镜像默认 UTC | 挂载 `/etc/localtime` 或设置 `TZ` 环境变量 |

### 14.3 调试常用手法

```bash
# 查看容器内进程与资源
docker top <c>
docker stats <c>

# 查看网络
docker network inspect <net>

# 覆盖入口进入容器排查
docker run -it --rm --entrypoint sh <image>

# 查看构建失败时的中间层（保留构建缓存）
docker build --rm=false -t app:debug .
docker run -it app:debug sh
```

---

## 15. 学习路线与练习建议

### 15.1 学习路线

```
阶段一：概念与命令
  ├─ 镜像/容器/仓库 三个核心概念
  ├─ 镜像拉取、容器运行/停止/删除/日志
  └─ 练习：运行 nginx、ubuntu，体验 exec/ps/logs

阶段二：镜像构建
  ├─ Dockerfile 指令逐个上手
  ├─ CMD vs ENTRYPOINT 区别
  └─ 练习：为你的项目写 Dockerfile，构建并运行

阶段三：数据与网络
  ├─ 卷、绑定挂载、容器间共享
  ├─ 自定义网络、端口映射、容器名互访
  └─ 练习：部署 "Web + MySQL" 双容器应用并持久化数据

阶段四：Compose 与编排
  ├─ compose.yaml 多服务编排
  ├─ depends_on、healthcheck、环境变量
  └─ 练习：用 Compose 一键启动全栈应用（前端+后端+DB+Redis）

阶段五：生产实践
  ├─ 多阶段构建、镜像瘦身
  ├─ 安全基线（非 root、只读、资源限制）
  ├─ 日志轮转、系统清理
  └─ 练习：部署生产级应用，配置健康检查与重启策略

阶段六：生态延伸（可选）
  ├─ containerd 与 OCI 规范
  ├─ Docker Swarm（原生编排）
  ├─ Kubernetes（容器编排的事实标准）
  └─ 镜像仓库私有化（Harbor 等）
```

### 15.2 动手练习建议（由浅入深）

1. 跑通 `hello-world`，理解镜像拉取与容器生命周期
2. 用 `nginx` 跑一个静态网站，理解端口映射与挂载
3. 写一个 Python/Node 应用 Dockerfile，体验多阶段构建
4. 部署 `WordPress + MySQL` 或用 Compose 起一个全栈应用
5. 将镜像推到 Docker Hub / 私有仓库，体验分发
6. 给生产容器加健康检查、资源限制、非 root 用户，做一次安全加固

### 15.3 官方文档入口

| 资源 | 地址 |
|---|---|
| Docker 官方文档 | https://docs.docker.com |
| Dockerfile 参考 | https://docs.docker.com/reference/dockerfile |
| Compose 参考 | https://docs.docker.com/compose/compose-file/ |
| Docker Hub | https://hub.docker.com |
| Docker 官方示例仓库 | https://github.com/docker/awesome-compose |

---

## 附录：常用命令速查卡

```bash
# ── 镜像 ──
docker pull <img>              # 拉取
docker images                  # 列表
docker rmi <img>               # 删除
docker build -t <name> .       # 构建
docker save/load               # 导出/导入

# ── 容器 ──
docker run -d -p <host>:<con> --name <n> <img>   # 运行
docker ps [-a]                 # 查看
docker logs -f <c>             # 日志
docker exec -it <c> bash       # 进入
docker start/stop/restart <c>  # 生命周期
docker rm [-f] <c>             # 删除
docker cp <c>:<path> <local>   # 拷贝

# ── 数据 ──
docker volume create/ls/rm/prune
# 挂载：-v <vol>:/path 或 --mount type=volume,...

# ── 网络 ──
docker network create/ls/inspect/rm
# 端口：-p 8080:80

# ── Compose ──
docker compose up -d
docker compose ps/logs/exec/down/config

# ── 清理 ──
docker system df
docker system prune [-a] [--volumes]
```

---

*本资料基于 Docker 官方文档与通用实践整理，命令在不同版本间可能有细微差异，遇到疑问以 `docker <command> --help` 与官方文档为准。*
