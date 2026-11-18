# App 前端接口

本文定义前端调用的 App HTTP 接口及 WebSocket 消息。请求和响应使用 JSON，时间为 Unix 毫秒，业务字段使用 camelCase。
接口使用平台身份认证，并检查目标网关及数据的访问权限。网关通信使用 gRPC 双向流。

## 通用结果

响应体使用以下联合结构：

```typescript
type ApiResult<T> =
  | { code: 200; data: T }
  | { code: number; message: string };
```

成功 code 固定为 200；失败 code 及含义见下表。HTTP 状态码表达请求处理结果。

| code | 含义 |
| --- | --- |
| 200 | 查询、草稿保存或发布提交成功 |
| 400 | 字段、参数或配置非法 |
| 401 | 未认证 |
| 403 | 无目标访问权限 |
| 404 | 网关或查询目标不存在 |
| 409 | 当前状态不支持操作 |
| 429 | 处理资源不足 |
| 500 | 平台处理或存储失败 |
| 503 | 目标网关离线或服务暂不可用 |
| 504 | 等待网关结果超时 |

## 配置接口

路径前缀为 `/api/gateway/{gatewayId}/config`。

| 方法与路径 | 请求 | 成功响应 data |
| --- | --- | --- |
| GET `/draft` | 无 | DraftView |
| PUT `/draft` | 完整 GatewayConfiguration | 保存后的 DraftView |
| POST `/publish` | 无；提交当前已保存草稿 | PublishedView |
| GET 空路径 | 无 | PublishedView |

