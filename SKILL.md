---
name: commit-and-merge
description: 自动提交当前分支已暂存的文件并合并到目标分支（默认 develop，可通过 --preview/--master 指定），提交前执行 lint 检查。适用于用户需要提交代码、合并分支、部署到测试/预发/生产环境，或输入 "/commit-and-merge" 时。
---

# 提交并合并

> 自动化 lint 检查、提交本地**已暂存**变更并合并到目标分支的完整工作流。

## 命令格式

```
/commit-and-merge [--<目标分支>] ["提交信息"] [--no-verify] [--dry-run]
```

参数**位置无关**，可任意组合与调换。

### 保留标志白名单

下列以 `--` 开头的参数**不作为分支名**，而是 Skill 自身的功能开关：

| 参数 | 说明 | 默认行为 | 必填 |
|------|------|---------|------|
| `--no-verify` | 跳过 Lint 检查 + Git 钩子（`pre-commit` / `commit-msg`） | 执行 lint + hooks | 否 |
| `--dry-run` | 模拟执行：走完前置检查 + 解析 + 状态扫描，在第 5 步和第 8 步前停下，输出**将要执行**的 git 命令清单 | 真实执行 | 否 |

> ⚠️ **保留标志优先级最高**：解析时先扫描白名单，再识别分支名。避免把 `--version`、`--help` 等误识别为分支名。

### 常用快捷分支

| 参数 | 目标分支 | 环境 | CI 触发 | 必填 |
|------|---------|------|---------|------|
| `--develop` | develop | 测试 | `npm run devbuild` | 否 |
| `--preview` | preview | 预发 | `npm run prebuild` | 否 |
| `--master` | master | 生产 | `npm run probuild`（手动） | 否 |

> 不指定时默认 `--develop`。

### 自定义分支

支持任意合法 git 分支名（含 `/`、`-`、`_`、`.`），例如 `--feature/login`、`--release/v1.0`。

> 自定义分支**不触发预定义 CI 行为**，由项目 CI 配置决定。

### 使用示例

```bash
# 默认：合并到 develop，message 由 AI 建议后确认
/commit-and-merge

# 合并到 preview
/commit-and-merge --preview

# 使用指定 message 合并到 develop
/commit-and-merge "feat: 优化列表加载性能"

# 指定 message 合并到 preview
/commit-and-merge --preview 'fix: 修复登录失效问题'

# 跳过所有检查合并到 develop
/commit-and-merge --no-verify

# 紧急修复合并到 master 并跳过检查
/commit-and-merge --master "fix: 紧急修复" --no-verify

# 仅合并（无变更场景，默认行为）
/commit-and-merge --preview
# → 检测到工作区干净，自动进入仅合并流程

# Dry-Run：完整走前置检查 + 状态扫描，输出命令清单后停下
/commit-and-merge --preview --dry-run
```

---

## 自检清单

每次执行前**逐项检查**：

- [ ] 当前是 Git 仓库（`git rev-parse --is-inside-work-tree`）
- [ ] 远程 `origin` 已配置（多 remote 时询问优先使用哪个）
- [ ] 无进行中的 merge / rebase
- [ ] 非 Detached HEAD
- [ ] 工作区有 staged 文件 **或** 用户确认仅合并
- [ ] 并发锁可用（`.git/commit-and-merge.lock`）
- [ ] 非浅克隆（若是，警告并提供解决方案）
- [ ] GPG 签名配置（若启用，校验签名可用）

---

## 前置检查

| 检查项 | 验证方式 | 不满足时行为 |
|--------|---------|-------------|
| Git 仓库 | `git rev-parse --is-inside-work-tree` | 终止 |
| `origin` 远程 | `git remote get-url origin` | 终止：提示 `git remote add origin <url>` |
| 多远程处理 | 存在 `upstream` / `fork` 等多 remote | 询问优先使用哪个 remote 推送 |
| 无 merge/rebase | 检查 `MERGE_HEAD` / `rebase-merge` 目录 | 暂停：等待用户处理 |
| 非 Detached HEAD | `git symbolic-ref HEAD` | 警告 + 询问是否继续 |
| `node_modules` 存在 | 检查目录 | 标记 `SKIP_LINT` |
| 浅克隆 | `git rev-parse --is-shallow-repository` | 警告：`fetch --unshallow` 后再继续 |
| GPG 签名 | `git config --get commit.gpgsign` | 启用时校验 `gpg --list-secret-keys` |
| 并发锁 | 检查 `~/.git/commit-and-merge.lock` 是否存在 | 提示已有实例运行，询问是否继续 |

---

## 状态变量

