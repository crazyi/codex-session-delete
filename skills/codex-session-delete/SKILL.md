---
name: codex-session-delete
description: 删除一个 Codex 会话，并生成可撤销备份。也支持恢复、查看备份和按项目批量删除。当用户要求删除会话、或明确要求使用 codex-session-delete 时使用。
---

# codex-session-delete

删除一个 Codex 会话，并生成可撤销备份。也支持恢复、查看备份和按项目批量删除。

## 何时使用

用户要求删除会话、或明确要求使用 `codex-session-delete` 时使用本 skill。

## 输入

- `session_id`：要删除的会话 ID，例如 `019ec98d-50af-7a01-a1dd-266343be0236`。

## 流程

> 本 skill 来自 Codex 插件。脚本随插件发布在 `scripts/codex-session-delete`，以相对路径调用即可。

1. 查看备份：`python3 scripts/codex-session-delete list`。
2. 删除会话：`python3 scripts/codex-session-delete delete "<session_id>"`。
3. 读取输出，确认 `session_id`、`undo_token`、`backup_path`。
4. 如失败，检查脚本输出；失败时脚本不会修改数据。
5. 恢复会话：`python3 scripts/codex-session-delete restore "<backup_path>"`。

## 成功标准

- 目标会话不再出现在 `~/.codex/sqlite/state_5.sqlite` 的 `threads` 表。
- 对应 rollout 文件被删除。
- 存在备份文件。