无独立草稿时，GET /draft 返回当前已发布配置。PUT 保存当前网关的完整草稿，devices、communication、properties 数组表示完整集合，不是增量；遗漏的对象从草稿中移除。配置编辑及按钮流程见[前端设计](./frontend.md#配置编辑)。
发布时比较完整草稿与已发布配置；发布成功后保留对应草稿内容，失败时保留草稿供修改。
App 发布时校验通信配置至少包含一个功能；存在空 properties 的通信配置时返回 400，保持已发布版本不变。删除或迁移最后一个功能时须同时移除空通信配置，Gateway 直接应用 App 已校验的配置。
发布结果表示配置已提交，网关离线仍可发布；版本通知及等待由 App 配置发布流程处理。

| 结构 | 字段 |
| --- | --- |
| GatewayConfiguration | gatewayId：字符串；reportInterval：正整数，默认 1000；devices：完整设备配置数组 |
| DraftView | version：当前发布版本；config：GatewayConfiguration |
| PublishedView | version：发布版本；config：GatewayConfiguration |

设备、通信和功能字段遵循设备模型。HTTP 配置包含 remark、unit 等展示字段；生效版本由 App 管理。
草稿中新建的设备或通信配置省略 id 时，由 App 保存时分配并在响应中返回；功能 id 必须由前端生成并随完整配置提交，缺失或同设备内重复时返回 400。App 保存草稿时校验标识及结构，发布时校验完整业务配置。ID 的唯一范围遵循设备模型。
同设备内迁移通信配置时必须携带已有功能 ID。尚未发布配置时 version=0，首次发布为 1。

## 运行状态

`GET /api/gateway/{gatewayId}/status`

| StatusView 字段 | 含义 |
| --- | --- |
| gatewayId | 网关 ID |
| connection | online、offline、unknown；表示当前网关流连接状态 |
| version | 当前发布版本 |
| runtime | 最近网关状态快照，包含 timestamp、version、status；status 与 StatusResp 相同，含设备、连接、功能总数及非正常连接及存储状态；尚未收到时为空 |
| deviceTotal | 当前配置中的设备总数 |

runtime.status.connectionIssues 按当前配置提供 deviceId、communicationId、status、timestamp，包含 UNKNOWN 或异常连接，正常连接省略；runtime.status.storage 始终包含 configDatabase、collectionDatabase，每项提供 status 和 timestamp。字段含义与 StatusResp 一致。connection 为当前流状态，runtime 为最近快照，离线时保留快照及其生成时间。
功能读数由最新记录的 value/code 独立派生，连接异常变化保持功能最新记录。

## 采集历史

`GET /api/gateway/{gatewayId}/history`

| 查询参数 | 规则 |
| --- | --- |
| deviceId | 必填，设备 ID |
| propertyId | 可重复传入；省略时查询该设备全部功能 |
| from、to | 必填，左闭右开的采集时间范围，from < to |
| pageSize | 必填，正整数 |
| cursor | 首次省略，续页原样携带上一页 nextCursor |

成功响应 data 为 HistoryPage：

| 字段 | 含义 |
| --- | --- |
| data | HistoryRecord 数组，每项包含 recordId、sample、storedAt |
| nextCursor | 下一页游标；空字符串表示结束 |

sample 包含 deviceId、propertyId、timestamp、value、transformType 及 code；value 为零时保留零值。
数值存在时输出 value，float32 可省略 transformType，float64 提供 transformType=float64；异常仅输出 code，不输出 value 和 transformType。
历史不存在匹配记录时返回空数组，错误结果按通用状态码返回。续页保持筛选条件一致。
同毫秒的多条历史记录分别返回。查询以 deviceId、propertyId 定位，功能在同设备内迁移通信配置时无需调整查询参数。

## 网关状态变动历史

`GET /api/gateway/{gatewayId}/status-history`

App 校验访问权限后，通过当前网关流代理 StatusHistoryReq，查询 Gateway 本地已入库的状态变动历史。状态内容是查询主体，上报成功或失败仅为附带属性。

| 查询参数 | 规则 |
| --- | --- |
| from、to | 必填，左闭右开的状态历史时间 occurredAt 范围，from < to |
| pageSize | 必填，正整数 |
| cursor | 首次省略，续页原样携带上一页 nextCursor，保持时间范围一致 |

成功响应 data 为 StatusHistoryPage，包含 data：StatusHistoryRecord 数组及 nextCursor：字符串。主体字段为 eventId、gatewayId、occurredAt、version、connectionIssues、storageIssues、storedAt；附带的上报属性为 requestId、completedAt、success、failureReason。字段含义见[持久化结构](./database.md#gateway-状态变动历史)。
occurredAt 为状态快照生成时间，具体状态项保留其状态开始时间。connectionIssues、storageIssues 保存快照的非正常部分，正常快照对应 [] 和 {}；恢复可从后续快照中对应异常消失判断。success 仅表示上报是否确认成功，与状态是否正常分别展示。
记录按 occurredAt、eventId 稳定排序，空 nextCursor 表示结束；待写缓存不进入查询。App 直接转交网关查询结果，网关离线返回 503，超时返回 504，本地查询失败按业务结果映射。
success 始终输出布尔值，失败时明确返回 false。

## 业务事件

`GET /api/gateway/{gatewayId}/events`

| 查询参数 | 规则 |
| --- | --- |
| from、to | 必填，左闭右开的事件发生时间范围，from < to |
| deviceId、propertyId、type | 可选筛选 |
| pageSize | 必填，正整数 |
| cursor | 首次省略，续页原样携带上一页 nextCursor |

成功响应 data 为 EventPage，包含 data：EventRecord 数组及 nextCursor：字符串。
EventRecord 按持久化事件结构输出 camelCase 字段；无关联对象时对应 ID 为 null，details 为结构化业务字段。
事件按 occurredAt、eventId 稳定排序。空 nextCursor 表示结束，续页保持筛选条件一致。

## WebSocket 实时订阅

连接入口为同源 `/api/ws`，HTTPS 页面使用 wss。连接复用平台登录凭证，App 校验连接身份及每次订阅的网关访问权限。WebSocket 提供当前状态、最新读数及配置变化通知；配置保存、发布和历史查询使用上述 HTTP 接口。

### 订阅与取消

客户端连接成功后发送订阅请求，每个订阅对应一个网关，subscriptionId 在当前连接内唯一：

```json
{"type":"subscribe","subscriptionId":"sub_001","gatewayId":"gateway_001"}
```

取消时使用原标识：

```json
{"type":"unsubscribe","subscriptionId":"sub_001","gatewayId":"gateway_001"}
```

App 注册订阅后，先发送该订阅的 snapshot，再发送后续变化。注册、捕获当前视图和登记后续变化需要衔接，初始快照发送期间产生的变化保留到快照之后发送。取消后停止投递，客户端忽略已取消或旧连接的在途消息。

### 服务端消息

所有消息包含 type、subscriptionId、gatewayId 和 data，按同一连接的发送顺序处理。subscriptionId 和 gatewayId 对应原订阅，标识失效时不更新页面。

| type | data | 含义 |
| --- | --- | --- |
| snapshot | status：StatusView；records：LatestRecordView[] | 初次订阅的当前运行状态及全部现有最新记录，无记录的功能不进入 records |
| recordsChanged | records：LatestRecordView[] | 实际更新的功能最新记录，按功能标识更新对应项 |
| runtimeChanged | runtime：StatusView.runtime | 完整替换网关运行快照及连接问题，包括异常恢复后的移除 |
| connectionChanged | connection：online、offline 或 unknown | 网关与 App 的当前流连接状态变化 |
| configChanged | version：当前发布版本 | 已发布配置变化，客户端重新读取已发布配置及运行状态；仅展示字段变化时也通知，version 可以不变 |
| unsubscribed | 空对象 | 对应订阅已取消 |
| error | code：数字；message：字符串 | 订阅或消息处理错误，错误码含义与 HTTP 通用结果一致 |

LatestRecordView 包含 deviceId、propertyId、timestamp、reportedAt，及互斥的 value 或 code。timestamp 为采集时间，reportedAt 为批次上报时间；数值精度使用 transformType，float32 可省略，float64 显式提供，异常不提供数值及精度字段。正常零值照常传递。

snapshot 替换对应网关的运行状态和最新记录集合，recordsChanged 只更新消息中的功能。配置删除后的当前功能集合由已发布配置决定，前端读取配置后清除已删除功能的当前记录；HTTP 返回的草稿不参与运行视图过滤。

```json
{
  "type": "recordsChanged",
  "subscriptionId": "sub_001",
  "gatewayId": "gateway_001",
  "data": {
    "records": [
      {"deviceId":"device_001","propertyId":"property_001","timestamp":1794355200000,"reportedAt":1794355200100,"value":0}
    ]
  }
}
```

### 断线与恢复

WebSocket 连接断开后，该连接上的订阅全部结束。重新连接使用新的 subscriptionId 建立订阅，通过 snapshot 重新获取当前视图；服务端不回放断线期间的历史消息。发送队列有界，按对象合并当前变化，慢连接无法继续消费时关闭连接，由重新订阅恢复。
未认证时拒绝连接或结束已有会话，订阅无权限时返回 403 错误且不建立该订阅；错误消息关联原订阅标识。认证失效后客户端停止自动重连，登录恢复后重新建立连接。
