---
name: commit-and-merge
description: 自动提交当前分支已暂存的文件并合并到目标分支（默认 develop，可通过 --preview/--master 指定），提交前执行 lint 检查。适用于用户需要提交代码、合并分支、部署到测试/预发/生产环境，或输入 "/commit-and-merge" 时。
---

# 提交并合并

自动化 lint 检查、提交本地**已暂存**变更并合并到目标分支的完整工作流。

## 命令格式

```
/commit-and-merge --<目标分支名> ["提交信息" 或 '提交信息']
```

> **参数顺序灵活**：两个参数**可任意组合**和**调换位置**：
> - `/commit-and-merge --feature/x "message"`
> - `/commit-and-merge "message" --feature/x`
> - 分支参数可省略（默认 `--develop`）
> - commit message 可省略（进入第 5 步询问）
> - 两者可单独使用，也可同时使用
>
> **分支名任意**：`--<目标分支名>` 不限于常用分支，**支持任意分支名**（含 `/`、`-` 等），例如：
> - `--develop` / `--preview` / `--master`（常用快捷分支，带默认 CI 行为）
> - `--feature/login`、`--hotfix-123`、`--release/v1.0`（自定义分支）

### 参数说明

#### 常用快捷分支（带默认 CI 行为）

| 参数 | 目标分支 | 环境 | CI 触发行为 | 是否必填 |
|------|---------|------|------------|----------|
| `--develop` | develop | 测试环境 | 自动构建（`npm run devbuild`）| ❌ |
| `--preview` | preview | 预发环境 | 自动构建（`npm run prebuild`）| ❌ |
| `--master` | master | 生产环境 | 手动触发（`npm run probuild`）| ❌ |

> 不指定分支参数时，默认使用 `--develop`。

#### 自定义分支（任意分支名）

| 参数 | 目标分支 | 说明 | 是否必填 |
|------|---------|------|----------|
| `--<任意名称>` | 任意已存在的分支 | 例：`--feature/login`、`--release/v1.0`、`--user/lijinhai` | ❌ |

> - 自定义分支**不触发预定义的 CI 行为**，由项目的 CI 配置决定
> - 自定义分支名支持任意合法 git 分支名（含 `/`、`-`、`_`、`.` 等）
> - 若指定分支在本地和远程都不存在，提示用户并停止

### 使用示例

```bash
/commit-and-merge
# 默认：仅提交已暂存文件，合并到 develop，message 由 AI 建议后确认

/commit-and-merge --preview
# 合并到 preview

/commit-and-merge --master
# 合并到 master

/commit-and-merge "feat: 优化列表加载性能"
# 使用指定 message，合并到 develop

/commit-and-merge --master 'fix: 修复登录失效问题'
# 使用指定 message，合并到 master

/commit-and-merge --preview "feat: 新增用户管理"
# 分支在前：合并到 preview

/commit-and-merge "feat: 新增用户管理" --preview
# 分支在后（等价写法）：合并到 preview

/commit-and-merge --feature/login "feat: 登录改版"
# 合并到自定义分支 feature/login

/commit-and-merge --release/v1.2.0
# 合并到自定义分支 release/v1.2.0

/commit-and-merge 'feat: 紧急修复' --master
# 分支在后 + 单引号
```

## 前置检查

流程启动前自动验证环境，任一条件不满足则**终止并提示用户**：

| 检查项 | 验证方式 | 不满足时行为 |
|--------|---------|-------------|
| Git 仓库 | `git rev-parse --is-inside-work-tree` | 终止：「当前目录不是 Git 仓库」 |
| origin 远程 | `git remote get-url origin` | 终止：提示执行 `git remote add origin <url>` |
| 无 merge/rebase | 检查 `MERGE_HEAD` / `rebase-merge` 目录 | 暂停：等待用户处理完成后继续 |
| 非 Detached HEAD | `git symbolic-ref HEAD` | 警告：询问是否继续（可能导致提交无法被分支引用） |
| node_modules 存在 | 检查目录 | 标记 `SKIP_LINT`，Lint 步骤自动跳过 |

