# 项目实战

# **项目概述（产品需求）**

构建一个**实时小组协作聊天室**（Mini Team Chat）：

- 用户注册/登录（JWT），每个用户可创建/加入聊天室（room）。
- 聊天消息通过room-service 处理并广播；支持历史消息查询（存 PostgreSQL）。
- 前端为 Next.js + React：登录页、房间列表、房间内实时聊天（WebSocket）。
- 后端微服务间通过 gRPC（Protobuf）通信：user-service提供用户查询/鉴权，room-service 调用它做鉴权/查用户信息。
- 强调：Go 的并发（worker pool、消息队列/通道）、WebSocket 双端实现（Go 与 浏览器 JS）、JWT 鉴权中间件、GORM + PostgreSQL、Protobuf 设计。

# **总体架构（建议）**

- repo 根：/infra（docker-compose/Postgres）,/services/user,/services/room,/web（Next.js）
- 用户微服务：user-service（Kratos + gRPC + GORM + Postgres）
  - 管理用户、注册、登录、JWT 颁发、用户存储
  - 提供 gRPC 服务 User.GetUser, User.VerifyToken
- 房间微服务：room-service（Kratos + gRPC + GORM + Postgres + WebSocket）
  - 管理房间、消息持久化、WebSocket 连接池与广播、并发消息处理
  - 调用user-service 的 gRPC 验证用户/查询用户信息
- 前端：web（Next.js）
  - 登录/注册页面（调用 user-service REST/gRPC gateway 或者暴露 HTTP API）
  - 房间列表与聊天页（使用 WebSocket 连接到 room-service）
- infra:docker-compose 启动 Postgres（用户表/消息表各自一个 DB 或同一 DB 的不同 schema），可选 kratos dev config。

***

# **Day-by-day 任务（五天）**

## **Day 1 — 环境与基础后端（身份认证服务：user-service）**

**目标产出：** 可运行的user-service，支持注册、登录，JWT 签发与验证；数据库能存用户记录；生成 Protobuf 文件并运行 gRPC 服务接口。

**任务清单**

1. 建 repo skeleton：services/user，初始化 Go module。
2. 设计 DB：users 表（id uuid, username unique, email, password\_hash, created\_at）。
3. 使用 GORM 连接 PostgreSQL，做 AutoMigrate。
4. 实现注册接口（HTTP/gRPC）：接受 username/email/password，存 hashed password（bcrypt）。
5. 实现登录接口：验证后返回 JWT（RS256 或 HS256，建议 HS256 简单实现）。
6. 定义 Protobuf（user.proto）并生成 gRPC 代码：服务 UserService 包含 GetUser, VerifyToken RPC。

```protocol buffers 
syntax = "proto3";
package user;
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserReply);
  rpc VerifyToken(VerifyTokenRequest) returns (VerifyTokenReply);
}
message GetUserRequest { string id = 1; }
message GetUserReply { string id = 1; string username = 2; string email = 3; }
message VerifyTokenRequest { string token = 1; }
message VerifyTokenReply { bool ok = 1; string user_id = 2; }
```


1. 写 JWT 中间件（用于 Kratos HTTP/gRPC side）并提供 VerifyToken RPC。
2. 本地运行：docker-compose启动 Postgres；验证注册/登录能成功，能通过 gRPC 调用VerifyToken。

**评审点**

- 密码是否安全（bcrypt）；JWT 模式明确；Protobuf、gRPC 正确生成且能互调；自动迁移成功。

***

## **Day 2 — 房间服务骨架（room-service）与前端基础**

**目标产出：** room-service能通过 gRPC 与user-service 验证用户；实现房间 CRUD；前端建立项目并实现登录页 + 房间列表（静态调用 user-service HTTP 或 API）。

**任务清单**

1. services/room skeleton，Go module，Kratos 服务框架。
2. 设计 DB：rooms（id, name, owner\_id, created\_at），messages（id, room\_id, sender\_id, content, created\_at）。
3. 实现房间 CRUD（创建房间需验证 JWT -> 调用user-service.VerifyToken）。
4. 在room-service中集成 gRPC client 调用UserService.
5. 前端web 初始化 Next.js + React：
   - 页面：/login, /rooms, /rooms/\[id]
   - 实现登录表单，提交到user-service 登录接口（HTTP）。