> **统一命名规范**：所有布尔状态变量采用 `SCREAMING_SNAKE_CASE`，下划线分隔。

| 变量 | 含义 | 设置时机 |
|------|------|---------|
| `SOURCE_BRANCH` | 当前所在分支 | 第 2 步 |
| `TARGET_BRANCH` | 目标合并分支 | 第 1 步 |
| `NO_VERIFY` | 是否跳过 lint + hooks | 第 1 步 |
| `DRY_RUN` | 是否模拟执行 | 第 1 步 |
| `SKIP_LINT` | 是否跳过 lint | 前置检查 / `NO_VERIFY` / `MERGE_ONLY` |
| `SAME_BRANCH` | 源 = 目标 | 第 2 步 |
| `MERGE_ONLY` | 仅合并，无 commit | 第 2 步子流程 A |
| `AUTO_STASH_CREATED` | 自动创建的 stash | 第 7 步（切换失败保护） |
| `USER_STASH_EXISTS` | 用户在第 2 步手动 stash | 第 2 步子流程 C/D 选项 5 |
| `STAGED_FILES` | 已暂存文件列表 | 第 2 步 |
| `UNSTAGED_FILES` | 未暂存文件列表 | 第 2 步 |
| `UNTRACKED_FILES` | 未跟踪文件列表 | 第 2 步 |
| `COMMIT_MESSAGE` | 最终提交信息 | 第 5 步 |
| `COMMIT_HASH` | 提交后的 SHA | 第 5 步 |
| `PUSH_RETRY_COUNT` | Push 重试计数 | 第 6 步 |
| `CONFLICT_FILES` | 冲突文件列表 | 第 8 步 |
| `CONFLICT_RESOLVED_BY` | 冲突处理方式（manual / ai / abort） | 第 8 步 |

---

## 工作流程

按以下步骤**顺序执行**。

### 第 1 步：解析参数

**解析顺序**（**严格按此优先级**）：

1. **保留标志白名单** → 扫描 `--no-verify`、`--dry-run`
2. **快捷分支** → 扫描 `--develop` / `--preview` / `--master`
3. **自定义分支** → 其他 `--xxx`（必须以 `--` 开头且非空）
4. **提交信息** → 成对 `"..."` 或 `'...'`
5. **未命中** → `TARGET_BRANCH=develop`，提示用户确认 message

**校验规则**：

