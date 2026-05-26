# 02 - settings.local.json 规范

> 规定 `.claude/settings.local.json` 的内容结构与推荐配置。

---

## 1. 定位

`.claude/settings.local.json` 是 **项目级** Claude Code 配置：

- **permissions** —— 自动放行的工具调用，避免每次都要确认
- **hooks** —— 在某些事件触发时自动执行的脚本（如 PreCommit / PostToolUse）
- **environment** —— 环境变量
- **disabled-skills** —— 禁用的 Skill

> ⚠️ 这是 `local.json`——属于本机/项目设置，**不入版本控制**。需团队共享的设置放 `.claude/settings.json`（去掉 `.local`）。

---

## 2. 顶层结构

```json
{
  "permissions": { ... },
  "hooks": [ ... ],
  "env": { ... },
  "disabledSkills": [ ... ]
}
```

---

## 3. permissions 配置（推荐）

### 3.1 自动放行 Bash 命令

```json
{
  "permissions": {
    "allow": [
      "Bash(go build *)",
      "Bash(go vet *)",
      "Bash(go fmt *)",
      "Bash(go test *)",
      "Bash(go run *)",
      "Bash(git status)",
      "Bash(git log *)",
      "Bash(git diff *)",
      "Bash(git show *)",
      "Bash(git branch)",
      "Bash(ls *)",
      "Bash(find *)",
      "Bash(grep *)",
      "Bash(rg *)",
      "Bash(cat *)",
      "Bash(head *)",
      "Bash(tail *)",
      "Bash(wc *)",
      "Bash(mkdir *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)"
    ]
  }
}
```

### 3.2 自动放行其他工具

```json
{
  "permissions": {
    "allow": [
      "Read(*)",
      "Write(*)",
      "Edit(*)",
      "Glob(*)",
      "Grep(*)"
    ]
  }
}
```

读写文件已是基础操作，建议全部放行。删除/破坏性命令保持需确认。

---

## 4. hooks 配置（推荐）

### 4.1 PostToolUse: 文档同步提醒

```json
{
  "hooks": [
    {
      "name": "remind-doc-sync-on-go-edit",
      "event": "PostToolUse",
      "matcher": {
        "tool": "Edit",
        "filePathPattern": "*.go"
      },
      "command": "echo '⚠️ 注意：本次编辑了 .go 文件，请按 CLAUDE.md 范围判定表检查是否需要同步 docs/ 文件'"
    }
  ]
}
```

### 4.2 Stop: 文档同步声明检查

```json
{
  "hooks": [
    {
      "name": "enforce-doc-sync-statement",
      "event": "Stop",
      "command": "scripts/check_doc_sync_statement.sh"
    }
  ]
}
```

`scripts/check_doc_sync_statement.sh` 检查 AI 最近一条回复末尾是否有「文档同步：...」声明。

### 4.3 PreToolUse: 防止误删 docs/

```json
{
  "hooks": [
    {
      "name": "prevent-docs-deletion",
      "event": "PreToolUse",
      "matcher": {
        "tool": "Bash",
        "commandPattern": "rm.*docs/"
      },
      "command": "exit 1",
      "blockMessage": "禁止用 rm 直接删除 docs/ 下文件，请用 Edit 工具或显式确认"
    }
  ]
}
```

---

## 5. env 配置

```json
{
  "env": {
    "GO111MODULE": "on",
    "GOPROXY": "https://goproxy.cn,direct",
    "TLOG_OUTPUT_PATH": "log",
    "PROJECT_ENV": "dev"
  }
}
```

---

## 6. disabledSkills 配置

如果某些 Skill 在当前项目中不适用（如项目未启用 Kafka），可禁用：

```json
{
  "disabledSkills": [
    "new-mq-consumer"
  ]
}
```

被禁用的 Skill 在 AI 调用时立即报错，避免误用。

---

## 7. 完整示例

```json
{
  "permissions": {
    "allow": [
      "Bash(go build *)",
      "Bash(go vet *)",
      "Bash(go fmt *)",
      "Bash(go test *)",
      "Bash(git status)",
      "Bash(git log *)",
      "Bash(git diff *)",
      "Bash(ls *)",
      "Bash(find *)",
      "Bash(grep *)",
      "Bash(rg *)",
      "Bash(mkdir *)",
      "Read(*)",
      "Write(*)",
      "Edit(*)",
      "Glob(*)",
      "Grep(*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)"
    ]
  },
  "env": {
    "GO111MODULE": "on",
    "GOPROXY": "https://goproxy.cn,direct"
  },
  "disabledSkills": []
}
```

---

## 8. 与 settings.json（共享） vs settings.local.json（本机）的区别

| 文件 | 用途 | 入 git |
|------|------|--------|
| `.claude/settings.json` | 团队共享配置（hooks 强制项 / 通用 permissions） | ✅ |
| `.claude/settings.local.json` | 本机偏好（个人 env / 调试 hooks） | ❌（应在 .gitignore） |

通常项目模板下只放 `settings.local.json` 示例（本规范模板就是如此），团队级 `settings.json` 由各项目自行决定。

---

## 9. 维护

| 触发 | 动作 |
|------|------|
| 新增项目级工具 | permissions.allow 追加 |
| 出现误操作风险 | permissions.deny 追加 + hooks 拦截 |
| 新增团队级强制规则 | 移到 `.claude/settings.json`（共享） |

---

## 10. 模板

可直接复制到新项目的骨架：

`aiweave/templates/.claude/settings.local.json`（如有需要可生成）。
