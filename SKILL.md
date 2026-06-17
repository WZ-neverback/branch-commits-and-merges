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

在开始第 0 步前，先验证环境是否就绪：

**1. 是否为 Git 仓库**

```bash
git rev-parse --is-inside-work-tree
```

- 若报错 `not a git repository`：告知用户「当前目录不是 Git 仓库」，终止流程

**2. 远程仓库（origin）是否已配置**

```bash
git remote get-url origin
```

- 若报错 `fatal: No such remote 'origin'` 或返回空/无效 URL：告知用户「当前 Git 仓库未配置 origin 远程仓库，无法执行推送/拉取操作。请先执行 `git remote add origin <url>` 配置远程仓库」，**终止流程**
- 若配置正常 → 继续

**3. 是否存在进行中的合并或 rebase**

```bash
git status --porcelain | grep -E "^(UU|AA|DD|AU|UA|DU|UD)"
# 同时检查
test -f .git/MERGE_HEAD && echo "merge in progress"
test -d .git/rebase-merge && echo "rebase in progress"
test -d .git/rebase-apply && echo "rebase in progress"
```

- 若存在：**提示用户先完成或中止当前操作**，不继续执行
- 选项：`✅ 已完成，继续本流程`（用户手动处理后）/ `❌ 取消`

**4. 是否为 Detached HEAD 状态**

```bash
git symbolic-ref HEAD
```

- 若报错 `fatal: ref HEAD is not a symbolic ref`：警告用户处于游离状态，询问是否切回分支或继续（⚠️ 继续可能导致提交无法被分支引用）

**5. node_modules 是否存在（为第 2 步 Lint 准备）**

- 若 `node_modules` 目录不存在，标记 `SKIP_LINT_REASON="node_modules 未安装"`，第 2 步自动跳过 lint 检查

---

## 工作流程

按以下步骤**顺序执行**，任何步骤失败时停止并向用户报告。

### 第 0 步：解析命令参数

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

### 第 1 步：检查 Git 状态

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
  - 仍走正常提交流程（第 2~6 步）
  - 提交并推送成功后，**跳过第 7~10 步**（无需切换分支、拉取、合并、推送目标分支）
  - 直接进入**第 11 步**输出汇总报告，在报告中标注「源分支与目标分支相同，已跳过合并」
- ❌ **不同**：继续正常流程

#### 子流程 A：完全无变更（工作区干净）

- 告知用户，询问是否仍需继续执行**仅合并**操作
- 选项：仅合并 / 取消
- 若用户选择「仅合并」：标记 `MERGE_ONLY=true`，**跳过第 2~5 步**（无变更可提交，无需 lint、提交、推送源分支），直接进入第 6 步推送当前分支（若有未推送 commit）或第 7 步切换目标分支

#### 子流程 B：仅 `STAGED_FILES` 有内容（最常见，无需询问）

- 直接进入第 2 步

#### 子流程 C：仅 `UNSTAGED_FILES` / `UNTRACKED_FILES` 有内容（无已暂存）

此时**没有可直接提交的内容**，必须先暂存。使用 AskUserQuestion：

1. `git add -A` 后一起提交（推荐）— 包含未跟踪文件
2. `git add -u` 仅 add 已修改/删除 — **排除**未跟踪文件（适合不想提交 IDE 配置的场景）
3. 手动指定要 add 的文件 — 用户在"其他"中输入文件路径，执行 `git add <file1> <file2> ...`
4. 跳过 commit，仅执行合并

#### 子流程 D：三者都有（`STAGED` + `UNSTAGED` +/`UNTRACKED`）

使用 AskUserQuestion：

1. 仅提交已暂存文件（推荐）— 其他保留在工作区
2. `git add -A` 暂存所有后一起提交 — 把未暂存和未跟踪都纳入本次 commit
3. `git add -u` 暂存已修改后一起提交 — **排除**未跟踪文件
4. stash 未暂存内容后仅提交已暂存 — 执行 `git stash push -u`（含未跟踪），然后仅 commit 已暂存部分

