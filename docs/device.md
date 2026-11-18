# 设备模型

本文定义设备配置、采集记录、错误码及运行状态。

## 设备配置

层级为设备 → 通信配置 → 功能。一个设备归属一个网关，可以包含多个通信配置，每个通信配置包含多个功能。

| 对象 | 标识 | 必填字段 |
| --- | --- | --- |
| 设备 | `id`，平台内唯一 | `id、gatewayId、communication` |
| 通信配置 | `id`，平台内唯一 | `id、protocol、config、properties` |
| 功能 | `id`，设备内唯一 | `id、config`；轮询功能还需正数 `samplingInterval` |

当前功能由 gatewayId、deviceId、propertyId 的组合区分，ID 在当前配置中满足上表唯一范围。
新增功能 ID 由前端随机生成，App 校验非空及同一设备内唯一；功能在同一设备内迁移通信配置时复用 ID，最新读数及历史查询使用相同标识。
设备和功能可包含展示字段 remark，功能可包含展示字段 unit。

| 功能通用字段 | 规则 |
| --- | --- |
| samplingInterval | uint32 毫秒，轮询采样间隔，正数；订阅功能可省略 |
| transformType | 输出精度，float32 或 float64，默认 float32 |
| storeHistory | 是否保存采集历史，布尔值，默认 true |
| config | 协议专属的读取、解码及质量规则 |

`communication`、`properties` 为数组，发布配置中的每个通信配置必须至少包含一个功能。删除或迁移最后一个功能时，应同时移除对应通信配置；保留空通信配置的发布请求由 App 直接拒绝。前端校验用于提示，App 发布校验保证配置完整性，Gateway 按已发布配置解析和应用。
协议枚举为 `MODBUS_TCP、MQTT、OPC_UA、TCP`，App 按对应协议规则校验连接及功能参数。

```json
{
  "id": "device_001",
  "gatewayId": "gateway_001",
  "communication": [
    {
      "id": "communication_001",
      "protocol": "MODBUS_TCP",
      "config": {"host": "192.168.1.100", "port": 502, "timeout": 2000},
      "properties": [
        {
          "id": "property_001",
          "storeHistory": true,
          "transformType": "float32",
          "samplingInterval": 1000,
          "config": {
            "slaveId": 1,
            "functionCode": 3,
            "address": 1,
            "dataType": "float32",
            "byteOrder": "ABCD",
            "validRange": {"min": -40, "max": 125}
          }
        }
      ]
    }
  ]
}
```

### Modbus TCP

| 字段位置 | 参数 | 规则 |
| --- | --- | --- |
| 通信 `config` | `host、port、timeout` | 均必填；timeout 为正数，连接及响应分别计时 |
| 通信 `config` | `idleTimeout` | 正数毫秒，默认 30000，映射到库的 IdleTimeout，空闲达到该时长时关闭连接 |
| 功能 `config` | `slaveId、functionCode、address、dataType、byteOrder` | 均必填，定义读取与解码 |
| 功能 `config` | `validRange` | 可选，转换后数值的有效范围 |

`address` 为从零开始的协议地址。`dataType` 包括 `int16、uint16、int32、uint32、float32` 等，
App 发布时校验支持的功能码、地址和类型，占用寄存器数按类型推导。同一连接的功能可使用不同 `slaveId`。

16 位字节序为 `AB、BA`；32 位为 `ABCD、BADC、CDAB、DCBA`，字母表示接收到的字节排列。
驱动按寄存器及类型配置解码 Modbus TCP 协议响应，转换为统一采集记录。配置合法性由 App 发布校验负责，Gateway 处理实际连接、读取及解码结果。

### MQTT

网关连接外部 broker，通过功能配置订阅消息。

