# OpenClaw WebChat 多客户端用户消息同步设计

> 摘要：当前服务端已按 `[userId, appId]` 保存双方历史，并会向同一用户的所有浏览器广播 agent 消息，但不会广播刚收到的 user 消息。
> 推荐由浏览器在发送前生成 UUID，作为端到端 `messageId`；服务端落库、转发 Plugin、广播浏览器时保持不变。
> 服务端应向包括发送端在内的所有同 `userId` 浏览器广播，前端以 `messageId` upsert，使该广播同时充当确认并消除本地回显重复。
> 浏览器消息事件继续使用 `type: "message"`，以必填 `from: "user" | "agent"` 区分角色；通过协议版本门控保护旧前端。
> Plugin 协议无须改变；建议服务端同时做有限窗口的幂等检查，避免重发导致重复历史和重复触发 Agent。
> 本轮仅给出设计，不修改代码、服务或生产配置。

## 修订（2026-10-21 15:23）：取消兼容层，直接单版本上线

主人明确：**服务端和客户端会一起更新，不需要兼容旧版**。据此本方案原「兼容性与发布顺序」一节整体作废，简化为：

- **删除 `protocolVersion` 协商**：`register` 帧不再声明版本；`registered` 回包不发 capability。
- **删除 v1/v2 分支**：`routeBrowserMessage()` 无条件向同 `userId` 的全部 browser socket 广播 user 帧。
- **前端严格解析**：`type:"message"` 必须带 `from`，取值只能是 `user`/`agent`；缺失或未知则丢弃并 `console.warn`，不做「默认当 agent」的猜测。
- **服务端仍宽松接收入站**：浏览器发来的 `messageId` 缺失时仍生成后备 ID（防止手写脚本/异常客户端打挂），但**不再为旧前端保留任何出站兼容分支**。
- **`clientId` 降级为纯诊断字段**：前端仍生成并随消息上行，服务端只透传、不入历史、不参与任何判断。
- **发布方式**：`server.js` 与 `public/app.js` 属于同一批产物，一次同步到线上并重启 `webchat.service` 即可，无需分阶段。

前述「幂等检查」「messageId 端到端贯通」「前端 upsert」「历史保留 ID」全部保留，是本次实现的核心。

## 范围与目标

目标是在多个浏览器以相同 `userId` 使用 WebChat 时，一端成功发送的文字能实时出现在其余端，并同时保证：

- 发送端的乐观回显不会因服务端回广播而重复。
- 重连后的历史与实时消息可按稳定标识合并。
- 同一条重发消息不会重复进入历史或重复派发给 Agent。
- 不引入数据库、消息队列或前端框架；继续使用内存、JSON Lines、原生 WebSocket 和小函数。

本方案不改变会话粒度。当前所有持有相同 `userId` 的浏览器共享该用户在各 `appId` 下的消息；没有 room/session 维度。

## 代码事实核对

题述核心判断正确，按当前源码核对如下：

- `server/server.js` 的 `browsers` 是 `Map<userId, Set<ws>>`。`registerBrowser()` 只按 `userId` 注册，并依次发送 `registered`、`app_list`、`history`；不存在浏览器 room/session。
- `routeBrowserMessage()` 会先确认目标 Plugin 在线，再调用 `appendHistory()` 保存 user 消息，随后只向 Plugin 发送 `incoming`；没有向浏览器集合广播该消息。这正是实时缺失的直接原因。
- `routePluginOutgoing()` 保存 agent 消息后，用 `browsers.get(userId)` 广播 `type: "message", from: "agent"`。`handlePluginTyping()` 也按同一集合广播。
- 历史键确为 `JSON.stringify([userId, appId])`，每个键在内存中最多保留 `HISTORY_LIMIT = 100` 条；持久化文件是 `server/messages.ndjson`，格式为 JSON Lines。
- `getHistoryForUser(userId)` 返回 `{ [appId]: messages[] }`；`broadcastAppList()` 只处理 App 列表。
- `server/public/app.js` 当前把实时 `type: "message"` 一律作为 agent 渲染；历史会把 `from === "user"` 映射为 user，但丢弃 `messageId`。`addMessage()` 只保存 `{ role, content, timestamp }`，没有去重。
- `sendMessage()` 先添加本地 user 消息，再发 `{ type: "message", appId, content }`，没有携带 `messageId`。因此若直接增加服务端回广播，发送端会出现两个气泡。
- Plugin 接收 `incoming` 后派发给 OpenClaw，回复再以 `outgoing` 返回；本需求不需要修改 Plugin。

