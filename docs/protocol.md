# 通信协议

App 与 Gateway 通过 gRPC 双向流通信，消息使用 Protobuf。完整定义见[Protobuf 数据结构](./protobuf.md)，业务字段含义见[设备模型](./device.md)。

## 接入与双向流

Gateway 使用 TLS，通过认证元数据提交网关 ID 和 Token，双向流绑定认证身份。
每个 Gateway 维护一条长期双向流，承载下表中的全部消息。
双方可通过已有流主动发送消息。
流结束后，该流上尚未完成的交互返回通信失败；后续处理遵循对应业务规则。

服务入口为：

```protobuf
service GatewayService {
  rpc Connect(stream GatewayMessage) returns (stream AppMessage);
}
```

GatewayMessage、AppMessage 以 oneof 区分业务消息。每个请求携带 request_id，对应响应回显该标识。
有响应的交互设置等待超时；长期流由连接生命周期管理。消息串行写入，接收循环持续读取并按类型分发。

| Req | 请求方向 | Resp | 业务含义 |
| --- | --- | --- | --- |
| ConfigReq | Gateway → App | ConfigResp | 获取配置 |
| ReportReq | Gateway → App | — | 上报周期最新记录 |
| StatusReportReq | Gateway → App | ReportResp | 同步配置总数、非正常连接及存储状态 |
| ConfigVersionReq | App → Gateway | ConfigVersionResp | 通知配置版本，响应确认已接收 |
| StatusReq | App → Gateway | StatusResp | 查询运行状态 |
| HistoryReq | App → Gateway | HistoryResp | 查询采集历史 |
| StatusHistoryReq | App → Gateway | StatusHistoryResp | 查询网关本地状态变动历史 |

App 从流的认证身份识别网关，消息中的 gatewayId 必须与绑定身份一致。
连接保活使用 gRPC Keepalive；双向流的消息按各发送方向保持顺序。

### 时间与关联

| 字段 | 含义 |
| --- | --- |
| Gateway 业务消息 timestamp | 本次消息生成时间；StatusReportReq 为快照生成时间 |
| Sample.timestamp | 结果到达网关的采集时间，查询或重发保持原值 |
| App 业务消息 timestamp | 消息生成时间 |

requestId 标识一次请求，用于将同一双向流中并行交互的响应交给对应等待方；响应回显该标识。batchId 标识内容确定的采集上报批次。流发送成功表示传输层已接收消息，有响应的交互由响应确认业务处理结果；ReportReq 为单向发送。

## 配置同步

配置请求、响应及版本通知均使用 Protobuf。配置响应使用单条消息；通用字段使用明确类型，协议专属 config 使用 JSON 字符串字段。
ConfigReq.version 为本地生效版本；响应 baseVersion 回显请求版本，version 为目标版本，reportInterval 为当前上报间隔。
更新对象包含设备模型定义的完整采集字段，通信组携带 ID、协议及连接参数，展示字段保留在平台。

| 请求版本 | 响应 |
| --- | --- |
| 0，App 当前版本也为 0 | 同版本无增量：baseVersion=0、version=0、devices=[]、ignore=[]、reportInterval=1000 |
| 0，App 已有发布配置 | 当前全量配置，忽略列表为空 |
| 非零且等于当前版本 | devices 为空，全部当前设备进入顶层忽略列表，保留当前 reportInterval |
| 非零且小于当前版本 | 更新项及忽略列表 |
| 大于当前版本 | 当前全量配置及版本，替换本地集合 |

Gateway 无本地配置时从版本 0 发起请求，App 尚无发布配置时也使用版本 0，首次发布配置使用版本 1。同版本请求同样返回 ConfigResp；无增量不表示省略响应。

| 层级 | 更新项 | ignore | 删除集合 |
| --- | --- | --- | --- |
| 设备 | devices：版本大于请求版本的设备 | 版本小于或等于请求版本的全部设备 ID | 本地设备减去两者并集 |
| 更新设备内功能 | 通信组中版本大于请求版本的功能 | 该设备内版本小于或等于请求版本的全部功能 ID | 本地功能减去两者并集 |

更新项与忽略列表互斥，并集为当前完整集合。忽略设备保留内部配置，忽略功能保留通信归属。
功能删除判断在整个设备范围执行。更新设备的 communication 与 ignore 均为空时，表示删除该设备全部功能；
顶层 devices 与 ignore 均为空时，表示设备集合为空。ignore 仅用于同步消息。

每个通信组只携带需要更新的功能，无更新项的通信组不发送；已发布完整配置中的每个通信配置至少包含一个功能。App 发布时拒绝空通信配置并保证配置完整性，Gateway 按集合规则合并和应用，无需重复校验业务配置。

以下 JSON 展示解码后的业务字段：

```json
{
  "requestId": "config_req_001",
  "gatewayId": "gateway_001",
  "timestamp": 1794355200000,
  "version": 1
}
```

