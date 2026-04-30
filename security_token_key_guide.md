# 安全鉴权、Token 与 Key 管理指南

本文档整理项目中涉及的密码 hash、Token、Key、权限校验、key 接口保护、限流、日志审计、配置管理、错误码、接口调试和测试等知识。

核心原则：

```text
Token 管权限
Key 管加密
客户端不可信
服务端做最终裁决
敏感信息不进日志、不进 git、不公开暴露
```

## 1. 密码 Hash

密码不能明文存数据库，也不应该用可逆加密保存。正确做法是保存密码 hash。

流程：

```text
注册/改密码：
明文密码 -> Argon2id/bcrypt -> password_hash -> 存数据库

登录：
用户输入密码 -> 使用同算法验证 password_hash -> 成功/失败
```

需要掌握：

```text
hash vs encryption
salt
pepper
Argon2id
bcrypt
密码策略
密码重置
```

建议：

```text
优先使用 Argon2id
不要自己设计密码算法
不要存明文密码
不要把密码写日志
```

学习资料：

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

## 2. Access Token 与 Refresh Token

### Access Token

用途：

```text
访问普通业务 API
```

示例：

```http
GET /api/v1/videos
Authorization: Bearer access_token
```

特点：

```text
短期有效
证明用户已登录
不存敏感数据
```

建议有效期：

```text
15 分钟 - 2 小时
```

可以包含：

```text
user_id
device_id
role
exp
jti
```

不要包含：

```text
密码
AES key
MinIO secret
refresh token
```

### Refresh Token

用途：

```text
换新的 access token
```

特点：

```text
有效期更长
安全要求更高
服务端最好可撤销
```

建议有效期：

```text
7 - 30 天
```

管理方式：

```text
数据库保存 refresh_token_hash
绑定 user_id
绑定 device_id
支持退出登录后撤销
支持异常时全部撤销
```

不要明文保存 refresh token，建议只保存 hash。

## 3. JWT / PASETO

JWT 和 PASETO 都是 Token 格式。

JWT 特点：

```text
生态大
使用广
配置错误风险较多
```

PASETO 特点：

```text
设计更保守
减少算法选择错误
适合短期授权 token
```

需要理解：

```text
claims
签名
过期时间 exp
issuer
audience
jti
防篡改
防重放
不要把敏感信息放 token
```

项目建议：

```text
access token：JWT 或 PASETO
play token：PASETO 或短期 JWT
```

学习资料：