#### 子流程 E：用户选择"其他"（高级操作入口）

当用户通过"其他"输入时，按需执行以下操作（执行完毕**回到对应子流程**继续）：

- **查看差异**：
  - `git diff` — 查看未暂存修改
  - `git diff --cached` — 查看已暂存内容
  - `git diff <file>` — 查看指定文件差异
- **打开交互式 add**：`git add -p` 或 `git add -i`（用户自行挑选 hunk）
- **取消部分已暂存**：`git restore --staged <file>` 或 `git reset HEAD <file>`（将已暂存文件退回未暂存）
- **撤销未暂存修改**（⚠️ 危险操作，需二次确认）：
  - `git restore <file>` 撤销指定文件
  - `git restore .` 撤销所有未暂存修改
- **stash 高级用法**：
  - `git stash push -u` — stash 包含未跟踪
  - `git stash push -m "message"` — 带备注
  - `git stash pop` — 恢复
- **暂存特定文件**：用户给出文件路径后执行 `git add <path>`

#### 安全约束

- **任何 `restore` / `checkout --` / `reset --hard` 类破坏性操作**必须先 AskUserQuestion 二次确认，并明确告知会丢失未保存的工作
- 暂存完成后，**重新执行 `git status` 和 `git diff --stat --cached` 确认状态**，再进入第 2 步

### 第 2 步：运行 Lint 检查

> **跳过条件**：若 `MERGE_ONLY=true`（子流程 A 仅合并）或 `SAME_BRANCH=true` 且无暂存变更，直接跳过本步，进入第 3 步

**前置检查：node_modules 是否存在**

若前置检查阶段已标记 `SKIP_LINT_REASON="node_modules 未安装"`：
- **自动跳过 Lint 检查**，提示用户：`node_modules 未安装，自动跳过 lint 检查（建议执行 npm install 后重试）`
- 在汇总报告中标注：`Lint 检查：跳过（node_modules 未安装）`

否则，执行 lint 命令，根据返回结果分类处理：

```bash
npm run lint
```

**情况 A：命令报错 `Missing script: lint`（项目未配置 lint 脚本）**
- 默认**跳过 Lint 检查**，继续执行第 3 步
- 提示用户：`项目未配置 lint 脚本，自动跳过本步`
- 在汇总报告中标注：`Lint 检查：跳过（未配置 lint 脚本）`
- **不当作错误**，不中断流程

**情况 B：项目已配置 lint 脚本，但因缺少配置文件或依赖而失败**

当 `npm run lint` 报错包含以下任意特征时：
- `ESLint configuration not found`
- `No ESLint configuration found`
- `Cannot find module`（eslint 相关）
- `Failed to load config`（eslint 配置相关）

处理方式：**默认跳过**，同情况 A
- 提示用户：`项目未配置 ESLint 环境，自动跳过 lint 检查`
- 在汇总报告中标注：`Lint 检查：跳过（未配置 ESLint）`
- **不当作错误**，不中断流程

**情况 C：项目已配置 lint 脚本，且正常运行**

- Lint 通过 → 继续执行第 3 步
- Lint 失败（代码问题）：
  1. 向用户展示 lint 错误输出
  2. 询问用户：自动修复（如可修复）、手动修复，或跳过 lint 继续
  3. 若用户选择跳过，附带警告继续执行
  4. 若用户修复，重新运行 lint 直到通过

> 提示：如需强制检查，可先在 `package.json` 中添加 `lint` 脚本或在项目中配置 ESLint 后再次执行本 skill。

### 第 3 步：展示变更摘要

运行：

```bash
git diff --stat --cached
```

向用户展示清晰的变更摘要：
- 已暂存文件数量
- 列出已暂存的文件名
- **重要提醒**：未暂存/未跟踪文件**不**会包含在本次提交中