6. 实现房间列表页面，调用room-service 的 HTTP 接口拉取房间列表（有鉴权 header: Bearer JWT）。

**评审点**

- gRPC client 调用正确；前端能登录并拿 JWT；房间创建流程把用户 id 正确关联为 owner。

***

## **Day 3 — WebSocket 实时通信 & 前端连接**

**目标产出：** WebSocket 双向连接实现，前端可连到room-service 并实时收发消息；后端持久化消息到 PostgreSQL；实现基本广播机制。

**任务清单**

1. 在room-service内暴露 WebSocket endpoint（例如/ws?room\_id=xxx），WebSocket 握手时验证 JWT（使用user-service.VerifyToken）。
2. 设计连接管理（简单连接池/room map）：map\[roomID]RoomHub 每个 hub 管理客户端 list。
3. 实现消息接收与广播：客户端发消息 -> hub 接收到后：
   - 入库（messages 表）
   - 并发广播给 room 中的所有已连接客户端
4. 前端房间页实现 WebSocket 客户端：
   - 建立连接时给 server 带Authorization: Bearer \<JWT>（或在 query token）
   - 接收消息并追加到聊天窗口；支持发送消息。
5. 测试：多个浏览器窗打开同一房间，实现实时聊天。

**并发/稳定性任务（要点）**

- 后端 hub 用 channel 做消息队列；hub 内部使用 goroutine 处理广播（避免阻塞写）。
- 在写 WebSocket 时做 write timeout；处理连接断开并清理。
- 推荐实现一个 worker pool（可选高级）：当消息入库和处理都要并发处理时，用固定数量 goroutine（workers）从消息 channel 取任务处理（persist + 推送），避免瞬时流量撑爆 DB。

**评审点**

- 消息能实时广播，入库成功；并发写入不会丢消息（至少在常见情况下）；连接断开时资源清理正常。

***

## **Day 4 — 并发控制、消息一致性、历史与分页、前端优化**

**目标产出：** 完善并发逻辑（worker pool、速率限制）、消息历史分页查询、前端实现消息分页与滚动优化、基本压力测试。

**任务清单**

1. 在room-service 中实现 worker pool：
   - msgInboundChan -> 多个 worker goroutine 负责 message persist + 推送
   - 需要幂等/顺序保证：按消息 create time/order 保存
2. 加入简单速率限制（per-socket 或 per-user）防刷：token bucket 或计数器。
3. 实现消息历史查询接口（REST/gRPC），支持分页（limit, offset 或 cursor）。
4. 前端实现消息懒加载/分页：滚动到顶部触发加载上一页历史消息。
5. 写简单压力/并发测试脚本（Go 或 k6），模拟 N 个客户端同时在同一房间发送消息，观察消息丢失或延迟。
6. 修复发现的竞态/死锁问题（建议使用go test -race）。

**评审点**

- worker pool 正确处理并发请求；历史分页返回正确顺序；前端滚动体验良好；基本压力测试无明显故障。

***

## **Day 5 — 安全、测试、文档与部署**

**目标产出：** 完整运行的 demo（docker-compose），测试覆盖（单元 + 集成基本用例），项目文档与演示视频/说明，代码 review & 评分。

**任务清单**

1. 完善安全：
   - JWT 过期/刷新策略（建议提供 refresh token endpoint 或短期 access token + refresh token）
   - WebSocket 握手时 token 过期处理
   - SQL 注入检查（使用 GORM parameter binding）
2. 补充测试：
   - user-service 单元测试（注册/登录）
   - room-service 单元测试（hub 管理、消息处理）
   - 集成测试：启动 Postgres + 两个服务（可以用 docker-compose），用测试脚本登录并在房间内发送消息，校验 DB 中记录与接收端消息。