### 与题述相比的补充或差异

1. 当前 `server/server.js::routeBrowserMessage()` **已经支持浏览器提供可选的 `messageId`**：有值就沿用，无值才生成 `user_...`；同一个 ID 也已传给 `appendHistory()` 和 Plugin 的 `incoming`。缺的是前端生成/发送、浏览器广播和去重，而不是服务端完全没有贯通能力。
2. 内存中的单条历史对象是 `{ from, appId, content, timestamp, messageId }`；`userId` 不在该对象内，而在 Map key 中。写入 JSON Lines 的对象包含 `userId`。
3. 服务端只有在目标 Plugin 在线时才保存 user 消息。Plugin 不可用时会直接向发送端返回 `error`，该消息不会进入历史。因此“刷新后全量可见”只适用于服务端已接受并保存的消息。
4. `loadMessageHistory()` 只加载包含字符串 `messageId` 的合法日志行；更老、缺少该字段的日志行会被跳过。
5. 当前浏览器端所谓“登录”实际只是提交客户端自声明的 `userId/userName`，未看到用户鉴权。本文中的“同一个用户”严格指相同 `userId`。
6. `server/test-ws.mjs` 仍使用旧的 `agent_list/agentId` 协议，不能直接作为当前协议的可靠回归测试。
7. 题述行号是近似值；当前源码中 `registerBrowser`、`routeBrowserMessage`、`handlePluginTyping`、`routePluginOutgoing` 分别从约 284、374、422、451 行开始，不影响事实判断。

## 协议设计

### 选择：复用 `type: "message"`，以 `from` 区分角色

浏览器入站消息事件统一为：

```json
{
  "type": "message",
  "from": "user",
  "appId": "wch_abc123",
  "content": "你好",
  "messageId": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": 1792576800123,
  "clientId": "page_9d1f..."
}
```

agent 消息保持同一事件类型，只将 `from` 设为 `agent`。字段约束：

| 字段 | 方向 | 约束与作用 |
|---|---|---|
| `type` | Server → Browser | 固定为 `message` |
| `from` | Server → Browser | 必须为 `user` 或 `agent`，前端不得再默认全部为 agent |
| `appId` | 双向 | 消息所属 App |
| `content` | 双向 | 文本内容 |
| `messageId` | 双向 | 全局高概率唯一 UUID；去重、确认与历史合并的主键 |
| `timestamp` | Server → Browser | 服务端首次接受消息的时间；确认时替换本地临时时间 |
| `clientId` | Browser → Server → Browser | 可选的页面实例标识，仅用于诊断/来源提示，不参与正确性判断，也不写历史 |

`type` 表示事件类别，`from` 表示消息作者，这比维护两套几乎相同的 `message`/`user_message` 处理逻辑更简单，也与现有 agent 帧已经携带 `from: "agent"` 的方向一致。

但**不能只在服务端直接复用现有 `type: "message"` 后就上线**。当前旧前端会将所有这类帧硬编码为 agent，导致其他客户端发来的 user 消息显示在左侧、错误地移除 agent typing，而且发送端还会重复显示。必须先让新前端按 `from` 解析并支持去重，或使用下述版本门控原子上线。

若无法做版本门控，备选方案是把 user 广播命名为 `type: "user_message"`：旧前端会忽略未知事件，不会错画角色。但它会形成两套消息分支，未来仍需统一历史/实时合并逻辑，因此不是首选。

### 浏览器注册与来源标识（已按上节修订：无版本门控）

> 以下 `protocolVersion` 部分**已作废**，保留 `clientId` 仅作诊断。注册帧实际只需 `userId` / `userName` / 可选 `clientId`。

新前端注册时增加：

```json
{
  "type": "register",
  "userId": "u_123",
  "userName": "Alice",
  "protocolVersion": 2,
  "clientId": "page_9d1f..."
}
```

服务端把 `protocolVersion` 和 `clientId` 放进 `browserMeta`；缺失版本视为 v1。

