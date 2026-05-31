# DEV.md — 二次开发工作流

## 远程仓库配置

| 远程名称 | 地址 | 用途 |
|---------|------|------|
| `origin` | `git@github.com:bennyworks/new-api.git` | 你的 fork，推送目标 |
| `upstream` | `https://github.com/QuantumNous/new-api.git` | 原项目，拉取上游更新 |

## 分支策略

```
upstream/main  ──●──●──●──●──●──●──  (原项目主分支，不断更新)
                   ↘        ↗
origin/main    ──●──●──●──●──●──●──  (你的 main，紧跟 upstream)
                        ↘
origin/dev     ──────────▲──▲──▲──  (你的开发分支，累积二开改动)
```

核心原则：**用独立分支管理你的二开代码，`main` 分支保持与 upstream 同步。**

## 初始化开发分支

```bash
# 1. 基于当前 main 创建你的开发分支
git checkout -b dev

# 2. 在上面做你的二开修改，然后提交
git add ...
git commit -m "feat: 自定义某某功能"

# 3. 推送到你的 fork
git push -u origin dev
```

所有二开代码都在 `dev` 分支上迭代。

## 跟进 Upstream 更新

当原项目有新版本时：

```bash
# 1. 切到 main，拉取上游更新
git checkout main
git pull upstream main

# 2. 推送到你的 fork（保持 fork 的 main 也是最新的）
git push origin main

# 3. 把上游更新合并到你的开发分支
git checkout dev
git merge main

# 4. 解决冲突（如果有），然后推送
git push origin dev
```

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
| 修改配置文件 | 把敏感配置抽成 `config.local.yml`，加入 `.gitignore` |
| 修改某个模块 | 尽量通过接口/插件机制扩展，而不是直接改原代码 |
| 新增文件/模块 | 放在独立目录下（如 `custom/`），不会和 upstream 冲突 |
| 前端主题/样式 | 用独立主题目录，不走原主题文件 |
| 数据库迁移 | 新增独立迁移文件，不修改上游已有迁移 |

## 日常操作速查

| 操作 | 命令 |
|------|------|
| 同步上游最新代码 | `git checkout main && git pull upstream main && git push origin main` |
| 把上游更新合入二开 | `git checkout dev && git merge main` |
| 查看二开改了哪些 | `git diff main...dev` |
| 查看二开改了哪些文件 | `git diff main...dev --stat` |
| 查看某个文件的改动 | `git diff main...dev -- path/to/file` |
| 查看二开的提交历史 | `git log main..dev` |

## 冲突解决流程

```bash
# 1. 合并时出现冲突
git merge main
# CONFLICT: ...

# 2. 查看冲突文件
git status

# 3. 手动解决冲突，编辑冲突文件
# 搜索 <<<<<<< 标记，保留需要的代码

# 4. 标记已解决
git add <conflicted-file>

# 5. 完成合并
git commit

# 6. 推送
git push origin dev
```

## 完全舍弃二开、重新开始

如果你想把 dev 完全重置为与 upstream 一致，保留之前的二开记录：

```bash
# 备份旧分支
git branch dev-backup dev

# 基于最新 main 重建 dev
git checkout main
git checkout -b dev-new
# 然后手动移植需要的改动
```