3. 部署/运行脚本：docker-compose.yml 启动 Postgres、user-service、room-service（可把前端放 dev 模式运行）。
4. 文档：README（如何运行、接口说明、Protobuf 概要、数据库 schema、如何测试）。
5. 演示：提交 5–10 分钟演示视频或现场 demo，展示用户注册/登录、创建房间、多人实时聊天、历史加载。

**评审点**

- Demo 能跑通；主要功能都有测试；文档清晰；安全基本措施到位。

***

# **关键技术细节 / 示例片段**

## **示例 GORM model（Go）**

```go 
type User struct {
  ID        uuid.UUID `gorm:"type:uuid;primaryKey"`
  Username  string    `gorm:"uniqueIndex;size:64"`
  Email     string    `gorm:"uniqueIndex;size:128"`
  Password  string    `gorm:"size:128"` // bcrypt hash
  CreatedAt time.Time
}
type Room struct {
  ID       uuid.UUID `gorm:"type:uuid;primaryKey"`
  Name     string
  OwnerID  uuid.UUID `gorm:"type:uuid"`
  CreatedAt time.Time
}
type Message struct {
  ID        uuid.UUID `gorm:"type:uuid;primaryKey"`
  RoomID    uuid.UUID `gorm:"type:uuid;index"`
  SenderID  uuid.UUID `gorm:"type:uuid"`
  Content   string    `gorm:"type:text"`
  CreatedAt time.Time `gorm:"index"`
}
```


## **JWT 签发/验证（要点）**

- 登录成功：生成access\_token（短期，如 15m），可选生成refresh\_token（长期）。
- token 内容（claims）：sub(user\_id),exp,iat,room(可选).
- WebSocket 握手时验证 token 并关联user\_id 到连接。
- 在 gRPC 调用中也可传 metadata（authorization: Bearer …），或用 VerifyToken RPC。

## **WebSocket 骨架（Go）**

- hub struct 管理register,unregister,broadcast channels。
- 每个 client runreadPump/writePump goroutine，read 把消息放入 hub inbound channel，write 从 client’s send chan 写向 socket。
- 在 write 时设置 write deadline、使用 mutex 控制并发写入（每个 websocket 只允许一个 writer）。

## **前端 WebSocket（JS 简化）**

```javascript 
const ws = new WebSocket(`wss://your-host/ws?room_id=${roomId}`);
ws.onopen = () => { ws.send(JSON.stringify({ type: 'auth', token: accessToken })); };
ws.onmessage = (ev) => { const msg = JSON.parse(ev.data); /* append UI */ };
function sendMessage(text) {
  ws.send(JSON.stringify({ type: 'msg', content: text }));
}
```


（实际可把 token 放在 subprotocol 或 header，某些浏览器不支持 header — 可通过 query token，但注意安全）

***

# **Protobuf / gRPC 关键接口（建议）**

user.proto（前文示例）

room.proto:

```protocol buffers 
syntax = "proto3";
package room;
service RoomService {
  rpc CreateRoom(CreateRoomRequest) returns (CreateRoomReply);
  rpc ListRooms(ListRoomsRequest) returns (ListRoomsReply);
  rpc GetMessages(GetMessagesRequest) returns (GetMessagesReply);
}
message CreateRoomRequest { string name = 1; string owner_id = 2; }
message CreateRoomReply { string id = 1; }
message ListRoomsRequest {}
message RoomInfo { string id = 1; string name = 2; string owner_id = 3; }
message ListRoomsReply { repeated RoomInfo rooms = 1; }
message GetMessagesRequest { string room_id = 1; int32 limit = 2; int32 offset = 3; }
message MessageInfo { string id = 1; string room_id = 2; string sender_id = 3; string content = 4; string created_at = 5; }
message GetMessagesReply { repeated MessageInfo messages = 1; }
```


***

# **交付 & 评分标准**

- 功能实现：
  - 注册/登录/JWT
  - 房间 CRUD 与权限
  - 实时消息
  - 历史分页
- 代码质量与并发处理：
  - 代码清晰、模块化、并发正确（race-free）
- 测试：单元+集成、压力测试报告
- 文档与演示：README、运行说明、演示视频/现场 demo
- 安全与稳健性：速率限制、JWT 过期处理、错误处理

***
