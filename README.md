# commit-and-merge

> 自动化 Git 工作流：lint 检查 → 提交已暂存变更 → 合并到目标分支。
> Automated git workflow: lint → commit staged changes → merge into target branch.

## 功能

- 自动 lint 检查（可自定义命令）
- 提交本地**已暂存**变更
- 合并到指定目标分支
- 推送源分支与目标分支
- 支持 Dry-Run 模拟执行
- 并发安全（自动锁）

## 命令格式

```
/commit-and-merge [--<目标分支>] ["提交信息"] [--no-verify] [--dry-run] [--lint-cmd "<cmd>"]
```

参数位置无关，可任意组合。

### 保留标志（**不作为分支名**）

| 参数 | 说明 |
|------|------|
| `--no-verify` | 跳过 lint + Git 钩子 |
| `--dry-run` | 模拟执行，第 5 步和第 8 步前停下 |
| `--lint-cmd "<cmd>"` | 自定义 lint 命令（覆盖 `npm run lint`） |

### 常用分支

| 参数 | 目标分支 | 环境 | CI 触发 |
|------|---------|------|--------|
| `--develop` | develop | 测试 | `npm run devbuild` |
| `--preview` | preview | 预发 | `npm run prebuild` |
| `--master` | master | 生产 | `npm run probuild`（手动） |

> 不指定时默认 `--develop`。自定义分支名（如 `--feature/login`）不触发预定义 CI。

## 用法

```bash
# 默认：合并到 develop，message 由 AI 建议后确认
/commit-and-merge

# 合并到 preview
/commit-and-merge --preview

# 使用指定 message
/commit-and-merge "feat: 优化列表加载性能"

# 跳过检查合并到 preview
/commit-and-merge --preview --no-verify

# Dry-Run：模拟执行（不真实写 git）
/commit-and-merge --preview --dry-run

# pnpm 项目（自定义 lint）
/commit-and-merge --lint-cmd "pnpm lint" --develop

# 完整组合
/commit-and-merge --preview "fix: 修复 bug" --no-verify --lint-cmd "pnpm lint:fix"
```

## 工作流程（10 步）

1. **解析参数** — 保留标志白名单 + 分支名 + 提交信息
2. **检查 Git 状态** — 分类已暂存/未暂存/未跟踪，按需引导暂存；无变更默认仅合并
3. **Lint 检查** — `$LINT_CMD`（默认 `npm run lint`），失败时先自动修复
4. **确认目标分支** — 展示分支信息与 CI 行为
5. **预览并提交** — Dry-Run 检查点 + 格式校验 + AI 建议 message
6. **推送源分支** — 指数退避 3 次重试，处理 `non-fast-forward`
7. **切换并更新目标分支** — 自动 stash 保护 + 拉取最新
8. **合并源分支** — Dry-Run 检查点 + 冲突处理（手动 / AI 辅助 / 中止）
9. **推送目标分支并切回源分支** — Stash 恢复（区分自动 / 用户）
10. **输出汇总报告** — 中英双语 + emoji 状态

> 完整规范、异常处理、安全约束见 [SKILL.md](./SKILL.md)。

## 前置条件

- 当前目录为 Git 仓库
- 已配置 `origin` 远程仓库（多 remote 时需指定优先项）
- 无进行中的 merge / rebase
- 非 Detached HEAD 状态
- 非浅克隆（若是，先 `git fetch --unshallow`）

## 注意事项

- **仅提交已暂存文件**，不自动执行 `git add`
- 源分支 = 目标分支时，自动跳过合并
- 目标分支不存在时提示用户并终止
- 分支保护时提示通过 Pull Request 合并
- 破坏性操作（`restore`、`reset --hard`）需二次确认
- 无变更时**默认执行仅合并**，不再询问
- Lint 失败时**先自动修复**，无法修复则跳过
- `--no-verify` 跳过检查时，汇总报告明确标注
- Dry-Run 模式下任何 git 写操作前必须等待用户确认
- 自动 stash 与用户 stash 区分处理，避免误恢复

## FAQ

### 为什么我修改了文件但提示「工作区干净」？

工具**只提交已暂存（staged）的文件**，不会自动执行 `git add`。请在执行前手动暂存：

```bash
git add src/components/Login.vue
git add .
```

然后再运行 `/commit-and-merge`。