### 第 4 步：确认目标分支

> 默认行为：第 0 步已解析出 `TARGET_BRANCH`（无论是否带 `--` 参数），**直接展示让用户确认一次**，无需再次选择。

**4.1 展示目标分支信息**

根据 `TARGET_BRANCH` 的类型展示：

- **常用分支**（`develop` / `preview` / `master`）：显示其环境与 CI 行为

  | 分支 | 环境 | CI 触发行为 |
  |------|------|-------------|
  | `develop` ✅ 默认 | 测试环境 | 自动构建（`npm run devbuild`）|
  | `preview` | 预发环境 | 自动构建（`npm run prebuild`）|
  | `master` | 生产环境 | 手动触发（`npm run probuild`）|

- **自定义分支**（如 `feature/login`、`hotfix-123`、`release/v1.0`）：显示分支名 + 提示"自定义分支无预定义 CI 行为"

**4.2 校验分支存在性**

```bash
git branch -a | grep -E "(^|/)TARGET_BRANCH$"
```

- ✅ 存在（本地或远程）→ 继续
- ❌ 不存在 → 提示用户「分支 `xxx` 不存在，请先创建或检查名称」并停止

**4.3 用户确认**

使用 AskUserQuestion 询问：

| 选项 | 后续动作 |
|------|---------|
| **「✅ 确认合并到此分支」** | 继续第 5 步 |
| **「✏️ 改为其他分支」** | 用户在"其他"中输入新分支名 → 回到 4.1 |
| **「❌ 取消」** | 终止流程 |

记录确认结果为 `TARGET_BRANCH`。

### 第 5 步：预览并提交

**5.1 预览待提交内容**

```bash
git diff --cached --stat        # 改动概要
git status                       # 确认状态
```

向用户展示：
- 即将提交的文件清单（路径 + 改动行数）
- **重要提醒**：未暂存/未跟踪文件**不**会包含在本次提交中

**5.2 确认 commit message**

- **情况 A**：第 0 步已通过 `"..."` 或 `'...'` 提供 → **展示该 message 让用户确认**
- **情况 B**：未提供 → 分析已暂存 diff 生成符合规范的 message 建议（`feat(scope): description` / `fix(scope): description`）

**5.3 用户确认（无论是否提供 message 都要走这一步）**

使用 AskUserQuestion 询问，提供 4 个选项：

| 选项 | 后续动作 |
|------|---------|
| **「✅ 确认提交」** | 使用当前 message 执行 `git commit` |
| **「✏️ 修改 message」** | 用户在"其他"中输入新 message → 回到 5.3 再次确认 |
| **「👀 查看完整 diff」** | 执行 `git diff --cached` 展示完整内容 → 回到 5.3 |
| **「❌ 取消」** | 终止整个流程 |

**5.4 执行提交**

```bash
# 注意：仅提交已暂存文件，不再执行 git add
git commit -m "USER_COMMIT_MESSAGE"
```

若提交失败，展示错误并停止。

### 第 6 步：推送当前分支

**6.1 预检查：远程分支是否存在**

```bash
# 检查远程是否已有该分支
git ls-remote --heads origin SOURCE_BRANCH

# 或检查本地是否有上游
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>$null
```

根据结果决定推送方式：
- **远程已存在 / 本地有上游** → `git push origin SOURCE_BRANCH`
- **远程不存在（新分支首次推送）** → `git push -u origin SOURCE_BRANCH`（同时设置上游跟踪）

**6.2 预览推送内容**

```bash
# 已有上游：显示将推送的 commits
git log origin/SOURCE_BRANCH..SOURCE_BRANCH --oneline 2>$null
# 新分支：显示最近 commits
git log --oneline -3
```

向用户展示：
- 将要推送的 commit 列表（含 hash + message）
- 推送目标（origin/SOURCE_BRANCH）
- 是否为首次推送（设置上游）

**6.3 用户确认**

