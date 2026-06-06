# DEV.md — 二次开发工作流

## 远程仓库配置

| 远程名称 | 地址 | 用途 |
|---------|------|------|
| `origin` | `git@github.com:bennyworks/new-api.git` | 你的 fork，推送目标 |
| `upstream` | `https://github.com/QuantumNous/new-api.git` | 原项目，拉取上游更新 |

## 分支策略

```
upstream tags  ──●──────────●──────────●──  (原项目稳定发布版本)
                     ↘          ↘
origin/dev      ──────▲──▲──────▲──▲──────  (工作台：merge upstream tag + 二开改动)
                         ↘          ↘
origin/main     ──────────●──────────●─────  (生产线：只接受 dev 合并，构建镜像 & 发布)
```

核心原则：
- **所有二开代码和 upstream 同步都在 `dev` 分支上进行**
- **`dev` 是工作台**：在这里 merge upstream tag、解决冲突、写新功能
- **`main` 是生产线**：只接受来自 `dev` 的合并（`git merge --no-ff dev`），不在 `main` 上直接提交或解决冲突
- **冲突隔离**：upstream tag 先合入 `dev`，充分测试后再合入 `main`，避免冲突污染生产环境

## 初始化开发分支

```bash
# 1. 添加 upstream remote（仅首次）
git remote add upstream https://github.com/QuantumNous/new-api.git

# 2. 拉取上游 tags
git fetch upstream --tags

# 3. 基于最新上游 tag 创建 main（生产线）
git checkout main
git merge $(git describe --tags --abbrev=0 upstream/main)
git push origin main

# 4. 基于 main 创建 dev 分支（工作台）
git checkout -b dev main

# 5. 在 dev 上做二开修改，然后提交
git add ...
git commit -m "feat: 自定义某某功能"

# 6. 测试通过后，合入 main 并推送
git checkout main
git merge --no-ff dev
git push origin main dev
```

所有二开代码都在 `dev` 分支上迭代，`main` 只通过 `git merge --no-ff dev` 接收更新。

## 跟进 Upstream 更新

当原项目发布新版本（tag）时，遵循 **先 dev 后 main** 的流程：

```bash
# 1. 拉取上游最新 tags
git fetch upstream --tags

# 2. 查看可用 tag
LATEST_TAG=$(git describe --tags --abbrev=0 upstream/main)
echo "最新 tag: $LATEST_TAG"

# 3. 在 dev 上合并新 tag（冲突在这里解决）
git checkout dev
git merge $LATEST_TAG

# 4. 解决冲突（如果有），然后充分测试
# 编辑冲突文件 → git add . → git commit

# 5. 测试通过后，将 dev 合并到 main（用 --no-ff 保留合并节点）
git checkout main
git merge --no-ff dev

# 6. 推送两条分支
git push origin main dev
```

> **为什么先在 dev 合并？** 冲突只发生在 `dev`（工作台），`main`（生产线）接收的始终是经过测试的干净合并，不会在部署环节遇到意外冲突。如果 `dev` 上合并失败，可以 `git merge --abort` 重来，对 `main` 零影响。

## 使用 Rebase 保持历史整洁

如果 merge 频繁导致历史混乱，可以改用 rebase：

```bash
# 在 dev 分支上
git checkout dev
git rebase main

# 解决冲突后，继续 rebase
git rebase --continue

# ⚠️ 只有你自己的分支可以 force push
git push -f origin dev
```

## 减少合并冲突的最佳实践

| 场景 | 处理方式 |
|------|---------|
| 修改前端页面 | 只改 `web/default/src/` 下的文件，和后端路径天然隔离 |
| 修改配置文件 | 把敏感配置抽成 `config.local.yml`，加入 `.gitignore` |
| 修改某个模块 | 尽量通过接口/插件机制扩展，而不是直接改原代码 |
| 新增文件/模块 | 放在独立目录下（如 `custom/`），不会和 upstream 冲突 |
| 数据库迁移 | 新增独立迁移文件，不修改上游已有迁移 |

## 日常操作速查

| 操作 | 命令 |
|------|------|
| 在 dev 上合入上游 tag | `git checkout dev && git merge $(git describe --tags --abbrev=0 upstream/main)` |
| 将 dev 发布到 main | `git checkout main && git merge --no-ff dev && git push origin main dev` |
| 查看当前跟踪的 tag | `git describe --tags --abbrev=0 main` |
| 查看可用的上游 tag | `git tag --sort=-creatordate \| head -10` |
| 查看二开改了哪些 | `git diff main...dev` |
| 查看二开改了哪些文件 | `git diff main...dev --stat` |
| 查看某个文件的改动 | `git diff main...dev -- path/to/file` |
| 查看二开的提交历史 | `git log main..dev` |

## 冲突解决流程

```bash
# 1. 在 dev 上合并 upstream tag 时出现冲突
git checkout dev
git merge v1.2.3
# CONFLICT: ...

# 2. 查看冲突文件
git status

# 3. 手动解决冲突，编辑冲突文件
# 搜索 <<<<<<< 标记，保留需要的代码

# 4. 标记已解决
git add <conflicted-file>

# 5. 完成合并
git commit

# 6. 测试通过后，合入 main
git checkout main
git merge --no-ff dev
git push origin main dev
```

## 完全舍弃二开、重新开始

如果你想把 dev 完全重置为与上游一致，保留之前的二开记录：

```bash
# 备份旧分支
git branch dev-backup dev

# 基于最新 main 重建 dev
git checkout main
git checkout -b dev-new
# 然后手动移植需要的改动
```
