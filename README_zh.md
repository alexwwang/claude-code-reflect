# claude-code-reflect

> *"认识自己，是一切智慧的开端。" — 亚里士多德*

[English](README.md)

一个 Claude Code 插件，在你纠正模型错误时自动检测、后台执行根因分析，并草拟记忆文档等你审批。

## 它做什么

当你纠正 Claude 的错误时（比如 "不对，这个 API 应该用 callback"），reflect 会：

1. **检测**纠正信号（支持英文和中文）
2. **启动**后台 Claude 会话分析错误发生的原因（WHY）
3. **草拟**记忆文档（反馈记忆、项目笔记）等你审查
4. **等待**你的批准后才写入任何内容

分析使用结构化根因分类：假设错误（Assumption Error）、上下文遗漏（Context Gap）、错误抽象（Wrong Abstraction）、范围错配（Scope Mismatch）、过时知识（Stale Knowledge）、过度工程（Over-Engineering）、规格不足（Under-Specification）。

## 安装

```
/plugin marketplace add https://github.com/alexwwang/claude-code-reflect
/plugin install claude-code-reflect
# 完全重启 Claude Code（不是 /reload-plugins）
```

> **注意：** 仅维护 `standalone` 分支。`main`（依赖 OMC）分支已废弃。
> 本插件与 oh-my-claudecode 完全兼容——各系统管理各自的数据目录（`.reflect/` vs `.omc/`），`~/.claude/projects/*/memory/` 中的反馈记忆使用 Claude Code 原生格式，两个系统都能识别。

## 使用方法

### 触发反思

当你纠正 Claude 后，调用技能：

```
/claude-code-reflect:reflect
```

也可以直接说出纠正内容然后调用。技能会扫描最近的对话轮次寻找纠正信号。

### 审查结果

后台分析完成后，审查草拟文档：

```
/claude-code-reflect:reflect review ref-20260404a
```

你可以全部批准、修改后批准、变更范围、提供更多上下文重新分析、或放弃。

### 检查进度

查看正在运行的后台分析状态：

```
/claude-code-reflect:reflect inspect ref-20260404a
```

显示会话状态、日志，以及如何直接恢复 subagent 会话。

## 快速验证：安装后测试

安装并重启 Claude Code 后，按以下步骤验证插件工作正常：

**测试 1：技能发现**
```
# 在 Claude Code 中输入：
/claude-code-reflect:reflect

# 预期：Claude 会用 AskUserQuestion 询问你要反思什么
# 如果看到 "skill not found"，需要完全重启 Claude Code（不是 /reload-plugins）
```

**测试 2：中文纠正端到端测试**
```
# 在 Claude Code 中进行以下对话：

You: 介绍一下 Go 的 append() 函数。
Claude: [给出一些解释，例如 "append() 总是创建新的 slice"]
You: 不对，append() 在容量足够时会复用底层数组。
You: /claude-code-reflect:reflect

# 预期流程：
# 1. Claude 检测到 "不对" 作为直接否定信号
# 2. Claude 执行原子准备并启动后台会话，告诉你 session UUID
# 3. 主对话正常继续
# 4. 后台完成后你收到通知
# 5. 运行：/claude-code-reflect:reflect review ref-xxxxxxxx
# 6. 你看到 RCA 报告和带范围判断的草拟记忆文档
```

**测试 3：英文纠正端到端测试**
```
You: What does Python's sorted() return?
Claude: [说错了，例如 "sorted() sorts the list in place"]
You: Incorrect, sorted() returns a new list, it doesn't modify the original.
You: /claude-code-reflect:reflect

# 预期：同测试 2 的流程
```

**测试 4：审查流程**
```
# 后台任务完成后：
/claude-code-reflect:reflect review ref-xxxxxxxx

# 预期：Claude 展示：
#   - 根因分析（类别、推理链）
#   - 严重度评级
#   - 草拟的记忆文档，含范围判断（USER-LEVEL 或 PROJECT-LEVEL）、理由和目标路径
#   - 选项：全部批准 / 修改后批准 / 变更范围 / 重新分析 / 放弃
```

