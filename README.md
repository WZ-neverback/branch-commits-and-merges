# commit-and-merge

自动提交当前分支已暂存的文件并合并到目标分支，提交前执行 lint 检查。

## 功能

- 自动 lint 检查
- 提交本地**已暂存**变更
- 合并到指定目标分支
- 推送源分支与目标分支

## 用法

```bash
/commit-and-merge ["提交信息"] [--<目标分支>]
```

### 常用分支（快捷参数）

| 参数 | 目标分支 | 环境 | CI 触发 |
|------|---------|------|--------|
| `--develop` | develop | 测试环境 | `npm run devbuild` |
| `--preview` | preview | 预发环境 | `npm run prebuild` |
| `--master` | master | 生产环境 | `npm run probuild`（手动） |

> 不指定分支时，默认使用 `--develop`。

### 自定义分支

支持任意合法分支名（如 `--feature/login`、`--release/v1.0`）：

```bash
/commit-and-merge "feat: 登录改版" --feature/login
```

自定义分支不触发预定义 CI 行为。

### 示例

```bash
/commit-and-merge
# 默认：提交已暂存文件，合并到 develop，message 由 AI 建议后确认

/commit-and-merge --preview
# 合并到 preview

/commit-and-merge "feat: 优化列表加载性能"
# 使用指定 message，合并到 develop

/commit-and-merge --master 'fix: 修复登录失效问题'
# 使用指定 message，合并到 master
```

## 工作流程

1. **解析参数** — 提取目标分支和提交信息
2. **前置检查** — Git 仓库、origin 配置、合并/rebase 状态、游离 HEAD、node_modules
3. **检查 Git 状态** — 分类已暂存/未暂存/未跟踪文件，按需引导暂存
4. **Lint 检查** — `npm run lint`（缺少配置时自动跳过，不中断流程）
5. **确认目标分支** — 展示分支信息并确认
6. **预览并提交** — 展示变更摘要，确认/修改 message，执行 `git commit`
7. **推送源分支** — 首次推送自动设置上游
8. **合并到目标分支** — 切换目标分支 → `pull` → `merge` → `push`
9. **切回原分支** — 自动恢复 stash（如有）
10. **输出汇总报告** — 完整记录提交、推送、合并状态

## 前置条件

- 当前目录为 Git 仓库
- 已配置 `origin` 远程仓库
- 无进行中的合并或 rebase
- 非 Detached HEAD 状态

## 注意事项

- **仅提交已暂存文件**，不再自动执行 `git add`
- 源分支与目标分支相同时，自动跳过合并步骤
- 目标分支不存在时提示用户并终止
- 检测到分支保护（如 `protected branch`）时，终止流程并提示通过 Pull Request 合并
- 任何破坏性操作（`restore`、`reset --hard` 等）均需二次确认
