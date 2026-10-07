# Agent Lab P1-A：实验规格与证据内核

日期：2026-10-07。状态：首个独立子项目规格；范围从[产品规格](./2026-10-07-agent-lab-codex-plugin-design.md)拆分。

## 1. 交付结果

在没有模型凭据、没有 Codex 插件安装和没有前端的情况下，用户可以初始化一个工作区，校验并展开实验规格，将规格冻结为不可变对象，追加证据事件，导出和验证证据包。退出并重开进程后，同一冻结对象和事件仍可读取；重复保存不会产生重复实验，冲突提交会明确报错。

这验证实验平台的基础正确性，不产生模型效果排名。P1-B 才增加真实请求、策略与调度，P1-C 增加 Codex 接入，P1-D 增加默认看台。A 可独立安装、使用和测试。

## 2. 命令与数据

```text
agentlab --workspace PROJECT init --json
agentlab --workspace PROJECT spec validate experiments/temporal-baseline.json --json
agentlab --workspace PROJECT spec freeze experiments/temporal-baseline.json --key local-001 --json
agentlab --workspace PROJECT spec inspect FROZEN_ID --json
agentlab --workspace PROJECT evidence append FROZEN_ID --kind note --file evidence.json --json
agentlab --workspace PROJECT evidence list FROZEN_ID --after 0 --json
agentlab --workspace PROJECT export FROZEN_ID DESTINATION --json
agentlab bundle verify DESTINATION --json
```

PROJECT 是研究工作区根目录，数据落在 PROJECT/.agentlab。命令名称由本项目定义。追加 note 仅是人工或测试记录，不允许伪装成运行器的 model.response；B 增加带租约的内部事件提交接口。

工作区文件：workspace.json、metadata.sqlite3、objects/。workspace.json 保存格式版本与 workspace_id，不包含密钥。schema 版本为 1。SQLite 中保存 frozen_specs、freeze_keys、events，公共 ID 与哈希之间用显式元数据关联；事件全工作区递增序号。

冻结对象保存规范规格 JSON、hash、workspace_id 和矩阵。事件 payload 为 JSON 产物引用，list 读取时验证产物。JSON 导出包含 frozen.json、events.json 与 objects/<sha>；manifest 列出全部文件的哈希和工作区身份。

## 3. 文件职责

| 文件 | 职责 |
| --- | --- |
| pyproject.toml | 包元数据、命令入口、核心依赖与测试依赖 |
| src/agentlab/errors.py | 稳定错误码、公共异常 |
| src/agentlab/specs.py | 不可变规格模型、规范 JSON、矩阵展开 |
| src/agentlab/workspace.py | 显式工作区初始化与身份读取 |
| src/agentlab/artifacts.py | 内容寻址、原子写入、读回验真 |
| src/agentlab/store.py | SQLite schema、事务、冻结与事件追加 |
| src/agentlab/bundles.py | 创建导出与完整性验证 |
| src/agentlab/cli.py | 参数、输入文件、JSON 响应、错误码；只调用应用接口 |
| experiments/temporal-baseline.json | 两变体、两场景、两重复的离线规格样例 |
| tests/core/ | 规格、持久化、幂等、事件、产物、导出与命令验收 |

Store 是 A 的应用服务，拥有数据库操作；CLI 不执行 SQL。B 在同样的语义之上扩展提交、租约和 HTTP 服务，不将 A 的数据库操作暴露给 worker。

## 4. 不变量

- I1：排序不同但语义相同的 JSON 产生同一 hash；不接受非有限数值或未知字段。
- I2：展开数量先检查，不能先创建超大矩阵；重复 ID 不通过校验。
- I3：同键同内容返回原 FrozenSpec，同键不同内容报 IDEMPOTENCY_CONFLICT，事务内没有孤儿冻结对象。
- I4：冻结对象返回后不可原地修改；读取副本的修改不影响数据库。
- I5：事件关联现有 FrozenSpec，序号稳定；重开进程能完整读取。
- I6：产物 ID 只能是合法 hash，读回哈希不一致报 INTEGRITY_ERROR；同内容写入幂等。
- I7：导出只写一个全新目标目录，验真拒绝路径穿越、符号链接、缺失文件、内容篡改和未声明文件。
- I8：JSON 模式不混入诊断文字，错误有稳定退出码。

## 5. 测试与完成条件

测试覆盖语义而非每个字段的重复快照：哈希与展开、预算与重复 ID、重复和冲突冻结、事务重开、按序分页事件、内容篡改、导出篡改和 CLI 退出码。核心不访问网络，不需要 API key。

完整验收是在新的临时工作区中通过命令初始化、校验、冻结、追加 note、退出重开、导出和验真；再修改导出的 frozen.json，验证命令必须失败。包安装后的入口可运行，而非仅在源目录 import 成功。

设计决策 D1：宿主、运行器和裁判分离；D2：规格与证据不可变，重跑和重判分新增版本；D3：本地工作区和应用服务拥有状态。A 实现遵守这些已有方向。A 不选择外部模型鉴权、跨进程调度或第三方工具恢复语义；这些边界属于 B/C 的规格和实现评审。

详细计划：[P1-A 实施计划](../plans/2026-10-07-agent-lab-core.md)。