```json
{
  "requestId": "config_req_001",
  "timestamp": 1794355200000,
  "baseVersion": 1,
  "version": 2,
  "reportInterval": 1000,
  "ignore": ["device_002"],
  "devices": [
    {
      "id": "device_001",
      "gatewayId": "gateway_001",
      "ignore": ["property_001"],
      "communication": [
        {
          "id": "communication_001",
          "protocol": "MODBUS_TCP",
          "config": {"host": "192.168.1.100", "port": 502, "timeout": 2000},
          "properties": [
            {
              "id": "property_002",
              "storeHistory": true,
              "transformType": "float32",
              "samplingInterval": 1000,
              "config": {
                "slaveId": 1,
                "functionCode": 3,
                "address": 3,
                "dataType": "float32",
                "byteOrder": "ABCD"
              }
            }
          ]
        }
      ]
    }
  ]
}
```

示例中 device_002 和 property_001 保持不变，property_002 为更新项；本地 device_001 的其他功能属于删除集合。
ConfigVersionReq 携带 gatewayId、timestamp 及最新发布 version。Gateway 收到后返回 ConfigVersionResp，回显 requestId；响应 body 仅包含 timestamp，处理结果由外层 code、message 表达，成功表示通知已接收。Gateway 按本地版本判断是否需要发起 ConfigReq，由 App 返回 ConfigResp。

## 采集批次

