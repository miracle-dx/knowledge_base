---
name: claude-code-skill-实战总结
description: Claude Code 项目级 skill 的创建、调用和 code-review/simplify 指令的实战经验总结
date: 2026-06-16
metadata:
  type: reference
---

# Claude Code Skill 实战总结

2026-06-16，在 `industrial-distribution-dashboard` 项目中实战了 Claude Code 的 skill 系统和代码审查能力。

---

## 一、项目级自定义 Skill

### 格式要求

每个 skill 是 `.claude/skills/<skill-name>/SKILL.md` 文件夹结构：

```
.claude/skills/
  query-list-page/
    SKILL.md        ← 主文件，必须命名 SKILL.md（大小写敏感）
```

### SKILL.md 必须有 frontmatter

```markdown
---
name: query-list-page
description: 生成标准查询列表页（SearchForm + JXTable + Pagination）
whenToUse: 创建管理后台列表页时
triggers:
  - "查询列表"
  - "列表页"
---

# 正文内容...
```

- `name` — skill 的唯一名称
- `description` — 简短描述，Claude 用来判断是否触发
- `whenToUse` — 使用场景说明
- `triggers` — 触发关键词列表

### 加载方式

项目启动时自动扫描 `.claude/skills/` 子目录，注册为 `/<目录名>` 命令。输入 `/query-list-page` 即可调用。

### skill 与测试的自检机制

在 SKILL.md 末尾添加「执行完成后必须自检」段落，每次加载 skill 都会带上：

```markdown
## 执行完成后必须自检

1. **类型检查**: `npx tsc --noEmit`
2. **构建验证**: `npx vite build 2>&1 | tail -20`
3. **自动修复 lint**: `npx eslint src/pages/XxxManage --fix`
4. **依赖检查**: 如有新 import 的包，需确认 package.json 存在
```

但注意：**skill 只保证自身文件中的指令每次都执行**。像「自动生成测试用例」这种逻辑，需要显式写在 SKILL.md 里才生效。如果要跨会话持续生效，可以用 hooks（`.claude/settings.local.json` 的 `hooks.after_task`）。

---

## 二、code-review 指令

### 使用方式

```
/code-review                    ← 只报告，不改代码
/code-review --fix              ← 报告 + 自动修复
/code-review --comment          ← 作为 PR comment 输出
```

### 高 effort 审查流程

```
Phase 0: 收集 diff（git diff HEAD）
Phase 1: 7 个独立角度并行审查
  - Angle A: 逐行 diff scan（逐行审查每段 hunk）
  - Angle B: 删除行为审计（被删除的代码是否有对应替代）
  - Angle C: 跨文件追踪（函数调用链检查）
  - Reuse: 重复代码（现有工具类可替代）
  - Simplification: 不必要的复杂度
  - Efficiency: 效率问题（冗余计算、闭包泄漏）
  - Altitude: 深度不足（当前修复是否是表层修复）
Phase 2: Verify（1票验证，recall-biased）
  - 每个发现独立验证：CONFIRMED / PLAUSIBLE / REFUTED
  - 保留 CONFIRMED 和 PLAUSIBLE
Phase 3: 输出 top ≤10 条，JSON 格式
```

### 实战发现的典型问题

**1. Stale closure（最严重）**
```jsx
// ❌ 错误：setParams 后 fetchData 读的是旧 params
const fetchData = useCallback(async (page) => {
  const res = await Service.queryList({ ...params, page })  // params 是闭包旧值
}, [params])

const handleSearch = () => {
  setParams(queryParams)
  fetchData(1)  // 闭包捕获的 params 还没更新！
}

// ✅ 正确：fetchData 直接接收最新参数
const fetchData = useCallback(async (searchParams, page) => {
  const res = await Service.queryList({ ...searchParams, page })
}, [])  // 零依赖
```

**2. antd 依赖位置**
antd 作为运行时包应放在 `dependencies` 而非 `devDependencies`，否则 `npm ci --only=production` 部署会白屏。

**3. RangePicker 判空**
```jsx
// ❌ 错误：[null, null] 是 truthy 数组
if (values.createDate) {
  values.createDate[0].format('YYYY-MM-DD')  // null.format() 抛错
}

// ✅ 正确
if (Array.isArray(values.createDate) && values.createDate[0])
```

**4. catch 绑定 error**
```jsx
catch { message.error('删除失败') }                    // ❌ 看不到具体错误
catch (err) { message.error(err.message || '删除失败') }  // ✅ 可见错误详情
```

---

## 三、simplify 指令

### 使用方式

```
/simplify                     ← 扫描所有变更文件，自动应用简化修复
/simplify src/pages/          ← 限定目录
```

### 4 角度审查 + 自动修复

和 code-review 的区别：

| | code-review | simplify |
|--|:---:|:---:|
| 找 bug | ✅ 是 | ❌ 不找 |
| 找重复/可简化 | ✅ 附带 | ✅ 主力 |
| 自动修改文件 | ❌（需 `--fix`） | **✅ 直接改** |
| 典型结果 | 报告 10 条发现 | 重构 5-8 处文件 |

### 实战应用效果

| 重构项 | 方式 | 效果 |
|--------|------|------|
| Service CRUD 重复 | 抽取 `createService(basePath)` 工厂 | 每个 Service 减少 ~70% 代码量 |
| 日期处理重复 | 抽取 `extractDateRange()` + `formatDate()` | 两页面消除重复逻辑 |
| stale closure | `fetchData` 改为参数传递 | useCallback 零依赖，彻底消除闭包过期 |
| 路由 | 添加 react-router-dom | App 支持 `/products` 和 `/users` |
| console.error 改 message.error | 用户可见错误提示 | UX 改进 |

---

## 四、测试配置（Vitest + RTL）

### 技术栈

| 工具 | 用途 |
|------|------|
| **Vitest** | 测试运行器（与 Vite 原生集成） |
| **@testing-library/react** | React 组件渲染测试 |
| **@testing-library/jest-dom** | DOM 断言（toBeInTheDocument） |
| **jsdom** | 浏览器环境模拟 |

### 关键配置

vite.config.js 中：
```js
/// <reference types="vitest" />
test: {
  environment: 'jsdom',
  globals: true,
  setupFiles: './src/test/setup.js',
},
```

setup.js：
```js
import '@testing-library/jest-dom'
```

### antd 组件测试需要 mock

```js
beforeAll(() => {
  // matchMedia
  Object.defineProperty(window, 'matchMedia', {
    value: vi.fn().mockImplementation((query) => ({ matches: false, ... })),
  })
  // ResizeObserver
  class ResizeObserverMock {
    observe = vi.fn()
    unobserve = vi.fn()
    disconnect = vi.fn()
  }
  window.ResizeObserver = ResizeObserverMock
})
```

---

## 五、总结

1. **Skill 系统**：把常用模板写成 skill，能显著提升效率。关键是写清楚 description 和 triggers，让 Claude 能自动匹配合适场景。
2. **code-review vs simplify**：code-review 适合质量门禁（找 bug + 报告），simplify 适合日常重构（直接修重复代码）。
3. **React 闭包陷阱**：useCallback 依赖对象类型（params、pagination）时极易出 stale closure bug，最佳实践是**函数直接接收最新值作为参数**。
4. **抽取时机**：当第二个类似页面出现时，可以同步抽取公共工具类（createService、useCrudList 等），避免等第三个页面时再大改。
