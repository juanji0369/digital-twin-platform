# 持久化结构

本文定义持久化字段、关联约束和归档结构。字段名使用 snake_case，时间使用 Unix 毫秒。DuckDB 的时间字段使用 UBIGINT，时间值必须大于 0；SQLite 的版本使用 INTEGER 并约束非负。

App 与 Gateway 的配置各自保存在本地 SQLite 数据库，分别与 App 事件库、Gateway 业务库使用独立文件。Gateway 业务库在同一 DuckDB 文件内分表保存采集历史和状态变动历史。配置库启用 WAL，每个连接设置 synchronous=FULL、foreign_keys=ON；每个配置库的写事务由所属存储模块串行执行，读取使用独立连接。

## App SQLite

配置草稿以单个网关的完整 JSON 独立保存；发布时将配置变更与版本更新放在同一事务提交。

| 表 | 字段与类型 | 主键、关联与约束 |
| --- | --- | --- |
| `gateways` | `id TEXT`、`version INTEGER`、`report_interval INTEGER` | `id` 主键；未发布时版本为 0，首次发布为 1，后续单调递增 |
| `gateway_drafts` | `gateway_id TEXT`、`config TEXT` | `gateway_id` 主键并引用网关；保存完整草稿，独立于发布配置 |
| `devices` | `id TEXT`、`gateway_id TEXT`、`version INTEGER`、`remark TEXT` | `id` 主键；`gateway_id` 引用网关；索引 `(gateway_id, version)` |
| `communications` | `id TEXT`、`device_id TEXT`、`protocol TEXT`、`config TEXT` | `id` 主键；`device_id` 引用设备；唯一约束 `(device_id, id)`，用于功能归属校验 |
| `properties` | `device_id TEXT`、`id TEXT`、`communication_id TEXT`、`version INTEGER`、`remark TEXT`、`unit TEXT`、`sampling_interval INTEGER`、`transform_type TEXT`、`store_history INTEGER`、`config TEXT` | 主键 `(device_id, id)`；`(device_id, communication_id)` 引用通信配置的 `(device_id, id)`；索引 `(device_id, version)` 和 `(device_id, communication_id)` |

remark、unit 可空，保存展示信息。version 表示网关发布版本或对象最后一次采集配置变化所属的版本。
轮询功能 sampling_interval 为正数，订阅功能可空。transform_type 默认 float32，约束取值为 float32 或 float64。
通信配置通过复合外键保证功能归属同一设备；各表 config 使用 TEXT 保存合法 JSON，按解析后的字段值比较。主键、外键字段及 config 显式设置 NOT NULL。
properties.store_history 为 NOT NULL DEFAULT 1，使用 CHECK (store_history IN (0, 1))，0 表示不入库，1 表示入库，接口使用布尔值。gateways.report_interval 为正数，默认 1000。
对象 ID 满足当前配置表的主键及关联约束。version 使用 CHECK (version >= 0)，report_interval 使用 CHECK (report_interval > 0)，sampling_interval 设置时必须大于 0。
功能 ID 满足同设备内唯一约束，同设备内迁移通信配置复用 ID。发布事务检查每个通信配置至少包含一个功能，存在空通信配置时整体拒绝。

## Gateway SQLite

完整配置和版本在同一事务提交。

| 表 | 字段与类型 | 主键与用途 |
| --- | --- | --- |
| gateway_config | gateway_id TEXT、version INTEGER、config_json TEXT | gateway_id 主键；完整配置及版本，供启动恢复 |

config_json 保存 reportInterval、完整设备配置及功能参数（含 storeHistory），version 为本地配置版本，使用 CHECK (version >= 0)。进程重启按持久化配置初始化；无本地配置时以版本 0 请求 App。

## Gateway 采集历史

采集历史统一写入 DuckDB 的 history 表，并归档为同结构的 Parquet 文件，默认保留 7 天，按 collected_at 清理。记录结构如下：

