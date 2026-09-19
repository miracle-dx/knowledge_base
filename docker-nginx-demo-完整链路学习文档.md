# Docker + Nginx + Java + MySQL 完整链路学习文档

> 适合初学者：本文以 `docker-nginx-demo` 项目为例，说明浏览器如何通过 Nginx 调用 Java 服务，再由 Java 查询 MySQL，最后把数据返回到页面。

---

## 一、项目最终实现了什么

打开浏览器访问：

```text
http://localhost:8080
```

点击页面上的“查询用户列表”按钮，页面会显示 MySQL 中的用户数据。

完整调用链路如下：

```text
浏览器
  |
  | 访问 http://localhost:8080
  v
Nginx 容器
  |
  | 静态页面：返回 html/index.html
  |
  | 接口请求：转发 /api/users
  v
Java Spring Boot 服务
  |
  | 查询 users 表
  v
MySQL 数据库
  |
  | 返回用户记录
  v
Java -> Nginx -> 浏览器页面
```

可以把它理解成：

- 浏览器：提出请求
- Nginx：接待请求并分流
- Java：处理业务
- MySQL：保存和提供数据
- 浏览器：展示结果

---

## 二、项目目录结构

```text
docker-nginx-demo/
├── docker-compose.yml                 # 启动 Nginx 容器
├── conf/
│   └── nginx.conf                     # Nginx 配置
├── html/
│   └── index.html                     # 前端静态页面
├── java-user-service/
│   ├── pom.xml                        # Java/Maven 依赖配置
│   ├── README.md                      # Java 服务说明
│   └── src/main/
│       ├── java/com/example/userservice/
│       │   ├── UserServiceApplication.java  # Java 启动类
│       │   ├── controller/
│       │   │   └── UserController.java       # 接口控制器
│       │   ├── entity/
│       │   │   └── User.java                 # 用户实体
│       │   └── repository/
│       │       └── UserRepository.java       # 数据库访问接口
│       └── resources/
│           └── application.properties        # Java 和数据库配置
└── sql/
    └── mysql-init.sql                  # 创建数据库、表和模拟数据
```

---

## 三、各个服务使用的端口

| 服务 | 地址 | 用途 |
|---|---|---|
| Nginx | `localhost:8080` | 浏览器访问入口 |
| Java | `localhost:8081` | 用户列表接口 |
| MySQL | `localhost:3306` | 数据库服务 |

这里有一个容易混淆的地方：

- Nginx 容器内部监听的是 `80` 端口
- Docker 把宿主机的 `8080` 映射到容器的 `80`
- 所以浏览器访问的是 `8080`，不是 `80`

对应的配置是：

```yaml
ports:
  - "8080:80"
```

格式是：

```text
宿主机端口:容器端口
```

---

## 四、第一步：准备 MySQL 数据库

### 4.1 启动 MySQL

确认本机 MySQL 服务已经启动。

Windows 可以通过：

```text
任务管理器 -> 服务 -> MySQL -> 启动
```

也可以在 PowerShell 查看：

```powershell
Get-Service *mysql*
```

### 4.2 使用 DBeaver 连接 MySQL

在 DBeaver 中创建 MySQL 连接：

```text
Host: localhost
Port: 3306
Username: root
Password: 你的 MySQL 密码
```

先不要在数据库名中填写 `demo_db`，因为它可能还没有创建。

### 4.3 执行数据库脚本

打开项目中的：

```text
sql/mysql-init.sql
```

脚本主要做三件事：

1. 创建 `demo_db` 数据库
2. 创建 `users` 表
3. 插入 5 条模拟用户数据

如果 DBeaver 的 `Ctrl + Enter` 只能执行单条语句，可以分三段执行。

第一段：

