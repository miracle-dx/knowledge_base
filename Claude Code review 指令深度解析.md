## 一、概述

之前那份文档是基于命令使用经验**推测**的，这份是直接读了 Claude Code 插件市场的源码后整理的，准确还原了两个 review 命令的内部工作机制。

源码位置：`~/.claude/plugins/marketplaces/claude-plugins-official/plugins/`

涉及两个相关命令：
- `pr-review-toolkit/commands/review-pr.md` — 多 agent 并行审查
- `code-review/commands/code-review.md` — 严谨的 PR 评审 + 自动评论

---

## 二、两个 review 命令对比总览

| 维度 | `/pr-review-toolkit:review-pr` | `/code-review` |
|------|-------------------------------|---------------|
| **使用场景** | 本地代码 / PR 提交前自检 | 已有 PR，最终评审 |
| **执行方式** | 多 agent 顺序或并行 | 多阶段流水线（8 步） |
| **输出** | 终端报告（critical/important/suggestions） | 直接评论到 GitHub PR |
| **去伪机制** | 无 | 置信度评分系统（>80 才报告） |
| **依赖工具** | git + 可选 gh | 必须 gh，所有交互都通过 gh |
| **agent 模型** | Sonnet（专家级） | Haiku（轻活）+ Sonnet（重活）混合 |
| **复杂度** | 中 | 高 |

---

## 三、`/pr-review-toolkit:review-pr` 详解

### 3.1 命令元信息

```yaml
---
description: "Comprehensive PR review using specialized agents"
argument-hint: "[review-aspects]"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task"]
---
```

`Task` 工具的存在说明它要**调度子 agent**，不是单纯的指令脚本。

### 3.2 工作流程（7 步）

```
1. 确定评审范围
   ├─ git status 看哪些文件变了
   ├─ 解析参数（comments / tests / errors / types / code / simplify / all）
   └─ 默认跑全部

2. 列出可用评审维度（6 种）

3. 识别变更文件
   ├─ git diff --name-only
   ├─ gh pr view（如果 PR 已存在）
   └─ 按文件类型决定哪些 agent 适用

4. 决定启用哪些 agent
   ├─ 始终启用：code-reviewer
   ├─ 如果有测试文件：pr-test-analyzer
   ├─ 如果有注释/文档变化：comment-analyzer
   ├─ 如果有错误处理变化：silent-failure-hunter
   ├─ 如果有类型定义变化：type-design-analyzer
   └─ 通过评审后：code-simplifier（润色）

5. 启动 agent
   ├─ Sequential（默认）：一个一个跑，便于交互
   └─ Parallel（可选）：并行跑，速度快

6. 聚合结果
   ├─ Critical Issues（必须修）
   ├─ Important Issues（建议修）
   ├─ Suggestions（可选）
   └─ Positive Observations（亮点）

7. 给出 Action Plan
```

### 3.3 6 个专门 agent 的职责

| Agent | 干什么 | 触发条件 |
|-------|-------|---------|
| `comment-analyzer` | 验证注释和代码是否一致，找过时注释，查文档完整性 | 改了注释/文档 |
| `pr-test-analyzer` | 评审测试覆盖率，找关键缺失，评估测试质量 | 改了测试文件 |
| `silent-failure-hunter` | 找静默失败、空 catch、缺失日志 | 改了错误处理 |
| `type-design-analyzer` | 分析类型封装、不变量表达、类型设计质量 | 加/改了类型 |
| `code-reviewer` | 检查 CLAUDE.md 合规性、bug、整体质量 | 永远启用 |
| `code-simplifier` | 简化复杂代码，提升可读性 | 通过评审后润色 |

### 3.4 使用方式

```bash
# 全套评审（默认）
/pr-review-toolkit:review-pr

# 只查测试和错误处理
/pr-review-toolkit:review-pr tests errors

# 只查注释
/pr-review-toolkit:review-pr comments

# 只做代码简化
/pr-review-toolkit:review-pr simplify

# 全部并行跑
/pr-review-toolkit:review-pr all parallel
```

### 3.5 集成工作流

**提交前：**
```
写代码 → /pr-review-toolkit:review-pr code errors → 修 critical → 提交
```

**建 PR 前：**
```
暂存改动 → /pr-review-toolkit:review-pr all → 修所有 critical/important → 复查 → 建 PR
```

**收到 PR 反馈后：**
```
修改 → 跑特定维度复查 → 验证 → push
```

---

## 四、`/code-review` 详解（更狠的那个）

### 4.1 命令元信息