- agent 消息继续发给 v1/v2，确保旧行为不退化。
- user 实时广播只发给 v2 浏览器。v1 仍可发送消息，也能在刷新后的历史里看到 user 消息，但不会实时同步，且不会误画为 agent。
- `registered` 可回传 `protocolVersion: 2` 和 `capabilities: ["user-message-sync"]`，便于前端确认服务端能力。新前端连接旧服务端时没有该能力，就保留现有本地回显逻辑；不能假定会收到 user 回广播。

静态页面和服务端通常同时部署，但浏览器缓存可能留下旧 `app.js`，所以仍建议做该门控。待所有客户端自然升级后，也可长期保留 v1 分支，成本很低。

## 发送者去重与幂等

### 推荐：客户端生成 canonical `messageId`，广播给所有端，前端 upsert

不推荐排除发送 socket。排除虽然能避免眼前的双气泡，却失去了服务端接受确认；连接在发送后立即中断时，发送端无法判断是否落历史，重连后的历史也缺少稳定键来对齐本地行。它还会让“发送 socket”和同一页面重连后的新 socket难以对应。

也不推荐先使用一个完全不同的临时 ID、再由服务端生成真实 ID。该方案需要额外的 `clientMessageId → messageId` 映射和确认协议。这里浏览器可以直接生成 UUID，把它作为 canonical ID，临时 ID 与真实 ID 相同，逻辑更小。

推荐流程：

1. 浏览器生成 `messageId = crypto.randomUUID()`，以 `{ messageId, role: "user", content, timestamp: localNow, status: "pending" }` 乐观插入。
2. 浏览器发送同一个 `messageId`。服务端验证并沿用它。
3. 服务端首次接受后保存消息，将同一对象广播给该 `userId` 的全部 v2 socket，**包括发送 socket**，再转发 Plugin（或保持当前保存后转发的顺序）。
4. 每个前端按 `(appId, messageId)` 查找：不存在则插入；存在则更新 `timestamp`、`status: "sent"` 和规范字段，不再 push。
5. 服务端错误帧应带回 `appId`、`messageId` 和错误码；发送端把对应行标为 `failed`。用户重试时复用原 ID，而不是生成新 ID。

服务端还应做幂等检查，否则 UI 虽然不重复，重发仍可能重复写历史并让 Agent 处理两次。无需数据库：在 `[userId, appId]` 的内存历史中按 `messageId` 检查即可。

- ID 未出现：正常保存、广播、派发 Plugin。
- ID 已出现且 `from/content` 相同：不再保存、不再派发 Plugin；重新广播已保存对象，作为幂等确认。
- ID 已出现但内容或角色不同：返回 `duplicate_message_id_conflict`，不得覆盖旧消息。

该去重窗口与当前内存历史上限一致，为最近 100 条/用户/App。若要求在消息被截断后仍可无限期重放去重，就需要独立持久幂等索引，超出本次“不引入数据库或重型依赖”的范围。UUID 冲突本身可忽略，主要场景是网络重试。

### 断线、重连和离线补齐

- 页面仍存活的短暂断线：保留 `pending/failed` 行。重连收到 `history` 后按 ID 合并；历史中存在则标为 `sent`，不存在则保留为 `failed` 或让用户显式重试同一 ID。
- 发送后连接断开、结果不确定：不要立即换 ID重发；先等重连历史。若历史没有该 ID，再复用原 ID重试。
- 页面刷新：当前没有前端持久发送队列，未被服务端接受的 pending 消息会丢失。已接受的消息会由历史恢复。这是可接受的最小范围；若要刷新后自动重发，需另行设计 `localStorage/IndexedDB` outbox。
- 其他主机离线：上线注册时由现有 `history` 补齐最近 100 条，无需额外离线推送机制。

## `messageId` 完整字段流转

```text
A 浏览器
  crypto.randomUUID() = M1
  本地 upsert(M1, user, pending)
  └─ {type:message, appId, content, messageId:M1, clientId:A}
                         │
                         ▼
Chat Server / routeBrowserMessage
  校验 M1 → 在 key=[userId,appId] 内做幂等检查
  appendHistory(..., messageId:M1) → {from:user,...,messageId:M1,timestamp:T1}
  persistMessage(...M1,T1)
  ├─ broadcast browsers[userId]
  │    {type:message, from:user, appId, content, messageId:M1,
  │     timestamp:T1, clientId:A}
  │      ├─ A：命中 M1，pending → sent，不新增
  │      └─ B：未命中 M1，新增 user 消息
  └─ Plugin {type:incoming,...,messageId:M1}
                 │
                 ▼
OpenClaw：现有 Plugin 将 M1 用作 MessageSid

重连/刷新
  Server {type:history, messages:{appId:[...{from,user,...,messageId:M1,timestamp:T1}]}}
  Browser 按 (appId,M1) 合并，而非按 content/timestamp 猜测
```