使用 AskUserQuestion 询问：

| 选项 | 后续动作 |
|------|---------|
| **「✅ 确认推送」** | 执行 push |
| **「❌ 取消推送」** | 保留本地 commit，终止流程，输出"未推送"状态报告 |

**6.4 执行推送**

```bash
# 情况 1：远程已存在
git push origin SOURCE_BRANCH

# 情况 2：新分支首次推送（自动设置上游）
git push -u origin SOURCE_BRANCH
```

**6.5 推送失败处理**

若 push 失败（常见原因：远程有新提交、权限不足、网络问题）：

1. 若错误信息为 `rejected (non-fast-forward)` 或 `failed to push`：
   ```bash
   # 先 rebase 同步远程变更
   git fetch origin SOURCE_BRANCH
   git rebase origin/SOURCE_BRANCH
   
   # 若 rebase 出现冲突 → 参考第 8 步的冲突处理流程
   ```
   然后**重新执行 6.4**

2. 若错误信息包含分支保护相关关键字（如 `protected branch`、`GH006`、`hook declined`、`protected branch update failed` 等）：
   - 告知用户：「分支 `SOURCE_BRANCH` 已开启分支保护，禁止直接推送。请通过 Pull Request 进行代码合并。」
   - **跳过后续所有步骤**，直接进入第 11 步输出汇总报告，标注推送失败原因

3. 若错误为 `Permission denied` / `Authentication failed`：
   - 报告错误并停止，提示用户检查 SSH Key / Token 权限

4. 若错误为网络问题：
   - 重试一次，仍失败则报告

5. 若重试后仍失败 → 报告错误，保留本地状态不丢失

**6.6 推送成功验证**

```bash
git ls-remote --heads origin SOURCE_BRANCH
```

确认远程已存在该分支，输出 commit hash 与本地一致。

### 第 7 步：切换到目标分支并拉取最新代码

> **跳过条件**：若 `SAME_BRANCH=true`（源分支与目标分支相同），跳过第 7~10 步，直接进入第 11 步输出汇总报告

**7.1 确定目标分支本地是否存在**

```bash
git branch --list TARGET_BRANCH
git ls-remote --heads origin TARGET_BRANCH
```

- ✅ 本地已存在 → 继续 7.2
- ❌ 本地不存在、远程存在 → 执行：
  ```bash
  git checkout -b TARGET_BRANCH origin/TARGET_BRANCH
  ```
  完成后跳过 7.2，直接到 7.3
- ❌ 本地和远程都不存在 → 提示用户「分支 `TARGET_BRANCH` 不存在」并停止

**7.2 切换到目标分支**

```bash
git checkout TARGET_BRANCH
```

若 checkout 失败（常见原因：未提交的修改与目标分支冲突）：

1. **自动 stash 保护**：
   ```bash
   git stash push -m "commit-and-merge auto stash"
   ```
2. 重新执行 `git checkout TARGET_BRANCH`
3. 标记 `STASH_CREATED=true`，在第 10 步切回源分支后自动恢复 stash

若仍失败：报告错误并停止

**7.3 拉取目标分支最新代码**

```bash
git pull origin TARGET_BRANCH
```

- ✅ 拉取成功 → 继续第 8 步
- ❌ 拉取失败（冲突）：
  1. 展示冲突文件列表：`git diff --name-only --diff-filter=U`
  2. 询问用户处理方式：
     - **「✅ 自动 rebase 解决」** — 执行 `git pull --rebase origin TARGET_BRANCH`，若仍有冲突参考第 8 步冲突处理流程
     - **「❗ 手动解决」** — 用户自行处理后点「✅ 已完成」
     - **「❌ 中止合并」** — 执行 `git merge --abort` 或 `git rebase --abort`，切回源分支，终止流程

### 第 8 步：合并源分支

执行：

```bash
git merge SOURCE_BRANCH --no-edit
```

