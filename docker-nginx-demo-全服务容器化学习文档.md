# Docker + Nginx + Java + MySQL 全服务容器化学习文档

> 本文基于当前 `docker-nginx-demo` 项目，面向初学者说明：如何把前端、Nginx、Java 服务和 MySQL 都放进 Docker 容器，并通过 Compose 和流水线部署。

---

## 一、这次改造解决了什么问题

之前的结构是：

```text
浏览器
  ↓
Nginx 容器
  ↓
宿主机上的 Java
  ↓
宿主机上的 MySQL
```

改造后的结构是：

```text
浏览器
  ↓
Nginx 容器
  ↓
Java 容器
  ↓
MySQL 容器
```

也就是说，运行项目不再依赖宿主机上单独安装的 Java 和 MySQL。

只要机器安装了 Docker，就可以通过 Docker Compose 启动整套服务。

完整链路：

```text
浏览器访问 localhost:8080
        ↓
Nginx 容器监听 80
        ↓
Nginx 返回前端页面，或代理 /api/users
        ↓
Java 容器监听 8081
        ↓
Java 通过 JPA 查询 MySQL 容器
        ↓
MySQL 返回 users 表数据
        ↓
Java → Nginx → 浏览器
```

---

## 二、当前有哪些容器

当前 Compose 中有三个服务：

| Compose 服务 | 容器名称 | 作用 |
|---|---|---|
| `mysql` | `demo-mysql` | 保存用户数据 |
| `user-service` | `demo-user-service` | 提供 Java API |
| `nginx` | `demo-nginx` | 提供网页和反向代理 |

查看容器：

```powershell
docker compose ps
```

正常状态类似：

```text
demo-mysql          Up (healthy)
demo-user-service   Up
demo-nginx          Up
```

---

## 三、端口应该怎样理解

当前端口关系：

```text
浏览器访问 Windows:8080
        ↓
Nginx 容器:80
        ↓
Java 容器:8081
        ↓
MySQL 容器:3306
```

Compose 中只有 Nginx 映射到宿主机：

```yaml
nginx:
  ports:
    - "8080:80"
```

格式是：

```text
宿主机端口:容器端口
```

Java 使用：

```yaml
expose:
  - "8081"
```

`expose` 只让同一个 Docker 网络中的其他容器访问，不直接暴露给 Windows 宿主机。

MySQL 没有写 `ports`，所以数据库也不直接暴露到宿主机，只能被 Compose 网络中的 Java 访问。

这样更接近生产环境：

```text
外部只访问 Nginx
Java 和 MySQL 不直接暴露
```

---

## 四、Compose 网络中的服务名

这是全容器化最关键的变化。

在容器里，不能这样连接 MySQL：

```text
localhost:3306
127.0.0.1:3306
```

因为容器里的 `localhost` 指向当前容器自己。

Java 连接 MySQL 使用 Compose 服务名：

```text
mysql:3306
```

配置如下：

```yaml
DB_URL: jdbc:mysql://mysql:3306/demo_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC&characterEncoding=UTF-8&useUnicode=true
```

Nginx 连接 Java 也使用服务名：

```text
user-service:8081
```

当前 Nginx 配置的关键部分：

```nginx
location /api/ {
    resolver 127.0.0.11 ipv6=off;
    set $user_service user-service:8081;
    proxy_pass http://$user_service;
}
```

总结：

| 连接对象 | 地址 |
|---|---|
| 浏览器访问 Nginx | `localhost:8080` |
| Nginx 访问 Java | `user-service:8081` |
| Java 访问 MySQL | `mysql:3306` |

Compose 会自动创建网络，并自动把服务名解析成容器 IP。

---

## 五、MySQL 容器和数据库数据

MySQL 使用官方镜像：

```yaml
image: mysql:8.4
```

这相当于启动一个包含 MySQL 服务的容器。

### 5.1 数据保存在哪里

配置：

```yaml
volumes:
  - mysql_data:/var/lib/mysql
```

表示：

```text
容器内 /var/lib/mysql
        ↓
Docker 数据卷 mysql_data
        ↓
宿主机磁盘
```

