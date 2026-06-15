# Claude Code 前端开发最佳实践

> 整理自 Anthropic 官方指南、社区实战经验与多位资深开发者的实践总结
> 最后更新：2025年（持续更新中）

---

## 目录

1. [Claude Code 的本质定位](#1-claude-code-的本质定位)
2. [CLAUDE.md 配置最佳实践](#2-claudemd-配置最佳实践)
3. [前端设计美学与 Skills](#3-前端设计美学与-skills)
4. [核心工作流模式](#4-核心工作流模式)
5. [上下文管理](#5-上下文管理)
6. [权限与安全](#6-权限与安全)
7. [自动化与 Hooks](#7-自动化与-hooks)
8. [高级技巧（LSP、后台代理、多Agent）](#8-高级技巧)
9. [前端实战案例](#9-前端实战案例)
10. [参考来源](#10-参考来源)

---

## 1. Claude Code 的本质定位

### 1.1 不是代码生成器，而是 Agent 工具

Claude Code（CC）是一个纯粹的 **Agent 工具**，而非传统意义上的代码生成器。它与普通工具的本质区别：

| 维度 | 传统代码生成工具 | Claude Code (Agent) |
|------|-----------------|---------------------|
| 交互模式 | 单次请求-响应 | 持续多轮对话 |
| 上下文 | 无累积 | 完整维护对话历史和工作区状态 |
| 任务理解 | 按提示生成片段 | 理解完整开发流程 |
| 自主性 | 被动响应 | 主动规划、探索、执行 |

### 1.2 正确的使用心智模型

- **把 Claude 当作一个非常快速的实习生**——它有完美记忆但经验有限
- **指令要具体**——Claude 不会读心术
- **使用迭代工作流**——第一次尝试很少是最终方案
- **利用 Claude 的优势**：模式识别、文档能力、系统性问题解决

### 1.3 前端开发的特殊性

> "当你要求 LLM 构建一个落地页而不加指导时，它几乎总是使用 Inter 字体、白色背景上的紫色渐变、以及最少的动画。"

前端开发中，Claude 容易陷入 **分布收敛（Distributional Convergence）** 问题——即生成"最安全"、最常见的 UI 设计。因此前端开发需要更明确的 **美学引导**。

---

## 2. CLAUDE.md 配置最佳实践

### 2.1 文件放置位置（优先级从高到低）

| 位置 | 说明 | 适用场景 |
|------|------|---------|
| `./CLAUDE.md` | 仓库根目录，检入 git | 团队共享 |
| `./CLAUDE.local.md` | 仓库根目录，加入 .gitignore | 个人本地配置 |
| 父目录 `CLAUDE.md` | 运行 claude 目录的上级 | Monorepo |
| 子目录 `CLAUDE.md` | 处理子目录文件时按需拉入 | 按模块细化 |
| `~/.claude/CLAUDE.md` | 主文件夹 | 全局偏好 |

### 2.2 推荐的 CLAUDE.md 结构

```markdown
# 项目: [项目名称]

## 启动命令
- `npm run dev` : 开发服务器
- `npm run build` : 构建项目
- `npm run test` : 运行测试
- `npm run lint` : 代码检查
- `npm run typecheck` : 类型检查

## 代码风格
- 使用 ES 模块 (import/export)，不用 CommonJS (require)
- 尽可能解构导入：`import { foo } from 'bar'`
- 使用 TypeScript 严格模式
- 组件使用函数式 + Hooks

## 项目架构
- 组件库: `src/components/`
- 页面: `src/pages/`
- API: `src/services/`
- 类型定义: `src/types/`
- 工具函数: `src/utils/`

## 工作流规则
- 分支命名: `feature/xxx` 或 `fix/xxx`
- 代码修改后必须运行 `npm run typecheck`
- 新功能必须编写测试
```

### 2.3 调优技巧

- **保持简短**：Claude 可靠遵循约 150-200 条指令。过长会导致随机忽略。
- **聚焦项目独特性**：告诉它项目的"奇怪"之处，而非通用知识。
- **解释"为什么"**：例如"使用 TypeScript 严格模式，因为曾因隐式 any 导致生产 bug"。
- **持续更新**：按 `#` 键快速添加指令；重复纠正同个问题时立即写入。
- **使用强调词**：用 `IMPORTANT`、`YOU MUST` 加重关键规则。
- **每一条规则自问"没它会怎样？"**，如果大概率也能做对，则是冗余。

### 2.4 前端专用 CLAUDE.md 示例

```markdown
## UI 组件指南
- 使用 Tailwind CSS 类名，禁止内联 style
- 组件接受 className prop 以支持外部覆盖
- 使用 React Hook Form 处理表单，Zod 做验证
- 所有 API 请求通过 useQuery/useMutation 封装（TanStack Query）

## UI 约束
- 颜色使用主题变量，不硬编码色值：`text-primary` 而非 `text-blue-500`
- 响应式断点：sm(640) / md(768) / lg(1024) / xl(1280)
- 避免使用 Inter 和 Roboto 字体（过度使用）
- 优先使用 CSS 动画而非 JS 动画

## 测试规范
- 组件测试使用 React Testing Library
- 优先测试用户行为而非实现细节
- E2E 测试使用 Playwright
```

---

## 3. 前端设计美学与 Skills

### 3.1 前端美学 Prompt（~400 tokens）

Anthropic 官方推荐的前端美学引导 Prompt：

```xml
<frontend_aesthetics>
你倾向于收敛到通用、"在分布上"的输出。
在前端设计中，这会产生用户所谓的"AI 风格"美学。
避免这种情况：创建有创意、独特、令人惊喜和愉悦的前端。

重点关注：
- **字体**：选择美观、独特、有趣的字体。避免 Arial 和 Inter 等通用字体；
  选择能提升前端美感的独特字体。
- **色彩与主题**：坚持连贯的美学。使用 CSS 变量保持一致性。
  大胆的主色配锐利强调色，优于均匀分布的小心调色板。
- **动效**：使用动画实现效果和微交互。HTML 优先 CSS-only 方案，
  React 中可用 Motion 库。聚焦高光时刻：一个精心编排的页面加载
  配合交错显示（animation-delay）比散落的微交互更令人愉悦。
- **背景**：营造氛围和层次感，而非默认使用纯色。
  叠加 CSS 渐变、几何图案或匹配整体美学的上下文效果。

避免通用的 AI 生成美学：
- 过度使用的字体（Inter、Roboto、Arial、系统字体）
- 陈词滥调的配色方案（特别是白色背景的紫色渐变）
- 可预测的布局和组件模式
- 缺乏上下文特色的千篇一律设计

要有创意地诠释，做出真正为上下文设计的不拘一格的选择。
在亮/暗主题、不同字体、不同美学之间变换。
</frontend_aesthetics>
```

### 3.2 字体选择指导

```xml
<use_interesting_fonts>
字体即时传达品质感。避免使用无聊的通用字体。

绝不使用：Inter、Roboto、Open Sans、Lato、系统字体

好的字体的示例：
- 代码风格：JetBrains Mono、Fira Code、Space Grotesk
- 编辑风格：Playfair Display、Crimson Pro
- 技术风格：IBM Plex 系列、Source Sans 3
- 独特风格：Bricolage Grotesque、Newsreader

配对原则：高对比度 = 有趣。展示体 + 等宽体、衬线体 + 几何无衬线体。
使用极端值：100/200 重量 vs 800/900，而非 400 vs 600。
尺寸跳跃 3x+，而非 1.5x。
选一个独特的字体，果断使用。从 Google Fonts 加载。
</use_interesting_fonts>
```

### 3.3 Skills：按需加载的动态上下文

**Skills** 是存储在专门目录中的指令和上下文资源，按需激活，为特定任务类型提供专门指导，**不产生永久的上下文开销**。

```
~/.claude/skills/
├── frontend-aesthetics.md   # 前端美学指导
├── react-component.md       # React 组件开发规范
├── tailwind-patterns.md     # Tailwind CSS 模式
└── api-integration.md       # API 集成模式
```

Skills 的核心价值：在需要时注入精确的领域知识，不需要时零开销。

### 3.4 Skills vs CLAUDE.md 的选择策略

| 场景 | 用 CLAUDE.md | 用 Skills |
|------|-------------|----------|
| 项目通用的构建/测试命令 | ✅ | ❌ |
| 编码风格和约定 | ✅ | ❌ |
| 项目架构说明 | ✅ | ❌ |
| 前端高级美学指导 | ❌ | ✅ |
| 特定框架的深度模式 | ❌ | ✅ |
| 部署约定、内部 API 文档 | ❌ | ✅ |

---

## 4. 核心工作流模式

### 4.1 探索 → 规划 → 编码 → 提交（官方推荐）

```
阶段 1: 探索
  "先读取认证模块和相关测试文件，不要写任何代码。"

阶段 2: 规划
  "思考如何添加 OAuth 支持。制定详细计划，考虑：
   - 与现有认证流程的集成
   - 数据库 schema 变更
   - 测试策略
   - 安全考虑"

阶段 3: 编码
  "现在按照计划实现 OAuth 功能。"

阶段 4: 提交
  "运行所有测试，修复问题，按 conventional commit 格式提交。"
```

### 4.2 测试驱动开发（TDD）

```
1. 编写测试（基于预期输入/输出）
2. 运行测试，确认失败，不许实现
3. 提交测试
4. 编写代码使测试通过，禁止修改测试
5. 提交代码
```

**前端 TDD 特别提示：**
- 组件测试用 React Testing Library，测试用户行为而非实现
- 禁止在实现过程中修改测试
- 优先单测，不跑全量测试套件（性能考虑）

### 4.3 视觉开发工作流

```
"设置 Puppeteer MCP 用于截图。"
"这是设计稿 [拖入图片]。实现它，截图，迭代。"
```

适用于：
- UI 组件实现后截图验证
- CSS 布局问题排查
- 响应式设计验证
- 给 Claude 看错误截图进行调试

### 4.4 前端 UI 开发闭环

```
"实现这个登录页面，然后：
1. 启动开发服务器：!npm run dev
2. 用 Chrome 打开 localhost:3000/login
3. 截图验证视觉表现
4. 检查控制台是否有报错
5. 直到感觉对劲为止"
```

**关键原则**：始终给 Claude 一种验证自己工作的方法。如果 Claude 能看到自己代码的运行结果，代码质量会提升 2-3 倍。

### 4.5 安全 YOLO 模式

```bash
# 在 Docker Dev Container 或隔离环境中使用
claude --dangerously-skip-permissions

# 更安全的方式：限制工具范围
claude --allowedTools "Edit,Bash(eslint:*)" "修复 src/ 中所有 ESLint 错误"
```

**仅推荐在隔离的容器环境**中使用完全跳过权限的模式。

---

## 5. 上下文管理

### 5.1 黄金法则

> **200k 上下文是甜蜜的陷阱**。上下文使用率达到 20-40% 时模型性能已明显下降，/compact 无法挽回。

### 5.2 管理策略

| 策略 | 操作 | 适用场景 |
|------|------|---------|
| `/clear` | 清除对话历史，保留 CLAUDE.md | 任务切换 |
| `/compact` | 摘要压缩对话 | 复杂调试中途 |
| 分割对话 | 每个功能/任务使用独立对话 | 大型项目 |
| 外部记忆 | 写计划到 `SCRATCHPAD.md` | 跨会话任务 |
| 果断重开 | `/clear` 重新开始 | 对话脱轨时 |

### 5.3 前端开发中的上下文管理

- **复杂任务先开计划模式**（双击 Shift+Tab）：说清楚改哪些文件、为什么改、怎么验证
- **不相关任务间先 `/clear`**：避免多个话题污染上下文
- **使用 `@文件路径` 直接点名**：
  ```
  重点看：
  @src/components/LoginForm.tsx
  @src/hooks/useAuth.ts
  先分析调用关系，再给修改方案。
  ```
- **用 `/btw` 处理插问**：临时性附带问题不塞进主流程
- **子 Agent 做"先研究，再汇报"**：调查任务不撑脏主上下文

---

## 6. 权限与安全

### 6.1 权限层级

| 操作类型 | 默认行为 | 建议策略 |
|---------|---------|---------|
| 读取操作 | 自动批准 | 保持默认 |
| 写入操作 | 需要批准 | 配置信任工具 |
| Bash 命令 | 需要批准 | 按需允许 |

### 6.2 常用权限配置

```bash
# 会话中设置始终允许
/permissions add Edit
/permissions add "Bash(git commit:*)"
/permissions add "Bash(git push:*)"
/permissions add "Bash(npm run *)"
```

### 6.3 团队权限（`.claude/settings.json`）

```json
{
  "allowedTools": [
    "Edit",
    "Bash(git commit:*)",
    "Bash(npm run *)",
    "mcp__github__*"
  ]
}
```

### 6.4 前端安全注意事项

- **不要在 CLAUDE.md 中记录密码或密钥**
- API 密钥使用环境变量（`.env` 文件，不提交 git）
- 敏感变更（认证、支付、数据操作）必须人工审查
- `.gitignore` 排除敏感文件

---

## 7. 自动化与 Hooks

### 7.1 PostToolUse Hook：自动格式化

每次编辑后自动运行 Prettier：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{
          "type": "command",
          "command": "npx prettier --write \"$CLAUDE_FILE_PATH\" 2>/dev/null || true"
        }]
      }
    ]
  }
}
```

### 7.2 PostToolUse Hook：类型检查

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{
          "type": "command",
          "command": "npx tsc --noEmit --pretty 2>&1 | head -20"
        }]
      }
    ]
  }
}
```

### 7.3 PreToolUse Hook：危险命令拦截

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "if echo \"$TOOL_INPUT\" | grep -qE 'rm -rf|drop table|truncate'; then echo 'BLOCKED: destructive command' >&2; exit 2; fi"
      }]
    }]
  }
}
```

### 7.4 Headless Mode 批量任务

```bash
# 批量处理低耦合任务
for file in $(cat files-to-migrate.txt); do
  claude -p "把 $file 从旧写法迁移到新写法，并保持测试通过" \
    --allowedTools "Edit,Bash" &