| 字段位置 | 参数 | 类型与规则 |
| --- | --- | --- |
| 通信 config | broker | 非空字符串，包含协议、主机和端口 |
| 通信 config | clientId | 非空字符串，同一 broker 上使用独立客户端标识 |
| 通信 config | username、password | 可选字符串，按 broker 认证要求填写 |
| 通信 config | timeout | 正数毫秒，连接等待超时 |
| 通信 config | keepaliveTimeMs | 正数毫秒，默认 10000，映射到库的 KeepAlive；要求为整秒 |
| 通信 config | tls | 可选对象，保存 CA、客户端证书及私钥配置 |
| 功能 config | topic、qos | topic 为非空订阅过滤器，qos 为 0、1、2，默认 0 |
| 功能 config | payloadType | json 或 bytes，声明消息载荷格式 |
| 功能 config | valuePath | json 时必填，JSONPath 表达式，必须定位一个数值 |
| 功能 config | byteFrom、byteTo、dataType、byteOrder | bytes 时必填，定义左闭右开字节范围及数值解码规则 |
| 功能 config | validRange | 可选，转换后数值的有效范围 |

MQTT 库提供完整消息，驱动按功能规则提取数值，再执行精度转换和质量检查。json 载荷按 UTF-8 解析；valuePath 无匹配、匹配多项或类型不符合时输出 DECODE_ERROR。bytes 载荷使用与 TCP 相同的字段解码规则。broker 连接状态表示该通信配置的连接情况。

例如功能 config 为：

```json
{"topic": "sensor/temperature", "qos": 1, "payloadType": "json", "valuePath": "$.temperature"}
```

收到 {"temperature": 23.5} 时提取 23.5。

### OPC UA

| 字段位置 | 参数 | 类型与规则 |
| --- | --- | --- |
| 通信 config | endpoint | 非空字符串，服务器地址 |
| 通信 config | auth | 认证对象，选择匿名、用户名密码或证书认证，提供所选方式需要的凭据 |
| 通信 config | securityMode、securityPolicy | 字符串，选择服务器支持的安全模式和策略；签名或加密方式提供客户端证书与私钥 |
| 通信 config | timeout、sessionTimeout | 正数毫秒，分别用于连接及请求超时、请求的会话超时 |
| 功能 config | nodeId | 非空字符串，目标节点标识 |
| 功能 config | mode | poll 或 subscribe，分别为轮询和订阅 |
| 功能 config | publishingInterval、samplingInterval | 订阅方式的正数毫秒发布、采样间隔；轮询周期使用功能通用 samplingInterval |

OPC UA 按 nodeId 获取协议定义的节点值和质量码，驱动转换为统一采集记录。

轮询和订阅均复用通信配置的客户端及会话。订阅间隔及会话超时以服务器协商后的值为准；订阅保活使用协议的发布间隔和保活计数，由驱动与库管理。

### TCP

| 字段位置 | 参数 | 类型与规则 |
| --- | --- | --- |
| 通信 config | listenHost、listenPort | 网关本地监听 IP 和端口，端口为 1–65535 |
| 通信 config | deviceIp | 设备固定 IP，同一监听地址下唯一匹配一个通信配置 |
| 通信 config | keepaliveTimeMs | 正数毫秒，默认 10000，作为 TCP 保活探测空闲时间 |
| 通信 config | framing | 必填对象，声明完整报文的分帧规则 |
| 功能 config | byteFrom、byteTo | 无符号整数，完整报文内左闭右开的字节区间，byteFrom < byteTo |
| 功能 config | dataType、byteOrder | 目标字段的类型和字节序；dataType=text 时按文本数值处理，byteOrder 仅二进制数值必填 |
| 功能 config | encoding | 文本数值的字符编码，当前使用 utf-8；去除首尾空白后解析数值 |

按远端 IP 匹配设备，远端源端口不参与设备标识。功能字段提取针对完整报文执行，字节区间、类型及编码依据设备协议填写。

framing 使用以下两种结构之一：

| mode | 配置字段 | 规则 |
| --- | --- | --- |
| fixedLength | frameLength | uint32，正数字节数；每 frameLength 字节组成一帧 |
| delimiter | delimiter | 非空字符串，按 UTF-8 编码为结束符字节序列；完整匹配结束符后得到一帧，帧内容不包含结束符 |