`appendHistory()` 最好返回最终保存的规范消息对象（包括服务端 `timestamp`）；落盘、广播和历史内存都使用该对象，避免各处各自生成时间或 ID。agent 路径也应沿用这一返回对象，使实时帧与刷新后的历史字段一致。

## 前端数据与渲染改造

推荐把前端消息结构扩为：

```js
{
  messageId,
  role: 'user' | 'agent',
  content,
  timestamp,
  status: 'pending' | 'sent' | 'failed',
  clientId // 可选，仅实时诊断用途
}
```

`addMessage()` 可改为 `upsertMessage(appId, message, options = {})`，或者保持名称但将签名改为 `addMessage(appId, { messageId, role, content, timestamp, status, clientId }, options)`。其行为应是：

1. 校验 `appId/content/role`。
2. 有 `messageId` 时查找同 App 已有行；命中则原位合并并返回 `{ inserted: false }`。
3. 未命中才 push，并返回 `{ inserted: true }`。
4. 只有 `inserted === true` 且当前不在该 App 聊天页时才增加 unread；确认/重复帧不能增加 unread。
5. 只有 agent 消息到达时才移除 agent typing。user 同步帧不应清除 Agent 的输入状态。

`handleServerMessage()` 对实时消息必须执行：

```js
const role = data.from === 'user' ? 'user' : data.from === 'agent' ? 'agent' : null;
if (!role) return; // v2 严格解析，未知来源不猜
upsertMessage(appId, {
  messageId: data.messageId,
  role,
  content: data.content,
  timestamp: data.timestamp,
  status: 'sent',
  clientId: data.clientId,
});
```

历史处理也必须保留 `messageId`，并与内存中的 pending 行合并，而不是盲目覆盖。兼容旧历史时，可以为缺 ID 的记录保留 `messageId: null` 并直接显示，但不能用内容作为永久去重键；服务端当前加载规则意味着重启后一般不会下发这种旧记录。

`sendMessage()` 应先确认 WebSocket 为 OPEN，再生成 ID并乐观插入；或在断线时明确插入 `failed`。当前逻辑先插入、后检查连接，会产生一个既未发送又没有失败标记的气泡，应一并修正。

## 历史一致性

对于已经由服务端接受的消息，当前历史模型已经满足跨端刷新一致性：B 发出的 user 消息以 `[同一 userId, appId]` 为 key 写入；A 刷新或重连后，`registerBrowser()` 调用 `getHistoryForUser(userId)`，会取到该 App 下包含 B 消息的数组。A/B 不按主机分区，因此双方看到同一份最近 100 条历史。

实时缺陷不在历史 key，而在 `routeBrowserMessage()` 保存后没有浏览器广播。仍需注意三个边界：

- 目标 Plugin 不在线时，当前服务端在保存之前报错，所以该 user 消息不会出现在后续历史。
- 内存和重启加载都只保留每个 `[userId, appId]` 最近 100 条，这里的“全量”是该限制内的全量。
- 多条消息的最终顺序以单个 Chat Server 接受并赋予的 `timestamp`/数组顺序为准；不同主机的本地时钟不应作为权威排序依据。

## 多端 typing 与未读

现有 `typing_start/typing_error` 是 **Agent 回复状态**，从 Plugin 发出并广播给同用户所有浏览器。它不表示另一浏览器上的用户正在输入，本需求也不需要增加用户输入状态。

不要把 user 消息同步或未来的用户输入提示复用为现有 `typing_start`，否则会与“Agent 正在回复”的 UI 混淆。如果以后确有需求，可单独设计 `user_typing`，携带 `appId/clientId`，服务端排除来源 `clientId`，并增加节流和超时；这应作为独立功能。

未读规则建议保持“按当前页面是否正在查看该 App”判断，而不是按角色判断：