done
wait
```

### 7.5 自定义斜杠命令

在 `.claude/commands/` 下创建 markdown 文件，输入 `/` 即可调用。

**示例：`fix-github-issue.md`**

```markdown
请分析并修复 GitHub 问题：$ARGUMENTS。

按以下步骤：
1. 使用 `gh issue view` 获取问题详情
2. 理解问题描述的问题
3. 搜索代码库中的相关文件
4. 实施必要的更改
5. 编写并运行测试验证
6. 确保代码通过 linting 和类型检查
7. 创建描述性提交
8. 推送并创建 PR
```

使用方式：`/project:fix-github-issue 1234`

---

## 8. 高级技巧

### 8.1 LSP 集成（v2.0.74+）

Claude Code 原生支持 LSP（Language Server Protocol），提供 IDE 级别代码智能：

| LSP 工具 | 用途 |
|----------|------|
| `goToDefinition` | 跳转到符号定义 |
| `findReferences` | 查找所有引用 |
| `documentSymbol` | 文件符号概览 |
| `hover` | 文档和类型信息 |

**前端（TypeScript/JavaScript）安装：**

```bash
npm install -g @vtsls/language-server typescript
```

**使用示例：**

```
✅ "找到所有调用 authenticate() 的函数"
✅ "显示 UserConfig 在哪里定义"
✅ "找到所有引用 OLD_API_URL 的地方并更新为 NEW_API_URL"
```

### 8.2 后台代理（Background Agents）

用 `&` 前缀在后台运行任务：

```
& 运行全量测试套件
& 部署到 staging 环境并运行冒烟测试
```

管理命令：
- `/agents` 查看运行中的后台代理
- `/stats` 包含后台代理活动信息

**适用场景：**
- 长时间运行的任务（测试、构建、迁移、部署）
- 并行工作（跑后端测试同时处理前端）
- 监控任务（`& 监视日志并在出错时告警`）

### 8.3 多 Claude 工作流

**方案一：一个写代码，另一个验证**
- 终端 1：Claude 写代码
- 终端 2：审查代码
- 终端 3：根据反馈编辑

**方案二：Git Worktrees（推荐）**

```bash
# 创建独立工作区
git worktree add ../project-feature-a feature-a

