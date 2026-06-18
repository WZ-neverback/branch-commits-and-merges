# 发布与使用流程

> `commit-and-merge` Skill 的 npm 发布与使用文档。
> 记录完整的发布步骤、常见问题处理，以及使用方式。

## 包信息

| 项 | 值 |
|----|----|
| **包名** | `commit-and-merge` |
| **版本** | `0.1.0` |
| **作者** | `wz-never-back` |
| **License** | MIT |
| **仓库源** | `https://registry.npmjs.org/` |
| **打包内容** | `SKILL.md`、`README.md`、`package.json` |
| **包大小** | 11.1 kB（unpacked 32.9 kB） |

## 一、首次发布流程

### 1. 准备 `package.json`

```json
{
  "name": "commit-and-merge",
  "version": "0.1.0",
  "description": "自动化 Git 工作流：lint 检查 → 提交已暂存变更 → 合并到目标分支",
  "type": "module",
  "main": "SKILL.md",
  "files": [
    "SKILL.md",
    "README.md"
  ],
  "keywords": [
    "qoder",
    "trae",
    "skill",
    "git",
    "commit",
    "merge",
    "workflow"
  ],
  "author": "wz-never-back",
  "license": "MIT"
}
```

### 2. 切换到官方 npm 源

```bash
# 检查当前源
npm config get registry

# 切换到官方源（默认可能是 npmmirror）
npm config set registry https://registry.npmjs.org/

# 验证
npm config get registry
# → https://registry.npmjs.org/
```

### 3. 检查包名是否可用

```bash
npm view commit-and-merge name version --registry=https://registry.npmjs.org/
```

- 返回 404 → 可用 ✅
- 返回版本号 → 已被占用，需改名（加 scope 如 `@wz-never-back/commit-and-merge`）

### 4. 创建账号 + 开启 2FA

1. 访问 https://www.npmjs.com/signup 注册账号
2. 访问 https://www.npmjs.com/settings/<username>/tfa 开启 2FA
   - **Authy / Google Authenticator**：TOTP（6 位码，30 秒刷新）
   - **安全钥匙（WebAuthn）**：浏览器生物识别 / 硬件 key

### 5. 创建 Granular Access Token（推荐）

> **说明**：用 Granular Token + Bypass 2FA 可以一劳永逸地避免每次发布都要输入 OTP。

访问 https://www.npmjs.com/settings/<username>/tokens：

1. 删除旧 token（如有）
2. 点击 **"Generate New Token"** → **"Granular Access Token"**（不要选 Classic Token）
3. **Token name**：自定义，如 `publish-commit-and-merge`
4. **Expiration**：选 7 天或 30 天
5. ⚠️ **勾选 "Bypass two-factor authentication"**（关键，页面最底部）
6. **Packages and scopes**：选 **"All packages"** + **"Read and write"**
7. 点 **"Generate Token"** → **立刻复制** `npm_xxxxx`（关掉页面就看不到了）

### 6. 写入 token 到 `.npmrc`

在项目根目录创建 `.npmrc`：

```powershell
# PowerShell
"//registry.npmjs.org/:_authToken=npm_你的token" | Out-File -Encoding ascii .npmrc
```

或手动创建文件 `.npmrc`，内容：

```
//registry.npmjs.org/:_authToken=npm_xxxxx
```

### 7. ⚠️ 把 `.npmrc` 加入 `.gitignore`

```bash
echo ".npmrc" >> .gitignore
```

> 防止 token 泄露到 git 仓库。

### 8. 执行发布

```bash
npm publish
```

成功输出：

```
npm notice Publishing to https://registry.npmjs.org/
+ commit-and-merge@0.1.0
```

## 二、版本更新流程

```bash
# 1. 修改代码 / 文档
# 2. 更新版本号
npm version patch   # 0.1.0 → 0.1.1（bug 修复）
npm version minor   # 0.1.0 → 0.2.0（新功能，向后兼容）
npm version major   # 0.1.0 → 1.0.0（破坏性变更）

# 3. 发布
npm publish
```

> `npm version` 会自动创建 git tag 和 commit。

## 三、撤销发布

