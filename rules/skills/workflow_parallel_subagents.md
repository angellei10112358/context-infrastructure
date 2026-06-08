# 并行 Subagent 工作流

## 元数据

- **类型**: Workflow
- **适用场景**: 用 `multi_tool_use.parallel` 并行执行多个 `functions.task` subagent
- **创建日期**: 2026-02-20
- **最后更新**: 2026-06-07

---

## 何时使用并行模式

满足以下全部条件时，才值得并行：

1. **任务可拆分**：能分解为至少 2 个相对独立的子任务
2. **子任务有规模**：每个子任务预期需要 ≥5 个 tool call 才能完成
3. **子任务有价值**：并行执行比串行执行能显著节省时间

不满足时，直接串行执行，不要为了并行而并行。

---

## 并行执行流程

### 1. 评估与分割

识别 3-5 个关键维度后，根据任务类型确定 overlap：

| 任务类型 | Overlap 范围 | 原因 |
|---------|-------------|------|
| 调研/创造性任务 | 30% - 50% | 交叉验证、查漏补缺 |
| 代码/执行任务 | 0% - 20% | 效率优先，减少重复 |

### 2. 并行启动

当前 OpenCode 环境的正确并行方式是：在同一条 assistant 消息里，用 `multi_tool_use.parallel` 包裹多个 `functions.task` 调用。单独连续调用多个 `task`，即使文字上说“并行”，实际也是串行。

```json
{
  "tool_uses": [
    {
      "recipient_name": "functions.task",
      "parameters": {
        "description": "官方叙事",
        "subagent_type": "general",
        "prompt": "读取/搜索官方来源，提取 claim、URL、原文摘录，写入 tmp/<session>/tier1_official.md",
        "task_id": "",
        "command": ""
      }
    },
    {
      "recipient_name": "functions.task",
      "parameters": {
        "description": "独立体验",
        "subagent_type": "cheap_glm",
        "prompt": "搜索独立用户体验、社区讨论、GitHub issues，写入 tmp/<session>/tier3_independent.md",
        "task_id": "",
        "command": ""
      }
    }
  ]
}
```

`subagent_type` 是 OpenCode 原生 agent 名，不是模型名，也不是旧 `category`。OpenCode 会执行 `agent.get(subagent_type)`；找不到同名 agent 就报 `Unknown agent type`。

常用 `subagent_type`：

| subagent_type | 适用场景 |
|---|---|
| `general` | 通用任务；未配置模型时继承主会话模型。适合普通并行执行，但不要假设它是便宜模型 |
| `explore` | 代码库内部快速搜索、文件定位、架构理解 |
| `reasoning_gpt` | 高可靠推理、工程判断、方案设计、复杂代码审查 |
| `writer_deepseek` | 中文写作、改稿、风格润色、最终 prose polishing；避免高隐私材料 |
| `cheap_glm` | 低成本初筛、分类、提纲、轻量调研和非关键总结 |
| `private_ds4` | 本地 DS4 路线，适合隐私敏感、本机执行优先、低成本草稿 |
| `ollama_kimi` | Ollama Cloud Kimi K2.6，zero-data-retention，较便宜，适合隐私姿态要求高但无需最强模型的任务 |
| `ollama_deepseek_pro` | Ollama Cloud DeepSeek V4 Pro，zero-data-retention，较贵，适合隐私姿态要求高且需要更强 DeepSeek Pro 的任务 |

每个 subagent 的 prompt 应包含：
- 具体负责的维度/范围
- 预期的 overlap 区域（让 agent 知道其他人也在看这部分）
- 输出格式要求
- 输出落盘路径（例如 `tmp/<session_slug>/tier3_independent.md`）

### 3. 等待与整合

`multi_tool_use.parallel` 会在同一轮 tool response 中返回所有子任务结果；无需也不能调用 `background_output`。每个 `functions.task` 返回的 `task_id` 只用于后续需要恢复同一个 subagent 会话时使用，不是并行等待句柄。

整合步骤：

1. 读取每个 subagent 写入的 artifact 文件。
2. 对重叠区域做交叉验证：多 agent 共同发现 → 可信度高；单一来源 → 标注待验证；矛盾信息 → 标注并分析原因。
3. 把整合结果写入 session 目录，例如 `phase3_synthesis.md`、`fact_check.md`、`brainstorm_synthesis.md`。

## 路由决策

先按数据敏感性分流，再按任务能力分流：

1. 高隐私、不能出本机：优先 `private_ds4`。如果任务超出本地 Flash 能力，暂停并让用户决定是否用 Ollama Cloud zero-data-retention 路线。
2. 需要 zero-data-retention 但可以走云：轻量任务用 `ollama_kimi`，高质量推理或写作用 `ollama_deepseek_pro`。
3. 中文写作质量优先且内容不敏感：用 `writer_deepseek`。
4. 复杂工程判断、计划、架构、代码审查：用 `reasoning_gpt`。
5. 便宜、可粗糙、可重跑的初筛任务：用 `cheap_glm`。
6. 代码库探索、定位文件、回答内部结构问题：用 `explore`。

不要在 prompt 里用旧的 `category="deep"`、`category="writing"`、`librarian`、`ultrabrain`、`glm51` 这类路由名。除非它们已经在当前 `opencode.json` 里被注册成同名 agent，否则 `task` 工具不能调用它们。

不要为了“稳定”默认给 agent 配 `temperature: 0`。很多 provider/model 有自己的采样策略；DS4 还会对协议结构做 deterministic handling，但长文本 payload 强行确定性可能导致重复。默认不设置 temperature，让 provider/server 使用自己的默认值；只有明确知道某个模型需要固定采样时再配置。

---

## 示例

### 调研任务（30-50% overlap）

```
调研「某技术框架的采用情况」
├─ Agent 1（general）：官方叙事 + 产品定义
├─ Agent 2（cheap_glm）：独立体验 + 社区反馈
├─ Agent 3（reasoning_gpt）：失败边界 + 竞品对比
└─ Overlap：社区和企业案例都有覆盖，可交叉验证
```

### 代码任务（0-20% overlap）

```
实现「用户认证系统」
├─ Task 1：认证核心逻辑 + Token 管理
├─ Task 2：数据库模型 + 迁移脚本
├─ Task 3：API 端点 + 测试用例
└─ Overlap：接口定义处有少量重叠，确保对接正确
```

---

## 注意事项

- **不要过度并行**：2-3 个精心设计的 subagent 通常优于 5 个松散的
- **prompt 质量**：subagent 的 prompt 要足够具体，否则结果会很浅
- **成本意识**：并行会消耗更多 token，评估是否值得
- **中间结果**：默认不需要把每个 subagent 的原始输出都落盘；但 research / writing workflow 应把关键中间工件整理到 `tmp/<session_slug>/`
- **旧写法禁用**：不要使用 `mcp_task(run_in_background=True)`、`background_output`、`category` 或 `load_skills`；当前工具 schema 只接受 `description`、`prompt`、`subagent_type`、`task_id`、`command`