镜像保存 MySQL 程序，数据卷保存真正的数据库数据。

查看数据卷：

```powershell
docker volume ls
```

查看详细信息：

```powershell
docker volume inspect docker-nginx-demo_mysql_data
```

### 5.2 初始化脚本

配置：

```yaml
- ./sql/mysql-init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
```

第一次初始化空数据卷时，MySQL 会执行：

```text
sql/mysql-init.sql
```

脚本会：

1. 创建 `demo_db`
2. 创建 `users` 表
3. 插入 mock 用户

重要：初始化脚本只在数据目录为空时执行。

重新启动容器不会重复执行。

### 5.3 重置数据库

如果只是停止容器：

```powershell
docker compose down
```

数据卷仍然保留。

如果执行：

```powershell
docker compose down -v
```

会删除数据卷，数据库数据也会删除。

下次启动时，初始化脚本会重新执行。

> `down -v` 是破坏性操作，学习环境可以使用，重要数据不要随便执行。

---

## 六、MySQL 的健康检查和启动顺序

Compose 中 MySQL 配置了健康检查：

```yaml
healthcheck:
  test: ["CMD-SHELL", "mysqladmin ping -h localhost -uroot -p$${MYSQL_ROOT_PASSWORD} --silent"]
  interval: 5s
  timeout: 5s
  retries: 20
```

它会定期检查 MySQL 是否已经能够接受连接。

Java 依赖 MySQL 健康：

```yaml
depends_on:
  mysql:
    condition: service_healthy
```

所以启动顺序是：

```text
MySQL 容器启动
  ↓
MySQL 健康检查通过
  ↓
Java 容器启动
  ↓
Nginx 容器启动
```

`depends_on` 解决的是启动顺序，不代表应用永远不会出现连接问题；生产系统通常还会增加应用自身的重试和健康检查。

---

## 七、Java Dockerfile：多阶段构建

文件：

```text
java-user-service/Dockerfile
```

内容分成两个阶段。

### 7.1 构建阶段

```dockerfile
FROM maven:3.9.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY maven-settings.xml /root/.m2/settings.xml
RUN mvn -B -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -B -DskipTests package
```

这个阶段包含：

- Maven
- JDK 17
- 源代码
- Maven 依赖

它负责把 Java 源码编译成 JAR：

```text
/app/target/*.jar
```

`maven-settings.xml` 用于配置 Maven 镜像，避免默认访问 Maven Central 时网络失败。

### 7.2 运行阶段

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "app.jar"]
```

运行阶段只保留：

- Java JRE
- 最终 JAR

不会把 Maven、源代码和构建缓存带进最终镜像。

这就是多阶段构建，优点是最终镜像更小、更适合生产。

---

## 八、Nginx Dockerfile

根目录文件：

```text
Dockerfile
```

内容：

```dockerfile
FROM nginx:alpine
COPY conf/nginx.conf /etc/nginx/conf.d/default.conf
COPY html /usr/share/nginx/html
EXPOSE 80
```

构建过程：

1. 拉取 `nginx:alpine`
2. 把本地 Nginx 配置复制到容器
3. 把本地前端页面复制到容器
4. 生成自己的 Nginx 镜像

当前没有把前端目录挂载进去，因此修改页面后需要重新构建镜像：

```powershell
docker compose up -d --build
```

---

## 九、一次 `docker compose up -d --build` 发生了什么

执行：

```powershell
docker compose up -d --build
```

Docker 会依次完成：

### 第 1 步：读取 Compose

读取：

```text
docker-compose.yml
```

确定需要创建哪些服务、网络、数据卷和依赖关系。

### 第 2 步：检查基础镜像

Java 需要：

```text
maven:3.9.9-eclipse-temurin-17
eclipse-temurin:17-jre
```

Nginx 需要：

```text
nginx:alpine
```

MySQL 需要：

```text
mysql:8.4
```

如果本地没有，Docker 会尝试拉取。

### 第 3 步：读取 Dockerfile

Docker 按 Dockerfile 执行 `FROM`、`COPY`、`RUN`、`EXPOSE` 等指令。

### 第 4 步：构建自己的镜像

生成类似镜像：

```text
docker-nginx-demo-user-service
 docker-nginx-demo-nginx
