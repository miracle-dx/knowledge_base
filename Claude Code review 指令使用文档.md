## 一、指令简介

`/review` 是 Claude Code 内置的**代码审查指令**，用于自动化对 GitHub Pull Request 进行专业代码评审。它会调用 `gh` CLI 工具拉取 PR 的元信息和 diff，然后由 Claude 进行多维度分析，输出结构化的 review 报告。

简单说：把人工写 review 的活交给 AI，省时间、查漏洞、补建议。

---

## 二、指令工作流程

```
用户输入: /review [PR编号]
         │
         ▼
┌─────────────────────────────┐
│ 1. 检查参数                  │
│    - 有 PR 号 → 跳到第 3 步  │
│    - 无 PR 号 → 执行第 2 步  │
└─────────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ 2. gh pr list                │
│    列出当前仓库所有未关闭 PR │
│    用户挑一个再继续          │
└─────────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ 3. gh pr view <number>       │
│    获取 PR 标题、描述、作者  │
│    、目标分支等元信息        │
└─────────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ 4. gh pr diff <number>       │
│    拉取完整 diff 内容        │
└─────────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ 5. Claude 分析并输出 review  │
└─────────────────────────────┘
```

---

## 三、Review 评审维度

`/review` 会从以下五个角度系统性地分析代码：

### 1. 代码正确性（Correctness）
- 逻辑错误、边界条件漏洞
- 闭包陷阱、异步竞态条件
- 空指针、未定义引用
- 异常处理缺失

### 2. 项目规范（Conventions）
- 命名风格是否统一
- 是否符合项目已有架构模式
- 注释、文档是否到位
- 文件组织是否合理

### 3. 性能影响（Performance）
- 时间复杂度（如 O(n²) → O(n)）
- 不必要的重复计算
- 内存泄漏（未清理的 timer、监听器）
- React 渲染性能（key、memo、闭包陈旧值）

### 4. 测试覆盖（Test Coverage）
- 是否有对应单测
- 边界用例是否覆盖
- mock 是否合理

### 5. 安全性（Security）
- SQL 注入、XSS、命令注入
- 敏感信息硬编码
- 权限校验缺失
- 第三方依赖风险

---

## 四、使用前提

| 依赖 | 安装方式 | 说明 |
|------|----------|------|
| `gh` CLI | `winget install GitHub.cli` (Win) / `brew install gh` (Mac) | GitHub 官方命令行工具 |
| `gh` 认证 | `gh auth login` | 首次使用需要登录 GitHub |
| 当前目录 | 必须在 git 仓库内 | 且仓库的 origin 指向 GitHub |

验证安装：
```bash
gh --version
gh auth status
```

---

## 五、使用示例

### 示例 1：不带参数，列出所有 PR

```
/review
```

Claude 会先跑 `gh pr list`，显示形如：

```
#42  优化首页加载速度          feature/perf-home    user-a
#41  修复登录跳转 bug          fix/login-redirect   user-b
#40  接入支付宝 SDK            feat/alipay          user-c
```

然后等你说一个编号继续。

### 示例 2：直接指定 PR 编号

```
/review 42
```

Claude 直接拉 #42 的信息和 diff 进行 review。

### 示例 3：审查本地未提交改动

`/review` 设计上是面向 PR 的，但如果只想看本地改动，可以直接对 Claude 说：

```
帮我 review 一下当前未提交的改动
```

或针对某个文件/某段代码：

```
帮我 review @src/App.jsx#49-57
```

---

## 六、输出报告结构

`/review` 输出的报告通常包含：

### 📌 PR 概览
- 这个 PR 做了什么
- 涉及哪些文件、模块

### 🐛 问题清单（按严重程度分类）
- 🔴 **高危**：必须修复，否则会引入 bug 或安全漏洞
- 🟡 **中等**：建议修复，影响可维护性或性能
- 🔵 **轻微**：风格、命名等小建议

### ✨ 改进建议
- 重构提议
- 复用机会
- 抽象建议

### 🎯 健壮性建议
- 边界处理
- 错误处理
- 清理逻辑

### ✅ 亮点（可选）
- 写得好的地方也会肯定

---

## 七、典型问题 Claude 能发现

来自实战的真实例子（参考审查 [src/App.jsx:49-57](src/App.jsx#L49-L57) 时发现）：

### React 闭包陷阱
```js
// ❌ 错误
setTimeout(() => {
  setTodos(todos.filter(t => !t.completed))  // todos 是旧快照
}, 300)

// ✅ 正确
setTimeout(() => {
  setTodos(prev => prev.filter(t => !t.completed))
}, 300)
```

### O(n²) 优化
```js
// ❌ 多此一举
const ids = todos.filter(t => t.completed).map(t => t.id)
todos.map(todo => ids.includes(todo.id) ? ... : todo)

// ✅ 直接判断
todos.map(todo => todo.completed ? ... : todo)
```

### 重复逻辑提取
- 发现两个函数有 90% 相似的"动画后删除"逻辑
- 建议提取为 `animateAndRemove(predicate)` 通用方法

### 资源清理
- setTimeout 没有清理 → 组件卸载后 setState 报错
- 建议用 `useRef` 存 timer ID 并在 cleanup 中清除

---

## 八、最佳实践

### ✅ 推荐做法
1. **每个 PR 提交后立即 `/review`**：在 merge 前发现问题
2. **结合人工 review**：AI 查模式问题，人查业务逻辑
3. **保留 review 历史**：把重要建议保存为团队的最佳实践文档
4. **review 后追加测试**：AI 提出的边界 case 转化为单元测试

### ⚠️ 注意事项
1. **AI 不是万能的**：业务上下文、团队约定 AI 不一定知道
2. **大 PR 拆小**：超过 500 行的 diff，AI review 质量会下降
3. **敏感信息**：包含密钥、内部 URL 的 PR 谨慎使用
4. **手动验证**：AI 提出的修复建议要自己跑一遍确认

---

## 九、与其他指令对比

| 指令 | 适用场景 | 输入 |
|------|----------|------|
| `/review` | 审查 GitHub PR | PR 编号 |
| `/security-review` | 安全专项审查 | 当前分支 diff |
| `/init` | 初始化 CLAUDE.md | 无 |
| `simplify` skill | 重构改进现有代码 | 当前改动 |

---

## 十、扩展：自定义 review 规则

可以在 `.claude/CLAUDE.md` 中添加项目特定的审查规则，让 `/review` 自动套用：

```markdown
# 代码审查规则

- 所有 setState 必须使用函数式形式
- React 组件必须用 PropTypes 或 TypeScript
- 数据库查询必须使用参数化语句
- 所有 setTimeout/setInterval 必须有清理逻辑
```

这样 `/review` 在分析时会优先检查这些点。

---

## 十一、总结

`/review` 把代码审查从"凭经验"变成"按维度系统性扫描"，是个**省时不省质量**的工具。

适合：
- ✅ 单人开发想要"第二双眼睛"
- ✅ 小团队没有专职 reviewer
- ✅ 大 PR 需要快速识别风险点
- ✅ 学习新代码库时帮助理解

不适合：
- ❌ 替代人工的最终审查
- ❌ 业务逻辑正确性的判断
- ❌ 没有联网/没装 `gh` 的环境