ReportReq 使用 CollectionReq，每个 (deviceId, propertyId) 组合最多一条记录，可包含正常或异常结果，用于更新功能最新记录。
CollectionReq 包含 gatewayId、batchId、timestamp 和 records：Sample 数组。Sample 的结果使用 oneof，在默认 float32 的 value、显式 float64 的 value_float64 和异常 code 中选择一项。
oneof 必须设置一项，显式数值 0 为有效零值。gatewayId、batchId、deviceId、propertyId 非空，timestamp 为正数，records 非空。错误码见[采集记录](./device.md#采集记录)。
ReportReq 为单向消息，App 不发送处理响应。Gateway 写流完成或失败后释放本批，后续周期发送新记录。App 处理失败由 App 记录。
批次生成规则见[Gateway 采集与周期上报](./gateway.md#采集与周期上报)，接收处理规则见[App 周期最新记录](./app.md#周期最新记录)。

## 状态上报

StatusReportReq 携带一个完整状态快照，包含 gatewayId、快照生成时间 timestamp、生效配置 version 和 status；gatewayId 与当前流身份一致。ReportResp 通过外层 requestId 关联，外层 code、message 表达接收结果，响应 timestamp 为 App 处理时间。

StatusReportReq 与 StatusResp 共用 RuntimeStatus 结构，包含 deviceTotal、connectionTotal、propertyTotal、connectionIssues 和 storage。三个总数取当前生效配置，connectionTotal 按通信配置计数。

connectionIssues 仅包含 UNKNOWN 或异常的连接，同一连接只出现一次；正常连接省略。列表为空表示快照对应配置中的连接均已确认正常。App 按快照 version 理解归属，当前视图按现有配置匹配对象。

| ConnectionIssue 字段 | 含义 |
| --- | --- |
| deviceId、communicationId | 设备与通信配置标识，定位一个连接 |
| status | UNKNOWN 或连接异常语义码；TCP 帧超限使用 FRAME_TOO_LARGE |
| timestamp | 进入当前状态的时间，Unix 毫秒；UNKNOWN 取初始化或重建时间 |

storage 始终包含 configDatabase 和 collectionDatabase，每项具有 status、timestamp：

configDatabase 表示 SQLite 配置库；collectionDatabase 表示保存采集历史和状态变动历史的 DuckDB 业务库。

| 字段 | 含义 |
| --- | --- |
| status | UNKNOWN：尚未确认；OK：已确认正常；异常使用语义码，例如 CONFIG_WRITE_FAILED、HISTORY_WRITE_FAILED、EVENT_WRITE_FAILED |
| timestamp | 当前状态首次确认或变化时间，Unix 毫秒；UNKNOWN 取初始化时间 |

完整快照替换上次连接问题及存储状态。快照业务内容变化统一节流上报，平台流首次就绪或重连就绪时补当前完整快照；调度规则见 [Gateway 状态维护与上报](./gateway.md#状态维护与上报)。相同状态重复确认时保持状态项 timestamp 不变。具体诊断信息由本地记录。
每个快照只发送一次，Gateway 根据 ReportResp、通信错误或等待超时为状态历史附加上报结果，失败快照不补发。配置应用引起的上报使用应用完成且连接状态更新后的完整视图；设备连接失败进入连接问题，不阻止配置生效或快照发送。
状态变化只更新内存和待发送快照；上报尝试结束后，Gateway 将快照异常部分连同上报属性保存为状态历史，App 通过 StatusHistoryReq 代理查询。App 按 gatewayId、快照 timestamp 和内容识别重复快照。
状态事件提交后确认，重复快照直接返回成功，写入失败返回 INTERNAL。

## 历史查询

HistoryReq 携带设备 ID、propertyIds、from、to、pageSize 及 cursor，时间范围为左闭右开；propertyIds 为空表示该设备全部功能。
HistoryResp 通过外层 requestId 关联，直接提供 records：HistoryRecord 数组及 nextCursor。HistoryRecord 包含 recordId、sample 和 storedAt。
同毫秒多条记录保持独立，页面之间按稳定记录位置续查；nextCursor 为空表示结束。

## 状态历史查询

StatusHistoryReq 由 App 代理发往目标 Gateway，携带 from、to、pageSize、cursor 及请求生成时间 timestamp。时间范围按状态历史时间 occurredAt 取左闭右开区间，pageSize 为正数；首次 cursor 为空，续页保持筛选条件不变。
StatusHistoryResp 回显本次查询的 requestId，包含响应 timestamp、records：StatusHistoryRecord 数组及 nextCursor。记录以 eventId、gatewayId、occurredAt、version、connectionIssues、storageIssues、storedAt 表达历史状态，附带 requestId、completedAt、success、failureReason 表达对应上报尝试，持久化含义见[网关状态变动历史](./database.md#gateway-状态变动历史)。
响应记录的 gatewayId 从网关运行配置补充，本地 status_history 表仅保存历史记录自身的字段。
occurredAt 取状态快照生成时间，具体状态项保留其状态开始时间。connectionIssues 保存快照中的全部 UNKNOWN 或异常连接；storageIssues 仅保存状态非 OK 的 configDatabase、collectionDatabase 项。恢复通过后续快照中对应异常消失体现，完全正常的快照保留空异常集合。success 与状态是否正常分别表达，failureReason 成功时为空，失败时包含响应错误码、离线、发送错误或未确认成功等原因。
查询只返回已提交记录，按 occurredAt、eventId 稳定排序；nextCursor 为空表示结束。App 将网关结果直接返回前端，网关离线、查询超时或本地读取失败返回对应业务错误。

## 状态消息

StatusReq 对应 StatusResp，通过 requestId 关联。StatusResp 包含 gatewayId、读取时间 timestamp、生效配置 version 及 status：

```json
{
  "gatewayId": "gateway_001",
  "timestamp": 1794355200000,
  "version": 2,
  "status": {
    "deviceTotal": 1,
    "connectionTotal": 2,
    "propertyTotal": 4,
    "storage": {
      "configDatabase": {"status": "OK", "timestamp": 1794355199000},
      "collectionDatabase": {"status": "UNKNOWN", "timestamp": 1794355199000}
    },
    "connectionIssues": [
      {"deviceId": "device_001", "communicationId": "communication_002", "status": "UNKNOWN", "timestamp": 1794355199000}
    ]
  }
}
```

StatusResp 与 StatusReportReq 使用相同 status 结构，状态查询用于显式查询。字段说明如下：

```text
gatewayId             // 网关标识，与认证身份一致。
timestamp             // StatusResp 为读取时间，StatusReportReq 为快照生成时间，Unix 毫秒。
version               // 本次状态对应的生效配置版本。
status                // 完整状态。
  deviceTotal         // 生效配置中的设备总数。
  connectionTotal     // 生效配置中的通信配置总数，不表示已连接数量。
  propertyTotal       // 生效配置中的功能总数。
  connectionIssues[]  // 全部 UNKNOWN 或异常连接，正常连接省略。
    deviceId          // 所属设备标识。
    communicationId   // 通信配置标识。
    status            // UNKNOWN 或连接异常语义码。
    timestamp         // 进入当前状态的时间，Unix 毫秒，始终提供。
  storage             // 当前存储状态，两项均始终提供。
    configDatabase    // 配置数据库当前状态。
      status          // UNKNOWN、OK 或存储异常语义码。
      timestamp       // 当前状态首次确认或变化时间，Unix 毫秒。
    collectionDatabase // 业务数据库当前状态，涵盖采集历史及状态历史记录。
      status          // UNKNOWN、OK 或存储异常语义码。
      timestamp       // 当前状态首次确认或变化时间，Unix 毫秒。
```

## 结果与错误

流建立、认证及传输故障使用 gRPC 状态。单次业务响应由外层 GatewayMessage 或 AppMessage 的 code、message 统一表达，code 使用 gRPC 标准状态码数值；成功时 body 直接提供业务字段。gRPC 原生状态用于整条流调用，流内单次业务错误由统一消息层返回。
有响应的业务交互失败时回显关联标识，同一流继续处理其他交互。ReportReq 处理失败由 App 记录。

| code | 含义 |
| --- | --- |
| OK | 业务处理成功 |
| INVALID_ARGUMENT | 消息字段或参数非法 |
| UNAUTHENTICATED | 身份认证失败 |
| PERMISSION_DENIED | 身份与目标归属不符 |
| NOT_FOUND | 配置或查询目标不存在 |
| FAILED_PRECONDITION | 当前状态不支持操作 |
| RESOURCE_EXHAUSTED | 处理资源不足 |
| DEADLINE_EXCEEDED | 单次请求等待超时 |
| UNAVAILABLE | 网关未连接或通信暂不可用 |
| INTERNAL | 服务处理或本地存储失败 |

参考：[gRPC 双向流](https://grpc.io/docs/what-is-grpc/core-concepts/)、[Keepalive](https://grpc.io/docs/guides/keepalive/)、[状态码](https://grpc.io/docs/guides/status-codes/)。