**检查输出是否正确：**
- .reflect/reflections/ref-xxxxxxxx/report.md 存在且包含完整 RCA
- .reflect/reflections/ref-xxxxxxxx/state.json 显示 status: "pending_review"，artifacts 数组包含 scope 和 scope_reasoning
- 批准后，反馈记忆文件存在于 ~/.claude/projects/*/memory/

## 你会看到什么

### RCA 报告

运行 /reflect review 后，你会看到类似这样的输出：

```
## 根因分析
- 类别: Wrong Abstraction（错误抽象）
- 根因: 模型将 Go 的 append() 抽象为必然重新分配内存的操作，
  将"容量不足时重新分配"的特殊情况提升为了一般保证。
- 推理链:
  1. 正确理解了 append() 在容量不足时会触发重新分配
  2. 错误地将此提升为旧 slice 隔离的保证
  3. 未考虑容量充足时 append() 写入共享内存的情况

## 防止重犯
当推理条件性行为时，始终识别互补情况并明确陈述两个分支。

## 严重度: Important

## 草拟文档
[artifact-001] go-append-slice-isolation
范围:   用户级（USER-LEVEL）
理由:   Go 语言语义错误，跨项目适用
目标:   ~/.claude/projects/.../memory/feedback_go-append-isolation.md
```

### 反馈记忆（批准后）

批准的文档会写入 Claude Code 反馈记忆。未来的对话会自动加载此记忆：

```markdown
---
name: go-append-slice-isolation
description: Go append() 不保证 slice 隔离
type: feedback
---

Go 中 append() 在容量足够时可能复用现有底层数组。

**Why:** append() 仅在容量不足时分配新的底层数组。
如果现有数组有空间，append 会写入共享内存。

**How to apply:** 当代码依赖旧 slice 在 append 后保持不变时，
使用 copy() 或三索引切片强制隔离。
```

这份记忆会在未来的会话中自动加载到上下文中，防止同样的错误再次发生。

### 使用后的文件结构

```
.reflect/reflections/
  ref-20260404a/
    report.md       <- 完整 RCA 报告 + 草拟文档
    state.json      <- 状态: pending_review -> approved
  ref-20260404b/
    report.md
    state.json

~/.claude/projects/{project}/memory/
  MEMORY.md                        <- 索引（自动更新）
  feedback_go-append-isolation.md  <- 已批准的反馈记忆
```

## 工作原理

```
第 1 轮: 用户纠正 Claude -> /reflect -> 检测 + 原子准备 + 后台启动
第 1 轮: 主对话正常继续
   ...后台 subagent 执行 RCA，仅写入暂存目录 (.reflect/reflections/{id}/)...
第 2+ 轮: 用户运行 /reflect review ref-xxxx -> 查看 RCA + 带范围判断的草稿 -> 批准 -> 写入记忆
```

后台 subagent 以持久 Claude 会话方式运行（`claude --session-id`）。你可以随时打开新终端运行 `claude --resume <session_uuid>` 查看进度。

### 权限与安全模型

本技能使用三阶段权限设计：

| 阶段 | 会话类型 | `bypassPermissions` | 写入位置 |
|------|---------|---------------------|---------|
| 准备 | 主会话（单次原子 Bash） | 是 — 保证原子性 | 仅 `.reflect/reflections/` |
| 后台分析 | 后台 subagent | 否 | 仅 `.reflect/reflections/{id}/` |
| 审查 + 写入 | 交互式（恢复的）会话 | 否 | `.reflect/` + `~/.claude/` |

**准备阶段为何使用 `bypassPermissions`：** 准备阶段（uuidgen、mkdir、写 state.json、写 prompt、启动 subagent）合并为单次 Bash 调用。不使用 `bypassPermissions` 时，此调用在 `ask` 模式下会产生确认提示，制造中断窗口——用户的新消息可能穿插进来，导致 subagent 永远不会启动。由于所有操作都在项目根目录内（mkdir、文件写入），没有实质性的安全风险。

**subagent 为何不使用 `bypassPermissions`：** subagent 仅写入项目根目录——不需要提升权限。这避免了一个已确认的 Claude Code bug：使用 `bypassPermissions` 的后台 subagent 会被静默拒绝访问项目根目录之外的路径。

**范围判断：** 分析阶段，模型会判断每个 artifact 是**用户级**（通用知识错误，跨项目适用）还是**项目级**（代码库特定上下文）。你在审查时会看到判断理由，可以在写入前覆盖。

### 错误处理

如果后台 subagent 失败，技能会分类错误并提供相应的恢复选项：

| 错误类型 | 检测模式 | 恢复方式 |
|---------|---------|---------|
| API 连接问题 | `ECONNRESET`、`timeout` | 使用新会话重试 |
| 速率限制 | `rate limit`、`429` | 等待后重试 |
| 平台问题 | 其他失败 | 内联执行回退或放弃 |

### 三种模式

| 命令 | 何时使用 | 发生什么 |
|------|---------|---------|
| /reflect | 纠正 Claude 之后 | 扫描纠正信号，启动后台 RCA |
| /reflect review ref-xxxx | 后台完成后 | 展示 RCA + 草拟文档，请求批准 |
| /reflect inspect ref-xxxx | 后台运行中 | 展示会话状态、日志、调试方法 |

## 分支状态

> **`standalone` 是唯一维护的分支。** `main`（依赖 OMC）分支已废弃，不再更新。
>
> 本插件与 oh-my-claudecode 无冲突地共存。各系统管理各自的数据目录（`.reflect/` vs `.omc/`），`~/.claude/projects/*/memory/` 中的反馈记忆使用 Claude Code 原生格式，两个系统都能识别。

## 待改善项

以下问题在实际测试中发现，待后续改进。

1. **Subagent 模型配置缺失**

   启动 subagent 的 `claude -p` 命令未指定模型参数，可能使用默认模型而非主 session 的当前模型。

   *改进方向：* SKILL.md 中应指导从主 session 获取当前模型 ID，通过 `--model` 参数传递给 subagent。

2. **重试时 Session ID 冲突**

   subagent 启动失败后重试时，可能复用已注册的 session UUID，导致 `claude --resume` 行为不确定。

   *改进方向：* 每次重试（包括 `--deep` 重分析）必须生成新的 UUID，并在 `state.json` 中更新。

3. **Read 工具渲染与实际文件内容不一致**

   验证含 markdown 语法的文件（尤其是反引号代码块）时，Read 工具可能因嵌套 markdown 解析而渲染异常。磁盘上的实际文件内容是正确的——这是显示问题，不是数据损坏。

   *改进方向：* SKILL.md 中应提醒 subagent，验证含 markdown 语法的文件时使用 `Bash cat` 而非 Read 工具。

4. **错误恢复选项不充分**

   subagent 启动失败后，当前选项为"重试/内联回退/放弃"，未覆盖部分文件已写入的中间状态（如 `report.md` 已存在但 `state.json` 未更新）。

   *改进方向：* 增加"检查部分结果"选项，检测并利用已存在的部分文件继续完成操作。

5. **跨 Compaction 通知可靠性**

   standalone 分支使用文件写入（`.reflect/notifications.md`）。文件通知无法在 context compaction 后自动注入上下文。

   *改进方向：* 考虑在 SKILL.md 中增加 post-compaction 检查机制（如读取 `state.json` 扫描 `pending_review` 状态）。

### 已解决的问题

- ~~**后台 subagent 写入路径 bug**~~ — 通过写入路径重设计（v3）已解决。后台 subagent 现在仅写入项目内暂存目录。所有最终写入在交互式审查会话中执行，`~/.claude/` 访问正常。

- ~~**多步流程缺乏原子性保护**~~ — 通过将所有准备步骤合并为单次原子 Bash 命令已解决。步骤之间不再存在中断窗口。

## 迭代改进

本技能设计为通过实际使用不断迭代改进。推荐的工作流：

1. **在实际对话中使用**技能捕获模型错误
2. **审查反思结果**，识别根因分类中的模式
3. **调整 SKILL.md**，基于未被捕获或被错误分类的真实纠正案例
4. **更新纠正信号分类**，当你发现新模式时（如领域特定的纠正表达）
5. **调整 subagent 提示词**，如果 RCA 质量不满意

### 建议迭代的内容

- **纠正信号分类**（Step 2.1）：遇到新模式时补充
- **根因类别**：7 类分类法可根据你的领域调整
- **Subagent 提示词**（SUBAGENT_PROMPT 内）：调整 RCA 深度、添加领域特定指导
- **草拟文档模板**：自定义记忆文档格式以适配你的工作流
- **严重度阈值**：调整 Critical / Important / Minor 的判定标准

### 测试改动

```bash
# 编辑 SKILL.md 后，重启 Claude Code 重新加载插件
# 然后用已知的纠正模式测试

# 查看测试计划了解已覆盖的场景
cat skills/reflect/tests/TEST_PLAN.md
```

### 贡献改进

如果你改进了纠正检测或 RCA 质量，欢迎提 PR。最有价值的贡献：
- 新的纠正信号模式（特别是非英语的）
- 结合真实案例调整的根因分类
- 能产生更好 RCA 的 subagent 提示词改进

## 开发

```bash
git clone https://github.com/alexwwang/claude-code-reflect.git
cd claude-code-reflect
# standalone 是默认活动分支
```

## 许可证

MIT
