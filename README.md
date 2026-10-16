# CatBase Studio（猫窝）—— 纯 CatBase 写的动态网页 + 浏览器内 IDE

一个**全部用 CatBase 实现**的 Web 项目，目标是：

> 纯 CatBase 写的动态网页，访问 MySQL，调用各种包，可以在页面上直接写 CatBase 代码运行。

四个核心能力**已在本仓库实测跑通**：

| 设计目标               | 本项目的实现                                                                            | 状态                    |
| ------------------ | --------------------------------------------------------------------------------- | --------------------- |
| 纯 CatBase 写动态网页    | `catserve.cat` 用 CatBase 原生 TCP socket 当 HTTP 服务器；`pages/*.cat` 用 `print` 输出 HTML | ✅ 实测                  |
| 在页面上写 CatBase 直接运行 | `ide.html` 编辑器 + `POST /run`：服务端 `catbasecc run` 编译执行用户代码并返回 stdout               | ✅ 实测                  |
| 访问 MySQL           | `csql.cat` / `pages/db.cat` 经 `libc.system` 调 `mysql` 客户端执行 SQL 并渲染结果             | ✅ 编译/流程实测（需 MySQL 实例） |
| 调用各种包              | CatBase 的 `import "./x.cat"` 模块机制 = 包；FFI `Pointer` 还能直接调 opus/ffmpeg/png 等 C 库   | ✅ 验证                  |

---

## CatBase 是什么

CatBase 的设计者叫钟声，珠海豆子机器人科技的创始人，2009 年出过一本《Java程序员，上班那点事儿》。

他做 AI 应用研发时撞上一堵墙：用 Python 写起来爽，跑起来慢，还得装解释器；用 C 跑得快，指针和内存管理能把人折磨到脱发。

于是 CatBase 把两者的优点缝在一起，专为 AI 应用研发而生，目标是在极小运行环境下替代 C。

CatBase编程语言是为Ai应用研发而生，极小运行环境下可以替代C并支持python语法。

CatBase 诞生于对“简单”与“高效”的双重追求。我们希望创造一门无需学习复杂 C 语言、无需忍受 Python 性能瓶颈、无需关心内存与指针的语言，同时保留 Python 的易用性，并拥有 C 级别的性能。

它面向极小资源 Linux 环境与服务器环境，让开发者用最轻松的语法，写出最可靠、最高效的系统程序。

CatBase 是一门类 Python 语法的静态类型编程语言，编译为原生二进制、无需解释器即可运行。它支持 TCP/UDP/HTTP 网络编程、多线程、JSON 处理，并能通过 FFI 调用 `.so` / `.a` C 库，也可将自身编译为共享库，适合快速原型、网络服务与 AI 应用开发。

**获取 catbasecc 编译器**：从官网下载页 <http://catbase-lang.com/download.html> 或 GitHub Releases（<https://github.com/DoRobotAi/catbase-lang/releases>）获取对应平台的二进制包，解压后编译器位于 `bin/catbasecc`。使用方法：

```bash
./bin/catbasecc hello.cat          # 编译为原生可执行文件
./bin/catbasecc run hello.cat      # 编译并直接运行
```

---

## 快速开始

```bash
# 1) 确保 catbasecc 已安装并可用（见上文「CatBase 是什么」），然后启动服务器
catbasecc run catserve.cat
# 打开浏览器： http://127.0.0.1:8080/

# 2) 在页面里写 CatBase 代码，点 Run —— 服务端即时编译运行
# 3) 动态页示例：
#      GET /page/hello   -> 静态式动态页
#      GET /page/clock   -> 实时服务器时间（每秒刷新）
#      GET /page/db      -> 查 MySQL（按实际环境配置连接参数）
```

Docker 一键环境（本项目开发用）：

```bash
docker run -d --name catbase-dev --platform linux/amd64 <catbase 镜像>
docker cp -r . catbase-dev:/work/        # 把本目录拷进去
docker exec -it catbase-dev bash -c "cd /work && ./catbasecc run catserve.cat"
```

---

## 目录结构

