# 加密视频播放器项目学习路线与资料

本文档按项目所需能力拆分学习方向，目标是帮助你从零逐步掌握实现该项目所需的后端、数据库、视频处理、客户端、安全和部署知识。

推荐学习主线：

```text
Rust 后端 + PostgreSQL + HLS/FFmpeg + 安全鉴权
```

客户端、对象存储、部署作为落地能力同步推进。

## 1. Rust 后端

### 学习目标

达到中高级工程能力：

- 能独立实现 REST API。
- 能使用 axum + tokio 写异步后端。
- 能处理登录、播放授权、key 鉴权、播放日志等接口。
- 能写清晰的服务分层。
- 能处理错误、日志、配置和测试。

### 核心知识

```text
Rust 所有权
生命周期基础
Result 错误处理
trait / generic
tokio 异步
axum REST API
serde JSON
tracing 日志
配置管理
测试
```

### 学习资料

- [The Rust Programming Language](https://doc.rust-lang.org/book/)
- [Rust By Example](https://doc.rust-lang.org/rust-by-example/)
- [tokio 官方教程](https://tokio.rs/tokio/tutorial)
- [axum 文档](https://docs.rs/axum/latest/axum/)

### 推荐书籍

- 《Programming Rust》
- 《Rust in Action》

### 练习项目

```text
1. 写一个 Rust REST API。
2. 实现用户登录接口。
3. 实现视频列表接口。
4. 实现 play token 签发接口。
5. 实现 key 鉴权接口。
6. 实现播放日志上报接口。
```

## 2. PostgreSQL / SQL / 表设计

### 学习目标

达到中高级数据库建模能力：

- 能设计合理表结构。
- 能写 JOIN 查询。
- 能设计索引。
- 能使用 EXPLAIN ANALYZE 分析慢查询。
- 能理解事务和并发问题。
- 能使用 migration 管理表结构。

### 核心知识

```text
SELECT / INSERT / UPDATE / DELETE
JOIN
GROUP BY
ORDER BY
LIMIT
主键
外键
唯一约束
NOT NULL
索引
联合索引
事务
隔离级别
EXPLAIN ANALYZE
表设计
一对多
多对多
```

### 学习资料

- [PostgreSQL 官方 Tutorial](https://www.postgresql.org/docs/current/tutorial.html)
- [PostgreSQL 官方文档](https://www.postgresql.org/docs/current/)
- [PostgreSQL Exercises](https://pgexercises.com/)
- [CMU 15-445 Database Systems](https://15445.courses.cs.cmu.edu/)

### 推荐书籍

- 《SQL Antipatterns》
- 《Database Design for Mere Mortals》
- 《Designing Data-Intensive Applications》

### 练习项目

```text
1. 设计 users 表。
2. 设计 videos 表。
3. 设计 video_permissions 表。
4. 写“查询某用户可播放视频”的 SQL。
5. 设计 playback_sessions 表。
6. 设计 playback_events 表。
7. 设计 transcode_jobs 表。
8. 给常用查询加索引。
9. 用 EXPLAIN ANALYZE 查看查询计划。
```

## 3. Rust + PostgreSQL

### 学习目标

- 能在 Rust 后端中稳定访问 PostgreSQL。
- 能使用连接池。
- 能写 migration。
- 能执行事务。
- 能把 SQL 查询结果映射到 Rust struct。

### 核心知识

```text
sqlx
连接池
migration
query!
query_as!
事务
错误处理
```

### 学习资料

- [sqlx 文档](https://docs.rs/sqlx/latest/sqlx/)
- [sqlx GitHub](https://github.com/launchbadge/sqlx)

### 练习项目

```text
1. 用 sqlx 连接 PostgreSQL。
2. 用 migration 创建 users 表。
3. 实现用户注册。
4. 实现用户登录。
5. 实现视频权限查询。
6. 实现播放会话创建。
7. 实现播放日志写入。
```

## 4. 对象存储 / MinIO / S3

### 学习目标

达到工程熟练：

- 能理解 bucket、object、object key。
- 能上传和下载大文件。
- 能设置私有 bucket。
- 能使用 S3 兼容 API。
- 能生成短期签名 URL。
- 能设计视频对象路径。

### 核心知识

```text
Bucket
Object
Object Key
Metadata
Access Key
Secret Key
Endpoint
Region
Bucket Policy
Presigned URL
Multipart Upload
生命周期管理
CDN 回源
```

### 学习资料

- [MinIO 官方文档](https://min.io/docs/minio/container/index.html)
- [Amazon S3 对象和桶文档](https://docs.aws.amazon.com/AmazonS3/latest/userguide/uploading-downloading-objects.html)
- [S3 Presigned URL 文档](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [aws-sdk-s3 Rust 文档](https://docs.rs/aws-sdk-s3/latest/aws_sdk_s3/)

### 推荐对象路径设计

```text
raw/videos/{video_id}/source.mp4
hls/videos/{video_id}/index.m3u8
hls/videos/{video_id}/segment_000.ts
hls/videos/{video_id}/segment_001.ts
covers/{video_id}.jpg
subtitles/{video_id}/zh.vtt
```

### 练习项目

```text
1. docker compose 启动 MinIO。
2. 创建 bucket。
3. 用 Rust 上传 MP4。
4. 用 Rust 下载 m3u8。
5. 列出 hls/videos/{video_id}/ 下的分片。
6. 生成短期 presigned URL。
7. 设置 bucket 为私有。
```

## 5. FFmpeg / HLS / 视频处理

### 学习目标

达到中高级视频处理工程能力：

- 理解 MP4、HLS、m3u8、分片、key、码率、转码。
- 能使用 FFmpeg 生成 HLS。
- 能使用 FFmpeg 生成 HLS AES-128 加密分片。
- 能生成多清晰度播放列表。
- 能排查播放失败。

### 核心知识

```text
MP4
TS
fMP4
H.264
H.265
AAC
HLS
m3u8
master playlist
media playlist
EXT-X-KEY
EXT-X-MAP
AES-128 HLS
多码率
字幕
封面图
ffmpeg
ffprobe
```

### 学习资料

- [FFmpeg Documentation](https://www.ffmpeg.org/documentation.html)
- [ffmpeg 命令文档](https://ffmpeg.org/ffmpeg-doc.html)
- [FFmpeg Formats / HLS muxer](https://ffmpeg.org/ffmpeg-formats.html)
- [RFC 8216 - HTTP Live Streaming](https://www.rfc-editor.org/rfc/rfc8216)

### 练习项目

```text
1. MP4 转 HLS。
2. HLS AES-128 加密。
3. 生成 master.m3u8。
4. 生成 720p / 1080p 多码率。
5. 使用 ffprobe 查看视频信息。
6. 修改 m3u8 练习排查播放失败。
7. 生成封面图。
8. 添加字幕文件。
```

## 6. Qt Widgets / C++ 客户端

### 学习目标

达到工程可落地：

- 能实现 Windows 桌面客户端。
- 能写登录页、视频列表页、播放器页。
- 能调用 REST API。
- 能使用 QSS 做简约界面。
- 能实现动态水印。

### 核心知识

```text
C++ 基础
Qt Widgets
QWidget
QMainWindow
QLayout
QFrame
QLabel
QPushButton
QLineEdit
QNetworkAccessManager
信号槽
QTimer
QThread / QtConcurrent 基础
QSS
资源文件
Windows 打包
```

### 学习资料

- [Qt Widgets 官方文档](https://doc.qt.io/qt-6/qtwidgets-index.html)
- [QNetworkAccessManager 文档](https://doc.qt.io/qt-6/qnetworkaccessmanager.html)
- [Qt Style Sheets Reference](https://doc.qt.io/qt-6/stylesheet-reference.html)

### 推荐书籍

- 《C++ GUI Programming with Qt》
- 《The Book of Qt 4》

### 练习项目

```text
1. 写登录窗口。
2. QNetworkAccessManager 调登录接口。
3. 写视频列表页面。
4. 写播放器页面布局。
5. 用 QSS 做简约样式。
6. QWidget 实现动态水印。
7. 处理 token 过期和错误提示。
```

## 7. libmpv 播放器集成

### 学习目标

达到工程熟练：

- 能把 libmpv 嵌入 Qt QWidget。
- 能播放本地 MP4。
- 能播放远程 HLS m3u8。
- 能播放 HLS AES-128。
- 能控制暂停、继续、seek、音量。
- 能监听播放事件和错误。

### 核心知识

```text
mpv_create
mpv_initialize
mpv_set_option
mpv_command
loadfile
mpv_wait_event
wid 嵌入
pause
seek
volume
播放状态监听
错误处理
```

### 学习资料

- [mpv manual stable](https://mpv.io/manual/stable)
- [libmpv API 文档](https://www.mintlify.com/mpv-player/mpv/embedding/libmpv)
- [mpv-examples / libmpv](https://github.com/mpv-player/mpv-examples/tree/master/libmpv)

### 练习项目

```text
1. Qt QWidget 嵌入 libmpv。
2. 播放本地 MP4。
3. 播放远程 m3u8。
4. 播放 HLS AES-128。
5. 监听播放错误。
6. 实现暂停、继续、seek、音量。
7. 实现播放进度同步到 UI。
```

## 8. 安全 / 鉴权 / Token

### 学习目标

达到中高级安全设计能力：

- 能区分认证和授权。
- 能设计登录系统。
- 能设计 access token / refresh token。
- 能设计短期 play token。
- 能保护 key 接口。
- 能记录安全审计日志。
- 能理解客户端不可信。

### 核心知识

```text
认证 Authentication
授权 Authorization
密码 hash
Argon2id
bcrypt
access token
refresh token
JWT
PASETO
session
play token
权限模型
HTTPS
rate limit
audit log
key 管理
KMS / Vault 基础
replay attack
token 泄露
客户端不可信
```

### 学习资料

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [JWT Introduction](https://jwt.io/introduction)
- [PASETO](https://paseto.io/)

### 练习项目

```text
1. 使用 Argon2id 保存密码 hash。
2. 实现 access token。
3. 实现 refresh token。
4. 实现 play token。
5. 实现 key 接口鉴权。
6. 记录 key 请求日志。
7. 检测同 token 多 IP 请求。
8. 实现 key 接口基础限流。
```

## 9. Docker / 部署 / 运维

### 学习目标

达到生产可用基础：

- 能用 Docker Compose 搭建开发环境。
- 能启动 PostgreSQL、MinIO、Redis、Rust API、Rust Worker。
- 能配置环境变量。
- 能看日志。
- 能做基础服务部署。
- 能理解备份和恢复。

### 核心知识

```text
Docker
Dockerfile
docker compose
镜像
容器
数据卷
网络
环境变量
服务启动顺序
Nginx / Caddy
HTTPS
Prometheus
Grafana
日志
备份恢复
```

### 学习资料

- [Docker Compose 文档](https://docs.docker.com/compose/)
- [Docker Compose CLI](https://docs.docker.com/reference/cli/docker/compose/)
- [How Compose Works](https://docs.docker.com/compose/compose-application-model/)

### 练习项目

```text
1. docker compose 启动 PostgreSQL。
2. docker compose 启动 MinIO。
3. 加入 Redis。
4. 加入 Rust API。
5. 加入 Rust Worker。
6. 配置环境变量。
7. 挂载数据卷。
8. 查看容器日志。
9. 做一次数据库备份和恢复。
```

## 10. 客户端安全基础

### 学习目标

先掌握基础保护，后续再深入：

- 理解客户端不可信。
- 能做基础完整性校验。
- 能做证书 pinning。
- 能理解设备 ID 和本地配置加密。
- 能理解加壳、混淆、虚拟化的取舍。

### 核心知识

```text
代码签名
完整性校验
证书 pinning
设备 ID
本地配置加密
基础反调试
Windows 进程/模块基础
加壳
混淆
虚拟化保护
```

### 学习建议

第一版只需要轻量掌握：

```text
设备 ID
动态水印
HTTPS
基础完整性校验
```

后续再学习：

```text
反调试
反 hook
反 dump
加壳
虚拟化保护
Windows 驱动基础
```

## 11. 推荐学习顺序

如果以项目落地为目标，建议按这个顺序学习：

```text
1. Rust Book
2. axum + tokio
3. SQL + PostgreSQL
4. sqlx
5. MinIO / S3
6. FFmpeg + HLS
7. Qt Widgets
8. libmpv
9. OWASP 鉴权安全
10. Docker Compose
```

## 12. 学习深度建议

| 模块 | 建议深度 | 原因 |
|---|---|---|
| Rust 后端 | 中高级 | 项目主线 |
| PostgreSQL | 中高级 | 表设计和性能很重要 |
| 安全鉴权 | 中高级 | 项目核心价值 |
| HLS / FFmpeg | 中高级 | 项目特色 |
| 对象存储 | 工程熟练 | 必备基础设施 |
| Qt Widgets | 工程熟练 | 客户端落地 |
| libmpv | 工程熟练 | 播放核心 |
| Docker / 部署 | 工程熟练 | 项目上线必备 |
| 客户端加固 | 逐步深入 | 后续增强 |
| DRM | 理解原理 | 后续路线 |

## 13. 阶段目标

### 第一阶段：跑通主链路

```text
Rust API
PostgreSQL
MinIO
FFmpeg HLS
Qt 播放
```

### 第二阶段：做稳

```text
权限模型
play token
key 鉴权
播放日志
限流
转码任务
错误处理
部署
```

### 第三阶段：做强

```text
多码率 HLS
CDN
KMS / Vault
多 key 分片
风控
客户端 pinning
完整性校验
```

### 第四阶段：形成差异化

```text
DASH / CENC
商业 DRM
客户端加固
安全审计
大规模视频分发优化
```

## 14. 总结

这个项目的核心竞争力不是单独会某一个技术，而是能把下面几块组合起来：

```text
Rust 后端
PostgreSQL 数据建模
MinIO/S3 对象存储
FFmpeg/HLS 视频处理
安全鉴权和 key 管理
Qt/libmpv Windows 客户端
Docker 部署
```

优先把下面四块学扎实：

```text
Rust 后端
PostgreSQL
HLS / FFmpeg
安全鉴权
```

这四块决定项目的主体质量和长期技术竞争力。