# 在每个 worktree 中启动 Claude
cd ../project-feature-a && claude

# 完成后清理
git worktree remove ../project-feature-a
```

### 8.4 Extended Thinking 触发词

| 触发词 | 思考预算 |
|--------|---------|
| `think` | 4,000 tokens |
| `think hard` / `think deeply` | 10,000 tokens |
| `ultrathink` | 31,999 tokens (最大) |

**前端适用场景：**
- 架构讨论（组件拆分、状态管理方案）
- 性能优化链路排查
- 复杂交互逻辑的多步推理

### 8.5 快捷键速查

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+C` | 取消操作 |
| `Esc` | 中断当前执行，保持上下文 |
| `Esc + Esc` | 回退历史，编辑提示重来 |
| `/clear` (或 `Ctrl+L`) | 清除终端 |
| `#` | 给 Claude 指令，自动写入 CLAUDE.md |
| `!` | 直接执行 shell 命令 |
| `Tab` | 自动补全 |
| `@文件路径` | 引用文件 |
| 双击 `Shift+Tab` | 进入计划模式 |
| `Alt+T` (Win) / `Option+T` (Mac) | 切换 Thinking 模式 |
| `&` | 后台运行任务 |

---

## 9. 前端实战案例

### 9.1 开发新页面

```
提示模板：
"在 src/pages/ 下创建用户设置页面，要求：
- 使用现有 Layout 组件包装
- 表单字段：昵称、邮箱、头像上传、密码修改
- 使用 React Hook Form + Zod 验证
- 遵循项目中已有的组件模式和样式约定
- 先读取 @src/pages/Profile.tsx 了解现有模式
- 完成后运行 !npm run typecheck 和 !npm run test"
```