```
catbase_studio/
├── catserve.cat        # 纯 CatBase HTTP 服务器 + 路由 + /run 编译器桥接
├── ide.html            # 浏览器内 IDE（编辑器 + Run 按钮 + 输出区）
├── csql.cat            # 访问 MySQL 的复用模块（含 FFI 限制说明）
├── pages/
│   ├── hello.cat       # 动态页示例 1
│   ├── clock.cat       # 动态页示例 2（实时时间）
│   └── db.cat          # 数据库驱动动态页示例（入口内联 FFI）
├── samples/            # 独立可运行的 CatBase 语言示例
│   ├── hello.cat       # 问候 + 当前时间
│   ├── clock.cat       # 打印本地时间
│   ├── fib.cat         # 斐波那契前 15 项
│   ├── str_demo.cat    # 字符串常用操作
│   └── sum100.cat      # 1..100 求和
└── README.md
```

---

## 架构

```
┌──────────────┐     HTTP      ┌──────────────────────────────────────────┐
│  浏览器       │ ───────────> │  catserve.cat  (CatBase, 原生 tcpsocket)   │
│  ide.html     │             │   bind/listen/accept/read/write           │
│  - 写代码     │ <─────────── │   路由: /  /run  /page/<name>              │
│  - Run        │    响应      │      │                                    │
└──────────────┘              │      ├─ GET /        -> 返回 ide.html      │
                              │      ├─ POST /run    -> 落盘+catbasecc run  │
                              │      └─ /page/NAME  -> 运行 pages/NAME.cat  │
                              │              │ 该 .cat 用 print 输出 HTML    │
                              │              ▼                              │
                              │   libc.system("mysql -h.. -u.. -e SQL")     │
                              │              │  (数据库动态页用)              │
                              └──────────────┴──────────────────────────────┘
```

（下方有更直观的图示。）

---

## 关键技术事实（已在 `catbase-dev` 容器内逐一验证）

1. **CatBase 原生就有完整的 TCP 服务端 API**：`tcpsocket()` → `bind(host,port)` → `listen()` → `accept()` → `client.read(n)` / `client.write()` / `close()`。所以"纯 CatBase 当 Web 服务器"无需任何 C 壳。
2. **FFI 能力很强**：`import "libxxx.so" as x; from x import fn(...)` 能链接 opus / vorbis / lame / ffmpeg / portaudio / png / jpeg / webp / libc 等真实 C 库；支持 `Pointer`、`int32_ptr` 出参。`catbasecc` 还支持 `-shared`/`-static` 把 CatBase 编成 `.so`（即"包"）。
3. **编译器桥接**：`catserve` 用 `libc.system(bytes("catbasecc run xxx.cat > out"))` 编译运行用户/页面代码，读回 `out` 作为 HTTP 响应体。
4. **内置能力**：`time()` / `localtime()` / `strftime()`、`file().read/write`、`len()`、`int()`（仅接受 bytes/str 字面量）、`str()`、`ord()`/`chr()`、`split/find/replace`。

---

## 已踩坑 & 绕过（写进代码注释，避免再踩）

| 现象                                  | 原因                                         | 本项目绕法                                            |
| ----------------------------------- | ------------------------------------------ | ------------------------------------------------ |
| `file("w")` 覆盖不截断                   | runtime 的 `File.write` 不 truncate          | `run_cat/run_page` 用**带时间戳的唯一临时文件名**，永不复用        |
| 列表字面量下标 `toks[0]` 报可空类型             | CatBase 索引返回 `?T`/unknown                  | 全部改用**循环变量下标** `toks[i]`（netbeep 同款手法）           |
| `int(str变量)` / `int(bytes变量)` 报错    | 该编译器 `int()` 实际要原始字节，且 `bytes(变量)`→unknown | 不用 Content-Length 解析；用**终止符 `<<<END>>>`** 判收全并剥离 |
| `find()` 返回可空类型，无 `null` 关键字        | `find` 返回 `?usize`                         | 用 `replace(x,"") != x` 判"是否包含"                   |
| **模块里不能调用 FFI**                     | 被 import 的文件里的 `libc.system` 报 undefined   | 涉及系统调用/数据库的代码**必须写在入口 .cat**（见 `pages/db.cat`）   |
| 异常 `e` 不能拼进字符串                      | `str + Exception` 不允许                      | 出错时返回固定提示，不拼接异常对象                                |
| `return file(...).read()` 报 unknown | 链式调用返回值 unknown                            | 先 `out:str = ...read()` 再 `return out`           |
| 不能用 `;` 语句分隔符                       | 语法不支持                                      | 每行一条语句（已写入代码风格）                                  |

