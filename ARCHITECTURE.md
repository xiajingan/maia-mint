# Mint 架构

> Mint 是自动任务业务应用，不是执行引擎。

## 1. 定位与边界

Mint 将联系人广播、群广播和群管理等业务意图解析成已授权、可审计的 Stem Task Command。它负责业务规则、目标解析、去重、频控、影响预览和结果汇总；Task/WorkflowRun/Todo/Execution 全部是 Stem 的规范运行事实，Mint 不直接控制 Celt/Vine，也不直连 Mud 数据库。

## 2. 结构

HTTP/Consumer → Use Cases → 业务策略与 Task Command → Mud Resolvers / Stem Gateway。Resolver 分别解析 Account、Contact/Group、Content、Terminal policy、Plugin/Action/Workflow，并逐个校验 tenant、角色、Creator/Owner/User/有效 Own、关联实体、输入 Schema、发布状态和版本。Mint 只生成目标候选、权限来源与版本引用；Stem 在提交时重验并固化权威任务快照。

Mint 私有存储只保存业务参数预设、策略、提交记录和业务结果投影；预设只能引用已发布 Action/Workflow 版本，不得保存 Step、分支、失败或重试控制。公共资产归 Mud，WorkflowRun/Todo/Execution 归 Stem。跨服务使用版本化 HTTP；私有变更以 Outbox 发布，执行事件以 Inbox + 来源水位消费。

## 3. 核心流程

草稿 → Resolver → 目标去重/频控 → 影响预览 → 确认 → 提交 Stem Task Command → 订阅执行事件 → 按目标聚合。广播/群管理必须引用 Mud 已发布 Workflow 或 Action；流程节点/分支/子流程由 Stem 运行，Action Step/Operation 由 Celt 执行，Mint 不解释执行结构。

每个目标候选分配不可变 `target_snapshot_id`；Task Command、Todo 与执行事件都携带它，拆成多个 Todo 时另带 `target_unit_id`，聚合按目标及 Stem 的替代/重试关系计算。预览与确认绑定目标集、Content/Action/Workflow 版本、权限来源、参数摘要和过期时间；提交前重验。

频控由 Mint 的 FrequencyReservation 聚合唯一裁决：在 `(tenant, account, target, policy, window)` 上事务性占用额度并设唯一约束；Stem 接受 Task 后标记 consumed，提交失败释放，取消是否返还由版本化策略明确且不可事后改写。Mint 先持久化 reservation、submission id/idempotency key，再调用 Stem；超时按 key 对账，不创建第二个 Task。取消/暂停/重试/回放由权限入口调用 Stem，Mint 只更新业务投影。

结果投影保存 Stem event cursor、新鲜度和来源版本；检测断档后从 Stem 快照/重放接口补齐，乱序按版本合并。页面下钻 Stem 事实时由 Stem 重新鉴权，过期投影不得显示为实时状态。

### 3.1 业务模块和领域模型

| 模块 | 聚合/服务 | 规则 |
|---|---|---|
| Campaign Draft | CampaignDraft、AudienceSpec、DeliverySpec | 可编辑意图，不产生执行；保存版本 |
| Resolver | ResolutionSet、RejectedTarget | 固定来源/权限/实体版本，失败可解释 |
| Audience | TargetSnapshot、TargetUnit、DedupPolicy | 联系人/群统一目标协议，目标类型规则分离 |
| Policy | FrequencyPolicy、RiskPolicy、SchedulingPolicy | 策略版本化，不把流程节点藏在策略中 |
| Reservation | FrequencyReservation、Submission | 原子额度和跨服务幂等 |
| Projection | CampaignOutcome、TargetOutcome | 只读 Stem 事件，带 cursor/freshness |

联系人广播、群广播、群管理共享“草稿→目标解析→策略→提交→结果”的主流程，仅目标校验和结果聚合策略不同。新增业务类型实现 CampaignTypePolicy，不复制提交和投影流水线。

### 3.2 对外契约

- Command：create/update/preview/confirm/submit campaign；每次写入带 draft version 和 idempotency key。
- Query：campaign、target rejection、outcome、freshness；大目标集只返回分页摘要/导出引用。
- Stem Task Command 固定 campaign/draft/target snapshot/asset versions/frequency reservation/confirmation。
- 事件：`campaign.submitted/outcome.changed/completed`，只表达 Mint 业务状态，不复制 Todo/Execution payload。

### 3.3 产品化与非功能

- 性能：百万目标采用流式解析和分片 TargetSnapshot，不在 API 内存构造全量列表。
- 高可用：reservation/submission 本地事务；Stem 未知结果对账；投影断档重放。
- 安全：目标、内容、账号、Action、Workflow 分别授权；预览下载短期授权；频率和黑名单服务端强制。
- 可运营：按 rejected/duplicate/rate-limited/skipped/failed 分类，不以总成功率掩盖单目标失败。

路线：M0 Resolver/低风险单目标 → M1 联系人广播 → M2 群广播 → M3 群管理 → M4 模板/计划/运营报表 → M5 大规模分片与开放 CampaignType 扩展。

## 4. 迁移

既有自动任务模板拆为“业务参数预设 + Mud Workflow/Action 版本引用”，步骤/分支迁入唯一 Workflow 模型。高风险预览只请求并引用 Stem Confirmation，Mint 不签发可授权外部副作用的 token。`maia-mint-v0.1.0` 完成旧 Task/Todo/控制/模板字段映射、消费者切流和数量对账；Test 验证创建、取消、部分失败和未知结果对账后，`maia-mint-v0.2.0` 删除旧 Task/Todo 写入、旧控制 API、旧模板流程字段和适配器，不双写。

## 5. 部署与质量

Python 3.12/FastAPI/Pydantic/SQLAlchemy/Alembic，依赖 Seed；独立 OCI/Helm release 和私有 Schema。Test 使用真实 Mud/Stem/Celt 最小闭环，契约替身只用于单元测试。关键指标为解析失败、目标规模、频控拒绝、提交延迟、完成率和人工处置率。