- ✅ 仅允许一个分支参数（`--no-verify` 可与分支共存）
- ✅ 分支名不含 `~` `^` `:` `?` `*` `[` `\` 等保留字符
- ✅ 提交信息 subject ≤ 72 字符
- ✅ 提交信息符合 Conventional Commits（`feat` / `fix` / `chore` / `docs` / `refactor` / `test` / `perf` / `build` / `ci` / `style` / `revert`）时优先；不符合时**提示但不阻断**
- ❌ 多分支参数 → 终止：「参数冲突」
- ❌ 非法分支名 → 终止
- ❌ 提交信息包含未转义引号 → 警告并自动转义

**Dry-Run 模式特殊处理**：

```
[DRY-RUN] 解析结果：
- 目标分支：preview
- 提交信息：（将由 AI 生成）
- --no-verify：false
- 模式：DRY-RUN（第 5 / 8 步前停下）
```

### 第 2 步：检查 Git 状态

```bash
git status
git branch --show-current
git diff --stat --cached
git diff --stat
git ls-files --others --exclude-standard
git rev-parse --is-shallow-repository    # 浅克隆检测
git submodule status                       # 子模块状态
```

**记录**：`SOURCE_BRANCH`、三类文件列表、浅克隆状态、子模块状态。

#### 源分支 = 目标分支

- ✅ **相同** → 标记 `SAME_BRANCH=true`，仍走提交+推送（若需），跳过第 7~9 步
- ❌ **不同** → 继续

#### 子流程 A：完全无变更（工作区干净）

- 告知用户工作区干净
- 自动标记 `MERGE_ONLY=true`，跳过第 3~5 步
- **不再询问**（README 与 SKILL 一致）
- 直接进入第 6 步（推送源分支若有未推送 commit）或第 7 步

#### 子流程 B：仅 `STAGED_FILES` 有内容

- 直接进入第 3 步

#### 子流程 C/D：存在未暂存或未跟踪文件

询问暂存策略：

| # | 选项 | 操作 | 备注 |
|---|------|------|------|
| 1 | 仅提交已暂存 | 无操作 | **默认** |
| 2 | `git add -A` | 暂存全部 | 含未跟踪文件 |
| 3 | `git add -u` | 暂存已跟踪 | 排除未跟踪 |
| 4 | 跳过 commit | 仅合并 | 标记 `MERGE_ONLY=true` |
| 5 | 高级操作 | 见下 | — |

**选项 5 高级操作**：

- `git diff` / `git diff --cached` — 查看差异
- `git add -p` / `git restore --staged <file>` — 精细控制
- `git stash push -u` / `git stash pop` — Stash 工作区（**标记 `USER_STASH_EXISTS=true`**）
- `git restore <file>` / `git checkout -- <file>` — ⚠️ 危险操作，需二次确认

### 第 3 步：Lint 检查

> **跳过条件**：`MERGE_ONLY` / `SKIP_LINT` / `NO_VERIFY` 任一为 true

**默认命令**：

```bash
npm run lint
```

| 结果 | 处理 |
|------|------|
| ✅ 通过 | 继续 |
| ⚠️ 未配置 lint 脚本 | 自动跳过，标注 `跳过（未配置）` |
| ⚠️ ESLint 配置缺失 | 自动跳过 |
| ❌ 代码问题失败 | 自动尝试 `npm run lint --fix`，无法修复则跳过并警告 |

> ⚠️ **--no-verify 与 lint 钩子冲突说明**：
> 若项目的 `pre-commit` 钩子**本身**包含 lint 流程：
> - 传入 `--no-verify` 时 → 第 3 步 Lint 已跳过 + 提交时也跳过钩子（无重复）
> - 未传 `--no-verify` 时 → 第 3 步 Lint **会**先执行一次（与钩子内 lint 重复）
> - 建议项目将 lint 完全交由 Skill 接管（移除 `pre-commit` 内的 lint 配置）

### 第 4 步：确认目标分支

展示并校验：

| 类型 | 展示 |
|------|------|
| 快捷分支 | 环境 + CI 行为 |
| 自定义分支 | 分支名 + "无预定义 CI 行为" |

`git branch -a` 校验存在性；不存在则终止。

> 多 remote 场景：若目标分支存在于多个 remote，询问优先从哪个 fetch。

### 第 5 步：预览并提交

**5.0 Dry-Run 检查点**（若 `DRY_RUN=true`）：

```
[DRY-RUN] 即将执行：
- git commit -m "USER_MESSAGE"
- git push origin SOURCE_BRANCH
- git checkout TARGET_BRANCH
- git pull origin TARGET_BRANCH
- git merge SOURCE_BRANCH --no-edit
- git push origin TARGET_BRANCH
- git checkout SOURCE_BRANCH

确认后输入「执行」继续；输入「取消」终止。
```

**5.1 预览变更**

展示 `git diff --cached --stat` + `git status`：

```
📋 已暂存变更 / Staged Changes:
 M src/components/Login.vue    (+24, -8)
 A src/utils/auth.ts            (+45, -0)

ℹ️ 未暂存 / 未跟踪文件不包含在本次提交中。
```

**5.2 确认 commit message**

| 情况 | 处理 |
|------|------|
| A：参数传入 message | 直接使用，**仅做格式校验**（长度 + Conventional Commits 提示） |
| B：未传入 | AI 分析 diff 生成建议 → 展示给用户确认/修改 → 等待确认 |

**格式校验规则**：

- Subject ≤ 72 字符（超过则警告，建议拆分）
- 空行后 Body ≤ 100 字符/行
- Conventional Commits 前缀（`feat` / `fix` 等）— **提示但不阻断**

**5.3 执行提交**

```bash
# 情况 A / B 通用
git commit -m "USER_COMMIT_MESSAGE"

