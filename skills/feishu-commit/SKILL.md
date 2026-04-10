---
description: >
  普通 commit，自动在 message 末尾注入飞书任务链接。
  触发方式：/feishu-commit <task_id_or_url>。
  当用户说"feishu-commit"、"带飞书链接提交"时使用。
---

# feishu-commit

正常提交代码，commit message 末尾自动追加飞书任务链接，其余与 `/commit` 完全一致。

## 步骤

### 1. 提取任务 ID

从用户输入中提取任务 ID：
- 纯 UUID：`c0063ff4-4388-4c3b-9587-6172cded109c`
- 飞书 URL：`https://applink.feishu.cn/client/todo/detail?guid=<uuid>&...`

构建任务链接：
```
https://applink.feishu.cn/client/todo/detail?guid=<task_id>
```

### 2. 正常 commit 流程

```bash
git status
git diff HEAD
git log --oneline -10
```

分析改动，生成 commit message（type + 简短描述，匹配仓库风格）。

只 add 改动的文件，绝不使用 `git add -A` 或 `git add .`。

### 3. 注入飞书链接

Commit message 固定格式：

```
<type>: <根据实际 diff 写的简短描述>

Feishu-Task: https://applink.feishu.cn/client/todo/detail?guid=<task_id>
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
```

`Feishu-Task` trailer 必须存在，用户不需要手动填写。