### 9.2 修复 UI Bug

```
提示模板：
"修复这个组件在移动端的布局问题：
- 问题：表格在小屏幕上溢出容器
- 查看 @src/components/DataTable.tsx
- 添加响应式方案：小屏幕显示卡片布局替代表格
- 使用项目中的 Tailwind 断点
- 改完后在浏览器验证"
```

### 9.3 重构组件

```
提示模板：
"重构这个类组件为函数组件 + Hooks：
- 文件：@src/components/UserList.tsx
- 保持相同的 props 接口
- 使用 useState + useEffect 替代 setState + lifecycle
- 保持现有测试通过
- 完成后运行 !npm run test"
```

### 9.4 调试 CSS/样式问题

```
提示模板：
"[截图粘贴] 截图中的这个按钮 hover 效果没出来。
查看 @src/components/Button.tsx
检查 CSS 选择器优先级和伪类实现
修复后截图确认"
```

### 9.5 性能优化

```
提示模板：
"分析 @src/pages/Dashboard.tsx 的渲染性能：
1. 找出不必要的重渲染
2. 检查 useMemo/useCallback 的使用
3. 检查列表组件是否有正确 key
4. 检查图片是否懒加载
5. 给出优化方案并实施
6. /ultrathink 对优化效果进行推理验证"
```

