# commit-and-merge

自动提交当前分支已暂存的文件并合并到目标分支，提交前执行 lint 检查。

## 功能

- 自动 lint 检查
- 提交本地**已暂存**变更
- 合并到指定目标分支
- 推送源分支与目标分支

## 用法

```bash
/commit-and-merge ["提交信息"] [--<目标分支>] [--no-verify]
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

### 可选标志

| 参数 | 说明 | 效果 |
|------|------|------|
| `--no-verify` | 绕过 Git 提交钩子和 Lint 检查 | 跳过 lint + `pre-commit` + `commit-msg` |

### 示例

```bash
/commit-and-merge
# 默认：提交已暂存文件，合并到 develop，message 由 AI 建议后确认

/commit-and-merge --preview
# 合并到 preview

/commit-and-merge "feat: 优化列表加载性能"
# 使用指定 message，合并到 develop

/commit-and-merge --preview 'fix: 修复登录失效问题'
# 使用指定 message，合并到 preview

/commit-and-merge --no-verify
# 跳过 lint 和 git hooks，合并到 develop

/commit-and-merge --preview "fix: 紧急修复" --no-verify
# 跳过所有检查，直接提交并合并到 preview
```

## 工作流程

1. **解析参数** — 提取目标分支和提交信息
2. **检查 Git 状态** — 分类已暂存/未暂存/未跟踪文件，按需引导暂存
3. **Lint 检查** — `npm run lint`（缺少配置、传入 `--no-verify` 时自动跳过；失败时先自动修复，无法修复则跳过）
4. **确认目标分支** — 展示分支信息与 CI 行为，校验分支存在性
5. **预览并提交** — 展示变更摘要，用户传入 message 时直接提交；AI 生成时需用户确认
6. **推送源分支** — 预览 commit 列表后自动 push，首次推送自动设置上游
7. **切换并更新目标分支** — 切换目标分支 → `pull` 拉取最新代码
8. **合并源分支** — `git merge`，冲突时提供手动/AI 辅助/中止选项
9. **推送目标分支并切回源分支** — push 目标分支 → 自动切回源分支并恢复 stash（如有）
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
- 无变更时**默认执行仅合并**，不再询问
- Lint 失败时**先自动修复**，无法修复则自动跳过
- 用户未提供 message 时，AI 分析 diff 生成建议，需用户确认或修改后提交
- 传入 `--no-verify` 时跳过 lint 和 git hooks，汇总报告中会明确标注

## 需手动确认的关键节点

| 阶段 | 触发条件 | 操作 |
|------|---------|------|
| **前置检查** | 存在进行中的 merge/rebase 或 Detached HEAD | 选择是否继续 |
| **Git 状态** | 无已暂存文件 / 多种变更混合 | 选择暂存策略（`add -A` / `add -u` / 手动指定 / 跳过） |
| **提交信息** | 未通过参数传入 message | 确认或修改 AI 生成的 message |
| **合并冲突** | Pull/Merge 时发生冲突 | 手动解决 / AI 辅助 / 中止合并 |
| **破坏性操作** | 执行 `restore`、`reset --hard` 等 | 二次确认 |
| **Stash 恢复** | 切回源分支后恢复 stash 冲突 | 手动解决或保留 stash |

> 除上述节点外，其余步骤（lint、push、merge、切换分支等）均为**自动执行**，无需用户干预。

## 正常输出报告示例

执行成功后会输出完整的汇总报告，示例如下：

```
✅ 提交 & 合并完成！
- 源分支：feature/user-login
- 目标分支：develop
- 源分支 = 目标分支：否
- Commit Message："feat(login): 新增扫码登录功能"
- Lint 检查：通过 ✅
- 文件变更：5 个文件（仅已暂存）
- Commit 状态：已提交 ✅
- 源分支推送：成功 ✅
- 合并状态：成功 ✅
- 合并至：develop
- 目标分支推送：是
- 当前分支：feature/user-login
- Stash 恢复：无 stash ✅
- CI 状态：构建已触发（npm run devbuild）
```

当源分支与目标分支相同时：

```
✅ 提交 & 合并完成！
- 源分支：develop
- 目标分支：develop
- 源分支 = 目标分支：是（已跳过合并）
- Commit Message："fix: 修复构建报错"
- Lint 检查：通过 ✅
- 文件变更：2 个文件（仅已暂存）
- Commit 状态：已提交 ✅
- 源分支推送：成功 ✅
- 合并状态：跳过（源=目标）⏭️
- 当前分支：develop
- CI 状态：构建已触发（npm run devbuild）
```

当使用 `--no-verify` 跳过检查时：

```
✅ 提交 & 合并完成！
- 源分支：hotfix/urgent-patch
- 目标分支：master
- 源分支 = 目标分支：否
- Commit Message："fix: 紧急修复线上 bug"
- Lint 检查：跳过（--no-verify）⏭️
- 文件变更：1 个文件（仅已暂存）
- Commit 状态：已提交（--no-verify）✅
- 源分支推送：成功 ✅
- 合并状态：成功 ✅
- 合并至：master
- 目标分支推送：是
- 当前分支：hotfix/urgent-patch
- Stash 恢复：无 stash ✅
- CI 状态：构建已触发（npm run probuild，手动）
```

## FAQ

### 为什么我修改了文件但提示“工作区干净”？

工具**只提交已暂存（staged）的文件**，不会自动执行 `git add`。请在执行前手动暂存目标文件：

```bash
git add src/components/Login.vue
git add .
```

然后再运行 `/commit-and-merge`。

### 工作区有未暂存或未跟踪的文件，会怎么处理？

工具会自动检测并提示你选择暂存策略：

| 选项 | 说明 |
|------|------|
| 仅提交已暂存（默认） | 未暂存和未跟踪的文件保留在工作区，不影响本次提交 |
| `git add -A` | 暂存全部变更（含未跟踪文件，注意可能包含 IDE 配置等） |
| `git add -u` | 仅暂存已跟踪文件的修改，**不包含**未跟踪的新文件 |
| 跳过 commit | 不提交任何内容，直接进入合并流程 |

### 发生合并冲突怎么办？

工具会暂停并展示冲突文件列表，提供四种处理方式：

| 选项 | 说明 |
|------|------|
| **手动解决** | 你自己编辑冲突文件，完成后点击「✅ 已完成，继续」 |
| **AI 辅助** | AI 分析冲突并给出建议，确认后自动合并（仍需你最终确认） |
| **中止合并** | 执行 `git merge --abort`，安全回退到合并前状态 |
| **查看冲突** | 先展示 `git diff` 详情，再决定处理方式 |

> AI 辅助合并遵循保守原则：保留双方有意义逻辑，不会静默删除代码，合并后会输出修改摘要。

### Lint 检查失败会中断流程吗？

不会。Lint 失败时的处理策略：

1. 先自动执行 `npm run lint --fix` 尝试修复
2. 若仍无法修复，**自动跳过**并在报告中标注，不会中断提交流程
3. 若未配置 lint 脚本或 `node_modules` 未安装，也会自动跳过

### 当前分支和目标分支相同时会发生什么？

正常执行提交和推送步骤，但**自动跳过合并**（无需自己合并自己），报告中标注「源分支与目标分支相同，已跳过合并」。

### `--no-verify` 会影响什么？

传入 `--no-verify` 后：

- **跳过 Lint 检查**（第 3 步）
- **跳过 Git 钩子**（`pre-commit` 和 `commit-msg`）
- 汇总报告中会明确标注「Lint 检查：跳过（--no-verify）」和「Commit 状态：已提交（--no-verify）」

> 建议在紧急修复、钩子卡住或已知 lint 误报时使用，正常情况下不建议跳过检查。

### 目标分支受保护（Protected Branch）怎么办？

工具检测到分支保护时会自动终止流程，并提示你通过 **Pull Request** 合并，而不会强制推送。

### 推送失败（non-fast-forward）怎么处理？

工具会自动执行 `git fetch` + `git rebase`，将本地提交变基到远程最新节点后重新推送，无需手动干预。