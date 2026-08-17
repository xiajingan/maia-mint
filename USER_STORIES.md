# Mint 用户故事

> 演进顺序：M0 Resolver/单目标 → M1 联系人广播 → M2 群广播 → M3 群管理 → M4 大规模 Campaign。完成必须包含目标快照、频控占用、Stem 对账和逐目标真实结果。

Seed 不独立迭代：Mint Story 的 Seed 子任务只登记在 `maia-seed:CONSUMERS.yaml`，本文件不复制映射。Story 进入 `ready` 前须存在对应单边记录并固定 wheel version + SHA-256；Test 验证的同一 digest 原样提升 release。Campaign/Resolver/频控业务语义不得下沉 Seed，禁止 Git/path 依赖。

| ID | 用户故事 | 验收标准 | 来源 | 状态 |
|---|---|---|---|---|
| MINT-001 | 作为 Mint，我希望通过稳定 Resolver 解析任务输入。 | 各 Resolver 独立；逐对象校验 Own/Use；错误可理解且有契约测试。 | MNT-005 | `draft` |
| MINT-002 | 作为用户，我希望预览并提交联系人广播。 | 目标去重、内容/账号/频率校验；影响可确认；结果逐联系人汇总。 | MNT-001 | `draft` |
| MINT-003 | 作为用户，我希望创建群广播。 | 群状态与权限校验；逐群真实结果；单群失败不覆盖其他结果。 | MNT-002 | `draft` |
| MINT-004 | 作为用户，我希望创建受控群管理任务。 | 仅发布 Action 可选；不可逆影响确认；变更前后证据和审计完整。 | MNT-003/CFG-006 | `draft` |
| MINT-005 | 作为运营人员，我希望查看自动任务与异常。 | 按类型/账号/状态筛选；可下钻 Stem 事实；投影延迟可见。 | MNT-004 | `draft` |
| MINT-006 | 作为系统，我希望可靠跟踪 Stem 结果。 | 事件重复不重复聚合；遗漏可按水位回补；未知状态不判成功。 | FND-005/020 | `draft` |
| MINT-007 | 作为发布负责人，我希望在 Test 跑通一个低风险广播。 | 使用真实 Mud、Stem 和测试 CVD；覆盖权限拒绝、取消、部分失败和证据。 | FND-021 | `draft` |

## 产品化细化故事

| ID | 用户故事 | 验收标准 | 来源 | 状态 |
|---|---|---|---|---|
| MINT-008 | 作为用户，我希望保存可复用 Campaign 草稿。 | 只保存业务参数与已发布资产引用；版本化编辑；不保存 Step/分支；过期引用提示迁移。 | MNT-001~003 | `draft` |
| MINT-009 | 作为用户，我希望了解目标为什么被排除。 | rejected target 按重复/越权/失效/黑名单/频控分类；样本和总量一致；敏感对象脱敏。 | MNT-001/002 | `draft` |
| MINT-010 | 作为平台，我希望并发提交仍遵守频控。 | reservation 唯一约束；并发测试不超额；提交失败补偿；取消返还按策略版本。 | MNT-001~003 | `draft` |
| MINT-011 | 作为用户，我希望大目标集异步解析。 | API 快速返回 operation；分片进度/ETA；取消停止新分片；百万目标不进入单进程内存。 | MNT-001/002 | `draft` |
| MINT-012 | 作为用户，我希望群管理先看到变更差异。 | before/after、不可逆项、权限和 Action 版本明确；确认绑定 diff；执行后证据对账。 | MNT-003 | `draft` |
| MINT-013 | 作为运营人员，我希望结果投影可证明新鲜。 | cursor/freshness/last reconciliation 显示；断档自动补；下钻由 Stem 实时鉴权。 | MNT-004 | `draft` |
| MINT-014 | 作为开发者，我希望扩展新 CampaignType。 | 只实现目标校验/聚合策略；复用草稿、Resolver、reservation、submission；契约测试证明无新流程引擎。 | MNT-005 | `draft` |