- B 收到 A 发来的新 user 消息时，如果 B 正在该 App 的聊天页，不增加未读；否则 `state.unread[appId] += 1`。
- A 收到自己的服务端确认时，因 M1 已存在，`inserted === false`，不得增加未读。
- `history` 是同步快照，不应给每条历史累计未读；当前代码也没有这样做。
- 同一逻辑继续适用于 agent 消息。切换到 App 时沿用现有行为清零。

## ~~兼容性与发布顺序~~（已作废，见开头修订节）

> 主人决定不做兼容层，以下内容仅作为历史记录保留，实现时一律跳过。

推荐兼容策略是宽松接收、严格发送、版本门控：

1. 服务端继续接受没有 `messageId` 的 v1 浏览器消息并生成后备 ID，保证旧前端能发送。
2. 新前端解析 `message` 时要求明确 `from`；对历史兼容现有 `from`，可继续兼容旧的 `role` 字段。
3. 服务端只向声明 `protocolVersion >= 2` 的浏览器发送 user 实时帧；向所有版本发送 agent 帧。
4. 新前端若注册确认没有 `user-message-sync` capability，按旧服务端模式运行：只做本地回显，不等待 user 回广播来完成基本显示。
5. 发布时先部署能识别 v2 但尚未向 v1 推 user 帧的服务端，再部署新前端；最后启用 v2 user 广播。若服务端和静态资源只能一起发布，版本门控仍能处理旧缓存页面。

Plugin 只看 `incoming/outgoing`，现有 `incoming.messageId` 已贯通为 OpenClaw 的 `MessageSid`，因此不需要感知浏览器间广播、`protocolVersion` 或 `clientId`，也不需要修改 `plugin/src/ws-client.js`、`plugin/src/channel.js`。

## 实施清单

| 风险 | 文件与函数 | 改动 |
|---|---|---|
| 低 | `server/server.js::registerBrowser` | 仅保存可选 `clientId`；不做版本协商、不回 capability（按修订节） |
| 中 | `server/server.js::routeBrowserMessage` | 提前规范化 `messageId`；幂等检查；让 `appendHistory` 返回规范消息；向同用户所有 v2 browser 广播 `from:user` 帧；错误带回 ID；首次消息才派发 Plugin |
| 中 | `server/server.js::appendHistory` | 检测同 key 下重复/冲突 ID，返回已有或新建的规范对象；只首次 persist |
| 低 | `server/server.js::routePluginOutgoing` | 使用 `appendHistory` 返回对象广播，并补齐 `timestamp`，使 agent 实时帧与历史同形 |
| 中 | `server/public/app.js::connectWebSocket` | 创建页面级 `clientId`，注册时声明 v2，记录服务端 capability |
| 中 | `server/public/app.js::sendMessage` | 生成 UUID；发送同一 ID；以 pending 状态乐观回显；断线/错误可识别为 failed |
| 高 | `server/public/app.js::addMessage`（建议改为 `upsertMessage`） | 消息对象保留 ID/status/timestamp；按 `(appId,messageId)` 更新或插入；只对真正新插入的非当前 App 消息加未读 |
| 中 | `server/public/app.js::handleServerMessage` | `message` 按 `from` 选择 role；只对 agent 清 typing；实时与 history 均按 ID 合并；错误关联 pending 行 |
| 低 | `server/public/app.js::renderMessages` | 可选展示 pending/failed 状态；若首版不加样式，至少保证状态不影响现有气泡渲染 |
| 中 | `server/test-ws.mjs` 或新增轻量 WS 集成测试 | 修正过期协议；覆盖双 browser、发送端去重、接收端实时、重发幂等和历史补齐 |
| 无 | `plugin/src/ws-client.js`、`plugin/src/channel.js` | 无需修改，只做回归验证 |

风险“高”不是指代码量大，而是 `addMessage` 同时影响实时、历史、未读和渲染，回归面最大。

### 最小可行改动集

若目标只限定为“在线 A/B 实时可见且发送端不重复”，最少需要：

1. `server/public/app.js` 发送前生成并携带 `messageId`，消息状态中保留 ID。
2. `server/server.js::routeBrowserMessage()` 保存后把 `{ type:"message", from:"user", appId, content, messageId, timestamp }` 广播给同 `userId` 的 v2 浏览器，包括发送者。
3. `server/public/app.js::handleServerMessage()` 按 `from` 选 role；`addMessage` 改为按 ID upsert。
4. history 映射保留 ID，避免重连后失去去重依据。