---

## 工作流程

按以下步骤**顺序执行**，任何步骤失败时停止并向用户报告。

### 第 1 步：解析命令参数

从用户输入的命令字符串中提取（**参数顺序无关**，可任意调换位置）：

- `TARGET_BRANCH`：扫描 `--<任意文本>` 模式（以 `--` 开头，后跟非空字符直到空格或字符串结束）
  - 命中 `--develop` / `--preview` / `--master` → 使用对应分支名（带默认 CI 行为）
  - 命中其他 `--xxx` → 视为自定义分支名，直接使用
  - 未命中 → 默认 `develop`
- `COMMIT_MESSAGE`：扫描成对出现的 `"..."` 或 `'...'`，命中即取其内部内容；未命中则后续步骤询问

> **解析规则**：
> - 分支参数和 commit message 参数**位置可互换**
> - 两者可单独使用，也可同时使用
> - 都不传则使用默认值（develop）+ 第 5 步询问 message
> - 若同时传入多个 `--` 开头参数（如 `--develop --preview`），提示用户参数冲突并停止
> - 自定义分支名支持任意合法 git 分支名（`/`、`-`、`_`、`.` 等）
> - 解析完成后**校验分支是否存在**（`git branch -a | grep`），不存在则提示并停止

### 第 2 步：检查 Git 状态

运行以下命令了解当前状态：

```bash
git status
git branch --show-current
git diff --stat --cached   # 查看已暂存文件
git diff --stat            # 查看未暂存修改
git ls-files --others --exclude-standard  # 查看未跟踪文件
```

- 记录**当前分支名**为 `SOURCE_BRANCH`
- 将变更分为三类：
  - `STAGED_FILES`（已暂存，"Changes to be committed"）
  - `UNSTAGED_FILES`（未暂存，"Changes not staged"，含修改/删除）
  - `UNTRACKED_FILES`（未跟踪，"Untracked files"）
- 默认行为：**只对 `STAGED_FILES` 执行 `git commit`**，不再自动 `git add`

#### 源分支 = 目标分支校验

检查 `SOURCE_BRANCH` 是否等于 `TARGET_BRANCH`：

- ✅ **相同**：标记 `SAME_BRANCH=true`
  - 仍走正常提交流程（第 3~6 步）
  - 提交并推送成功后，**跳过第 7~9 步**（无需切换分支、拉取、合并、推送目标分支）
  - 直接进入**第 10 步**输出汇总报告，在报告中标注「源分支与目标分支相同，已跳过合并」
- ❌ **不同**：继续正常流程

#### 子流程 A：完全无变更（工作区干净）

- 告知用户工作区干净，**默认执行仅合并**
- 自动标记 `MERGE_ONLY=true`，**跳过第 3~5 步**（无变更可提交，无需 lint、提交、推送源分支），直接进入第 6 步推送当前分支（若有未推送 commit）或第 7 步切换目标分支

#### 子流程 B：仅 `STAGED_FILES` 有内容（最常见，无需询问）

- 直接进入第 3 步

#### 子流程 C/D：存在未暂存或未跟踪文件

当工作区存在 `UNSTAGED_FILES` 或 `UNTRACKED_FILES` 时，询问用户暂存策略：

| 选项 | 操作 | 说明 |
|------|------|------|
| 1. 仅提交已暂存 | 无操作 | **默认**，其他变更保留在工作区 |
| 2. `git add -A` | 暂存全部 | 包含未跟踪文件 |
| 3. `git add -u` | 暂存已跟踪 | **排除**未跟踪文件（避免提交 IDE 配置等） |
| 4. 跳过 commit | 仅合并 | 不提交，直接进入合并流程 |
| 5. 其他 | 高级操作 | 见下方说明 |

