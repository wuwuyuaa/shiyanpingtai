# Agent Lab P1 公共契约

日期：2026-10-07。状态：实施规格；由[产品规格](./2026-10-07-agent-lab-codex-plugin-design.md)引用。

## 1. 规范对象与身份

协议版本为 `1`。内部 ID 使用随机 UUID；实验中的 variant、scenario 别名使用非空稳定字符串。UTC 时间使用含时区的 ISO 8601；统计耗时使用单调时钟。规格哈希使用 UTF-8、排序键、无空白分隔、禁止 NaN/Infinity 的规范 JSON，再取 SHA-256。一个规格版本含研究问题、模型连接版本、策略版本与配置、场景集版本及 ID、重复次数、资源限制和判分版本。

`FrozenSpec` 是不可变设计，`Submission` 是运行提交。相同规格可以有不同 Submission，用来显式独立重跑；同一提交的幂等键重放只返回同一 Submission。A 的 freeze 幂等键只约束保存同一 FrozenSpec；B 的 start 使用独立提交幂等键，两者不混用。

`Run` 由 `(submission_id, variant_id, scenario_id, repeat)` 确定，`Attempt` 表示 Run 的一次执行。服务选定的 Attempt 和 ScoreSet 必须显式保存；不自动选最高分。重新采样创建新 Run/Submission，重试失败执行创建新 Attempt，重新判分创建新 ScoreSet。

## 2. 核心规格示例

以下是结构示例，其数字不是实测结果。

```json
{
  "schema_version": 1,
  "question": "相同模型下哪种策略更少引用过期事实？",
  "dataset_id": "temporal-facts",
  "dataset_version": "1",
  "scenario_ids": ["policy-update", "missing-evidence"],
  "repeats": 2,
  "variants": [
    {"id": "window", "connection_id": "test-local", "connection_version": "1", "model": "deterministic", "parameters": {}, "strategy": "window", "strategy_version": "1", "config": {}},
    {"id": "summary", "connection_id": "test-local", "connection_version": "1", "model": "deterministic", "parameters": {}, "strategy": "summary", "strategy_version": "1", "config": {}}
  ],
  "limits": {"max_runs": 8, "max_model_calls": 32, "max_seconds": 300, "max_concurrency": 1},
  "evaluator_id": "temporal-answer",
  "evaluator_version": "1"
}
```

`config` 为适配器拥有的 JSON 数据，拒绝非 JSON 对象。连接和策略引用版本冻结后不可改变。预校验报告保存每个变体的能力条件；B 的 launch 再检查引用与凭据是否可用。没有执行适配器时，只能报告“规格有效”，不能报告“模型连接可用”。

矩阵按 variant、scenario、repeat 展开；repeat 从 0 开始。输入 ID 不得重复。展开前计算总数，超出 `max_runs` 直接拒绝，不先分配巨大列表再报错。实验调度可随机化顺序，矩阵身份不随顺序变化。

## 3. API、CLI、MCP

应用操作采用 `operation + arguments` 的稳定语义，HTTP 映射 `/api/v1/operations/{operation}`；MCP 使用分别命名的窄工具，调用同一操作。CLI 对参数做本地可读解析，再交给应用服务校验。

成功响应：`{schema_version, workspace_id, result, request_id}`。失败响应：`{schema_version, workspace_id, error: {code, message, details}, request_id}`；不得回显凭据。JSONL 每行一个事件对象。分页按稳定 ID/序号排序，游标包含过滤条件版本，不把浏览器当前页码作为证据 ID。

| 错误码 | 含义 | HTTP / CLI exit |
| --- | --- | --- |
| INVALID_SPEC | 格式、重复 ID 或参数错误 | 422 / 2 |
| BUDGET_EXCEEDED | 已超出可检查的资源限制 | 422 / 2 |
| IDEMPOTENCY_CONFLICT | 同键提交不同规范内容 | 409 / 3 |
| NOT_FOUND | 对象或产物不存在 | 404 / 4 |
| WORKSPACE_MISMATCH | 客户端工作区与服务不同 | 409 / 5 |
| INTEGRITY_ERROR | 产物或导出内容哈希不匹配 | 409 / 6 |
| CAPABILITY_UNSUPPORTED | 组合无法满足所声明能力 | 422 / 7 |
| CONNECTION_UNAVAILABLE | 未配置、鉴权失败或端点不可达 | 503 / 8 |
| STALE_ATTEMPT | 过期执行者提交结果 | 409 / 9 |

CLI `--json` 成功写 stdout，失败 JSON 也写 stdout，运行诊断写 stderr；人类模式失败说明写 stderr。`request_id` 只关联请求，不能代替变更操作的幂等键。

## 4. 单机状态与持久化

服务是状态与 SQLite 写入的所有者。它以事务同时保存冻结对象与幂等记录，或提交对象、展开 Run 与提交幂等记录。worker 只通过 IPC/服务调用返回观察与结果。SQLite 开启外键、WAL 和 busy timeout；单服务实例通过工作区锁排除第二个服务。数据 schema 版本与协议版本分开维护。

Run 状态：queued → running → succeeded/failed/interrupted；queued 可以直接 cancelled；running 在取消请求后完成清理进入 cancelled，清理前保留 running 与取消请求属性。服务不把收到取消请求等同于调用已撤销。实验按所有 Run 汇总，所有完成且有失败是 completed_with_failures；仅一部分终态仍显示运行或取消处理中。