| 字段 | 类型 | 内容 |
| --- | --- | --- |
| record_id | VARCHAR | 本地唯一标识，同毫秒多条记录保持独立 |
| device_id、property_id | VARCHAR | 采集时设备与功能标识 |
| collected_at | UBIGINT | 结果到达网关的时间 |
| value_float32 | FLOAT，可空 | float32 正常结果的有限数值 |
| value_float64 | DOUBLE，可空 | float64 正常结果的有限数值 |
| transform_type | VARCHAR，非空 | 采集时的 float32 或 float64 类型，异常记录也保留 |
| code | VARCHAR，可空 | 异常记录的非空统一错误码 |
| stored_at | UBIGINT | 本地写入时间 |

查询按 collected_at、record_id 稳定排序，支持时间范围和续页；配置删除保持已保存历史的原始标识。
正常和异常记录使用相同结构，通过 CHECK 约束只允许以下三种组合，数值 0 为有效值，以是否为 NULL 判断字段是否存在：

| 结果 | transform_type | value_float32 | value_float64 | code |
| --- | --- | --- | --- | --- |
| float32 正常结果 | float32 | 非 NULL | NULL | NULL |
| float64 正常结果 | float64 | NULL | 非 NULL | NULL |
| 异常结果 | float32 或 float64 | NULL | NULL | 非空错误码 |

存储缓存按采集时的精度选择对应数值字段，两个精度的记录在同一批次写入 history。查询直接读取该表及其归档文件，按 transform_type 将数值映射为业务 Sample 的 value 及精度，异常结果只输出 code；接口不暴露两个存储数值字段。Parquet 保留与 history 相同的双数值字段及互斥规则。
旧配置下的在途结果按采集时字段和 storeHistory 入库，不按当前通信归属重新过滤。功能精度变更不迁移历史记录；功能迁移复用 ID，历史查询无需调整。

## Gateway 状态变动历史