这是功能 MVP。服务端幂等检查、pending/failed UI、能力协商回包和自动化测试属于强烈建议的同批加固项；其中服务端幂等尤其能避免 WebSocket 结果不确定时重发导致 Agent 回答两次。

## 验证方案

以下步骤应在隔离的本地开发实例上执行，不操作系统级 `webchat.service`、Caddy 或生产配置。本轮不实际启动任何服务。

### 手工双标签验证

1. 准备一个已连接的测试 App/Plugin，并打开两个独立浏览器上下文 A、B（两个浏览器或普通窗口 + 隐私窗口更接近两台主机）。
2. A、B 都输入完全相同的 `userId`，进入同一个 `appId`。在 DevTools → Network → WS 中确认两边注册帧含 `protocolVersion: 2`，注册确认含 `user-message-sync`。
3. 清空视觉计数，A 发送唯一文本 `A-sync-<时间戳>`。记录 A 发出的 `messageId = M1`。
4. 预期 A 立即出现一个 user 气泡；收到回广播后仍只有一个，状态从 pending 变 sent。A 的 WS 收帧应为：

   ```text
   message from=user appId=<测试 App> messageId=M1 content=A-sync-...
   ```

5. 预期 B 无刷新实时出现同一文本，位于 user 一侧，收帧的 `messageId` 也是 M1；随后 agent 回复在 A/B 都各出现一次且 `from=agent`。
6. 反向由 B 发送 `B-sync-<时间戳>`，验证 A/B 对称，且每端 user、agent 各只有一份。
7. 在 B 切换到另一个 App/返回 Agent 列表，让 A 向原 App 再发一条。预期 B 对原 App 未读 `+1`；B 打开该 App 后清零。A 的确认不能产生未读。
8. 刷新 A，再重连 B。预期历史中同时存在 A/B 发出的消息，角色正确，`messageId` 与实时帧一致且无重复。
9. 在 DevTools 临时断开 B，A 再发一条后恢复 B。预期 B 通过注册后的 `history` 补齐该消息（受最近 100 条限制）。

### 重发与错误验证

1. 用 DevTools 或轻量 WS 脚本把同一 `{ userId, appId, content, messageId:M2 }` 连续发送两次。
2. 预期服务端第二次识别为幂等重放：历史只有一条 M2，A/B 各只有一个气泡，Plugin 只收到一次 `incoming`，Agent 只回复一次；第二次可重新广播同一规范对象作为确认。
3. 再用 M2 发送不同 content。预期发送端收到 `duplicate_message_id_conflict`，历史和其他客户端均不改变。
4. 发送到离线/不存在的 App。预期错误帧包含原 `messageId`，本地对应气泡变 failed；刷新后历史中没有该消息。

### 预期服务端/Plugin 观察结果

现有服务端日志至少会看到每个连接的：

```text
browser connected
registering browser: <同一 userId>
```

建议实现时增加不含正文的结构化调试日志（避免泄露聊天内容），以便验证：

```text
[browser-message] accepted userId=<id> appId=<id> messageId=M1 browsers=2
[browser-message] duplicate userId=<id> appId=<id> messageId=M2 action=ack-only
[browser-message] conflict userId=<id> appId=<id> messageId=M2
```

Plugin 侧应只看到每个首次 ID 一次派发；现有日志在回复开始时类似：

```text
[webchat] reply started user=<id> agent=<id>
```

验收标准是 WS 帧、DOM 气泡数、历史内容和 Plugin 派发次数四者同时成立，不能只看“B 页面出现了文本”。

## Open questions

1. `messageId` 的服务端幂等保证只覆盖当前每 App 最近 100 条是否可接受？若必须跨截断、跨长期重放保证，需要确认是否允许增加一个轻量持久幂等索引。
2. pending 消息在页面刷新后是否需要自动恢复/重发？若需要，应另行确认是否接受浏览器本地 outbox；本方案默认只处理页面存活期间的重连。
3. 是否需要在 UI 上区分“本机发送”和“同一账号的另一客户端发送”？若需要，可使用 `clientId` 做短期来源提示，但不应改变它们同属 user 角色的事实。
4. 当前 `userId` 是客户端自声明而非鉴权身份。多端同步会让知道某个 `userId` 的连接收到该用户消息；是否要把浏览器身份认证纳入后续独立安全设计？