租约以 `(attempt_id, generation)` 标识。接收事件时原子校验该标识，分配工作区递增序号并保存事件与产物引用；旧代际消息进入 late-observation 审计流，不能改变当前 Run 或指标。服务断电后数据库恢复，失效租约标 interrupted；P1 只支持显式从场景起点重跑，不声称在供应商请求未知终态时 exactly-once。

服务取消/暂停并发竞争、完成结果与取消请求的次序、租约过期后提交、幂等重放均有明确事务测试。暂停只阻止领取新任务，在途执行继续。

## 5. 事件与产物

事件字段：`event_id, sequence, workspace_id, frozen_spec_id, submission_id, run_id, attempt_id, kind, observed_at, payload, artifact_ids`；在相应阶段不存在的关联字段为 null。核心 kind：spec.frozen、run.started、memory.write、memory.retrieve、context.built、model.request、model.response、grade.recorded、run.finished、run.interrupted、late.observation。

事件保存供应商公开输入输出；额外摘要/提取调用以 purpose 字段区分。API 密钥、认证请求头、连接 token 不属于产物。对请求 body 的脱敏会影响复核时，应保留脱敏规则与标记；不得宣称被删内容仍然完整可复现。

`Artifact` 用 SHA-256 内容寻址，元数据有 MIME、字节数、来源与可见性。写入先在同一目录创建私有临时文件并原子 rename；读回核对大小和哈希。证据工具只接受 ID，不接受任意本地路径。A 提供对象存储与 journal 原语；B 负责具体记忆、请求与运行关联。

导出目录有 manifest、规格、事件与 artifacts，每项记录相对路径与 SHA-256；manifest 自身不列入其文件哈希清单。导出开始时固定 event_cutoff，持续追加事件不会使导出无限增长。只导出服务可见范围内的内容；禁止路径穿越、重复目标路径、符号链接和未列入 manifest 的文件。验真确认内部完整性；manifest hash 可另行保存为核对依据，但不提供签名来源证明。P1 支持自有目录导出与验真，不提供任意第三方 ZIP 的自动执行/导入功能。

## 6. 模型与策略

模型连接返回规范消息结果、供应商原响应引用、实际模型标识、用量与计时。未知字段用 null 并给出原因。OpenAI 原生使用 Responses，Anthropic 使用 Messages，兼容端点使用其声明的协议；无能力证明时不转译或悄悄丢弃参数。依赖和接口细节在 B 的专门实施计划中按官方文档固定。

策略输入为截至当前观察时间的公开历史与用户提问；判分答案存放独立的 evaluator fixture，worker 的模型请求构造器只能获得公共视图。每个 Attempt 建立独立记忆 namespace；退出时保留可检查快照，清理失败记录为状态限制。

Window 固定最近历史选择规则；Summary 固定摘要触发阈值、摘要模型及提示；Structured Memory 固定提取字段、更新冲突规则、检索排序及预算。所有额外调用计入 Run。上下文限制明确单位及执行级别，不能将不同模型的 token 数量当成同等信息量。

## 7. 指标定义

| 指标 | 分子 / 分母 | 缺失处理 |
| --- | --- | --- |
| planned_success_rate | 成功回答数 / 全部计划 Run 数 | 失败计未成功；未完成标阶段结果 |
| answered_accuracy | 正确回答数 / 有可判分回答的 Run 数 | 同时展示无法判分、执行失败和未完成数 |
| stale_fact_error_rate | 使用过期事实的回答数 / 有此规则且有可判分回答的 Run 数 | 不适用场景单列，不当成正确 |
| source_precision | 有效且支持该回答的来源数 / 声称的来源数 | 无来源时 null；应提供来源却未提供另记规则失败 |
| execution_failure_rate | 执行失败 Run 数 / 全部计划 Run 数 | interrupted/cancelled 分开计数 |
| known_cost_total | 已知价格与用量的费用之和 | 同时展示 unknown_cost_run_count，部分已知不能称总费用 |

配对比较先在相同 scenario 下汇总重复，再计算场景级差值；置信区间按场景 bootstrap，固定分析种子。少于 5 个有效配对场景只显示描述值与样本数；不报告显著性结论。原始 Run 仍全部可下载，不仅保留被选中的成功样本。

## 8. 安装和连接

插件执行 PATH 中安装好的 `agentlab mcp`，通过工作区配置连接服务。插件缓存位置不会成为工作区根目录，MCP 初始化不启动付费实验。工作区由显式参数/专用环境变量优先，随后使用启动 cwd；无法可靠解析时返回诊断，不选择任意最近目录。

服务 token 不出现在 URL、日志、模型工具结果中；浏览器写操作需要本地会话和来源验证。只读看台也包含研究证据，按同一工作区会话保护。宿主审批设置与平台资源限制职责不同，插件不得为了运行方便静默扩大 Codex 权限。

当前 MCP Python SDK 官方文档以 v2 为稳定线，服务类已改为 MCPServer；C 固定一条 SDK 版本线，并验证当前 Codex 客户端协议兼容性。插件包装遵循便携格式，必要兼容清单由同一元数据生成，不手工维护两套独立名称。

来源：[Codex 插件包装](https://developers.openai.com/plugins/build/plugins)、[MCP Python SDK](https://py.sdk.modelcontextprotocol.io/)。以上平台对象和统计定义是本项目设计，不声称来自外部协议。