---

## MySQL：进阶"纯线协议"路线（待联调）

当前 `db.cat` 走 `mysql` 客户端，简单可靠。若要做到**完全纯 CatBase、不依赖外部客户端**，可用 `tcpsocket()` 直连 3306 实现 MySQL 线协议：

1. 连接 → 解析握手包（取 8+12 字节 scramble）；
2. 鉴权：实现 `mysql_native_password` 的 `SHA1(pw) XOR SHA1(SHA1(pw)+scramble)`（SHA1 可用 FFI 调 OpenSSL，或纯 CatBase 实现——前提是编译器支持位运算 `& | ^ << >>`）；
3. `COM_QUERY` 发 SQL，解析 Resultset 包（列定义 + 行数据 LengthEncodedString）成 CatBase 列表；
4. 封装成 `csql.cat` 的 `db_query()`，页面里直接 `import "./csql.cat"` 调用。

这套线协议代码已规划，接入可达的 MySQL 实例后即可联调（开发沙箱中无 MySQL 且无外网，故未实跑）。

---

## ⚠️ 安全警告

`POST /run` 本质是**远程代码执行（RCE）**：任何人都能提交任意 CatBase 代码在服务端编译运行。

- **仅限本机 / 受信任内网**使用；
- 若要对外，必须：放到 Docker 里限制 CPU/内存/时间（`timeout`/`ulimit`/`seccomp`）、关闭网络出口、对提交做鉴权；
- 临时文件名带时间戳仅防文件残留，不等于沙箱隔离。

---

## 路线图

- [x] 纯 CatBase HTTP 服务器 + 浏览器 IDE + `/run` 编译器桥接
- [x] 动态页 `/page/*`（hello / clock / db）
- [ ] MySQL 纯线协议客户端（不依赖 mysql 客户端）
- [ ] 多用户 / 鉴权 / 资源限制沙箱
- [ ] 包注册表（`import` 社区包）
- [ ] 错误栈更友好、热重载页面

---

## 如何运行（Mac / Linux，Docker）

```bash
# 1. 起容器：务必带 -p 端口映射 + -e TZ=GMT-8（北京时区，见下方说明）
docker run -d --name catbase-dev -p 8080:8080 -e TZ=GMT-8 --platform linux/amd64 catbase

# 2. 把本仓库拷进去（docker cp 没有 -r 参数；catbase 是镜像名不是容器名）
docker cp . catbase-dev:/work/

# 3. 编译并后台启动服务器（catbasecc 在 /work/bin，不在 PATH）
docker exec -d catbase-dev bash -c "cd /work && /work/bin/catbasecc catserve.cat && ./catserve"

# 4. 浏览器打开 http://localhost:8080/
```

容器 stop 后再 start，`catserve` 后台进程不会自启（它是 exec 起的，不是容器 CMD），需重跑第 3 步。

### ⏰ 时区说明（重要）

镜像**没有 tzdata**，而 `localtime()`/`strftime()` 走 C 标准库，只认 `TZ` 环境变量、找不到时区文件就回退 UTC。  
所以容器默认显示 UTC，时钟会比北京慢 8 小时。

修复：给容器/进程设 `TZ=GMT-8`（glibc 的 POSIX TZ 写法，**不需要 zoneinfo 文件**）。

- 正确方向：`TZ=GMT-8` → 北京（UTC+8）。`TZ=UTC+8` 方向反了（会变成 UTC-8）。
- 注意：`TZ=Asia/Shanghai` 需要 tzdata，本镜像没装，**不生效**。

设了 `TZ=GMT-8` 后，容器内 `date` 与 `/page/clock` 都显示北京时间，但 zone 名会标成 `GMT`（只是缩写名，时间值正确）。