```bash
# 撤销指定版本（仅 72 小时内有效）
npm unpublish commit-and-merge@0.1.0

# 强制撤销整个包（需联系 npm 支持）
npm unpublish commit-and-merge --force
```

## 四、使用方式

### 4.1 作为 Trae IDE Skill 使用

```bash
# 全局安装
npm install -g commit-and-merge

# 查看 SKILL.md 内容
cat "$(npm root -g)/commit-and-merge/SKILL.md"
```

将 `SKILL.md` 复制到 Trae IDE 的 skill 目录：

- Windows: `%USERPROFILE%\.trae\skills\commit-and-merge\SKILL.md`
- macOS: `~/.trae/skills/commit-and-merge/SKILL.md`

### 4.2 在项目里引用

```bash
# 项目内安装
npm install --save-dev commit-and-merge
```

在 `package.json` 中通过 `files` 字段已经包含了 `SKILL.md`。

### 4.3 直接使用

```bash
# npx 方式（无需安装）
npx commit-and-merge --preview
```

## 五、常见问题（FAQ）

### Q1: `ENEEDAUTH` 报错

**原因**：token 没生效或写入位置不对。

**排查**：

```bash
# 1. 检查项目 .npmrc
cat .npmrc

# 2. 检查全局 .npmrc
cat ~/.npmrc   # Windows: C:\Users\<user>\.npmrc

# 3. 检查环境变量
echo $NPM_TOKEN
```

**解决**：在项目根目录创建 `.npmrc`，写入 `_authToken`。

### Q2: `403 Forbidden - Two-factor authentication required`

**原因**：token 没有 bypass 2FA 权限，或 token 过期。

**解决**：重新生成 Granular Token，**勾选 "Bypass two-factor authentication"**。

### Q3: WebAuthn 流程卡住

**现象**：`npm publish` 提示 `Open https://www.npmjs.com/login/xxx to use your security key`

**原因**：账号 2FA 用了浏览器安全钥匙认证。

**解决**：改用 TOTP 验证码（`--otp=6位码`），或重新生成带 Bypass 2FA 的 Token。

### Q4: `EOTP` 报错

**原因**：TOTP 验证码过期（30 秒有效）。

**解决**：

```bash
# 等待新码刷新后立即执行
npm publish --otp=新6位码
```

### Q5: `403 - You do not have permission to publish`

**原因**：包名被占用，或账号权限不足。

**解决**：

1. 改用 scope 命名：`@wz-never-back/commit-and-merge`
2. 或换名：`commit-and-merge-skill`、`wz-commit-merge` 等

### Q6: PowerShell `&&` 报错

PowerShell 不支持 `&&`，分开执行：

```powershell
# 错误
npm config set registry https://registry.npmjs.org/ && npm config get registry

# 正确
npm config set registry https://registry.npmjs.org/
npm config get registry
```

## 六、安全注意事项

1. **永远不要把 `.npmrc` 提交到 git**
   ```bash
   echo ".npmrc" >> .gitignore
   ```

2. **Token 设置合理过期时间**：不要选 "Never"

3. **最小权限原则**：只勾选 "Publish" 权限，不要给 "Read & Write" 全开

4. **泄露后立即撤销**：访问 https://www.npmjs.com/settings/<username>/tokens 删除

5. **不要在聊天/截图里发 token**：发包出问题只贴错误信息，不要贴 token

## 七、发布检查清单

发布前逐项确认：

- [ ] `package.json` 中 `name`、`version`、`author`、`license` 都正确
- [ ] 当前 npm 源是 `https://registry.npmjs.org/`
- [ ] `.npmrc` 中 token 有效（带 Bypass 2FA）
- [ ] `.npmrc` 已在 `.gitignore` 中
- [ ] 包名在 npm 上未占用
- [ ] `SKILL.md`、`README.md` 已更新

## 八、参考链接

- npm 官方文档：https://docs.npmjs.com/
- 发布包：https://docs.npmjs.com/cli/v10/commands/npm-publish
- Granular Token：https://docs.npmjs.com/creating-and-viewing-access-tokens
- 2FA 配置：https://docs.npmjs.com/configuring-two-factor-authentication
