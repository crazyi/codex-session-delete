---
name: cleanup
description: 删除一个 Codex 会话，并生成可撤销备份。也支持恢复、查看备份和按项目批量删除。当用户要求删除会话、或明确要求使用 cleanup 时使用。
---

# cleanup

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

## 按归档状态批量删除

`bulk-delete` 子命令只按 `cwd` / `rollout_path` 的子串匹配，没法直接表达"未归档"或"无项目"。用 sqlite 取出 ID 后循环 `delete` 即可，每条都会在 `~/.codex-session-delete/backups/` 留一份可 `restore` 的 JSON 备份。

### 只删未归档（`archived = 0`）

```bash
cd <插件目录>
DB="$HOME/.codex/state_5.sqlite"

# 预览
sqlite3 -header -column "$DB" \
  "SELECT id, substr(cwd,1,50) AS cwd, substr(title,1,40) AS title
   FROM threads WHERE archived = 0 ORDER BY updated_at_ms DESC;"

# 逐条删除
for sid in $(sqlite3 "$DB" "SELECT id FROM threads WHERE archived = 0"); do
  python3 scripts/codex-session-delete delete "$sid"
done
```

### Codex 侧栏"任务"分组（未归档 + 无项目）

Codex 桌面侧栏的 **"任务"分组** 就是 `git_branch` 和 `git_origin_url` 都为空的会话。删这一组：

```bash
cd <插件目录>
DB="$HOME/.codex/state_5.sqlite"

# 预览
sqlite3 -header -column "$DB" \
  "SELECT id, substr(cwd,1,50) AS cwd, substr(title,1,40) AS title
   FROM threads
   WHERE archived = 0
     AND (git_branch IS NULL OR TRIM(git_branch) = '')
     AND (git_origin_url IS NULL OR TRIM(git_origin_url) = '')
   ORDER BY updated_at_ms DESC;"

# 逐条删除
for sid in $(sqlite3 "$DB" "SELECT id FROM threads
   WHERE archived = 0
     AND (git_branch IS NULL OR TRIM(git_branch) = '')
     AND (git_origin_url IS NULL OR TRIM(git_origin_url) = '')"); do
  python3 scripts/codex-session-delete delete "$sid"
done
```

判定规则：

- `archived = 0` 未归档；`archived = 1` 已归档
- `git_branch` 和 `git_origin_url` 都为空 → 无项目会话（侧栏"任务"分组）
- 想只保留其中一个条件，去掉对应 WHERE 子句即可