**高级操作（选项 5）**：
- `git diff` / `git diff --cached` — 查看差异
- `git add -p` / `git restore --staged <file>` — 精细控制暂存
- `git stash push -u` / `git stash pop` — Stash 工作区
- `git restore <file>` / `git checkout -- <file>` — ⚠️ 危险操作，需二次确认

暂存完成后，**重新执行 `git status` 和 `git diff --stat --cached` 确认状态**，再进入第 3 步。

### 第 3 步：运行 Lint 检查

> **跳过条件**：`MERGE_ONLY=true`（仅合并）或 `SKIP_LINT` 已标记（node_modules 未安装）

执行 `npm run lint`，根据结果处理：

| 结果 | 处理方式 |
|------|----------|
| ✅ 通过 | 继续下一步 |
| ⚠️ 未配置 lint 脚本 | 自动跳过，不中断流程 |
| ⚠️ ESLint 配置缺失 | 自动跳过，不中断流程 |
| ❌ 代码问题失败 | 自动尝试 `--fix` 修复；无法修复则跳过并警告 |

**跳过时在汇总报告标注**：`Lint 检查：跳过（原因）`

### 第 4 步：确认目标分支

第 1 步已解析出 `TARGET_BRANCH`，直接展示并校验：

| 分支类型 | 展示信息 |
|---------|----------|
| 常用分支 (`develop`/`preview`/`master`) | 环境 + CI 行为 |
| 自定义分支 | 分支名 + "无预定义 CI 行为" |

校验 `git branch -a` 是否存在，**存在则自动继续**，不存在则终止。

### 第 5 步：预览并提交

**5.1 预览变更摘要**

展示 `git diff --cached --stat` + `git status`，包括：
- 已暂存文件清单（路径 + 行数）
- 提醒：未暂存/未跟踪文件**不**会包含在本次提交中

**5.2 确认 commit message**

- **情况 A**：第 1 步已通过 `"..."` 或 `'...'` 提供 → **无需确认，直接使用该 message**
- **情况 B**：未提供 → 分析已暂存 diff 生成符合规范的 message 建议（`feat(scope): description` / `fix(scope): description`），**展示给用户确认或修改，等待用户确认后再提交**

**5.3 自动提交**

- **情况 A**：用户已通过参数传入 message → **直接使用该 message 执行 `git commit`**，不再询问
- **情况 B**：用户未提供 message → 使用用户确认/修改后的 message 执行 `git commit`
- 提交后展示 commit hash 和 message

**5.4 执行提交**

```bash
# 注意：仅提交已暂存文件，不再执行 git add
git commit -m "USER_COMMIT_MESSAGE"
```

若提交失败，展示错误并停止。

### 第 6 步：推送源分支

检查远程分支是否存在 → 决定用 `push` 或 `push -u` → 预览 commit 列表 → **自动推送**

**失败处理**：

| 错误类型 | 处理方式 |
|---------|----------|
| `non-fast-forward` | `git fetch` + `rebase` 后重试 |
| 分支保护 | 提示通过 PR 合并，终止流程 |
| 权限问题 | 提示检查 SSH/Token，终止 |
| 网络问题 | 重试一次 |

### 第 7 步：切换并更新目标分支

> **跳过条件**：`SAME_BRANCH=true`（源分支 = 目标分支）

1. **切换目标分支**：本地不存在则基于远程创建，切换失败则自动 stash 保护
2. **拉取最新代码**：`git pull` 失败则询问用户 rebase/手动解决/中止

### 第 8 步：合并源分支

执行：

```bash
git merge SOURCE_BRANCH --no-edit
```

#### 情况 A：合并成功（"Merge made by..."）或 Fast-forward / Already up to date

- 继续执行第 9 步

#### 情况 B：出现合并冲突

展示冲突文件清单（`git diff --name-only --diff-filter=U`），询问用户处理方式：

