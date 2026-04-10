# feishu-tasks 插件功能流程图

## 总体架构

```
用户输入
  ├── /feishu-task  ──→ 拉取任务 + 实现代码（按需完成/评论）
  ├── /feishu-dev   ──→ 全自动开发流程（拉取→Plan→实现→验证→commit→push）
  └── /feishu-commit──→ 普通 commit + 自动注入飞书链接

共享底层：feishu_api.py（Python stdlib，零依赖）
```

---

## 一、feishu-task：任务处理

```mermaid
flowchart TD
    A([用户输入飞书任务 ID / URL]) --> B[check_config\n检查凭据]
    B -->|未配置| C[引导配置\nApp ID + App Secret\n → save_config]
    C --> B
    B -->|已配置| D[parse_task_id\n提取 UUID]
    D --> E[get_task\n拉取任务详情\n summary / description / due / comments]
    E --> F[展示任务信息给用户]
    F --> G[分析代码库 → 实现代码]
    G --> H{用户是否要求\n后续操作?}
    H -->|"评论/备注"| I[add_comment]
    H -->|"完成/关闭"| J[complete_task]
    H -->|无| K([结束])
    I --> K
    J --> K
```

---

## 二、feishu-dev：全自动开发流程

```mermaid
flowchart TD
    START([用户输入 /feishu-dev task_id]) --> P0

    subgraph P0["Phase 0 · 一次性配置"]
        P0A{CLAUDE.md 有\n项目路径?}
        P0A -->|有| P1
        P0A -->|没有| P0B[询问前后端路径\n建议写入 CLAUDE.md]
        P0B --> P1
    end

    subgraph P1["Phase 1 · 拉取 + 补问"]
        P1A[check_config] -->|未配置| P1B[引导配置]
        P1B --> P1A
        P1A -->|已配置| P1C[get_task\n拉取 summary + description + comments]
        P1C --> P1D{清晰度自检\n知道改哪个模块?\n知道改什么?\nBug 类有复现证据?}
        P1D -->|不满足| P1E[AskUserQuestion 补问]
        P1E --> P1D
        P1D -->|满足| P1F{是否需要进一步补问?\ngrep 命中多个候选文件?}
        P1F -->|需要| P1G[grep 代码库\n生成候选文件选项\n→ AskUserQuestion]
        P1G --> P2
        P1F -->|不需要| P2
    end

    subgraph P2["Phase 2 · Plan 确认"]
        P2A[读取目标文件\ngrep 具体代码块]
        P2A --> P2B{数据类 Bug?}
        P2B -->|是| P2C[强制全链路排查\n前端参数 → 后端 where/filter]
        P2C --> P2D
        P2B -->|否| P2D[生成 Plan\n要改什么 + 不动什么]
        P2D --> P2E{用户确认 Plan?}
        P2E -->|修正| P2D
        P2E -->|确认| P3
    end

    subgraph P3["Phase 3 · 全自动执行（无交互）"]
        P3A["创建分支\nfeat/feishu-{uuid前8位}"]
        P3A --> P3B[按 Plan 逐项修改代码\n仿写项目已有模式]
        P3B --> P3C{验证\nnpm build / py_compile}
        P3C -->|失败| P3D[自动修复\n最多 3 轮]
        P3D --> P3C
        P3C -->|成功| P3E[代码 Review\n范围检查 + 质量检查]
        P3E --> P3F[git add 精确文件\ncommit + Feishu-Task trailer]
        P3F --> P3G[git push origin branch\n绝不 push master/main/dev]
    end

    subgraph P4["Phase 4 · 收尾"]
        P4A[complete_task\n自动标记飞书任务完成]
        P4A --> P4B[输出收尾报告\n任务/改动/验证/commit/push 状态]
    end

    P3G --> P4A
    P3C -->|3轮后仍失败| STOP([暂停，报错请人介入])
```

---

## 三、feishu-commit：带链接提交

```mermaid
flowchart TD
    A([用户输入 /feishu-commit task_id]) --> B[提取 UUID\n构建飞书任务链接]
    B --> C[git status / diff / log\n分析改动]
    C --> D[生成 commit message\ntype + 简短描述]
    D --> E[git add 精确文件\n不用 git add -A]
    E --> F["git commit\n───────────────\nfeat: 描述\n\nFeishu-Task: https://...\nCo-Authored-By: Claude..."]
    F --> G([完成])
```

---

## 四、feishu_api.py：底层 API 层

| 命令 | 功能 |
|------|------|
| `check_config` | 检查 `~/.claude/plugins/cache/.../config.json` 是否存在 |
| `save_config <id> <secret>` | 保存凭据，清除 token 缓存 |
| `get_task <task_id>` | 获取 token（文件缓存，有效期内复用）→ 调飞书 API |
| `list_tasks [--completed]` | 列出待办或已完成任务 |
| `complete_task <task_id>` | PATCH `completed_at` 标记完成 |
| `add_comment <task_id> <text>` | POST 添加评论 |

**Token 缓存策略**：有效期内直接读文件，过期前 60s 自动刷新，避免频繁请求。

---

## 关键设计决策

| 设计点 | 说明 |
|--------|------|
| 人工介入只有 2 次 | feishu-dev 中：补问（~5s）+ Plan 确认（~30s） |
| 数据类 bug 强制全链路 | 不能只修前端，必须同时读后端接口代码 |
| commit 绝不 `git add -A` | 精确 add，避免误提交 `.env` 等文件 |
| 凭据与插件文件分离 | 存 `~/.claude/plugins/cache/`，更新插件不丢凭据 |
| 零依赖 | `feishu_api.py` 纯 stdlib，无需 pip install |