# NO_VERIFY=true 时
git commit -m "USER_COMMIT_MESSAGE" --no-verify
```

提交后展示 `COMMIT_HASH` 和完整 message。

### 第 6 步：推送源分支

**重试策略**：

| 错误类型 | 重试次数 | 退避 | 最终失败处理 |
|---------|---------|------|-------------|
| 网络超时 | 3 次 | 5s → 10s → 20s（指数） | 终止 |
| `non-fast-forward` | 1 次 | `fetch` + `rebase` 后重试 | 终止 |
| 分支保护 | 0 次 | — | 提示通过 PR 合并 |
| 权限问题 | 0 次 | — | 提示检查 SSH / Token |
| 子模块推送 | — | — | 提示运行 `git push --recurse-submodules=check` |

**首次推送**自动 `git push -u origin SOURCE_BRANCH`。

**预览 commit 列表**（`git log origin/SOURCE_BRANCH..SOURCE_BRANCH --oneline`）后再 push。

### 第 7 步：切换并更新目标分支

> **跳过条件**：`SAME_BRANCH=true`

1. **自动 stash 保护**：
   ```bash
   git stash push -u -m "commit-and-merge: auto-stash @ $(date +%s)"
   # 标记 AUTO_STASH_CREATED=true
   ```
2. **切换目标分支**：本地不存在则基于远程创建
3. **拉取最新**：`git pull`（失败见下方）

**Pull 失败处理**：

| 错误 | 处理 |
|------|------|
| 网络问题 | 重试（沿用第 6 步策略） |
| 冲突 | 询问：`rebase` / 手动解决 / **中止**（明确：执行 `merge --abort` → `checkout SOURCE_BRANCH` → `stash pop` 恢复 → 终止流程） |
| 浅克隆失败 | 提示 `git fetch --unshallow` |

> ⚠️ 「中止」选项必须明确告知用户：会切回源分支并恢复 stash，**源分支的 commit 不会被回滚**。

### 第 8 步：合并源分支

**8.0 Dry-Run 检查点**（若 `DRY_RUN=true`）：

```
[DRY-RUN] 即将执行合并：
- git merge SOURCE_BRANCH --no-edit

实际未执行，输入「执行」继续；输入「取消」中止合并并切回源分支。
```

**正常合并**：

```bash
git merge SOURCE_BRANCH --no-edit
```

| 结果 | 处理 |
|------|------|
| ✅ 成功（含快进 / 已是最新） | 第 9 步 |
| ❌ 冲突 | 见下方 |

**冲突处理**：

展示 `git diff --name-only --diff-filter=U` 冲突文件清单，提供：

| 选项 | 流程 |
|------|------|
| **手动解决** | 用户编辑 → 「✅ 已完成，继续」→ 验证 → `git add .` → 第 9 步 |
| **AI 辅助** | AI 分析 → **必须用户确认**（可选：调整/查看 diff/转手动）→ `git add .` → 第 9 步 |
| **中止合并** | `git merge --abort` → 切回源分支 → 若 `AUTO_STASH_CREATED=true` 则 `stash pop` → 终止 |
| **查看冲突** | `git diff` 展示详情 → 返回重新选择 |

**AI 辅助合并原则**：

- ✅ 保留双方有意义逻辑
- ❌ 不得静默删除代码
- ✅ 合并后必须输出修改摘要
- ✅ 涉及核心配置文件（`package.json` / `tsconfig.json` / `vite.config.*`）时**降级为手动**

### 第 9 步：推送目标分支并切回源分支

**9.1 推送目标分支**：`git push origin TARGET_BRANCH`

- 重试策略同第 6 步
- 分支保护 → 提示通过 PR 合并

**9.2 切回源分支**：`git checkout SOURCE_BRANCH`

**9.3 Stash 恢复**：

| 条件 | 操作 |
|------|------|
| `AUTO_STASH_CREATED=true` 且无冲突 | `git stash pop`，清理 `AUTO_STASH_CREATED=false` |
| `AUTO_STASH_CREATED=true` 且恢复冲突 | 提示用户：手动解决 / 保留 stash（`git stash list` 查看） |
| `USER_STASH_EXISTS=true` | **不自动恢复**，提示用户自行处理（避免与用户意图冲突） |
| 两者都为 false | 无操作 |

### 第 10 步：输出汇总报告

**纯中文模板**：

#### 正常完成

```
✅ 提交 & 合并完成！
━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 基本信息
  源分支：feature/user-login
  目标分支：preview
  相同分支：否

📝 提交
  Message："feat(login): 新增扫码登录功能"
  Hash：abc1234
  文件变更：5 个（仅已暂存）
  Commit 状态：已提交 ✅

🔍 Lint 检查
  结果：通过 ✅

🚀 推送
  源分支：成功 ✅
  目标分支：成功 ✅

🔀 合并
  状态：成功 ✅
  冲突文件：0

📦 Stash 恢复
  自动 Stash：无
  用户 Stash：无

🔧 当前状态
  当前分支：feature/user-login
  CI 状态：构建已触发（npm run prebuild）
━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 相同分支（跳过合并）

```
✅ 提交 & 合并完成！
━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 基本信息
  源分支：develop
  目标分支：develop
  相同分支：是（已跳过合并）

📝 提交
  Message："fix: 修复构建报错"
  Hash：def5678
  文件变更：2 个
  Commit 状态：已提交 ✅

🔍 Lint 检查
  结果：通过 ✅

🚀 推送
  源分支：成功 ✅
  目标分支：跳过（同分支）

🔀 合并
  状态：跳过（源=目标）⏭️

📦 Stash 恢复
  N/A

🔧 当前状态
  当前分支：develop
  CI 状态：构建已触发（npm run devbuild）
━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 仅合并（无变更）

```
✅ 仅合并完成！
━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 基本信息
  源分支：feature/user-login
  目标分支：develop
  提交步骤：跳过（无变更）⏭️
  文件变更：0