```yaml
---
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*),
               Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*),
               Bash(gh pr list:*)
description: Code review a pull request
disable-model-invocation: false
---
```

注意：**所有 Bash 都是白名单 gh 命令**，安全性高，不会乱跑命令。

### 4.2 工作流程（8 步流水线）

```
Step 1: 资格检查（Haiku agent）
        ├─ PR 关闭了？
        ├─ Draft 状态？
        ├─ 不需要 review（自动化 PR / 太简单）？
        └─ 已经 review 过？
        任一为是 → 直接退出

Step 2: 收集 CLAUDE.md（Haiku agent）
        ├─ 根目录 CLAUDE.md
        └─ 修改的目录里的 CLAUDE.md
        只返回路径，不返回内容

Step 3: 总结 PR（Haiku agent）
        └─ gh pr view，让 agent 总结改了什么

Step 4: 5 个 Sonnet agent 并行评审
        ├─ Agent #1: 检查 CLAUDE.md 合规性
        ├─ Agent #2: 浅扫明显 bug（避免读太多上下文，专注变更）
        ├─ Agent #3: 读 git blame 和历史，结合历史判断 bug
        ├─ Agent #4: 读以前 PR 的评论，看是否有适用建议
        └─ Agent #5: 读代码注释，看变更是否符合注释里的指引

Step 5: 置信度评分（每个 issue 派一个 Haiku agent）
        给每个 issue 打 0-100 分：
        ├─  0: 经不起推敲的假阳性
        ├─ 25: 可能是真问题，没法验证
        ├─ 50: 确认是真问题，但是 nitpick
        ├─ 75: 很可能是真 bug，影响功能
        └─ 100: 100% 确认是 bug

Step 6: 过滤
        score < 80 的全部丢掉
        如果没有 ≥80 的 → 直接退出

Step 7: 二次资格检查（Haiku agent）
        重做 Step 1，因为流程跑了几分钟，PR 状态可能变了

Step 8: gh pr comment 把结果评论到 PR
```

### 4.3 严格的"假阳性"过滤规则

源码列出的**不该报告**的问题：

| 分类 | 例子 |
|------|------|
| 历史遗留 | 不是这个 PR 引入的 pre-existing 问题 |
| 看着像 bug 实则不是 | 误报 |
| 资深工程师不会提的 nitpick | 比如换行、命名风格之类 |
| Linter / 类型检查器能抓的 | 类型错误、import 错误、格式问题（CI 会跑） |
| 测试覆盖率不足等通用质量问题 | 除非 CLAUDE.md 明确要求 |
| 代码里 `// eslint-disable` 主动忽略的 | 已知豁免 |
| 看上去是有意修改的 | 配套大变更 |
| 用户没改的行 | 不属于本 PR 范围 |

### 4.4 输出格式（直接评论到 PR）

**有问题时：**

```markdown
### Code review

Found 3 issues:

1. <bug 简述> (CLAUDE.md says "<...>")

<完整 sha 链接，比如 https://github.com/anthropics/claude-code/blob/1d54823877c4de72b2316a64032a54afc404e619/README.md#L13-L17>

2. <bug 简述> (some/other/CLAUDE.md says "<...>")

<完整 sha 链接>

3. <bug 简述> (bug due to <file and code snippet>)

<完整 sha 链接>

🤖 Generated with [Claude Code](https://claude.ai/code)

<sub>- If this code review was useful, please react with 👍. Otherwise, react with 👎.</sub>
```

**无问题时：**

```markdown
### Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

🤖 Generated with [Claude Code](https://claude.ai/code)
```

### 4.5 链接格式严格规定

源码原话翻译：

- 必须用**完整 git sha**，不能用 `$(git rev-parse HEAD)`，因为评论是直接渲染 Markdown，bash 不会执行
- 仓库名必须和被 review 的仓库匹配
- 文件名后跟 `#`
- 行号格式 `L[start]-L[end]`
- 至少要给目标行**前后各 1 行上下文**（看 5-6 行就链接 4-7）

正确示例：
```
https://github.com/anthropics/claude-cli-internal/blob/c21d3c10bc8e898b7ac1a2d745bdc9bc4e423afe/package.json#L10-L15
```

---

## 五、为什么这两个命令值得研究

### 5.1 模型选择策略

`code-review` 的设计很精明：

- **Haiku 干轻活**：状态判断、收集文件路径、总结 PR、置信度评分
- **Sonnet 干重活**：5 路并行深度评审

**省钱省时间**，不是所有 agent 都用最强模型。这是 LLM 工程的最佳实践。

### 5.2 分布式校验机制

