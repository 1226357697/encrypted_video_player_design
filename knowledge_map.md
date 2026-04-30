# 加密视频播放器项目知识地图

本文档归纳项目所需知识、每块应该学到什么程度，以及学习这些知识的意义。

项目目标不是单纯写一个播放器，而是完成一套：

```text
Rust 后端 + PostgreSQL + MinIO/S3 + FFmpeg/HLS + Qt/libmpv + 安全鉴权
```

组合起来的加密视频播放系统。

## 1. 项目整体需要哪些知识

这个项目可以拆成六大块：

```text
后端
数据库
对象存储
视频处理
客户端
安全鉴权
部署运维
```

每块都不需要一开始学到专家级，但要按项目主线逐步深入。

## 2. 后端知识

### 需要学什么

```text
Rust 基础语法
tokio 异步编程
axum REST API
serde JSON
错误处理
tracing 日志
配置管理
接口测试
```

### 在项目里负责什么

```text
用户登录
视频列表
播放授权
play token 签发
m3u8 动态返回
key 鉴权接口
播放日志接收
转码任务管理
```

### 应该学到什么程度

建议学到中高级。

你应该能做到：

```text
独立写 REST API
设计清晰的模块分层
处理异步请求
处理错误和日志
写安全敏感接口
写测试
部署运行服务
```

后端是这个项目的主线能力，决定系统是否可用、可维护、可扩展。

## 3. PostgreSQL / SQL / 表设计

### 需要学什么

```text
SQL 基础
SELECT / INSERT / UPDATE / DELETE
JOIN
GROUP BY
ORDER BY
LIMIT
主键
外键
唯一约束
索引
联合索引
事务
migration
EXPLAIN ANALYZE
```

### 表设计是什么

表设计就是决定：

```text
系统里有哪些数据
每类数据放在哪张表
每张表有哪些字段
表和表之间是什么关系
哪些字段必须唯一
哪些字段不能空
哪些字段需要索引
```

例如：

```text
users：用户
videos：视频
video_permissions：用户和视频的播放权限
video_keys：视频加密 key 记录
playback_sessions：播放会话
playback_events：播放事件
transcode_jobs：转码任务
```

### 表之间的关系

常见关系：

```text
一对一：一个用户对应一个用户详情
一对多：一个视频有多个播放事件
多对多：一个用户可以看多个视频，一个视频也可以被多个用户观看
```

多对多通常需要中间表。

在本项目中：

```text
users <-> video_permissions <-> videos
```

`video_permissions` 表用于表达哪个用户可以看哪个视频。

### 数据库优化是什么

数据库优化就是让查询更快、更稳定、更省资源。

常见优化包括：

```text
加合适的索引
避免全表扫描
减少不必要字段
分页查询
避免 N+1 查询
合理设计表结构
拆分冷热数据
归档历史日志
使用连接池
分析慢查询
```

常用工具：

```sql
EXPLAIN ANALYZE
SELECT ...
```

它可以告诉你数据库如何执行 SQL、有没有用索引、耗时在哪里。

### 在项目里负责什么

PostgreSQL 存：

```text
用户
视频信息
播放权限
视频 key 记录
播放会话
播放日志
key 请求日志
转码任务
设备信息
```

不存：

```text
MP4 文件
HLS 分片
封面大图
字幕文件
```

这些大文件应该放 MinIO/S3。

### 应该学到什么程度

建议学到中高级。

你应该能做到：

```text
设计项目表结构
写 JOIN 查询
设计常用索引
理解事务
用 migration 管理表结构
用 EXPLAIN ANALYZE 分析慢查询
```

很多人只会 CRUD，但不会表设计和查询优化。把这块学深，会明显提升工程能力。

## 4. 对象存储：MinIO / S3

### 感性理解

MinIO/S3 可以理解成：

```text
一个专门存海量文件的网络仓库
```

类比：

```text
Bucket = 仓库
Object = 仓库里的包裹
Object Key = 包裹编号 / 货架路径
Metadata = 包裹标签
Access Key / Secret Key = 仓库管理员钥匙
Policy = 谁能进仓库、谁能拿包裹的规则
Presigned URL = 临时取货码
```

### 需要学什么

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

### 在项目里负责什么

对象存储放大文件：

```text
原始 MP4
HLS m3u8
加密 ts/m4s 分片
封面图
字幕文件
```

推荐路径：

```text
raw/videos/{video_id}/source.mp4
hls/videos/{video_id}/index.m3u8
hls/videos/{video_id}/segment_000.ts
hls/videos/{video_id}/segment_001.ts
covers/{video_id}.jpg
subtitles/{video_id}/zh.vtt
```

### 对象存储和数据库的区别

PostgreSQL 存结构化信息：

```text
用户 ID
视频标题
播放权限
订单状态
播放日志
文件路径
```

MinIO/S3 存大文件：

```text
MP4
m3u8
ts 分片
封面
字幕
```

一句话：

```text
PostgreSQL 管信息
MinIO/S3 管文件
Rust 后端管权限
libmpv 管播放
```