- [JWT Introduction](https://jwt.io/introduction)
- [PASETO](https://paseto.io/)
- [OWASP REST Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html)

## 4. 权限校验

认证解决：

```text
你是谁
```

授权解决：

```text
你能不能做这件事
```

播放前必须校验：

```text
用户是否 active
视频是否 ready
用户是否有 video_permissions
权限是否过期
设备是否允许
play token 是否匹配 video_id
```

原则：

```text
默认拒绝
每个敏感接口都校验
不要只在前端判断
不要相信客户端传来的 user_id
后端从 token/session 中取用户身份
```

学习资料：

- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

## 5. Play Token

Play Token 是短期播放凭证，不等同于登录 Token。

登录 Token 证明：

```text
你是这个用户
```

Play Token 证明：

```text
你这一次可以播放这个视频
```

用途：

```text
请求 m3u8
请求 key
```

示例：

```text
/api/v1/videos/{video_id}/hls/index.m3u8?play_token=xxx
/api/v1/videos/{video_id}/key?play_token=xxx
```

建议包含：

```text
user_id
video_id
device_id
play_session_id
expire_at
nonce
```

建议有效期：

```text
5 - 30 分钟
```

不要包含：

```text
AES content key
用户密码
长期 secret
```

## 6. Key 接口保护

key 接口是本项目最关键的安全接口。

示例：

```http
GET /api/v1/videos/{video_id}/key?play_token=xxx
```

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

响应必须是原始 16 字节 AES key，不是 JSON。

推荐响应头：

```http
Content-Type: application/octet-stream
Cache-Control: no-store
Pragma: no-cache
```

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

不要记录：

```text
AES key
access token 原文
refresh token 原文
play token 原文
密码
```

## 7. 限流

限流用于防止某个用户、IP、设备或会话短时间大量请求。

项目重点限流：

```text
登录接口
播放授权接口
key 接口
上传接口
```

常见限流维度：

```text
IP
user_id
device_id
play_session_id
video_id
```

简单策略：

```text
每分钟最多 N 次
连续失败锁定
key 请求异常直接拒绝
同 token 多 IP 触发风控
```

后续可以用 Redis 实现：

```text
rate_limit:key:{ip}
rate_limit:key:{user_id}
rate_limit:key:{play_session_id}
```

学习资料：

- [OWASP REST Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html)

## 8. 日志审计

日志审计要回答：

```text
谁
什么时候
从哪里
对什么资源
做了什么
结果如何
失败原因是什么
```

项目必须记录：

```text
登录成功/失败
播放授权
m3u8 请求
key 请求
token 过期
权限拒绝
异常频率
播放事件
转码失败
```

日志字段建议：

```text
request_id
user_id
video_id
device_id
play_session_id
ip
user_agent
event
result
error_code
latency_ms
created_at
```

不要记录：

```text
密码
access token 原文
refresh token 原文
play token 原文
AES key
完整身份证/手机号等敏感信息
```

学习资料：

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)

## 9. 配置文件管理

配置不要写死在代码里。

常见配置：

```text
DATABASE_URL
MINIO_ENDPOINT
MINIO_ACCESS_KEY
MINIO_SECRET_KEY
ACCESS_TOKEN_SIGNING_KEY
PLAY_TOKEN_SIGNING_KEY
VIDEO_MASTER_KEY
FFMPEG_PATH
SERVER_PORT
LOG_LEVEL
ACCESS_TOKEN_TTL
PLAY_TOKEN_TTL
```

常见方式：

```text
.env
config.toml
环境变量
Docker Compose environment
Kubernetes Secret
Vault / KMS
```

Rust 常用：

```text
config crate
dotenvy
serde
```

原则：

```text
开发配置可以放 .env
提供 .env.example
.env 不提交 git
生产密钥走环境变量或密钥管理系统
Secret 不写入代码仓库
```

## 10. 错误码设计

不要只返回：

```json
{
  "error": "failed"
}
```

推荐格式：

```json
{
  "code": "PLAY_TOKEN_EXPIRED",
  "message": "play token expired",
  "request_id": "req_abc"
}
```

常见错误码：

```text
UNAUTHORIZED
FORBIDDEN
VIDEO_NOT_FOUND
VIDEO_NOT_READY
PLAY_TOKEN_EXPIRED
PLAY_TOKEN_INVALID
KEY_ACCESS_DENIED
RATE_LIMITED
TRANSCODE_FAILED
INTERNAL_ERROR
```

意义：

```text
客户端好处理
日志好搜索
问题好排查
接口更稳定
```

## 11. 日志排查

当用户反馈播放失败时，应能按链路排查：

```text
登录是否成功
/play 是否返回
m3u8 是否请求
key 是否请求
key 是否被拒绝
分片是否 404
libmpv 报什么错
```

后端日志应包含：

```text
request_id
user_id
video_id
play_session_id
status
error_code
latency
```

Rust 建议学习：

```text
tracing
tracing-subscriber
structured logging
request id middleware
```

## 12. 接口调试：Postman / curl

接口调试用于在客户端完成前先验证后端 API 是否正常。

需要掌握：

```text
HTTP method
Header
Body
Status code
JSON
Bearer token
二进制响应
```

curl 示例：

```bash
curl -H "Authorization: Bearer xxx" \
  http://localhost:8080/api/v1/videos
```

key 接口返回二进制，不是 JSON：

```bash
curl -o key.bin \
  "http://localhost:8080/api/v1/videos/v_123/key?play_token=xxx"
```

学习资料：

- [Postman Learning Center](https://learning.postman.com/)
- [everything curl](https://everything.curl.dev/)

## 13. 单元测试

单元测试测试一个小函数或小模块。

适合测试：

```text
密码 hash 验证
token 生成/验证
权限判断函数
错误码转换
m3u8 URL 重写
```

典型用例：

```text
过期 play token 必须失败
video_id 不匹配的 token 必须失败
无权限用户不能获取 key
错误码转换稳定
```

目标：

```text
保证关键安全逻辑不会被改坏
```

## 14. 集成测试

集成测试测试多个组件一起工作。

适合测试：

```text
登录 -> 获取视频列表 -> 请求播放 -> 请求 key
上传视频 -> 转码任务 -> MinIO 生成 HLS
权限过期 -> /play 被拒绝
无效 play token -> key 接口拒绝
```

需要：

```text
测试数据库
测试 MinIO
测试 HTTP server
```

Rust 可用：

```text
tokio::test
sqlx test db
testcontainers
reqwest
```

## 15. Token 与 Key 的区别

Token 和 Key 不要混在一起。

```text
Token：证明谁能做什么
Key：用于签名、加密、解密
```

一句话：

```text
Token 管权限
Key 管加密
```

## 16. 项目里的 Token

### Access Token

用途：

```text
访问普通业务 API
```

生命周期：

```text
生成：登录/刷新
存储：客户端内存或安全本地存储
有效期：短
撤销：通常等过期，或配合黑名单
```

### Refresh Token

用途：

```text
换新的 access token
```

生命周期：

```text
生成：登录
存储：客户端安全存储
服务端：保存 hash
有效期：长
撤销：退出登录/异常登录/密码修改
```

### Play Token

用途：

```text
授权某一次视频播放
```

生命周期：

```text
生成：用户点击播放
存储：短期使用
有效期：很短
撤销：播放会话结束、异常行为、权限变更
```

### Session ID

严格说它不是 Token，但很重要。

用途：

```text
标识一次播放会话
```

用于关联：

```text
/play 授权
m3u8 请求
key 请求
播放事件
异常检测
```

## 17. 项目里的 Key

### Password Hash Salt

用途：

```text
防止相同密码 hash 一样
防止彩虹表
```

通常由 Argon2id/bcrypt 库自动处理。

### Token Signing Key

用途：

```text
签名 access token / play token
```

管理原则：

```text
不能写死代码里
不能提交 git
通过环境变量或密钥管理系统加载
定期轮换
区分开发/生产环境
```

建议：

```text
access token signing key
play token signing key
可以分开
```

分开可以减少一个用途泄露影响全部 token 的风险。

### Refresh Token Secret / Hash

如果 refresh token 是随机值，服务端数据库保存 hash。

原则：

```text
refresh token 明文只返回给客户端一次
数据库只保存 hash
退出登录删除或标记 revoked
```

### Video Content Key

这是视频 AES key。

用途：

```text
加密 HLS 分片
播放器请求 key 后解密播放
```

第一版：

```text
每个视频一把 content key
```

后续增强：

```text
每个清晰度一把 key
每 N 个分片一把 key
每 N 分钟一把 key
```

管理原则：

```text
不要公开存储
不要放 m3u8 静态公开路径
不要放客户端配置
不要放 token 里
不要写日志
```

数据库建议保存：

```text
encrypted_content_key
key_version
video_id
iv
created_at
```

### Master Key / KMS Key

用途：

```text
保护 video content key
```

例如：

```text
content key 明文 -> 用 master key 加密 -> encrypted_content_key 存数据库
```

播放时：

```text
encrypted_content_key -> 后端解密 -> 返回给 libmpv
```

第一版可以简化：

```text
master key 从环境变量读取
```

生产更推荐：

```text
Vault
云 KMS
HSM
```

管理原则：

```text
master key 不进数据库
master key 不进 git
master key 不能写日志
生产环境使用 KMS/Vault
支持轮换
```

### MinIO / S3 Access Key

用途：

```text
后端访问对象存储
```

管理原则：

```text
只给后端/worker 使用
不要给客户端
不要提交 git
最小权限
区分读写权限
定期轮换
```

客户端不应该拿到 MinIO access key / secret key。

客户端如果要访问分片，使用：

```text
公开受控路径
presigned URL
CDN 签名 URL
```

### TLS Private Key

用途：

```text
HTTPS 证书私钥
```

管理原则：

```text
只在服务器或负载均衡中保存
不要进代码仓库
定期续期
使用自动化证书管理
```

## 18. 开发环境与生产环境管理方式

### 开发环境

可以使用：

```text
.env
docker-compose.yml
本地 config.toml
```

示例：

```text
DATABASE_URL=postgres://...
MINIO_ACCESS_KEY=...
MINIO_SECRET_KEY=...
ACCESS_TOKEN_SIGNING_KEY=...
PLAY_TOKEN_SIGNING_KEY=...
VIDEO_MASTER_KEY=...
```

注意：

```text
.env 不提交 git
提供 .env.example
```

### 生产环境

推荐：

```text
环境变量
Docker/Kubernetes Secret
Vault
云 KMS
CI/CD Secret
```

生产不建议把密钥写在普通配置文件里。

## 19. 完整播放过程中的 Token / Key

```text
1. 用户登录
   -> 后端签发 access_token + refresh_token

2. 用户请求视频列表
   -> 使用 access_token

3. 用户点击播放
   -> 后端校验 access_token 和权限
   -> 生成 play_session_id
   -> 签发 play_token

4. libmpv 请求 m3u8
   -> URL 带 play_token

5. libmpv 请求 key
   -> URL 带 play_token

6. 后端验证 play_token
   -> 查询 encrypted_content_key
   -> 使用 master key/KMS 解密 content key
   -> 返回 16 字节 AES key

7. libmpv 下载加密分片
   -> 使用 AES key 解密播放
```

## 20. 最重要的安全规则

```text
Token 不放 key
Key 不进日志
Secret 不进 git
Refresh token 服务端只存 hash
Content key 数据库存加密值
Master key 不进数据库
MinIO secret 不给客户端
play token 短期有效
key 接口必须鉴权
```

## 21. 推荐学习顺序

```text
1. HTTP / REST / curl / Postman
2. 密码 hash
3. access token / refresh token
4. JWT 或 PASETO
5. 权限校验
6. play token
7. key 接口保护
8. 错误码设计
9. tracing 日志
10. 配置文件管理
11. 限流
12. 单元测试
13. 集成测试
14. 日志排查
```

## 22. 总结

本项目的安全核心不是某一个算法，而是一整套链路：

```text
密码安全存储
登录 token
播放授权 token
权限校验
key 接口保护
key 生命周期管理
日志审计
限流和风控
配置和密钥安全
测试保障
```

最关键的边界：

```text
Token 证明权限
Key 用于加密
服务端保护 key
客户端只在播放时拿到必要 key
所有敏感行为都要可审计
```
