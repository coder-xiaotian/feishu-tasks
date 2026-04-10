# claude-feishu-tasks

Claude Code 插件：在对话中直接拉取飞书（Lark）任务并在代码库中实现。

**零依赖**：仅需系统自带的 Python 3，无需 `npm install` 或任何第三方库。

## 功能

包含三个独立 skill，按需选用：

| Skill | 触发方式 | 适用场景 |
|-------|----------|----------|
| `feishu-task` | 直接说飞书任务 ID / URL | 拉取任务详情 + 实现代码，按需评论/完成 |
| `feishu-dev` | `/feishu-dev <task_id>` | 全自动开发：拉取→补问→Plan→实现→验证→commit→push，人工只需确认 2 次 |
| `feishu-commit` | `/feishu-commit <task_id>` | 普通 commit，自动在 message 末尾注入飞书任务链接 |

## 安装

进入claude后，执行下面两个命令
```bash
/plugin marketplace add coder-xiaotian/feishu-tasks
/plugin install feishu-tasks
```

安装后重启 Claude Code，首次使用时会自动引导配置飞书凭据。

## 前置条件

- Python 3（macOS / Linux 系统自带）
- 飞书自建应用（见下方配置步骤）

## 飞书应用权限配置

1. 在[飞书开发者后台](https://open.feishu.cn/)创建自建应用，申请以下权限并发布版本（需企业管理员审批）：

    | 权限 | 说明 |
    |------|------|
    | `task:task:read` | 读取任务详情 |
    | `task:task:readonly` | 只读访问任务 |
    | `task:task:writeonly` | 更新任务状态（完成） |
    | `task:comment:write` | 添加任务评论 |
    | `task:attachment:read` | 读取任务附件 |
    | `task:tasklist:write` | 操作任务列表 |

    凭据首次配置后保存在 `~/.claude/plugins/cache/coder-xiaotian/feishu-tasks/config.json`，与插件文件分离，**更新插件不会丢失凭据**。

2. 使用下面请求将应用添加进你的任务清单
    ```shell
    curl -i -X POST 'https://open.feishu.cn/open-apis/task/v2/tasklists/:tasklist_guid/add_members?user_id_type=open_id' \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer <user_access_token>' \
    -d '{
        "members": [
            {
                "id": "<app_id>",
                "role": "editor",
                "type": "app"
            }
        ]
    }'
    ```
    > 把:tasklist_guid（任务清单）、<user_access_token>（用户token）、<app_id>（应用App ID0）分别替换成自己的

## 使用示例

```
# feishu-task：拉取任务并实现（UUID 或完整 URL 均可）
帮我完成飞书任务 6940dcc0-63f6-446c-a601-ec912923f243
处理这个飞书任务：https://applink.feishu.cn/client/todo/detail?guid=6940dcc0-63f6-446c-a601-ec912923f243

# feishu-task：列出 / 关闭任务
列出我的飞书任务
关闭飞书任务 6940dcc0-63f6-446c-a601-ec912923f243

# feishu-dev：全自动开发流程（推荐）
/feishu-dev 6940dcc0-63f6-446c-a601-ec912923f243

# feishu-commit：提交代码并关联飞书任务
/feishu-commit 6940dcc0-63f6-446c-a601-ec912923f243
```

## 脚本直接使用

```bash
CLAUDE_PLUGIN_ROOT=/path/to/plugin

# 检查凭据
python3 skills/feishu-task/feishu_api.py check_config

# 保存凭据
python3 skills/feishu-task/feishu_api.py save_config cli_xxx your_secret

# 获取任务
python3 skills/feishu-task/feishu_api.py get_task 6940dcc0-63f6-446c-a601-ec912923f243

# 列出任务
python3 skills/feishu-task/feishu_api.py list_tasks
python3 skills/feishu-task/feishu_api.py list_tasks --completed

# 完成任务
python3 skills/feishu-task/feishu_api.py complete_task 6940dcc0-63f6-446c-a601-ec912923f243

# 添加评论
python3 skills/feishu-task/feishu_api.py add_comment 6940dcc0-63f6-446c-a601-ec912923f243 "实现完成"
```