---
name: videoflow-upgrade
description: 升级 videoflow 到最新版本
trigger: /videoflow-upgrade、/升级videoflow、「升级 videoflow」
---

# videoflow-upgrade

从 GitHub 拉取最新版本，版本比对，备份旧版，失败自动回滚。

## 使用场景

- 用户主动调用 `/videoflow-upgrade` 升级
- 显示版本变化和更新内容

---

## 升级流程

### Step 1：检测安装位置

```bash
if [ -d "$HOME/.claude/skills/video-flow" ]; then
  INSTALL_DIR="$HOME/.claude/skills"
  echo "Install location: $INSTALL_DIR"
else
  echo "ERROR: videoflow not found in ~/.claude/skills/"
  exit 1
fi
```

### Step 2：获取当前版本

读取 `$HOME/.claude/skills/video-flow/../../../videoflow-upgrade/../VERSION` 不可靠，改为直接读：

```bash
CURRENT_VERSION=$(cat "$HOME/.claude/skills/videoflow-upgrade/VERSION" 2>/dev/null || echo "unknown")
echo "Current version: $CURRENT_VERSION"
```

### Step 3：获取远程版本

```bash
REMOTE_VERSION=$(curl -sL https://raw.githubusercontent.com/q17603008535-netizen/videoflow/main/VERSION 2>/dev/null || echo "")
if [ -z "$REMOTE_VERSION" ]; then
  echo "ERROR: Cannot fetch remote version. Check network connection."
  exit 1
fi
echo "Remote version: $REMOTE_VERSION"
```

### Step 4：比较版本

如果 `CURRENT_VERSION` 等于 `REMOTE_VERSION`，输出：

```
videoflow 已是最新版本（v{CURRENT_VERSION}），无需升级。
```

结束。否则继续。

### Step 5：备份当前版本

```bash
BACKUP_DIR="$HOME/.claude/skills/.videoflow-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

for skill in video-flow ip-archive boom-elements hook-anchor zhengwen-router shai-guocheng jiang-gushi shuo-chanpin shuo-guandian videoflow-upgrade; do
  [ -d "$HOME/.claude/skills/$skill" ] && cp -r "$HOME/.claude/skills/$skill" "$BACKUP_DIR/" 2>/dev/null || true
done

echo "Backup created: $BACKUP_DIR"
```

### Step 6：下载最新版本

```bash
TMP_DIR=$(mktemp -d)
git clone --depth 1 https://github.com/q17603008535-netizen/videoflow.git "$TMP_DIR/videoflow"

if [ $? -ne 0 ]; then
  echo "ERROR: Failed to clone repository. Check network and repository access."
  exit 1
fi

echo "Downloaded to: $TMP_DIR/videoflow"
```

### Step 7：替换旧版本

```bash
for skill in video-flow ip-archive boom-elements hook-anchor zhengwen-router shai-guocheng jiang-gushi shuo-chanpin shuo-guandian; do
  rm -rf "$HOME/.claude/skills/$skill"
done

cp -r "$TMP_DIR/videoflow"/video-flow "$HOME/.claude/skills/" 2>/dev/null
cp -r "$TMP_DIR/videoflow"/ip-archive "$HOME/.claude/skills/" 2>/dev/null
cp -r "$TMP_DIR/videoflow"/boom-elements "$HOME/.claude/skills/" 2>/dev/null
cp -r "$TMP_DIR/videoflow"/hook-anchor "$HOME/.claude/skills/" 2>/dev/null
cp -r "$TMP_DIR/videoflow"/zhengwen-router "$HOME/.claude/skills/" 2>/dev/null
cp -r "$TMP_DIR/videoflow"/shai-guocheng "$HOME/.claude/skills/" 2>/dev/null
cp -r "$TMP_DIR/videoflow"/jiang-gushi "$HOME/.claude/skills/" 2>/dev/null
cp -r "$TMP_DIR/videoflow"/shuo-chanpin "$HOME/.claude/skills/" 2>/dev/null
cp -r "$TMP_DIR/videoflow"/shuo-guandian "$HOME/.claude/skills/" 2>/dev/null

if [ $? -ne 0 ]; then
  echo "ERROR: File copy failed. Restoring from backup..."
  for skill in video-flow ip-archive boom-elements hook-anchor zhengwen-router shai-guocheng jiang-gushi shuo-chanpin shuo-guandian; do
    rm -rf "$HOME/.claude/skills/$skill"
    cp -r "$BACKUP_DIR/$skill" "$HOME/.claude/skills/" 2>/dev/null || true
  done
  echo "Restored from backup: $BACKUP_DIR"
  exit 1
fi

# 更新 videoflow-upgrade 本身的 VERSION 记录
cp "$TMP_DIR/videoflow/VERSION" "$HOME/.claude/skills/videoflow-upgrade/VERSION" 2>/dev/null || true

rm -rf "$TMP_DIR"
echo "Upgrade completed"
```

### Step 8：显示更新内容

读取 `$HOME/.claude/skills/video-flow/../README.md`（即 `$HOME/.claude/skills/README.md`，如存在），从中提取版本相关更新说明。

输出格式：

```
videoflow v{REMOTE_VERSION} — 从 v{CURRENT_VERSION} 升级成功！

更新内容：
- [从 README 或 CHANGELOG 提取的更新要点，如无则写「请查看 GitHub 仓库了解更新详情」]

升级完成！备份位置：{BACKUP_DIR}
```

### Step 9：询问是否清理备份

```
备份保留在：{BACKUP_DIR}

是否现在删除备份？（回复「删除」确认，其他回复跳过）
```

如果用户回复「删除」：
```bash
rm -rf "$BACKUP_DIR"
echo "备份已删除。"
```

否则不操作，备份保留。

---

## 错误处理

| 错误 | 处理方式 |
|---|---|
| 网络无法访问 GitHub | 提示用户检查网络，终止 |
| git clone 失败 | 提示错误信息，终止，不修改现有文件 |
| 文件复制失败 | 从备份恢复，提示恢复完成 |
| VERSION 文件不存在 | 当前版本标记为 unknown，仍执行升级 |

---

## 注意事项

- 只支持通过 `~/.claude/skills/` 安装的版本
- videoflow-upgrade 本身不会被替换（保留当前运行中的版本）
- 升级前自动备份所有 skill，失败时自动恢复
- 不需要用户手动操作 git
