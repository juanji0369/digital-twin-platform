# Protobuf 数据结构

本文件定义 App 与 Gateway 的双向流消息。业务示例使用 camelCase；协议专属 config 对象对应 config_json 字符串内容。
请求以 Req 结尾，响应以 Resp 结尾。请求携带 request_id，有响应的交互由响应回显该标识；ReportReq 为单向消息。响应状态由外层 code、message 统一表达，成功响应的 body 直接承载对应业务字段。
状态快照只发送一次，Gateway 保存状态变动历史，上报结果作为附带属性，由 App 代理分页查询。
时间为 Unix 毫秒，间隔和超时为毫秒。PropertyConfig.store_history 未设置时按 true 处理，采样间隔仅轮询功能必填。

```protobuf
syntax = "proto3";
package monitoring;

service GatewayService {
  rpc Connect(stream GatewayMessage) returns (stream AppMessage);
}

message GatewayMessage {
  string request_id = 1;
  uint32 code = 2; // 响应使用 gRPC 标准状态码数值，OK 为 0。
  string message = 3; // 响应说明。
  reserved 13, 19;
  reserved "device_status_req";
  oneof body {
    ConfigReq config_req = 11;
    CollectionReq report_req = 12;
    HistoryResp history_resp = 15;
    StatusResp status_resp = 16;
    ConfigVersionResp config_version_resp = 17;
    StatusReportReq status_report_req = 18;
    StatusHistoryResp status_history_resp = 20;
  }
}

message AppMessage {
  string request_id = 1;
  uint32 code = 2; // 响应使用 gRPC 标准状态码数值，OK 为 0。
  string message = 3; // 响应说明。
  oneof body {
    ConfigResp config_resp = 11;
    ConfigVersionReq config_version_req = 12;
    StatusReq status_req = 13;
    HistoryReq history_req = 14;
    ReportResp report_resp = 15;
    StatusHistoryReq status_history_req = 16;
  }
}

message ConfigReq {
  string gateway_id = 1;
  uint64 timestamp = 2;
  uint64 version = 3;
}

message ConfigResp {
  uint64 timestamp = 1;
  uint64 base_version = 2;
  uint64 version = 3;
  repeated string ignore = 4;
  repeated DeviceConfig devices = 5;
  uint32 report_interval = 6;
}

message DeviceConfig {
  string id = 1;
  string gateway_id = 2;
  repeated CommunicationConfig communication = 3;
  repeated string ignore = 4;
}

message CommunicationConfig {
  string id = 1;
  Protocol protocol = 2;
  string config_json = 3;
  repeated PropertyConfig properties = 4;
}

message PropertyConfig {
  string id = 1;
  optional uint32 sampling_interval = 2;
  TransformType transform_type = 3;
  optional bool store_history = 4;
  string config_json = 5;
}

enum Protocol {
  PROTOCOL_UNSPECIFIED = 0;
  MODBUS_TCP = 1;
  MQTT = 2;
  OPC_UA = 3;
  TCP = 4;
}

enum TransformType {
  FLOAT32 = 0;
  FLOAT64 = 1;
}

message ConfigVersionReq {
  string gateway_id = 1;
  uint64 timestamp = 2;
  uint64 version = 3;
}

message ConfigVersionResp {
  uint64 timestamp = 1;
}

message CollectionReq {
  string gateway_id = 1;
  string batch_id = 2;
  uint64 timestamp = 3;
  repeated Sample records = 4;
}

message StatusReportReq {
  string gateway_id = 1; // 上报网关标识。
  reserved 2, 4;
  reserved "batch_id", "records";
  uint64 timestamp = 3; // 完整快照生成时间。
  uint64 version = 5; // 快照对应的生效配置版本。
  RuntimeStatus status = 6; // 当前配置总数、非正常连接及存储状态。
}

message ConnectionIssue {
  string device_id = 1; // 所属设备标识。
  string communication_id = 2; // 通信配置标识。
  reserved 3, 5;
  reserved "code", "message";
  uint64 timestamp = 4; // 进入当前状态的时间，Unix 毫秒。
  string status = 6; // UNKNOWN 或连接异常语义码；正常连接不进入列表。
}

message StorageState {
  string status = 1; // UNKNOWN、OK 或存储异常语义码。
  uint64 timestamp = 2; // 当前状态首次确认或变化时间，Unix 毫秒。
}

message StorageStatus {
  StorageState config_database = 1; // 配置数据库当前状态，始终提供。
  StorageState collection_database = 2; // 业务数据库当前状态，涵盖采集历史和状态历史记录，始终提供。
}

message Sample {
  string device_id = 1;
  string property_id = 2;
  uint64 timestamp = 3; // 结果到达网关的采集时间。
  oneof result {
    float value = 4; // 默认 float32。
    double value_float64 = 5; // 显式标识 float64。
    string code = 6; // 非空统一错误码，与数值互斥。
  }
}

message ReportResp {
  reserved 1;
  reserved "batch_id";
  uint64 timestamp = 2;
}

message StatusReq {
  uint64 timestamp = 1;
}

message StatusResp {
  string gateway_id = 1; // 被查询网关标识。
  uint64 timestamp = 2; // 本次状态读取时间，Unix 毫秒。
  uint64 version = 3; // 当前生效配置版本。
  RuntimeStatus status = 4; // 当前配置总数、非正常连接及存储状态。
}

message RuntimeStatus {
  reserved 1, 2, 3, 4, 6, 7, 10;
  reserved "devices", "device_communications", "collection", "errors", "connection_errors";
  uint32 device_total = 5; // 当前生效配置中的设备总数。
  uint32 connection_total = 8; // 当前通信配置总数，含尚未连接的配置。
  uint32 property_total = 9; // 当前生效配置中的功能总数。
  repeated ConnectionIssue connection_issues = 11; // 当前全部 UNKNOWN 或异常连接。
  StorageStatus storage = 12; // 配置库和业务数据库当前状态，始终提供。
}

message HistoryReq {
  string device_id = 1;
  repeated string property_ids = 2;
  uint64 from = 3;
  uint64 to = 4;
  uint32 page_size = 5;
  string cursor = 6;
  uint64 timestamp = 7;
}

message HistoryResp {
  uint64 timestamp = 1;
  repeated HistoryRecord records = 2;
  string next_cursor = 3;
}

message HistoryRecord {
  string record_id = 1;
  Sample sample = 2;
  uint64 stored_at = 3;
}

message StatusHistoryReq {
  uint64 from = 1; // 状态历史 occurred_at 范围，左闭右开。
  uint64 to = 2;
  uint32 page_size = 3;
  string cursor = 4;
  uint64 timestamp = 5;
}

message StatusHistoryResp {
  uint64 timestamp = 1;
  repeated StatusHistoryRecord records = 2;
  string next_cursor = 3;
}

message StatusHistoryRecord {
  string event_id = 1;
  string gateway_id = 2; // 查询响应从网关运行配置补充。
  string request_id = 3; // 附带属性：对应单次 StatusReportReq。
  uint64 occurred_at = 4; // 状态快照生成时间，作为历史查询及排序依据。
  uint64 version = 5;
  uint64 completed_at = 6; // 附带属性：上报尝试完成时间。
  bool success = 7; // 附带属性：上报是否确认成功，与状态健康程度独立。
  string failure_reason = 8; // 附带属性：成功时为空，失败时提供原因。
  repeated ConnectionIssue connection_issues = 9;
  map<string, StorageState> storage_issues = 10; // 仅含非 OK 项，键为 configDatabase 或 collectionDatabase。
  uint64 stored_at = 11;
}
```

