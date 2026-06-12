# New API 二次开发 Git 协作与官方同步指南

> 适用于基于 [QuantumNous/new-api](https://github.com/QuantumNous/new-api) 进行长期二次开发的团队。  
> 本项目采用“双远程仓库 + 分支隔离”的方式：`main` 负责同步官方代码，`custom` 负责团队二次开发。

---

## 1. 仓库关系

| 名称 | 地址 | 用途 |
|---|---|---|
| `origin` | `https://github.com/lien0219/my-new-api.git` | 团队自己的 GitHub 仓库 |
| `upstream` | `https://github.com/QuantumNous/new-api.git` | New API 官方仓库 |

查看当前远程仓库：

```bash
git remote -v
```

正常情况下应类似：

```text
origin    https://github.com/lien0219/my-new-api.git (fetch)
origin    https://github.com/lien0219/my-new-api.git (push)
upstream  https://github.com/QuantumNous/new-api.git (fetch)
upstream  no_push (push)
```

建议禁止向官方仓库误推送：

```bash
git remote set-url --push upstream no_push
```

---

## 2. 分支约定

| 分支 | 用途 | 是否允许直接开发 |
|---|---|---|
| `main` | 同步官方 New API 源码 | 禁止 |
| `custom` | 团队二次开发集成分支 | 不建议直接提交 |
| `feature/*` | 新功能开发 | 允许 |
| `fix/*` | Bug 修复 | 允许 |
| `refactor/*` | 重构 | 允许 |
| `release/*` | 发布前测试与修复 | 按需创建 |

### 分支关系

```text
upstream/main
      │
      ▼
origin/main
      │
      ▼
origin/custom
      │
      ├── feature/user-center
      ├── feature/payment
      ├── fix/login-error
      └── refactor/channel-service
```

### 核心原则

1. `main` 只用于同步官方源码，不提交业务代码。
2. 团队二次开发代码统一合并到 `custom`。
3. 每个需求从最新的 `custom` 创建独立分支。
4. 功能完成后通过 Pull Request 合并到 `custom`。
5. 禁止向 `main` 强制推送。
6. 禁止将 `.env`、密码、Token、密钥提交到仓库。

---

## 3. 新成员首次拉取项目

### 3.1 克隆团队仓库

```bash
git clone https://github.com/lien0219/my-new-api.git
cd my-new-api
```

### 3.2 添加官方仓库

```bash
git remote add upstream https://github.com/QuantumNous/new-api.git
```

禁止误推官方仓库：

```bash
git remote set-url --push upstream no_push
```

检查：

```bash
git remote -v
```

### 3.3 拉取所有分支

```bash
git fetch --all --prune
```

切换到二次开发分支：

```bash
git switch custom
```

如果本地还没有 `custom`：

```bash
git switch -c custom --track origin/custom
```

---

## 4. 日常功能开发流程

开始开发前，先更新本地 `custom`：

```bash
git switch custom
git pull --ff-only origin custom
```

创建功能分支：

```bash
git switch -c feature/功能名称
```

例如：

```bash
git switch -c feature/user-management
```

完成开发后提交：

```bash
git status
git add .
git commit -m "feat: 新增用户管理功能"
git push -u origin feature/user-management
```

然后在 GitHub 创建 Pull Request：

```text
feature/user-management → custom
```

代码评审、测试通过后再合并。

---

## 5. 同步官方最新代码

官方更新必须先进入 `main`，验证后再合并到 `custom`。

建议由项目负责人统一执行。

### 5.1 确保工作区干净

```bash
git status
```

应看到：

```text
nothing to commit, working tree clean
```

如果有未提交代码，应先提交或暂存：

```bash
git stash push -u -m "临时保存：同步官方代码前"
```

---

### 5.2 获取官方最新代码

```bash
git fetch upstream
```

查看官方最近提交：

```bash
git log upstream/main --oneline -10
```

查看本地 `main` 与官方的差异：

```bash
git log main..upstream/main --oneline
```

---

### 5.3 更新本地 `main`

```bash
git switch main
git pull --ff-only origin main
git merge --ff-only upstream/main
```

说明：

- `git pull --ff-only origin main`：先确保本地 `main` 与团队仓库一致。
- `git merge --ff-only upstream/main`：只允许快进同步，避免在 `main` 上产生额外合并提交。

---

### 5.4 推送到团队仓库

```bash
git push origin main
```

此时：

```text
upstream/main = 本地 main = origin/main
```

---

### 5.5 将官方更新合并到 `custom`

```bash
git switch custom
git pull --ff-only origin custom
git merge main
```

如果没有冲突，完成测试后推送：

```bash
git push origin custom
```

完整同步命令：

```bash
git fetch upstream

git switch main
git pull --ff-only origin main
git merge --ff-only upstream/main
git push origin main

git switch custom
git pull --ff-only origin custom
git merge main
git push origin custom
```

---

## 6. 推荐的安全同步流程

官方更新可能影响现有二次开发功能，建议不要合并后立即部署。

### 6.1 创建同步分支

```bash
git switch custom
git pull --ff-only origin custom
git switch -c sync/upstream-YYYYMMDD
```

例如：

```bash
git switch -c sync/upstream-20260612
```

### 6.2 合并官方 `main`

```bash
git merge main
```

### 6.3 解决冲突并完成测试

至少检查：

- 项目能否正常启动
- 数据库迁移是否正常
- 登录、渠道、令牌等核心功能是否正常
- 自定义页面和接口是否受到影响
- Docker 构建是否成功
- 前后端自动化测试是否通过

### 6.4 推送同步分支

```bash
git push -u origin sync/upstream-20260612
```

然后创建 Pull Request：

```text
sync/upstream-20260612 → custom
```

评审、测试通过后再合并。

此方式比直接在 `custom` 执行 `git merge main` 更适合多人团队。

---

## 7. 处理代码冲突

执行：

```bash
git merge main
```

如果出现：

```text
CONFLICT (content): Merge conflict in ...
Automatic merge failed
```

查看冲突文件：

```bash
git status
```

冲突内容通常如下：

```text
<<<<<<< HEAD
团队二次开发代码
=======
官方最新代码
>>>>>>> main
```

处理原则：

1. 理解官方代码变更目的。
2. 保留仍然需要的二次开发逻辑。
3. 不要直接选择“全部保留当前”或“全部保留传入”。
4. 修改后执行完整测试。

解决完成后：

```bash
git add 冲突文件
git commit -m "merge: 同步官方更新并解决冲突"
git push
```

如果想取消本次合并：

```bash
git merge --abort
```

---

## 8. 同步前创建备份

同步官方代码前，可以创建备份分支：

```bash
git switch custom
git pull --ff-only origin custom
git branch backup/custom-before-sync-YYYYMMDD
git push origin backup/custom-before-sync-YYYYMMDD
```

例如：

```bash
git branch backup/custom-before-sync-20260612
git push origin backup/custom-before-sync-20260612
```

需要恢复时：

```bash
git switch custom
git reset --hard backup/custom-before-sync-20260612
git push --force-with-lease origin custom
```

> `git push --force-with-lease` 属于高风险操作，必须经过团队确认后再执行。

---

## 9. 官方同步出现异常

### 9.1 无法快进

如果执行：

```bash
git merge --ff-only upstream/main
```

出现：

```text
fatal: Not possible to fast-forward, aborting.
```

说明本地或远程 `main` 已经存在自定义提交。

先检查：

```bash
git log --oneline --graph --decorate --all -30
```

如果确认 `main` 应完全跟随官方，可由负责人执行：

```bash
git switch main
git fetch upstream
git reset --hard upstream/main
git push --force-with-lease origin main
```

> 执行前必须确认 `main` 上没有需要保留的团队代码。

---

### 9.2 本地存在未提交修改

可以临时保存：

```bash
git stash push -u -m "同步官方前临时保存"
```

同步完成后恢复：

```bash
git stash pop
```

---

### 9.3 GitHub 网络超时

如果团队成员使用 Clash Verge，且本地 HTTP 代理端口为 `7897`，可以配置：

```bash
git config --global http.proxy http://127.0.0.1:7897
git config --global https.proxy http://127.0.0.1:7897
```

查看配置：

```bash
git config --global --get http.proxy
git config --global --get https.proxy
```

不再需要代理时清除：

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

> 代理端口以每个人本机代理软件显示的实际端口为准，不一定都是 `7897`。

---

## 10. Commit 提交规范

推荐采用 Conventional Commits：

| 类型 | 用途 |
|---|---|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `refactor` | 代码重构 |
| `perf` | 性能优化 |
| `docs` | 文档修改 |
| `style` | 格式修改，不影响逻辑 |
| `test` | 测试相关 |
| `build` | 构建系统或依赖修改 |
| `ci` | CI/CD 修改 |
| `chore` | 其他维护工作 |
| `merge` | 合并或官方同步 |

示例：

```bash
git commit -m "feat: 新增自定义渠道配置"
git commit -m "fix: 修复令牌额度统计异常"
git commit -m "refactor: 重构模型路由逻辑"
git commit -m "docs: 更新部署说明"
git commit -m "merge: 同步 New API 官方更新"
```

---

## 11. Pull Request 规范

PR 标题示例：

```text
feat: 新增用户管理功能
fix: 修复渠道测试超时
sync: 同步官方 2026-06-12 更新
```

PR 描述建议包含：

```markdown
## 修改内容

- 新增……
- 调整……
- 修复……

## 影响范围

- [ ] 前端
- [ ] 后端
- [ ] 数据库
- [ ] Docker
- [ ] 配置文件

## 测试结果

- [ ] 本地启动成功
- [ ] 构建成功
- [ ] 核心功能测试通过
- [ ] 数据库迁移验证通过

## 相关截图或日志

填写截图、日志或说明。
```

---

## 12. 禁止提交的内容

以下文件或信息禁止提交：

```text
.env
.env.local
.env.production
数据库账号和密码
Redis 密码
GitHub Token
第三方 API Key
SESSION_SECRET
CRYPTO_SECRET
私钥和证书
运行日志
数据库文件
生产环境备份
```

提交前执行：

```bash
git status
git diff --cached
```

确认没有敏感信息后再提交。

如果密钥已经被提交，不能只删除文件，必须立即：

1. 吊销并重新生成密钥；
2. 清理 Git 历史；
3. 检查 CI/CD 和服务器配置；
4. 通知项目负责人。

---

## 13. 常用命令速查

### 查看当前分支

```bash
git branch --show-current
```

### 查看状态

```bash
git status
```

### 查看远程仓库

```bash
git remote -v
```

### 获取所有远程更新

```bash
git fetch --all --prune
```

### 更新当前分支

```bash
git pull --ff-only
```

### 查看分支图

```bash
git log --oneline --graph --decorate --all -30
```

### 查看官方新增提交

```bash
git log main..upstream/main --oneline
```

### 查看官方代码变更

```bash
git diff main..upstream/main
```

### 删除已经合并的本地分支

```bash
git branch -d feature/功能名称
```

### 删除远程分支

```bash
git push origin --delete feature/功能名称
```

---

## 14. 团队标准流程总结

### 功能开发

```text
更新 custom
    ↓
创建 feature/* 或 fix/* 分支
    ↓
开发、测试、提交
    ↓
推送远程分支
    ↓
创建 PR 到 custom
    ↓
代码评审
    ↓
合并
```

### 官方同步

```text
fetch upstream
    ↓
upstream/main 快进同步到 main
    ↓
推送 origin/main
    ↓
创建 sync/upstream-* 分支
    ↓
合并 main
    ↓
解决冲突并测试
    ↓
创建 PR 到 custom
    ↓
评审后合并
```

---

## 15. 最重要的三条规则

1. **`main` 只同步官方代码，不写业务代码。**
2. **所有团队二次开发代码最终合并到 `custom`。**
3. **官方更新先进入同步分支，测试通过后再合并到 `custom`。**