```

### 第 5 步：创建网络和数据卷

Compose 创建：

```text
docker-nginx-demo_default
docker-nginx-demo_mysql_data
```

### 第 6 步：创建并启动容器

启动：

```text
demo-mysql
demo-user-service
demo-nginx
```

### 第 7 步：浏览器访问

打开：

```text
http://localhost:8080
```

---

## 十、前端到数据库的完整请求

点击页面“查询用户列表”按钮时：

### 1. 浏览器请求 Nginx

```http
GET http://localhost:8080/api/users
```

### 2. Nginx 匹配 API 路径

```nginx
location /api/
```

### 3. Nginx 转发到 Java

```http
GET http://user-service:8081/api/users
```

### 4. Java Controller 接收

```java
@GetMapping("/users")
```

配合类上的：

```java
@RequestMapping("/api")
```

最终路径就是：

```text
/api/users
```

### 5. Java 查询 MySQL

```java
userRepository.findAll()
```

JPA 执行类似：

```sql
SELECT * FROM users;
```

### 6. 返回结果

```text
MySQL → Java → Nginx → 浏览器
```

新增、修改、删除的链路也是一样，只是 HTTP 方法不同：

```text
POST   /api/users       新增
PUT    /api/users/{id}  修改
DELETE /api/users/{id}  删除
GET    /api/users       查询
```

---

## 十一、修改代码后怎样更新容器

### 修改 Java

修改本地 Java 源码后执行：

```powershell
docker compose up -d --build user-service
```

### 修改前端或 Nginx

执行：

```powershell
docker compose up -d --build nginx
```

### 修改 Compose 或多个服务

执行：

```powershell
docker compose up -d --build
```

`--build` 会重新构建镜像。

仅执行：

```powershell
docker compose up -d
```

不会一定重新打包最新代码，只会根据已有镜像启动或重建容器。

---

## 十二、流水线是怎样工作的

文件：

```text
.github/workflows/ci-cd.yml
```

当前流水线触发条件：

```yaml
push:
  branches: [main]

pull_request:
  branches: [main]
```

含义：

- 推送到 `main`：构建并部署
- 创建 Pull Request：执行检查，但不部署生产

### 12.1 构建阶段

流水线会：

1. 拉取 GitHub 代码
2. 安装 JDK 17
3. 执行 `mvn verify`
4. 执行 `docker compose config`
5. 构建 Docker 镜像

### 12.2 部署阶段

只有构建成功并且分支是 `main` 才会部署：

```text
GitHub Actions
  ↓ SSH
服务器
  ↓ git pull
最新代码
  ↓ docker compose up -d --build