| 选项 | 流程 |
|------|------|
| **手动解决** | 用户编辑冲突文件 → 点击「✅ 已完成，继续」→ 验证冲突是否全部解决 → `git add .` → 继续下一步 |
| **AI 辅助** | AI 分析冲突并修改文件 → **必须用户确认**（可选：调整/查看 diff/转手动）→ `git add .` → 继续 |
| **中止合并** | `git merge --abort` → 切回源分支 → 终止流程 |
| **查看冲突** | `git diff` 展示详情 → 返回重新选择 |

**AI 辅助原则**：保留双方有意义逻辑、删除无意义重复、保持代码风格、**不得静默删除代码**、必须输出修改摘要。

### 第 9 步：推送目标分支并切回源分支

**9.1 推送目标分支**

`git push origin TARGET_BRANCH`

失败处理：分支保护 → 提示通过 PR 合并；其他错误 → `git pull --rebase` 后重试。

**9.2 切回源分支**

`git checkout SOURCE_BRANCH`

若有 `STASH_CREATED=true`（第 7 步自动 stash）：执行 `git stash pop`，恢复冲突则提示用户选择解决或保留 stash。

### 第 10 步：输出汇总报告

展示最终汇总：

```
✅ 提交 & 合并完成！
- 源分支：SOURCE_BRANCH
- 目标分支：TARGET_BRANCH（常用分支 / 自定义分支）
- 源分支 = 目标分支：是（已跳过合并） / 否
- Commit Message："USER_COMMIT_MESSAGE"
- Lint 检查：通过 ✅ / 跳过（未配置）⏭️ / 跳过（node_modules 未安装）⏭️ / 跳过（用户选择）⚠️ / 失败 ❌
- 文件变更：N 个文件（仅已暂存）
- Commit 状态：已提交 ✅ / 已取消 ❌
- 源分支推送：成功 ✅ / 首次推送（设置上游）✅ / 失败 ❌ / 已取消 ❌
- 合并状态：成功 ✅ / 冲突已解决（手动）✅ / 冲突已解决（AI）✅ / 已中止 ❌ / 跳过（源=目标）⏭️
- 冲突文件：M 个（如有）
- 冲突处理方式：手动 / AI 辅助（如有）
- 合并至：TARGET_BRANCH
- 目标分支推送：是 / 否
- 当前分支：TARGET_BRANCH / SOURCE_BRANCH
- Stash 恢复：已恢复 ✅ / 恢复冲突 ⚠️ / 未恢复（保留 stash）📦 / 无 stash ✅
- CI 状态：构建已触发（如适用）/ 自定义分支（无预定义 CI）
```

## 异常处理与安全约束

### 环境异常（前置检查未通过）

| 异常 | 处理 |
|------|------|
| 非 Git 仓库 / origin 未配置 | 终止流程，提示配置 |
| Detached HEAD | 警告，询问是否继续 |
| merge/rebase 进行中 | 暂停，等待用户处理 |

### 流程异常

| 异常 | 处理 |
|------|------|
| Checkout 失败 | 自动 stash 保护 → 重试 → 标记待恢复 |
| Push 失败（non-fast-forward） | `git fetch` + `rebase` → 重试 |
| Push 失败（权限） | 提示检查 SSH/Token |
| Push 失败（分支保护） | 提示通过 PR 合并 |
| Pull 冲突 | 询问用户 rebase/手动/中止 |
| Commit 失败 | 按错误分类处理（hook/邮箱/空提交） |
| Stash 恢复冲突 | 提示用户解决或保留 stash |

### 安全约束

- 仅提交已暂存文件，不自动 `git add`
- 破坏性操作（`restore`/`reset --hard`）**必须二次确认**
- 目标分支必须存在且合法（不含 `~` `^` `:` `?` `*` `[` 等保留字符）
- 源分支 = 目标分支时自动跳过合并