## 字段约束

- 请求携带非空 request_id 并设置一个 body；响应回显该标识。code、message 仅用于响应。
- 成功响应 code=0，设置对应响应 body；失败响应使用非零 code 和 message，body 可省略。单次业务失败保持双向流可用。
- 字段编号固定；删除字段时保留编号及名称。PROTOCOL_UNSPECIFIED 不能作为生效通信协议。
- 所有业务 ID 非空。Unix 毫秒时间戳及版本使用 uint64；毫秒间隔、计数、分页大小及响应状态码使用 uint32。version、base_version 允许为 0；时间戳、report_interval、轮询 sampling_interval 和 page_size 必须大于 0。
- ConfigVersionResp 确认版本通知已接收；配置获取使用独立的 ConfigReq → ConfigResp 交互。ConfigResp 直接提供 base_version、version、ignore、devices 和 report_interval，默认上报间隔 1000。
- Gateway 无本地配置时使用版本 0，App 尚无发布配置时版本为 0，首次发布配置版本为 1。同版本仍返回 ConfigResp；双方版本均为 0 时 devices、ignore 为空，非零同版本时 devices 为空、ignore 包含全部当前设备。
- 已发布完整配置中每个 CommunicationConfig.properties 非空，App 发布时负责完整性校验；同步增量仅发送含更新功能的通信组。
- StatusReportReq 携带一个完整状态快照：gateway_id、生成时间 timestamp、生效配置 version 和 RuntimeStatus 类型的 status，与 StatusResp 共用运行状态结构。每个快照只尝试上报一次。connection_issues 为空表示快照对应配置中的连接均已确认正常。
- RuntimeStatus.storage 的 config_database、collection_database 均必填，status 为 UNKNOWN、OK 或非空异常语义码；timestamp 始终提供，初始化为 UNKNOWN 时取初始化时间。
- ConnectionIssue 的 device_id、communication_id、status 非空，timestamp 始终提供；status 为 UNKNOWN 或连接异常语义码，同一连接在快照内唯一。UNKNOWN 的 timestamp 为初始化或重建时间，异常的 timestamp 为进入当前异常状态的时间；确认正常后从列表移除。
- 相同状态重复确认时保持状态项 timestamp 不变。状态上报触发与调度见 [Gateway 状态维护与上报](./gateway.md#状态维护与上报)。
- CollectionReq 对应单向 ReportReq，App 不发送处理响应；ReportResp 确认 StatusReportReq。
- CollectionReq.records 非空。Sample.result 必须设置一项：value 表示默认 float32，value_float64 表示 float64，code 表示异常；显式数值 0 为有效零值。
- 业务对象统一输出 value；float32 可省略 transformType，float64 携带 transformType=float64。异常仅输出非空 code，不输出数值及数值类型。
- 接入后由 Gateway 发起 ConfigReq。流身份通过认证元数据绑定，设备与功能集合以 App 当前配置为准；状态查询用于显式查询，不属于接入握手。
- cursor 绑定原查询的设备、功能和时间范围，调用方原样携带；续页保持筛选条件一致。历史记录按采集时间、record_id 稳定排序。
- StatusHistoryReq 的 cursor 绑定 occurred_at 时间范围；from < to，page_size 为正数。结果按 occurred_at、event_id 稳定排序，只包含已提交的状态历史。StatusHistoryRecord 的主体是状态快照异常部分，request_id、completed_at、success、failure_reason 为上报附带属性；storage_issues 只允许非 OK 项，恢复后对应异常从后续记录中消失。