### 安全原则

```text
原始 MP4 不公开
AES key 不放对象存储公开访问
m3u8 最好由后端动态返回
分片可以由对象存储/CDN 分发
对象 key 不要暴露敏感业务信息
```

重要直觉：

```text
仓库里可以放上锁的视频分片
钥匙不能挂在仓库门口
```

### 应该学到什么程度

建议学到工程熟练。

你应该能做到：

```text
创建 bucket
上传文件
下载文件
判断文件是否存在
删除文件
列出某个前缀下的文件
设置私有 bucket
服务端用 SDK 访问 MinIO
生成短期签名 URL
处理大文件上传
```

对象存储不需要一开始学得特别深，但必须能安全落地。

## 5. 视频处理 / HLS / FFmpeg

### 需要学什么

```text
MP4
H.264
H.265
AAC
HLS
m3u8
ts 分片
fMP4
EXT-X-KEY
AES-128 HLS
多码率
字幕
封面图
FFmpeg
ffprobe
```

### 在项目里负责什么

```text
上传 MP4
转码
切片
加密
生成 m3u8
生成封面
生成多清晰度
排查播放失败
```

### 应该学到什么程度

建议学到中高级。

你应该能做到：

```text
用 FFmpeg 把 MP4 转 HLS
生成 HLS AES-128 加密分片
理解 m3u8 中的 key URL
理解播放器如何请求分片和 key
生成多码率播放列表
用 ffprobe 分析视频信息
排查 m3u8、分片、编码导致的播放失败
```

这是项目的特色能力，比普通 CRUD 后端更有辨识度。

## 6. Qt / C++ 客户端