```sql
CREATE DATABASE IF NOT EXISTS demo_db
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

第二段：

```sql
USE demo_db;
```

第三段：

```sql
CREATE TABLE IF NOT EXISTS users (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    age INT NOT NULL,
    dept VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO users (name, email, age, dept) VALUES
('Alice', 'alice@example.com', 28, '研发部'),
('Bob', 'bob@example.com', 32, '产品部'),
('Charlie', 'charlie@example.com', 26, '运维部'),
('Diana', 'diana@example.com', 29, '市场部'),
('Ethan', 'ethan@example.com', 35, '人事部')
ON DUPLICATE KEY UPDATE
name = VALUES(name),
email = VALUES(email),
age = VALUES(age),
dept = VALUES(dept);
```

执行完后，在 DBeaver 左侧刷新连接，应该能看到：

```text
demo_db
└── Tables
    └── users
```

打开 `users` 表，应该能看到 5 条数据。

---

## 五、第二步：启动 Java 服务

### 5.1 在 IDEA 中导入项目

在 IDEA 中打开：

```text
E:\代码库\docker-nginx-demo\java-user-service
```

重点是打开包含 `pom.xml` 的 `java-user-service` 目录。

IDEA 会根据 `pom.xml` 自动下载 Spring Boot、JPA、MySQL 驱动等依赖。

### 5.2 Maven 依赖

`pom.xml` 中的重要依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

作用：

- 启动 Web 服务
- 提供 HTTP 接口
- 内置 Tomcat

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

作用：

- 让 Java 使用 JPA 访问数据库
- 自动生成常见查询逻辑
- 通过实体类映射数据库表

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

作用：

- 让 Java 能连接 MySQL

### 5.3 Java 数据库配置

配置文件是：

```text
java-user-service/src/main/resources/application.properties
```

关键配置：

```properties
server.port=8081

spring.datasource.url=jdbc:mysql://127.0.0.1:3306/demo_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
spring.datasource.username=${DB_USERNAME:root}
spring.datasource.password=${DB_PASSWORD:123456}
```

含义：

- Java 监听 `8081` 端口
- 连接本机 MySQL 的 `3306` 端口
- 使用 `demo_db` 数据库
- 默认用户名是 `root`
- 默认密码是 `123456`

如果你的 MySQL 密码不是 `123456`，修改为真实密码：

```properties
spring.datasource.password=你的真实密码
```

### 5.4 启动 Java

打开：

```text
src/main/java/com/example/userservice/UserServiceApplication.java
```

点击类左侧的绿色运行按钮，选择：

```text
Run UserServiceApplication
```

启动成功后，IDEA 控制台通常会出现类似信息：

```text
Tomcat started on port 8081
Started UserServiceApplication
```

### 5.5 先直接测试 Java 接口

打开浏览器访问：

```text
http://localhost:8081/api/users
```

如果正常，会返回：

```json
[
  {
    "id": 1,
    "name": "Alice",
    "email": "alice@example.com",
    "age": 28,
    "dept": "研发部"
  }
]
```

这一步成功，说明：

- Java 服务启动成功
- Java 能连接 MySQL
- 数据库存在
- `users` 表存在
- 用户数据查询成功

---

## 六、Java 代码是如何查询数据库的

### 6.1 实体类 User

文件：

```text
java-user-service/src/main/java/com/example/userservice/entity/User.java
```

它对应数据库中的 `users` 表：

```java
@Entity
@Table(name = "users")
public class User {
```

字段对应关系：

| Java 字段 | 数据库字段 |
|---|---|
| `id` | `id` |
| `name` | `name` |
| `email` | `email` |
| `age` | `age` |
| `dept` | `dept` |
| `createdAt` | `created_at` |

`@Entity` 表示这是数据库实体。

`@Table(name = "users")` 表示这个实体对应 `users` 表。

### 6.2 Repository

文件：

```text
java-user-service/src/main/java/com/example/userservice/repository/UserRepository.java
```

代码：

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

继承 `JpaRepository` 后，Spring Data JPA 会自动提供：

- `findAll()`：查询全部用户
- `findById()`：按 ID 查询
- `save()`：保存用户
- `deleteById()`：按 ID 删除

所以这里不需要手写 SQL，就可以查询用户列表。

### 6.3 Controller

文件：

```text
java-user-service/src/main/java/com/example/userservice/controller/UserController.java
```

核心代码：

```java
@RestController
@RequestMapping("/api")
public class UserController {

    @GetMapping("/users")
    public List<User> getUsers() {
        return userRepository.findAll();
    }
}
```

两个路径拼接起来就是：

```text
/api + /users = /api/users
```

当浏览器请求：

```http
GET /api/users
```

Java 就会执行：

```java
userRepository.findAll();
```

然后把查询结果自动转换成 JSON 返回。

---

## 七、第三步：启动 Nginx

项目使用 Docker Compose 启动 Nginx。

进入项目根目录：

```powershell
cd E:\代码库\docker-nginx-demo
```

启动：

```powershell
docker compose up -d
```

查看状态：

```powershell
docker compose ps
```

正常应该能看到：

```text
my-nginx   Up   0.0.0.0:8080->80/tcp
```

浏览器访问：

```text
http://localhost:8080
```

---

## 八、Nginx 配置详解

配置文件：

```text
conf/nginx.conf
```

当前配置核心内容：

```nginx
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://host.docker.internal:8081/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 8.1 `listen 80`

表示 Nginx 在容器内部监听 80 端口。

### 8.2 `root /usr/share/nginx/html`

表示网站根目录是：

```text
/usr/share/nginx/html
```

前端首页就是：

```text
/usr/share/nginx/html/index.html
```

### 8.3 `location /`

匹配普通页面请求，例如：

```text
/
/about
/user
```

### 8.4 `try_files $uri $uri/ /index.html`

表示按顺序查找：

1. 当前请求对应的文件
2. 当前请求对应的目录
3. 如果都不存在，就返回 `index.html`

`$uri` 是 Nginx 内置变量，表示当前请求路径。

例如请求：

```text
/about
```

此时 `$uri` 就是：

```text
/about
```

这段配置主要用于前端单页应用，避免刷新 `/about` 时出现 404。

### 8.5 `location /api/`

匹配所有以 `/api/` 开头的接口请求，例如：

```text
/api/users
/api/login
/api/orders
```

### 8.6 `proxy_pass`

```nginx
proxy_pass http://host.docker.internal:8081/api/;
```

表示把 Nginx 收到的 API 请求转发给宿主机的 Java 服务。

`host.docker.internal` 是 Docker 容器访问宿主机的特殊域名。

这里写成 `/api/` 是为了保证：

```text
浏览器请求：/api/users
Java 收到：/api/users
```

如果写成：

```nginx
proxy_pass http://host.docker.internal:8081/;
```

Nginx 可能会去掉 `/api/` 前缀，导致 Java 收到：

```text
/users
```

Java 没有 `/users` 接口，就会返回 404。

### 8.7 请求头转发

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

作用是把访问域名和客户端 IP 等信息传给 Java 服务。

---

## 九、前端页面如何调用接口

文件：

```text
html/index.html
```

按钮绑定了点击事件：

```javascript
btn.addEventListener('click', async function () {
    const response = await fetch('/api/users');
    const users = await response.json();
});
```

这里的 `/api/users` 是相对地址，浏览器会自动使用当前页面的域名和端口。

如果当前页面地址是：

```text
http://localhost:8080
```

那么实际请求地址就是：

```text
http://localhost:8080/api/users
```

注意：前端没有直接请求 `8081`，而是先请求 Nginx 的 `8080`，这样页面和接口使用同一个入口，不容易产生跨域问题。

---

## 十、一次完整请求的详细过程

假设用户点击按钮。

### 第 1 步：浏览器发送请求

```http
GET http://localhost:8080/api/users
```

### 第 2 步：请求到达 Nginx

Nginx 发现路径以 `/api/` 开头，于是匹配：

```nginx
location /api/
```

### 第 3 步：Nginx 转发请求

Nginx 将请求转发到：

```http
http://host.docker.internal:8081/api/users
```

### 第 4 步：Java 匹配 Controller

Java 中的：

```java
@RequestMapping("/api")
@GetMapping("/users")
```

组合出：

```text
/api/users
```

### 第 5 步：Java 查询数据库

Controller 调用：

```java
userRepository.findAll()
```

JPA 查询：

```sql
SELECT * FROM users;
```

### 第 6 步：MySQL 返回数据

MySQL 返回 Alice、Bob 等用户记录。

### 第 7 步：Java 转换为 JSON

Java 返回：

```json
[
  {
    "id": 1,
    "name": "Alice",
    "email": "alice@example.com",
    "age": 28,
    "dept": "研发部"
  }
]
```

### 第 8 步：Nginx 返回给浏览器

Nginx 把 Java 的响应转交给浏览器。

### 第 9 步：前端渲染表格

JavaScript 遍历用户数组，把每条用户信息放到 HTML 表格中。

---

## 十一、推荐的启动顺序

每次重新启动项目时，建议按这个顺序：

### 1. 启动 MySQL

确保 MySQL 服务正在运行。

### 2. 启动 Java

在 IDEA 中运行 `UserServiceApplication`。

### 3. 直接测试 Java

打开：

```text
http://localhost:8081/api/users
```

能返回 JSON 后再进行下一步。

### 4. 启动 Nginx

```powershell
cd E:\代码库\docker-nginx-demo
docker compose up -d
```

### 5. 访问前端

```text
http://localhost:8080
```

### 6. 点击按钮

点击“查询用户列表”。

---

## 十二、常见问题和排查方法

### 问题 1：`Unknown database 'demo_db'`

原因：MySQL 中还没有创建 `demo_db`。

解决：在 DBeaver 中执行：

```sql
CREATE DATABASE IF NOT EXISTS demo_db;
```

然后执行建表和插入数据语句。

---

### 问题 2：Java 接口返回 500

先看 IDEA 控制台的 `Caused by:`。

常见原因：

| 错误内容 | 原因 |
|---|---|
| `Unknown database 'demo_db'` | 数据库不存在 |
| `Access denied for user` | 用户名或密码错误 |
| `Table 'demo_db.users' doesn't exist` | 数据表不存在 |
| `Communications link failure` | MySQL 未启动或端口错误 |

---

### 问题 3：前端接口返回 404

先直接测试 Java：

```text
http://localhost:8081/api/users
```

如果 Java 直接访问正常，而 `8080/api/users` 返回 404，重点检查 Nginx 的 `proxy_pass`：

```nginx
location /api/ {
    proxy_pass http://host.docker.internal:8081/api/;
}
```

修改配置后重新加载 Nginx：

```powershell
docker exec my-nginx nginx -t
docker exec my-nginx nginx -s reload
```

---

### 问题 4：前端首页 404

检查容器是否运行：

```powershell
docker compose ps
```

检查 8080 端口：

```powershell
try {
    Invoke-WebRequest http://localhost:8080 -UseBasicParsing
} catch {
    $_.Exception.Message
}
```

检查 Nginx 日志：

```powershell
docker compose logs --tail 100 nginx
```

---

### 问题 5：修改 Nginx 配置后没有生效

因为配置文件被挂载到了容器中，但 Nginx 不一定会自动重新读取配置。

执行：

```powershell
docker exec my-nginx nginx -t
docker exec my-nginx nginx -s reload
```

`nginx -t` 用来检查语法。

`nginx -s reload` 用来重新加载配置。

---

### 问题 6：Maven 依赖显示红色

通常是 Maven 下载依赖失败。

检查 IDEA：

```text
Settings
-> Build, Execution, Deployment
-> Build Tools
-> Maven
```

确认：

- Maven 使用 `Bundled Maven`
- 没有勾选 `Work offline`
- Maven 能正常联网
- 点击 Maven 面板中的 `Reload All Maven Projects`

---

### 问题 7：中文显示乱码

如果浏览器正常、PowerShell 输出乱码，一般是 PowerShell 的显示编码问题，不代表接口数据损坏。

浏览器和 Java JSON 响应通常会正确显示中文。

---

## 十三、常用命令清单

### 查看 Nginx 容器

```powershell
docker compose ps
```

### 启动 Nginx

```powershell
docker compose up -d
```

### 停止 Nginx

```powershell
docker compose down
```

### 查看 Nginx 日志

```powershell
docker compose logs --tail 100 nginx
```

### 检查 Nginx 配置

```powershell
docker exec my-nginx nginx -t
```

### 重新加载 Nginx 配置

```powershell
docker exec my-nginx nginx -s reload
```

### 直接测试 Java 接口

```powershell
Invoke-WebRequest http://localhost:8081/api/users -UseBasicParsing
```

### 测试 Nginx 代理接口

```powershell
Invoke-WebRequest http://localhost:8080/api/users -UseBasicParsing
```

---

## 十四、应该记住的核心概念

### 1. Docker 容器

容器是一个隔离的运行环境。Nginx 运行在容器中，但 Java 和 MySQL 当前运行在 Windows 宿主机上。

### 2. Volume 挂载

`docker-compose.yml` 中：

```yaml
volumes:
  - ./conf/nginx.conf:/etc/nginx/conf.d/default.conf
  - ./html:/usr/share/nginx/html
```

左边是宿主机路径，右边是容器内路径。

例如：

```text
本地 html/index.html
        |
        v
容器 /usr/share/nginx/html/index.html
```

### 3. 端口映射

```yaml
ports:
  - "8080:80"
```

表示访问 Windows 的 `8080`，实际进入容器的 `80`。

### 4. 反向代理

Nginx 接收浏览器请求，再把请求转发给 Java，这就叫反向代理。

### 5. API 路径

Java Controller 中：

```java
@RequestMapping("/api")
@GetMapping("/users")
```

最终接口是：

```text
/api/users
```

### 6. 同源请求

前端访问 `/api/users`，实际请求的是当前网站的同一个端口 `8080`，由 Nginx 代理到 `8081`，因此前端不需要直接处理跨域。

---

## 十五、最终验收清单

按顺序确认：

- [ ] MySQL 服务正在运行
- [ ] `demo_db` 数据库存在
- [ ] `users` 表存在
- [ ] `users` 表中有 5 条模拟数据
- [ ] IDEA 中 Java 服务已启动
- [ ] `http://localhost:8081/api/users` 能返回 JSON
- [ ] Docker 中 `my-nginx` 容器处于 Up 状态
- [ ] `http://localhost:8080` 能打开页面
- [ ] 页面点击按钮后能显示用户列表
- [ ] Nginx 配置检查通过

Nginx 配置检查命令：

```powershell
docker exec my-nginx nginx -t
```

看到下面内容就表示配置语法正确：

```text
syntax is ok
configuration file /etc/nginx/nginx.conf test is successful
```

---

## 十六、一句话总结

这个项目的本质是：

```text
浏览器访问 Nginx，Nginx 提供前端页面并代理 API，Java 查询 MySQL，查询结果经过 Nginx 返回给浏览器展示。
```