#### 情况 A：合并成功（"Merge made by..."）或 Fast-forward / Already up to date

- 继续执行第 9 步

#### 情况 B：出现合并冲突（输出包含 `CONFLICT`）

**8.1 展示冲突概览**

```bash
git diff --name-only --diff-filter=U   # 冲突文件列表
git diff --name-only --diff-filter=U | wc -l   # 冲突文件数量
```

向用户展示：
- 冲突文件清单（路径 + 冲突 hunk 数量）
- 总冲突规模
- 提示：合并已暂停，不会丢失工作进度

**8.2 询问用户处理方式**

使用 AskUserQuestion 询问，提供 4 个选项：

| 选项 | 后续动作 |
|------|---------|
| 1. 手动解决 | 用户自行编辑冲突文件 → 必须点击"已完成，继续"按钮 |
| 2. AI 辅助解决 | AI 分析并修改 → 修改后**必须用户确认**才能继续 |
| 3. 中止合并 | `git merge --abort` 并切回源分支，终止流程 |
| 4. 仅查看冲突详情 | 执行 `git diff` 展示完整冲突 → 返回 8.2 |

**8.3 根据选择分支处理**

##### 选项 1：手动解决

1. 提示用户：「请在 IDE 中处理冲突文件，处理完成后请点击下方按钮」
2. **持续轮询确认**（使用 AskUserQuestion 循环询问），按钮选项固定为：
   - **「✅ 已完成，继续」** — 触发冲突验证
   - **「⏳ 还需要时间」** — 继续等待，间隔后再次询问
   - **「❌ 放弃，转 AI 处理」** — 切换到 AI 辅助
   - **「🛑 放弃，中止合并」** — 切换到中止合并
3. 用户点击「✅ 已完成，继续」后：
   ```bash
   # 验证所有冲突已解决
   REMAINING=$(git diff --name-only --diff-filter=U)
   ```
   - 若 `$REMAINING` 不为空：列出仍含冲突标记的文件，提示用户继续处理，**回到本步骤第 2 点继续轮询**
   - 若 `$REMAINING` 为空：
     ```bash
     git add .
     ```
     继续第 9 步

##### 选项 2：AI 辅助解决

1. AI 逐个读取冲突文件，分析冲突原因
2. AI 提出合并策略（基于代码语义、保留双方合理逻辑、保持原有风格）
3. AI 修改文件，移除 `<<<<<<<` / `=======` / `>>>>>>>` 标记
4. **关键：AI 处理完成后必须等待用户确认**，使用 AskUserQuestion：
   - **「✅ 确认通过，继续合并」** — 暂存并继续
   - **「🔧 部分文件需调整」** — 用户输入需调整的文件名，AI 重新处理后**再次确认**
   - **「👀 展示 AI 修改的 diff」** — 执行 `git diff <file>` 展示详情后**回到本步确认**
   - **「🛑 放弃 AI，转手动」** — 切换到选项 1
5. 用户最终确认通过后：
   ```bash
   git add .
   ```
   继续第 9 步
6. **AI 处理原则**：
   - 保留冲突双方都有意义的逻辑（如同时引入两个工具函数）
   - 删除无意义的重复（如同名 import 重复声明）
   - 保持代码风格与项目一致
   - **不得静默删除一方的代码而不告知用户**
   - 处理完成后必须**输出修改摘要**（哪些文件、改动行数、合并策略）

##### 选项 3：中止合并

```bash
git merge --abort
git checkout SOURCE_BRANCH
```

终止流程，跳到第 11 步输出**合并失败**的汇总报告。

##### 选项 4：仅查看冲突详情

```bash
git diff                    # 完整冲突
git diff <file>             # 指定文件
git diff --stat             # 概要
```

展示完成后**回到 8.2**重新询问。

### 第 9 步：Push 目标分支

```bash
git push origin TARGET_BRANCH
```

