# 对话总结：Agent 架构、工具 Router 分层、评测体系与 Prompt 工程

> 本文整理了我们此前围绕 nanobot 项目的连续讨论，合并了重复/语义相近的问题，按“问题 -> 方案 -> 实施建议”方式总结，便于后续落地。

---

## 1. 是否应该搭建 Agent 执行成功率与推理正确率评测体系？

### 结论
应该搭，而且可以基于项目现有能力做“低侵入增强”，不需要推翻主执行链。

### 思路
现有项目已经具备评测基础：

- Agent 主循环中已存在可观测轨迹：模型响应、tool_calls、tool_result、finish_reason、迭代次数等。
- 已有后验 evaluator 机制（执行后再次评估结果价值），说明“judge 模型评估”范式已被项目接受。
- cron 运行历史已记录 status/duration/error，说明项目已具备“运行记录”风格。
- Provider 层返回结构已标准化（content/tool_calls/finish_reason/reasoning 等），便于指标化。

### 推荐评价框架（分层）
- **L0 系统成功率**：请求完成率、错误率、P95、最大迭代命中率。
- **L1 任务成功率**：Success / Partial / Failure / False Success。
- **L2 工具质量**：工具选择是否合理、调用成功率、无效调用率、收敛效率。
- **L3 推理正确率**：最终答案正确性 + 工具结果是否被正确利用。
- **产品层**：用户重复提问率、纠错追问率、通知价值。

### MVP 落地顺序
1. 统一落盘执行轨迹。
2. 新增 task success judge（复用 evaluator 模式）。
3. 建 100~300 条离线 benchmark。
4. 每次改 agent/tool/provider 自动回归对比。

---

## 2. 如果加入 Tool Router，项目架构怎么画？

### 核心结构
建议在现有 `AgentLoop -> ToolRegistry.execute` 之间加入 `Tool Router`：

- Router 负责：工具分类、策略闸门、候选工具筛选。
- Registry 继续负责：参数校验与实际执行。
- 工具层可分为：filesystem、shell、web、mcp、spawn、cron、message。

### 价值
- 缩小单轮可见工具集合，减少模型误调用。
- 将风险控制（权限、确认、窗口）前移到调用前。
- 保持与现有工具注册/执行路径兼容。

---

## 3. 对于三层 MCP 工具体系（巡检/处置 -> 应用系统 -> 具体操作），如何分层调用到目标工具？

### 问题抽象
你希望工具按层级组织并可被稳定命中：

1. **第一层**：巡检类 / 处置类。
2. **第二层**：不同应用系统（k8s、db、mq、业务系统等）。
3. **第三层**：具体动作（重启、查进程、看日志等）。

### 方案
采用“逐层收敛路由”：

1. **L1 路由**：先判定巡检还是处置。
2. **L2 路由**：再判定目标系统域。
3. **L3 路由**：在候选集合内选具体操作。
4. **执行前策略闸门**：高风险动作强制确认（尤其处置类）。

### 关键实践
- 建立 Tool Catalog（l1/l2/l3/risk/requires_confirm/tags）。
- Router 输出“路由键”而非直接工具名，工具名映射由本地控制（更可审计）。
- MCP 侧使用 enabledTools 做预过滤，形成“注册时白名单 + 调用时路由筛选”双保险。

---

## 4. 你选择“第三方案”：Router 直接调用模型决策，但不想一次加载所有工具，怎么实现分层加载降 token？

### 最优实现：两阶段模型调用

#### 阶段 A：Router 模型（轻量）
- 输入：用户请求 + 少量上下文。
- 输出：结构化路由结果（l1/l2/intent/risk）。
- 不加载全量工具，仅提供路由输出 schema。

#### 阶段 B：Executor 模型（执行）
- 根据 Router 结果从 Catalog 动态选取 top-k 候选工具（如 8~15 个）。
- 仅将这批工具定义传给执行模型。
- 保持现有 tool_calls 执行链不变。

### 为什么节省 token
- 避免把几十/上百工具 schema 每轮都放入上下文。
- 路由模型与执行模型职责分离，提示更短、工具集合更小。
- 与现有 per-turn tools 传参机制天然兼容。

### 建议参数
- `MAX_TOOLS_PER_CALL = 12`（可在 8~15 之间调优）。
- 高风险 route 必须二次确认。
- 若命中率下降，按层级渐进扩容候选集而非直接放开全量工具。

---

## 5. 这个项目的 Prompt 工程是怎么设计的？

### 总体风格
不是“一个大 system prompt”，而是**多层提示编排**：

1. 主 Agent 系统提示（身份 + 规范 + runtime + workspace 信息）。
2. bootstrap 文件注入（AGENTS/SOUL/USER/TOOLS）。
3. 记忆上下文注入（MEMORY.md）。
4. 技能体系：always skills 全文 + skills summary 索引。
5. 工具循环提示：assistant/tool 消息持续回灌。
6. 子系统提示：memory consolidator 与 subagent 各有专用 prompt。
7. provider 适配层：消息清洗、tool schema 适配、prompt cache。

### 安全与鲁棒性细节
- 运行时上下文明确标注为 metadata（非指令）。
- 合并 runtime 与 user message，规避部分 provider 对连续同角色消息的兼容问题。
- 对 `<think>` 内容做剥离后再对外呈现。
- memory consolidation 强制（或尽量）走 save_memory tool，失败时有降级存档策略。

---

## 6. 综合落地建议（按优先级）

### P0（先做）
1. 引入 Router 两阶段（route -> execute）。
2. 构建 Tool Catalog（层级标签 + 风险标签）。
3. 执行模型仅加载候选工具子集。
4. 处置类统一确认闸门。

### P1（增强）
1. 引入路由置信度与回退策略（扩容候选集）。
2. 对 Router 输出做结构化审计日志。
3. 推出在线评测（成功率、误调用率、处置回退率）。

### P2（优化）
1. 用真实流量持续更新路由词典与工具映射。
2. 建立版本回归基准（路由准确率 + 任务成功率 + token 成本）。
3. 联动 prompt 缓存与动态工具加载，进一步降低成本。

---

## 7. 一句话总览

从我们整段讨论看，最推荐的路线是：

**在保持现有工具执行主链不变的前提下，增加“分层 Router + 动态工具子集加载 + 风险闸门 + 评测闭环”，同时沿用项目现有的模块化 prompt 体系。**

这条路线工程改造成本可控、可逐步上线、并且能同时提升成功率、可控性与 token 成本效率。