🔍 Lint 检查
  结果：跳过（无变更）⏭️

🔀 合并
  状态：成功 ✅（或 跳过 ⏭️ 若已是最新）

🚀 推送
  目标分支：成功 ✅
  源分支：无未推送 commit

🔧 当前状态
  当前分支：feature/user-login
  CI 状态：构建已触发（npm run devbuild）
━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### --no-verify 跳过检查

```
✅ 提交 & 合并完成（跳过检查）!
━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 基本信息
  源分支：hotfix/urgent
  目标分支：master
  相同分支：否

📝 提交
  Message："fix: 紧急修复"
  Hash：9abcdef
  Commit 状态：已提交（--no-verify）⚠️

🔍 Lint 检查
  结果：跳过（--no-verify）⏭️
  ⚠️ 警告：本次提交未经过 lint 与 git hooks 校验

🔀 合并
  状态：成功 ✅

🔧 当前状态
  当前分支：hotfix/urgent
  CI 状态：手动触发（npm run probuild）
━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### Dry-Run 模式

```
🔍 [DRY-RUN] 模拟执行报告
━━━━━━━━━━━━━━━━━━━━━━━━━━━
本模式**未实际执行任何 git 命令**，仅展示将要执行的操作。

📌 解析结果
  目标分支：preview
  提交信息：feat: 新功能
  --no-verify：false
  --dry-run：true

📋 状态
  源分支：feature/login
  Staged files：3
  Unstaged files：1
  Untracked files：0

🔧 计划执行
  1. npm run lint
  2. git commit -m "feat: 新功能"
  3. git push origin feature/login
  4. git checkout preview
  5. git pull origin preview
  6. git merge feature/login --no-edit
  7. git push origin preview
  8. git checkout feature/login

输入「执行」继续；输入「取消」终止。
━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 异常处理与安全约束

### 环境异常

| 异常 | 处理 |
|------|------|
| 非 Git 仓库 | 终止 |
| `origin` 未配置 | 终止 + 提示命令 |
| 多 remote | 询问优先使用哪个 |
| Detached HEAD | 警告 + 询问 |
| merge / rebase 进行中 | 暂停 + 等待 |
| 浅克隆 | 警告 + 提供 `fetch --unshallow` 方案 |
| GPG 签名失败 | 提示 `gpg --list-secret-keys` |
| 并发锁占用 | 提示已有实例，询问是否继续 |

### 流程异常

| 异常 | 处理 |
|------|------|
| Checkout 失败 | 自动 stash → 重试 → 标记 `AUTO_STASH_CREATED` |
| Push 网络问题 | 指数退避 3 次重试 |
| Push `non-fast-forward` | `fetch` + `rebase` 重试 |
| Push 权限 | 提示检查 SSH / Token |
| Push 分支保护 | 提示通过 PR |
| Pull 冲突 | rebase / 手动 / 中止（明确回退路径） |
| Commit 失败（hook） | 区分 hook / 邮箱 / 空提交 |
| Commit 失败（GPG） | 提示签名问题 |
| Stash pop 冲突 | 提示手动解决 / 保留 |
| 子模块状态异常 | 提示 `git submodule update` |

### 安全约束

- 仅提交已暂存文件，**不自动 `git add`**
- 破坏性操作（`restore` / `reset --hard`）**必须二次确认**
- 目标分支必须存在且合法（不含 `~` `^` `:` `?` `*` `[` `\`）
- 源分支 = 目标分支时跳过合并
- `--no-verify` 跳过检查时在汇总报告明确标注
- `AUTO_STASH_CREATED` 与 `USER_STASH_EXISTS` 区分处理
- Dry-Run 模式下**任何 git 写操作前必须停下等待确认**

---

## 附录

### A. Push 重试策略

```
重试次数：3
退避策略：5s → 10s → 20s（指数退避）
超时阈值：30s / 次
触发条件：网络超时 / 连接重置 / DNS 失败
不触发：非快进 / 权限 / 分支保护
```

### B. 与 README 关系

- **SKILL.md** 是唯一权威定义（单一事实来源）
- **README.md** 仅保留简介 + 命令格式 + FAQ，与本文件保持同步
- 双副本差异以 SKILL.md 为准

---

> 📌 本文档与 `README.md` 内容保持一致；如发现冲突，**以 SKILL.md 为准**。