若 push 失败：
- 若错误信息包含分支保护相关关键字（如 `protected branch`、`GH006`、`hook declined`、`protected branch update failed` 等）：
  - 告知用户：「目标分支 `TARGET_BRANCH` 已开启分支保护，禁止直接推送。请通过 Pull Request 进行代码合并。」
  - **跳过后续所有步骤**，直接进入第 11 步输出汇总报告，标注合并失败原因
- 否则，先执行 `git pull --rebase origin TARGET_BRANCH`，然后重试
- 若仍失败，报告错误

### 第 10 步：切回原分支

询问用户是否切回 `SOURCE_BRANCH`：

```bash
git checkout SOURCE_BRANCH
```

若 `STASH_CREATED=true`（第 7 步因 checkout 失败自动 stash）：

1. 切回源分支后，询问用户是否恢复 stash：
   ```bash
   git stash pop
   ```
2. 若恢复出现冲突：提示用户手动解决，或选择「❌ 不恢复，保留 stash」
3. 在汇总报告中标注：`Stash 恢复：已恢复 ✅ / 恢复冲突 ⚠️ / 未恢复（保留 stash）📦`

### 第 11 步：输出汇总报告

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

## 错误处理

- **非 Git 仓库**：告知用户当前目录不是 Git 仓库（前置检查 1）
- **远程仓库未配置**：告知用户未配置 origin，终止流程（前置检查 2）
- **Detached HEAD 状态**：警告用户并询问如何继续（前置检查 4）
- **进行中的 merge/rebase**：提示用户先完成或中止（前置检查 3）
- **存在 Stash**：合并后检查 `git stash list`，若有 stash 则告知用户是否需要 `git stash pop` 恢复
- **Stash 恢复冲突**：提示用户手动解决，或选择保留 stash
- **Checkout 目标分支失败**：自动 stash 保护 → 重试 checkout → 标记待恢复（第 7 步 7.2）
- **目标分支仅存在远程**：自动 `git checkout -b TARGET_BRANCH origin/TARGET_BRANCH`（第 7 步 7.1）
- **Pull 目标分支冲突**：展示冲突并询问用户处理（第 7 步 7.3）
- **源分支 = 目标分支**：走正常提交流程，跳过第 7~10 步，直接到第 11 步输出报告（第 1 步校验）
- **网络错误**：重试一次后明确报告
- **无已暂存变更**：根据第 1 步子流程 C 引导用户先暂存
- **未暂存/未跟踪文件**：根据第 1 步子流程 C / D / E 处理，不自动 add
- **破坏性操作（restore/reset --hard）**：必须二次确认，告知风险
- **无效分支参数**：分支不存在（本地和远程都没有）时，提示用户检查名称或先创建分支
- **分支名含特殊字符**：若分支名含 git 保留字符（`~` `^` `:` `?` `*` `[` 等），提示用户并停止
- **Commit 失败**：根据错误信息分类（hook 失败 / 邮箱未配置 / 空提交等），分别处理
- **推送失败（non-fast-forward）**：自动 fetch + rebase，rebase 冲突参考第 8 步处理
- **推送失败（权限不足）**：报告错误，提示检查 SSH Key / Token
- **新分支未设置上游**：自动使用 `git push -u origin <branch>`，无需用户干预

## 安全检查

执行前验证：
1. 用户处于项目根目录或 Git 仓库的子目录中（前置检查 1）
2. 远程仓库（origin）已配置（前置检查 2）
3. 无进行中的合并或 rebase（前置检查 3）
4. 无 Detached HEAD 状态（前置检查 4）
5. **目标分支存在**（本地或远程任一即可，通过 `git branch -a | grep TARGET_BRANCH` 检查）
6. 仅有已暂存的文件会被提交（默认行为，不再自动 `git add`）
7. 目标分支名合法（不含特殊字符如 `~` `^` `:` `?` `*` `[` 等，git 保留字符）
8. 若源分支 = 目标分支，跳过合并流程（第 1 步校验）