5 个 Sonnet agent 各自独立，**互不影响**。不同视角找到的问题：
- Agent #2: 看代码本身
- Agent #3: 看 git 历史
- Agent #4: 看历史 PR 讨论
- Agent #5: 看代码注释

避免单个 agent 视野盲区。

### 5.3 置信度过滤系统

**这是亮点**。Step 5 的评分机制解决了 LLM review 的最大痛点：**幻觉过多假阳性**。

> 一个 PR 报 30 个问题，开发者直接关闭不看了

通过让另一个 agent 给每个 issue 评分，**只保留 80 分以上**的，把噪音降到最低。这是个很巧妙的"自检"设计。

### 5.4 二次资格检查

Step 7 重做 Step 1，是个细节但很重要——评审跑几分钟，期间 PR 可能：
- 被作者关闭
- 转成 Draft
- 被别人 review 了

不重检就会出现"评论冲突"或"评审已关闭 PR"的尴尬。

---

## 六、实战使用建议

### 6.1 选哪个命令

| 场景 | 推荐 |
|------|------|
| 写代码中途自检 | `/pr-review-toolkit:review-pr code` |
| 提交前完整自检 | `/pr-review-toolkit:review-pr all` |
| 已建 PR 走自动 review 流程 | `/code-review` |
| 想集成到 CI | `/code-review`（输出格式标准） |
| 学习 LLM agent 编排 | 读 `code-review.md` 源码 |

### 6.2 怎么让 review 更准

`/code-review` 强烈依赖 **CLAUDE.md** 来判断"什么是问题"。在项目根目录写好：

```markdown
# CLAUDE.md

## 代码规范
- React setState 必须使用函数式形式
- 所有 setTimeout/setInterval 必须有清理逻辑
- 数据库查询必须使用参数化语句
- 禁止硬编码密钥
```

review agent 会按这个清单逐项扫描，命中的会直接报出来并引用 CLAUDE.md 原话。

### 6.3 让 review 不太啰嗦

如果 `/pr-review-toolkit:review-pr` 报得太多，加参数限定：

```bash
# 只看 critical 类的问题
/pr-review-toolkit:review-pr errors

# 别润色，避免 code-simplifier 强行改写
/pr-review-toolkit:review-pr code tests
```

### 6.4 验证置信度系统

跑完 `/code-review` 后，可以让 Claude 自己说说被过滤了多少 issue：

> "Step 4 总共发现了多少个 issue，Step 6 过滤后剩下几个，过滤率多少？"

通常过滤率 50-80%，正常。

---

## 七、源码深读路径

如果想自己写一个类似的 review 命令，看这些文件：

```
~/.claude/plugins/marketplaces/claude-plugins-official/plugins/
├── pr-review-toolkit/
│   ├── commands/review-pr.md          # 命令定义
│   └── agents/                         # 各专门 agent 的提示词
├── code-review/
│   └── commands/code-review.md        # 严谨流水线版
└── code-modernization/                # 类似设计可参考
    └── commands/modernize-*.md
```

每个 `.md` 文件就是一个命令的完整实现，**Markdown 即代码**。Claude Code 把命令当成提示词模板执行，没有黑盒。

---

## 八、与 GitHub Copilot Code Review 对比

| 维度 | Claude Code review 系列 | Copilot Code Review |
|------|------------------------|---------------------|
| 自定义 | 完全可自定义命令和 agent | 黑盒服务，只能开关 |
| 透明度 | Markdown 源码可读 | 不开放 |
| 多 agent | 6+ 专门 agent 协作 | 单 agent |
| 置信度过滤 | 显式 80 分过滤 | 黑盒 |
| CLAUDE.md 集成 | 强依赖 | 不支持 |
| 成本控制 | 自己选模型组合 | 按席位 |
| 集成复杂度 | 需要 gh + git | GitHub 原生 |

**Claude Code 优势**：可控、可审计、可定制；
**Copilot 优势**：开箱即用、深度集成 GitHub UI。

---

## 九、总结

`/review` 系列命令的精妙之处不在"AI 能 review 代码"，而在**工程化的 LLM 编排**：

1. **分工**：不同复杂度任务用不同模型（Haiku/Sonnet）
2. **并行**：独立子任务并行跑，节约时间
3. **去噪**：置信度评分过滤假阳性
4. **健壮**：二次状态检查避免冲突
5. **透明**：所有逻辑都在 Markdown 里，可读可改

**这是值得借鉴的 LLM Agent 设计模式**。下次自己设计 AI 工作流，可以套用这套思路。