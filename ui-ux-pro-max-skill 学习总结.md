---
name: ui-ux-pro-max 入门学习总结
description: 2026-05-27 首次学习 ui-ux-pro-max skill 的使用流程和关键认知
type: project
originSessionId: bbf53902-09a9-410b-b045-3548889dd5d1
---
## 核心流程

1. **分析需求** → 提取产品类型、风格关键词、行业
2. **`--design-system`** → 并行搜索 5 个域，输出完整设计系统（样式+配色+字体+效果+反模式）
3. **深挖细节** → 按需搜 `--domain ux|typography|chart|style` 等
4. **查栈指南** → `--stack html-tailwind`（默认栈）获取实现级最佳实践
5. **输出代码** → 基于设计系统生成 HTML/CSS 静态原型

## 关键认知

| 要点 | 内容 |
|------|------|
| Skill 机制 | 一个对话只加载一次，后续复用上下文 |
| 取舍原则 | 原型用原生（零依赖），生产用库 |
| 学习 vs 交付 | 学习模式聚焦设计系统和视觉规范，不做功能实现 |
| 设计一致性 | 同一项目各页面共享一套设计系统（蓝、Flat Design、Plus Jakarta Sans） |
| 进阶空间 | 多域交叉查询、推理引擎、反模式分析、Chart 域、Prompt 域、Markdown 输出 |

## 产出

- `taskflow-login.html` — 分栏布局 + 表单验证 + 暗色模式
- `taskflow-board.html` — 三列看板 + 卡片 + 优先级标签 + 拖拽 + 进度条

## 设计系统 (TaskFlow)

- **样式**: Flat Design（无阴影渐变，简洁线条）
- **主色**: `#2563EB` / `#3B82F6`
- **字体**: Plus Jakarta Sans
- **效果**: 150-200ms 过渡，hover 颜色/透明度变化
- **反模式**: 复杂上手流程 + 慢性能