新容器
```

流水线中执行的核心命令：

```bash
cd $DEPLOY_PATH
git pull --ff-only origin main
docker compose up -d --build --remove-orphans
docker compose ps
```

### 12.3 GitHub Secrets

需要在仓库：

```text
Settings
→ Secrets and variables
→ Actions
```

配置：

```text
DEPLOY_HOST
DEPLOY_USER
DEPLOY_SSH_KEY
DEPLOY_PATH
```

不要把 SSH 私钥、数据库密码提交到 Git 仓库。

---

## 十三、生产服务器上的首次部署

服务器需要安装：

- Git
- Docker
- Docker Compose

首次部署：

```bash
sudo mkdir -p /opt/docker-nginx-demo
cd /opt/docker-nginx-demo
git clone https://github.com/你的用户名/你的仓库.git .
docker compose up -d --build
```

后续流水线自动部署，或者手工执行：

```bash
cd /opt/docker-nginx-demo
git pull --ff-only origin main
docker compose up -d --build --remove-orphans
```

查看日志：

```bash
docker compose logs -f mysql
docker compose logs -f user-service
docker compose logs -f nginx
```

---

## 十四、常见问题

### 1. Java 报 `Unknown database 'demo_db'`

说明数据库没有初始化成功。

查看 MySQL：

```powershell
docker compose logs mysql
```

如果是全新学习环境，可以重置：

```powershell
docker compose down -v
docker compose up -d --build
```

注意这会删除数据库数据。

### 2. MySQL 报 `unknown variable 'default-character-set'`

MySQL 服务端启动参数应使用：

```yaml
command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```

不要把客户端参数 `default-character-set` 直接传给服务端。

### 3. Java 报 `Unsupported character encoding 'utf8mb4'`

JDBC URL 应使用：

```text
characterEncoding=UTF-8
```

MySQL 服务端仍可以使用：

```text
utf8mb4
```

### 4. 新增用户报邮箱重复

`users.email` 有唯一约束：

```sql
email VARCHAR(100) NOT NULL UNIQUE
```

同一个邮箱不能重复新增。

### 5. 修改代码后页面没有变化

重新构建：

```powershell
docker compose up -d --build
```

浏览器再执行：

```text
Ctrl + F5
```

### 6. Nginx 404 或 502

先确认容器：

```powershell
docker compose ps
```

再看日志：

```powershell
docker compose logs --tail 100 nginx
docker compose logs --tail 100 user-service
```

重点检查：

```text
Nginx → user-service:8081
Java → mysql:3306
```

不要在容器之间使用 `localhost`。

### 7. Docker 拉取基础镜像失败

Java Dockerfile 依赖：

```text
maven:3.9.9-eclipse-temurin-17
eclipse-temurin:17-jre
```

如果 Docker Hub 网络不可用，可以配置镜像加速；Maven 依赖则由 `maven-settings.xml` 使用国内 Maven 镜像。

---

## 十五、常用命令

启动并构建：

```powershell
docker compose up -d --build
```

查看状态：

```powershell
docker compose ps
```

查看所有日志：

```powershell
docker compose logs -f
```

查看某个服务：

```powershell
docker compose logs -f user-service
```

停止容器但保留数据：

```powershell
docker compose down
```

停止容器并删除数据卷：

```powershell
docker compose down -v
```

查看镜像：

```powershell
docker image ls
```

查看 Docker 磁盘占用：

```powershell
docker system df -v
```

检查 Compose 配置：

```powershell
docker compose config
```

---

## 十六、开发配置和生产配置

项目中有：

```text
docker-compose.yml
docker-compose.prod.yml
```

通常约定：

```text
docker-compose.yml       默认/开发环境
docker-compose.prod.yml  生产环境
```

使用生产文件：

```powershell
docker compose -f docker-compose.prod.yml up -d --build
```

当前两个文件结构接近，主要用于学习和演示。

更成熟的生产方式通常是：

```text
流水线构建镜像
  ↓
推送镜像仓库
  ↓
服务器 docker compose pull
  ↓
服务器启动固定版本镜像
```

而不是服务器每次现场编译源代码。

---

## 十七、最终验收清单

- [ ] `docker compose config` 通过
- [ ] `demo-mysql` 状态为 `healthy`
- [ ] `demo-user-service` 状态为 `Up`
- [ ] `demo-nginx` 状态为 `Up`
- [ ] `http://localhost:8080` 能打开页面
- [ ] `GET /api/users` 返回用户列表
- [ ] `POST /api/users` 能新增用户
- [ ] `PUT /api/users/{id}` 能修改用户
- [ ] `DELETE /api/users/{id}` 能删除用户
- [ ] MySQL 数据卷存在
- [ ] 修改 Java 后使用 `--build` 更新镜像
- [ ] 修改前端后使用 `--build` 更新 Nginx 镜像
- [ ] GitHub Actions 构建检查通过

---

## 十八、最重要的三句话

```text
镜像保存程序，数据卷保存数据库数据。
```

```text
容器访问宿主机使用 host.docker.internal，容器访问容器使用 Compose 服务名。
```

```text
本地代码修改后，Dockerfile 方式部署必须重新构建镜像。
```

最终架构可以记成：

```text
浏览器
  ↓ localhost:8080
Nginx 容器
  ↓ user-service:8081
Java 容器
  ↓ mysql:3306
MySQL 容器
  ↓ mysql_data
宿主机磁盘
```