### 工作区有未暂存或未跟踪的文件，会怎么处理？

工具会询问暂存策略：

| 选项 | 说明 |
|------|------|
| 仅提交已暂存（默认） | 未暂存和未跟踪的文件保留在工作区 |
| `git add -A` | 暂存全部变更（含未跟踪文件） |
| `git add -u` | 仅暂存已跟踪文件的修改 |
| 跳过 commit | 直接进入合并流程 |
| 高级操作 | 精细控制 / stash / 危险操作（需确认） |

### 发生合并冲突怎么办？

工具会暂停并展示冲突文件列表，提供四种处理方式：

| 选项 | 说明 |
|------|------|
| **手动解决** | 你自己编辑冲突文件，完成后点击「✅ 已完成，继续」 |
| **AI 辅助** | AI 分析冲突并给出建议，确认后自动合并（仍需最终确认） |
| **中止合并** | `git merge --abort`，安全回退到合并前状态 |
| **查看冲突** | 先展示 `git diff` 详情，再决定处理方式 |

> AI 辅助合并遵循保守原则：保留双方有意义逻辑、不会静默删除代码、核心配置文件降级为手动。

### Lint 检查失败会中断流程吗？

不会。失败处理策略：

1. 先自动执行 `$LINT_CMD --fix` 尝试修复
2. 若仍无法修复，**自动跳过**并在报告中标注
3. 若未配置 lint 脚本或 `node_modules` 未安装，也会自动跳过

### 当前分支和目标分支相同时会发生什么？

正常执行提交和推送，**自动跳过合并**（无需自己合并自己）。

### `--no-verify` 会影响什么？

- 跳过 Lint 检查（第 3 步）
- 跳过 Git 钩子（`pre-commit` 和 `commit-msg`）
- 汇总报告中明确标注「跳过（--no-verify）」

> 建议在紧急修复、钩子卡住或已知 lint 误报时使用。

### 目标分支受保护（Protected Branch）怎么办？

工具检测到分支保护时会**自动终止**流程，并提示通过 **Pull Request** 合并。

### 推送失败（non-fast-forward）怎么处理？

自动执行 `git fetch` + `git rebase`，变基到远程最新节点后重新推送。**网络问题**最多重试 3 次（指数退避 5s/10s/20s）。

### Dry-Run 模式有什么用？

完整走完前置检查 + 参数解析 + Git 状态扫描，在第 5 步（commit）和第 8 步（merge）前停下，输出**将要执行**的 git 命令清单，由用户确认后再真实执行。适合：

- 不熟悉 Skill 行为时先试运行
- 复杂合并前预演
- 教学/演示场景

### 自定义 lint 命令怎么用？

使用 `--lint-cmd "<cmd>"` 覆盖默认的 `npm run lint`：

```bash
# pnpm 项目
/commit-and-merge --lint-cmd "pnpm lint"

/commit-and-merge --lint-cmd "pnpm lint" --develop

# monorepo 指定子包
/commit-and-merge --lint-cmd "npm run lint -w @scope/pkg" --develop

# yarn 项目
/commit-and-merge --lint-cmd "yarn lint" --preview
```

> 自定义命令执行失败时，会自动尝试追加 `--fix` 参数再试。

### 浅克隆（shallow clone）仓库能用吗？

**能用但有限制**。`git pull` 和 `rebase` 在浅克隆上可能失败。建议先执行：

```bash
git fetch --unshallow
```

> Skill 会在第 2 步检测浅克隆并提示。

### GPG 签名提交支持吗？

支持。若项目启用了 `commit.gpgsign=true`，Skill 会校验 GPG 配置可用性，签名失败时给出明确提示。

### 多远程仓库（fork 工作流）怎么用？

Skill 会自动检测多 remote（如同时存在 `origin` 和 `upstream`），在以下节点询问优先使用哪个：

- 第 4 步：目标分支 fetch 来源
- 第 6 / 9 步：push 目标 remote

### Dry-Run 模式下能切换为真实执行吗？

可以。Dry-Run 模式在第 5 步和第 8 步前停下，提示「执行」或「取消」。输入「执行」后该步骤会真实执行，后续步骤仍受 Dry-Run 标记保护。

### 与 SKILL.md 冲突时以哪个为准？

**以 [SKILL.md](./SKILL.md) 为准**。本 README 是精简版，仅保留简介、命令格式和 FAQ。
