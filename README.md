# codex-session-delete

删除 Codex 会话，并生成可撤销备份。支持删除、恢复、撤销、备份列表查看和按项目批量删除。

## 适用场景

- 你需要在 Codex 中删除某个会话，但原版没有直接删除能力。
- 你想保留可恢复的备份，避免误删。
- 你希望用统一 CLI 处理多库场景。

## 安装

### 方式 1：通过 Codex marketplace 安装（推荐）

本仓库本身是一个 Codex 插件，可通过个人 marketplace 安装。

#### 一次性配置

```bash
# 1. clone 仓库（路径可自定义）
git clone https://github.com/crazyi/codex-session-delete.git ~/Desktop/1111/codex-session-delete

# 2. 在 ~/plugins/ 下放一个软链接，指向仓库（开发时改仓库内容，插件实时生效）
mkdir -p ~/plugins
ln -sfn ~/Desktop/1111/codex-session-delete ~/plugins/codex-session-delete

# 3. 写个人 marketplace 入口（首次安装时执行一次）
mkdir -p ~/.agents/plugins
```

`~/.agents/plugins/marketplace.json`：

```json
{
  "name": "personal",
  "interface": {
    "displayName": "Personal"
  },
  "plugins": [
    {
      "name": "codex-session-delete",
      "source": {
        "source": "local",
        "path": "./plugins/codex-session-delete"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Productivity"
    }
  ]
}
```

> `source.path` 是**相对个人 marketplace 根（`$HOME`）的路径**，必须以 `./` 开头。Codex 运行时只接受相对路径，硬写绝对路径会报 "plugin not found"。
> 软链接让仓库位置变化时，marketplace 入口无需修改。

#### 安装与升级

```bash
# 安装
codex plugin add codex-session-delete@personal

# 升级（每次改了代码后，让 Codex 重新读取）
codex plugin upgrade codex-session-delete@personal
```

`personal` 是默认 marketplace 名称，由 `~/.agents/plugins/marketplace.json` 隐式发现，不需要额外 `codex plugin marketplace add`。

### 方式 2：手动安装为个人 skill（不推荐，无法跟随 marketplace 升级）

```bash
git clone https://github.com/crazyi/codex-session-delete.git \
  "${CODEX_HOME:-$HOME/.codex}/skills/codex-session-delete"
```

## 快速开始

> 插件安装后，skill 内的命令以相对路径 `python3 scripts/codex-session-delete ...` 形式给出，Codex 会从插件根目录解析。

```bash
python3 scripts/codex-session-delete --help
```

## 常用命令

### 1. 验证会话是否存在

```bash
python3 scripts/codex-session-delete verify <session_id>
```

### 2. 删除会话

```bash
python3 scripts/codex-session-delete delete <session_id>
```

### 3. 查看备份列表

```bash
python3 scripts/codex-session-delete list
```

### 4. 撤销删除

```bash
python3 scripts/codex-session-delete undo <undo_token>
```

或直接恢复：

```bash
python3 scripts/codex-session-delete restore <backup_path>
```

### 5. 批量删除项目会话

```bash
# 预览项目相关会话
python3 scripts/codex-session-delete bulk-delete "<project>" --dry-run

# 执行删除
python3 scripts/codex-session-delete bulk-delete "<project>"

# 跳过指定会话
python3 scripts/codex-session-delete bulk-delete "<project>" --skip <session_id>
```

## 自定义备份目录

```bash
python3 scripts/codex-session-delete \
  delete <session_id> --backup-dir /tmp/my-backups
```

## 成功标准

- 目标会话不再出现在 `~/.codex/sqlite/state_5.sqlite` 的 `threads` 表。
- 对应 rollout 文件被删除。
- 存在备份文件，可用于恢复。

## 注意事项

- 删除前会先备份；若删除失败会尝试回滚。
- 如发现数据异常，优先使用 `undo <undo_token>` 恢复。
- 可同时查看 `~/.codex-session-delete/backups/` 了解历史备份。

## 故障排查

### 1. `verify` 返回 `NOT_FOUND`

- 确认 `session_id` 是否拼写正确。
- 检查 `~/.codex/sqlite/state_5.sqlite` 和 `~/.codex/state_5.sqlite` 是否存在。
- 该会话可能已被删除，可执行 `list` 查看历史备份。

### 2. `delete` 提示“Session ... not found”

- 先执行 `verify` 确认会话是否仍存在。
- 确认你使用的是正确的 `session_id`。
- 如果存在多个 state 库，当前脚本会自动扫描多个候选库；如果仍找不到，说明该会话可能已不在本地。

### 3. 删除失败或中断

- 脚本在数据库删除阶段失败时会回滚已修改的 state 库。
- rollout 文件删除或 memories 清理失败不会影响数据库事务，但请检查脚本输出。
- 如果数据库已回滚但备份文件已生成，可直接使用该备份。

### 4. 恢复失败

- 确认备份文件路径正确，且未手动修改 JSON。
- 如果提示已存在，脚本使用 `INSERT OR IGNORE`，不会覆盖已有数据。
- 如需强制恢复，可手动备份当前数据后再操作。

### 5. rollout 文件删除失败

- 可能是文件被 Codex 占用，关闭 Codex 后重试。
- 检查文件路径是否存在，以及当前用户是否有写权限。

### 6. 权限问题

- 确保当前用户对 `~/.codex`、`~/.codex/sqlite` 和 `~/.codex-session-delete/backups` 有读写权限。
- macOS/Linux 可使用 `ls -ld` 查看目录权限。

### 7. 多数据库场景

- 如果同时存在 `~/.codex/sqlite/state_5.sqlite` 和 `~/.codex/state_5.sqlite`，脚本会扫描并处理所有包含 `threads` 表的库。
- 如希望只操作某个库，可修改脚本中的 `DEFAULT_STATE_DB_CANDIDATES`。

### 8. 需要人工检查

- 查看备份目录：`ls -la ~/.codex-session-delete/backups`
- 查看当前 state 库：`sqlite3 ~/.codex/sqlite/state_5.sqlite "SELECT id, title FROM threads WHERE id = '<session_id>';"`