### 需要学什么

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
QSS
资源文件
Windows 打包
```

### 在项目里负责什么

```text
登录页
视频列表页
播放器页
调用 REST API
保存 token
传 play_url 给 libmpv
显示动态水印
上报播放日志
处理错误提示
```

### Qt Widgets + QSS 和 QML 的选择

以落地为目标，第一版建议：

```text
Qt Widgets + QSS
```

原因：

```text
libmpv 嵌入更直接
Windows 桌面体验稳定
表单和列表开发简单
开发成本低
更适合 MVP
```

QML 更适合：

```text
现代动画
沉浸式播放器
触控界面
复杂视觉效果
高定制 UI
```

但第一版不建议被 UI 技术栈拖慢。

### 应该学到什么程度

建议学到工程熟练。

你应该能做到：

```text
实现稳定 Windows 客户端
写登录页和播放器页
调用后端 REST API
用 QSS 做简约界面
实现动态水印
处理播放错误和 token 过期
```

## 7. libmpv 播放器集成

### 需要学什么

```text
libmpv 初始化
mpv_create
mpv_initialize
mpv_set_option
mpv_command loadfile
mpv_wait_event
wid 嵌入
暂停
继续
seek
音量
播放状态监听
错误处理
```

### 在项目里负责什么

libmpv 负责：

```text
加载 m3u8
解析 HLS
请求 key URL
下载加密分片
HLS AES-128 解密
音视频解码
渲染播放
```

Qt 负责：

```text
业务 API
UI
水印
播放控制按钮
日志上报
```

### 应该学到什么程度

建议学到工程熟练。

你应该能做到：

```text
把 libmpv 嵌入 QWidget
播放本地 MP4
播放远程 m3u8
播放 HLS AES-128
监听播放错误
实现进度、暂停、seek、音量控制
```

## 8. 安全和鉴权

### 需要学什么

核心概念：

```text
认证 Authentication：你是谁
授权 Authorization：你能不能做这件事
会话 Session：这次登录/播放的状态
Token：服务端签发的凭证
权限 Permission：用户是否能看某个视频
审计 Audit：记录谁在什么时候做了什么
```

具体知识：

```text
密码 hash
Argon2id / bcrypt
access token
refresh token
JWT / PASETO
play token
HTTPS
权限校验
短期 token
key 接口保护
限流
日志审计
动态水印
客户端不可信
```

### 认证要学什么

认证解决：

```text
这个用户是谁
```

你需要掌握：

```text
密码不能明文保存
密码要 hash
推荐 Argon2id
登录成功后签发 access token
refresh token 用于刷新登录状态
```

普通 API 使用：

```text
Authorization: Bearer xxx
```

### 授权要学什么

授权解决：

```text
这个用户能不能播放这个视频
```

播放前检查：

```text
用户是否 active
视频是否 ready
用户是否有 video_permissions
权限是否过期
设备是否允许
```

### play token 要学什么

play token 是短期播放凭证。

登录 token 证明：

```text
你是这个用户
```

play token 证明：

```text
你这一次可以播放这个视频
```

play token 应包含：

```text
user_id
video_id
device_id
play_session_id
expire_at
nonce
```

有效期建议：

```text
5 到 30 分钟
```

第一版可用于：

```text
/index.m3u8?play_token=xxx
/key?play_token=xxx
```

### key 接口安全要学什么

key 接口是最关键的安全点。

必须校验：

```text
play_token 是否有效
token 是否过期
user_id 是否匹配
video_id 是否匹配
device_id 是否匹配
用户是否仍有权限
视频是否仍可播放
请求频率是否异常
```

key 响应：

```http
Content-Type: application/octet-stream
Cache-Control: no-store
Pragma: no-cache
```

返回内容：

```text
原始 16 字节 AES key
```

不要返回 JSON。

必须记录日志：

```text
user_id
video_id
device_id
play_session_id
ip
user_agent
requested_at
result
failure_reason
```

### 动态水印和追责

客户端播放时，key 和明文数据最终会出现在客户端环境中。HLS AES-128 不是商业 DRM，不能彻底防内存抓 key 或录屏。

所以必须加入追责机制。

水印内容：

```text
user_id
手机号尾号
device_id
时间
session_id
```

水印策略：

```text
半透明
定时移动
不要固定角落
不要太容易裁剪
```

### 应该学到什么程度

建议学到中高级。

你应该能做到：

```text
设计登录系统
设计 access token / refresh token
设计 play token
设计权限表
保护 key 接口
记录审计日志
做基础限流和异常检测
理解客户端不可信
理解 HLS AES-128 的安全边界
```

安全鉴权是这个项目的核心价值之一。

## 9. Docker / 部署 / 运维

### 需要学什么

```text
Docker
Docker Compose
镜像
容器
数据卷
网络
环境变量
服务启动顺序
日志
备份恢复
Nginx / Caddy
HTTPS
Prometheus / Grafana 基础
```

### docker-compose 是什么

`docker-compose` 是一个用配置文件一次性启动多个服务的工具。

这个项目可能需要：

```text
PostgreSQL
MinIO
Redis
Rust API
Rust Worker
```

用 docker-compose 可以一条命令启动：

```bash
docker compose up
```

### 应该学到什么程度

建议学到工程熟练。

你应该能做到：

```text
用 Docker Compose 搭建开发环境
配置 PostgreSQL
配置 MinIO
配置 Redis
配置 Rust API
查看日志
管理环境变量
做基础备份恢复
```

部署能力决定项目能不能真正跑起来。

## 10. 客户端安全基础

### 需要学什么

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

### 第一版要做到什么程度

第一版轻量掌握即可：

```text
设备 ID
动态水印
HTTPS
基础完整性校验
```

后续再深入：

```text
证书 pinning
反调试
反 hook
反 dump
加壳
虚拟化保护
```

### 安全边界

不要把主要安全性建立在客户端加壳上。

第一版重点应该是：

```text
服务端权限控制
短期 token
key 接口鉴权
播放日志
动态水印追责
```

客户端安全用于提高攻击成本，不是最终信任来源。

## 11. 每块应该学到什么程度

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

## 12. 技术竞争力重点

如果想形成更强竞争力，不要平均用力。

重点做深：

```text
Rust 后端
PostgreSQL 建模和优化
HLS / FFmpeg 视频处理
安全鉴权和 key 管理
```

做到工程熟练：

```text
MinIO / S3
Qt Widgets
libmpv
Docker Compose
```

后续逐步深入：

```text
客户端加固
DASH / CENC
商业 DRM
CDN
KMS / Vault
风控系统
```

这个项目形成的核心竞争力是：

```text
音视频工程 + 后端工程 + 安全工程
```

这比普通 CRUD 后端更有辨识度。

## 13. 推荐学习顺序

```text
1. HTTP / REST 基础
2. Rust + axum 写简单 API
3. PostgreSQL + SQL + 表设计
4. sqlx 连接 PostgreSQL
5. MinIO 文件上传下载
6. FFmpeg 生成 HLS
7. HLS m3u8 / key / segment 原理
8. Qt Widgets 写登录页
9. QNetworkAccessManager 调后端 API
10. libmpv 嵌入 QWidget 播放 m3u8
11. 动态水印 QWidget
12. token 鉴权和 key 接口安全
13. Docker Compose 部署后端组件
```

## 14. 阶段性目标

### 第一阶段：能做出来

```text
Rust API
PostgreSQL
MinIO
FFmpeg HLS
Qt 播放
```

### 第二阶段：做稳定

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

### 第三阶段：做安全

```text
多 key 分片
key 请求滑动窗口
风控
证书 pinning
完整性校验
KMS / Vault
CDN 签名 URL
```

### 第四阶段：做差异化

```text
DASH / CENC
商业 DRM
客户端加固
安全审计
大规模视频分发优化
```

## 15. 总结

这个项目不是某一个单点技术难，而是系统集成能力要求高。

你需要建立的知识结构是：

```text
Rust 后端负责业务和鉴权
PostgreSQL 负责结构化数据
MinIO/S3 负责大文件
FFmpeg/HLS 负责视频切片加密
Qt/libmpv 负责 Windows 播放
安全鉴权负责权限、key 和追责
Docker 负责开发和部署环境
```

优先做深：

```text
Rust 后端
PostgreSQL
HLS / FFmpeg
安全鉴权
```

其他模块做到能稳定落地，再按需求逐步深入。