---

## 🎬 视频资源推荐

文字看累了？以下是整理的 Claude Code 相关视频教程，从入门到进阶，覆盖中英文资源。

---

### 🇨🇳 中文视频

| 序号 | 标题 | 平台 | 链接 | 说明 |
|------|------|------|------|------|
| 1 | **Claude Code 教程（2025最新版）** | B站 | [BV1CFe2zBEws](https://www.bilibili.com/video/BV1CFe2zBEws/) | 25集全系列，从安装到精通，B站最全 |
| 2 | **Claude Code 从 0 到 1 全攻略** | YouTube | [观看](https://www.youtube.com/watch?v=AT4b9kLtQCQ) | MCP/SubAgent/Skill/Hook/图片/上下文/后台任务/权限全覆盖 |
| 3 | **Claude Code 完整实战攻略（博客园总结）** | 博客园 | [文章总结](https://www.cnblogs.com/luweiseu/p/19718565) | 上述视频的文字版总结，可快速查阅 |
| 4 | **Claude Code 从入门到精通** | SegmentFault | [阅读](https://segmentfault.com/a/1190000047256519) | 最全配置指南和工具推荐 |

### 🇬🇧 英文视频

| 序号 | 标题 | 平台 | 链接 | 说明 |
|------|------|------|------|------|
| 1 | **Claude Code Tutorial (Net Ninja)** | YouTube | [播放列表](https://www.youtube.com/playlist?list=PL4cUxeGkcC9g4YJeBqChhFJwKQ9TRiivY) | **11集系列，945K+播放量**，含 Introduction、CLAUDE.md、Context、Tools、Planning、Slash Commands、MCP、Subagents、GitHub 等 |
| 2 | **Claude Code Tutorial for Beginners (Sabrina Ramonov)** | YouTube | [观看](https://www.youtube.com/watch?v=3HVH2Iuplqo) | 35分钟新手教程，178K播放，含 Blotato MCP 自动化工作流 |
| 3 | **How to Create Frontend with Claude Code** | YouTube | [观看](https://www.youtube.com/watch?v=l7TXmH3Copc) | 专注前端开发，UI 设计、代码生成全流程 |
| 4 | **Building Beautiful Websites with Claude Code** | YouTube | [观看](https://www.youtube.com/watch?v=86HM0RUWhCk) | 教你用 Claude Code 构建美观网站 |
| 5 | **Claude Code Setup That Actually Works (2025)** | YouTube | [观看](https://www.youtube.com/watch?v=P-5bWpUbO60) | 完整设置、安装、工作流教程 |
| 6 | **Design-to-Code Workshop (Claude Code + Cursor + Figma)** | YouTube | [观看](https://www.youtube.com/watch?v=SEy1WPjPF3k) | Figma 设计稿转代码实战工作坊 |
| 7 | **Claude Code Web Tutorial: Build & Deploy Apps** | YouTube | [观看](https://www.youtube.com/watch?v=b3QMFWLC8TY) | 浏览器中直接用 Claude 构建和部署 Web 应用 |
| 8 | **Automate Frontend with Claude Code (Real Case)** | wmedia.es | [阅读](https://wmedia.es/en/writing/ai-frontend-automation-claude-code) | 实战案例：从 1 小时到 3 分钟的前端自动化 |
| 9 | **Claude the Frontend Dev (Martijn Arts)** | Blog | [阅读](https://blog.martijnarts.com/claude-the-frontend-dev) | 用 Claude Code 开发 Leptos 组件的前端实战经验 |
| 10 | **Frontend Design Plugin Tutorial** | Blog | [阅读](https://thomas-wiegold.com/blog/claude-code-frontend-design-plugin) | 如何用官方 frontend-design 插件告别 AI 风格 |
| 11 | **How to Install Claude's Frontend Design Skill** | Blog | [阅读](https://muchendu.com/blog/claude-frontend-design-skill) | 前端设计 Skill 安装与使用指南 |
| 12 | **Frontend Masters: Free Claude Code Course** | Frontend Masters | [查看](https://frontendmasters.com/courses/claude-code) | 专业前端平台的免费 Claude Code 课程 |

### 💡 按学习路径推荐

```
初学者 👉 Net Ninja 播放列表（11集）→ Sabrina 教程 → B站25集系列
前端开发 👉 How to Create Frontend → Frontend Design Plugin → Figma Workshop
进阶用户 👉 从0到1全攻略 → SubAgent → MCP 集成 → Headless 自动化
```

---

## 10. 参考来源

| 来源 | 链接 | 类型 |
|------|------|------|
| Anthropic 官方最佳实践 | https://www.anthropic.com/engineering/claude-code-best-practices | 官方 |
| Improving Frontend Design through Skills | https://claude.com/blog/improving-frontend-design-through-skills | 官方 |
| Claude Code 文档 | https://docs.anthropic.com/en/docs/claude-code/overview | 官方 |
| Claude Code Best Practices (GitHub) | https://github.com/jmckinley/claude-code-resources | 社区 |
| Claude Code Best Practices 综合指南 | https://rosmur.github.io/claudecode-best-practices/ | 社区 |
| The Complete Claude Code Best Practices Guide | https://notes.muthu.co/2025/08/the-complete-claude-code-best-practices-guide/ | 社区 |
| Real-World Lessons Learned | https://johnoct.com/blog/2025/08/01/claude-code-best-practices-lessons-learned/ | 社区 |
| 50个日常使用技巧 | https://developer.aliyun.com/article/1728645 | 中文 |
| 8条黄金法则 | https://cloud.tencent.com/developer/article/2617720 | 中文 |
| 官方内部团队最佳实践翻译 | https://segmentfault.com/a/1190000047224398 | 中文 |
| Claude Code Frameworks & Sub-Agents | https://www.medianeth.dev/blog/claude-code-frameworks-subagents-2025 | 社区 |

---

> **核心原则总结：**
> 1. **规划先行** — 复杂任务先计划再执行
> 2. **上下文管理是基础** — 善用 `/clear`、CLAUDE.md、分割对话
> 3. **前端美学需主动引导** — 用 Skills 和 Prompt 打破 AI 的"分布收敛"
> 4. **验证闭环** — 始终让 Claude 验证自己的工作
> 5. **具体胜过泛泛** — 指令越具体，输出越精准
> 6. **持续沉淀规则** — 每次纠正都写入 CLAUDE.md
> 7. **自动化飞轮** — Hooks + Headless + 自定义命令构建自动化系统