业务库的 status_history 表保存网关状态变动历史，主体是状态快照的异常部分，上报结果作为附带属性。状态管理生成快照时固定其状态内容及历史时间，每次 StatusReportReq 尝试结束后附加上报结果，再交给存储缓存入库；状态变化及快照生成时只更新内存。历史记录对应实际尝试上报的快照，包含内容变化节流生成的快照及平台流就绪时的当前完整快照；触发规则见 [Gateway 状态维护与上报](./gateway.md#状态维护与上报)。

| 字段 | 类型 | 内容 |
| --- | --- | --- |
| event_id | VARCHAR | 事件唯一标识 |
| occurred_at | UBIGINT | 状态历史时间，取快照生成时间，作为查询、排序及清理依据 |
| version | UBIGINT | 快照对应的生效配置版本，可为 0 |
| connection_issues | JSON | 快照中的全部 UNKNOWN 或异常连接，保留设备、通信、状态及时间；无问题时为 [] |
| storage_issues | JSON | 快照 storage 中非 OK 的项，键为 configDatabase、collectionDatabase；均正常时为 {} |
| request_id | VARCHAR | 附带属性：对应单次上报请求 |
| completed_at | UBIGINT | 附带属性：上报尝试完成时间 |
| success | BOOLEAN | 附带属性：收到成功 ReportResp 为 true，其余尝试结果为 false |
| failure_reason | VARCHAR | 附带属性：成功时为空，失败时保存离线、发送错误、响应错误码或等待超时等原因 |
| stored_at | UBIGINT | 本地入库时间 |

状态内容使用生成快照时的不可变副本，入库时不重新读取当前状态。各状态项的 timestamp 保留进入该状态的时间；恢复后对应异常不出现在后续快照中，完全正常的快照保留空异常集合。success 与状态健康程度分别表达，失败仅表示未确认上报成功，不证明 App 未收到，失败快照不补发。
记录按 occurred_at、event_id 稳定排序，支持状态历史时间范围及分页，只有已提交记录进入查询；保留时间与容量使用独立运行配置，按 occurred_at 清理。App 通过网关流代理查询本地状态历史，返回结果不写入 App 事件库。敏感字段脱敏。

## App 事件库

事件使用 DuckDB + Parquet，包含网关上报事件，以及 App 本地运行产生的用户操作和报错事件。逻辑记录如下：

| 字段 | 类型 | 内容 |
| --- | --- | --- |
| event_id | VARCHAR | 事件唯一标识 |
| type | VARCHAR | 网关上报事件、App 用户操作事件及运行报错事件 |
| gateway_id、device_id、property_id | VARCHAR，可空 | 相关业务对象 |
| occurred_at | UBIGINT | 业务事实发生时间 |
| received_at | UBIGINT | App 接收或本地生成事件的时间 |
| stored_at | UBIGINT | 事件写入时间 |
| details | JSON | 必要业务字段；网关状态包含 version 和完整 status，配置更改包含更改项 |

网关状态事件关联 gateway_id，occurred_at 为快照生成时间；details.status.connectionIssues 保留 deviceId、communicationId、status 和状态开始时间，details.status.storage 保存两个数据库的 status 和 timestamp。完整快照作为一条事件保存，按 gatewayId、快照 timestamp 和内容识别重复快照，去重查询覆盖近期表及已归档文件。
details 保存结构化业务信息，敏感字段脱敏。

## DuckDB 与 Parquet 存储

Gateway 业务库与 App 事件库各自维护独立存储。近期记录批量提交到持久化 DuckDB 表，按时间段归档为 Parquet。
Gateway 存储缓存为采集历史和状态历史记录分别维护有界 channel，由单个写入协程按 reportInterval 定时读取本轮已排队记录。采集历史统一写入 history，按入队时的采集精度填充对应数值字段，状态历史写入 status_history；每张表的本轮批次分别在一个事务提交，处理结束后释放，失败不阻止其他表处理。SQLite 配置写入使用独立事务入口，不经过周期缓存。
保留期限和归档间隔使用运行配置，归档按时间组织。

### 归档标记与任务

各近期记录表均增加 archive_id：VARCHAR，可空，默认 NULL。NULL 表示尚未归入归档任务，该字段仅供内部存储管理。
归档任务保存在各自 DuckDB 的 archive_jobs 表：

| 字段 | 类型与约束 | 用途 |
| --- | --- | --- |
| archive_id | VARCHAR，主键 | 标识一组确定的归档记录 |
| source_table | VARCHAR | 对应近期记录表，限定为所属数据库中的可归档表 |
| file_path | VARCHAR，唯一 | 对应 Parquet 正式文件路径 |
| state | VARCHAR，pending 或 completed | 归档切换是否已提交 |

每个任务只处理 source_table 指定的一张表，查询和清理按目标表选择对应任务及文件，历史表与状态历史表分别归档。

### 执行顺序

1. 在一个 DuckDB 事务中创建 pending 任务，将选中的近期记录标记为该 archive_id。新写入记录保持 NULL。
2. 按 archive_id 导出记录到临时文件，完整落盘后原子更名为正式 Parquet 文件。临时文件与正式文件位于同一文件系统。
3. 在一个 DuckDB 事务中，删除近期表中对应 archive_id 的记录，并将任务状态设为 completed。事务提交后，该文件纳入查询。

归档记录在最终事务提交前保留，过期清理与归档协调，避免提前删除待归档记录。
查询使用同一 DuckDB 读取快照取得近期记录及 completed 文件清单，按清单读取 Parquet。
归档保持原始标识、时间和数值，同一记录在查询中只返回一次。

### 启动续做

存储模块启动后查询 pending 任务。正式文件尚未生成时，按原 archive_id 重新导出；正式文件已经生成时，继续执行删除记录及更新状态的事务。
completed 任务表示归档切换已经提交。任务状态与近期记录删除在同一事务中修改，用于确定是否还需续做。

过期清理覆盖近期记录和已归档文件，并同步更新对应文件登记信息。
Gateway 采集历史及状态历史记录分别设置容量上限，分别按采集日期、状态历史日期清除最旧一天的全部记录，覆盖相应近期表和 Parquet 文件；清理后仍达到上限时，继续清除下一天，直至低于上限。容量清理与归档由存储缓存模块协调。
查询与清理由同一存储模块协调，文件删除须等待引用该文件的在途查询结束。