framing.maxFrameLength 为必填 uint32 正数字节数，由前端配置，网关执行；表示帧内容上限，不包含结束符。固定长度模式要求 frameLength ≤ maxFrameLength。

固定长度以字节计数。分隔符模式要求帧内容中不包含结束符字节序列；功能 byteFrom、byteTo 从去除结束符后的帧起始位置计算。

以下为两种通信 config 中的 framing 示例，换行符使用 JSON 转义表示：

```json
{"mode": "fixedLength", "frameLength": 16, "maxFrameLength": 16}
```

```json
{"mode": "delimiter", "delimiter": "\r\n", "maxFrameLength": 1024}
```

## 采集记录

采集记录字段如下：

| 字段 | 类型与含义 |
| --- | --- |
| deviceId | 非空字符串，设备 ID |
| propertyId | 非空字符串，设备内功能 ID |
| timestamp | uint64，结果到达网关的 Unix 毫秒时间 |
| value | 正常结果的有限数值；默认 float32，零值为有效数值，与 code 互斥 |
| transformType | 有数值时的精度标记；默认 float32 可省略，float64 显式提供 |
| code | 异常结果的非空字符串，表示具体错误；异常不携带 value |

config.dataType 决定原始数据解码方式，transformType 决定输出精度；转换溢出按 CONVERSION_ERROR 处理。
记录通过网关、设备和功能 ID 关联，查询及传输保留采集时的时间、数值和 code。

### 质量

驱动完成解码、数值转换及质量检查，将结果统一映射为 code。布尔、字符串输入必须有明确转换规则。
`validRange.min/max` 为可选有限数值，包含边界，同时设置时要求 min ≤ max；该范围表达数据可信性。

| code | 错误原因 |
| --- | --- |
| CONNECTION_ERROR | 连接失败或中断 |
| TIMEOUT | 连接或响应超时 |
| PROTOCOL_ERROR | 协议错误响应、长度不符或响应不匹配 |
| DECODE_ERROR | 按类型或字节序解码失败 |
| CONVERSION_ERROR | 转换失败，或结果为 NaN、Infinity 等非有限数 |
| OUT_OF_RANGE | 有限数值超出有效范围 |
| PROTOCOL_QUALITY_BAD | 协议质量无效或不确定 |

全部检查通过时携带有限数值及必要精度标记。检查失败时仅携带 code，不携带数值及数值精度标记。协议原生质量为 bad 或 uncertain 时映射为 PROTOCOL_QUALITY_BAD，
能够明确识别为连接、超时或解码等错误时使用相应错误码。原始质量码及具体错误原因用于运行诊断。
异常 code 取表中值；记录变旧或配置变化保持原 code。

## 运行状态

| 对象 | 状态规则 |
| --- | --- |
| 网关与平台连接 | online：当前流可用；offline：流结束或确认不可达；unknown：尚未确定 |
| 功能读数 | unknown：没有采集记录；ok：最新记录为 value；error：最新记录为 code |
| 单个通信配置 | UNKNOWN：尚未确认；OK：已确认正常；异常使用语义码。connectionIssues 仅包含 UNKNOWN 或异常项，均携带进入当前状态的 timestamp |
| 存储 | configDatabase 表示 SQLite 配置库，collectionDatabase 表示包含采集历史及状态历史记录的 DuckDB 业务库；各项始终保存 status、timestamp，status 为 UNKNOWN、OK 或异常语义码，timestamp 为当前状态首次确认或变化时间 |

连接异常按 deviceId、communicationId 定位，设备作为连接的归属和查询入口，各通信配置分别展示。快照对应配置中的连接不在 connectionIssues 中，表示已确认正常。单次采集失败由对应功能的 code 表达，连接层确认故障时维护连接异常。
MQTT 的连接情况表示到外部 broker 的接收通道是否可用，不能据此判断发布设备在线。订阅通道正常但没有新消息不构成异常。
功能读数与连接异常分别展示。连接异常及恢复保持最新采集记录的内容和时间；配置仅保存配置字段，采集历史以互斥的 value/code 表达每次结果。
