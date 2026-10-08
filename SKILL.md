---
name: github-batch-publish
description: 把本地自定义技能（~/.workbuddy/skills/ 下的目录）批量发布到 GitHub，一个技能一个独立仓库，含密钥脱敏、配套 MCP 源码归置、仓库描述与话题标签回填。当用户说"把技能上传到 GitHub""技能发到 GH""批量发布技能""技能同步到 GitHub""把对应的 MCP 也传上去"时使用。三个必踩的坑：①本机 github.com:443 被墙但 ssh 22 端口通，推送必须走 SSH；②GitHub 创建仓库有二级限流，连续建约 10 个就被临时封禁（约 1 小时）；③技能里普遍有明文 API 令牌，必须先脱敏再传。
agent_created: true
---

# 技能批量发布到 GitHub

把 `~/.workbuddy/skills/` 下的自研技能批量推到 GitHub，**一个技能 = 一个独立仓库**。

## 铁律（违反会出事）

1. **先问清范围再动手**：哪些技能要传、公开还是私有、密钥怎么处理、MCP 放哪。不问清就动手 = 返工。
2. **脱敏在推送前，不是推送后**。技能里几乎一定藏着明文 `sk-` 令牌、`.env`、账号信息。先扫、先换、再传。
3. **推送必须走 SSH**（见下方网络章节），HTTPS 在本机不通。
4. **不要改用户的 git 代理配置**。用户配了 `127.0.0.1:7890`，代理没开时 git 会失败，但那不是让你去改它——用 SSH 绕开即可。

## 三个必踩的坑

### 坑 1：HTTPS 推送不通，必须走 SSH

在本机实测（2026-10-01）：

- `github.com:443` 直连 → 超时 / Connection reset（被墙）
- `api.github.com`、`codeload.github.com` 直连 → 正常
- `ssh github.com:22` → 正常

所以 `gh` 命令行能用（走 api 域名），但 `git push` 走 https 一定失败。

**解法**：

```bash
# 1. 生成密钥（如果 ~/.ssh/id_ed25519 不存在）
ssh-keygen -t ed25519 -N "" -C "sheen945-workbuddy" -f "$HOME/.ssh/id_ed25519"

# 2. 公钥加到账号
gh ssh-key add "$HOME/.ssh/id_ed25519.pub" --title "WorkBuddy AutoPublish"

# 3. 验证
ssh -o StrictHostKeyChecking=no -T git@github.com   # 应回 Hi sheen945!
```

仓库 remote 一律用 `git@github.com:sheen945/<repo>.git`。

注意：`git -c http.proxy= push` 和 `GIT_CONFIG_GLOBAL=空文件` 这两种绕过代理的招**都没用**（绕过代理后直连 443 依然不通）。别浪费时间。

### 坑 2：GitHub 二级限流

连续创建约 10 个仓库后，账号被临时封禁内容创建：

```
You have exceeded a secondary rate limit and have been temporarily blocked
from content creation.
```

REST（`POST /user/repos`）和 GraphQL（`gh repo create`）**共用同一个限制池**，换接口没用。

**解法**：等待重试。实测第 1 轮（10 分钟）仍失败，第 2 轮（20 分钟）成功。写个重试循环，每 10 分钟试一次，最多 8 轮：

```bash
for i in $(seq 1 8); do
  sleep 600
  OUT=$(python _tools/publish.py $REPOS 2>&1)
  echo "$OUT"
  echo "$OUT" | grep -q "成功 N 个" && break
done
```

### 坑 3：技能里的明文密钥

扫描时必须覆盖的模式（用 Python 写，别用 grep 递归——本机 Git Bash 的 `grep --include` 有坑）：

```
sk-[A-Za-z0-9]{16,}          真实令牌
gh[pousr]_[A-Za-z0-9]{20,}   GitHub token
Bearer [A-Za-z0-9\-_\.=]{20,}
(TOKEN|SECRET|PASSWORD|KEY)\s*[=:]\s*["']?\S{12,}
1[3-9]\d{9}                  手机号
```

**替换策略**：

- 代码里的 `KEY = "sk-xxx"` → 改成 `os.environ.get("ENV_NAME", "sk-YOUR_KEY_HERE")`，保留可运行性
- 文档 / 命令示例里的 → 直接换占位符
- **顺序很重要**：先处理代码赋值行，再替换残留令牌。反了的话代码行会被当成纯字符串替换，改不成 env 形式

**注意误伤**：`sk-based-asynchronous-programming`（WPF 技术名）含 `sk-` 前缀，用 `{24,}` 长度限制可以避开。

## 标准流程

### 第 1 步：盘点 + 问清范围

```bash
ls -1 ~/.workbuddy/skills/          # 列出技能
gh repo list sheen945 --limit 100   # 已有的仓库，避免重名
gh auth status                      # 确认登录
```

带 `__skillhub` 后缀的是从技能商店下载的版本，一般不算自研，默认排除——但要问用户。

问清四件事：**范围**（哪些技能）、**可见性**（公开/私有）、**脱敏策略**、**MCP 怎么放**。

### 第 2 步：建工作区、复制、排除构建产物

工作区放在项目目录下 `<项目>/gh-publish/`，每个技能一个子目录。

复制时排除：

- 目录：`build`、`dist`、`__pycache__`、`.git`、`node_modules`、`.venv`、`chroma`、`logs`、`models`
- 后缀：`.pyc`、`.log`、`.db`、`.exe`、`.dll`、`.zip`
- 文件：`.env`、`.DS_Store`

保留 git 关联，方便后续更新。

### 第 3 步：脱敏 + 生成配套文件

每个仓库补上：

- `README.md` —— 由 `SKILL.md` 生成（剥掉 YAML frontmatter），否则 GitHub 首页什么都不显示
- `.gitignore` —— 排除 `.env`、`__pycache__`、`node_modules`、`*.db`、`build/`、`dist/`
- `.env.example` —— 有环境变量需求的技能才加

### 第 4 步：并入配套 MCP

自建 MCP 的源码位置通常在 `~/.workbuddy/mcp.json` 里能看到（`args` 指向的 .py 文件）。

放在对应技能仓库的 `mcp/` 子目录下，并写 `mcp/README.md`：目录结构 + `mcp.json` 配置片段（路径用占位符）+ 依赖说明。

**不要传**：数据库文件、向量索引（chroma）、嵌入模型（动辄 184M）。

### 第 5 步：建仓库 + 推送

```bash
# 新仓库：先建远端，再 SSH 推
gh repo create sheen945/<repo> --public|--private --description "<中文描述>"
cd <repo-dir>
git init -q -b main && git add -A && git commit -q -m "feat: 初始提交 —— <repo>"
git remote add origin git@github.com:sheen945/<repo>.git
git push -u origin main

# 已有仓库：clone 下来覆盖内容，保留提交历史
git clone git@github.com:sheen945/<repo>.git /tmp/x
# 清空工作区（留 .git）→ 复制新内容 → commit → push

# 回填描述与话题
gh repo edit sheen945/<repo> --description "..." --add-topic a,b,c
```

**注意**：`gh repo create --source=. --push` 会用 https 推送从而失败。要拆成「建仓库」和「SSH 推送」两步。

### 第 6 步：核验

```bash
gh api "repos/sheen945/<repo>" --jq '"\(.visibility)\t\(.default_branch)"'
gh api "repos/sheen945/<repo>/git/trees/HEAD?recursive=1" --jq '[.tree[]|select(.type=="blob")]|length'
```

**线上再扫一遍密钥**（不能只信本地脱敏），确认推送的是脱敏版。

## 可见性判断标准

含以下内容 → 私有：

- 真实令牌 / `.env`
- 平台账号运营类技能（闲鱼、小红书、巨量引擎等）
- 个人记忆档案、含真名与公司信息

通用方法论、无账号凭据 → 公开。

## 已发布的 MCP 型仓库（发布/更新时照抄这套规格）

已知带配套 MCP 的仓库，各自有必须遵守的排除规则：

- **`powerbi-crawler-data-prep`** → MCP 放 `mcp/powerbi-mcp/`
  - ⛔ **必须排除 `adomd/`**：97 MB 的微软 ADOMD/TOM DLL，既是版权物又太大。排除后整体仅 0.97 MB
  - 排除项：`adomd/`、`logs/`、`__pycache__/`、`*.dll`、`*.nupkg`、`*.p7s`
  - `mcp/README.md` 里必须写清「DLL 从 NuGet 自行获取」并给出两个 NuGet 包名，否则别人拿到跑不起来
- **`obsidian-notes-rag`** → `mcp/notes-rag/`
- **`memory-vault-sync`** → `mcp/mem0-shared-memory/` + `mcp/obsidian-vault/`

MCP 的 `.gitignore` 会一并带过去（嵌套生效），一般无需改；但**先看一眼**里面有没有排除掉你其实要传的目录。

## ⭐ 推送后必做：两级核验（2026-10-08 血泪补充）

**只跑 `gh repo view` 或看仓库描述是不够的。** 实测踩过：

> `powerbi-crawler-data-prep` 的仓库描述从 2026-09-30 起就写着「技能 + 配套 Power BI MCP 增强包」，
> 但线上 tree 里**根本没有 `mcp/` 目录** —— 那个增强包当时压根没推成功，白白漏了近 40 天。
> **仓库描述会骗人，文件树不会。**

所以核验必须做两步：

**① 文件树对账**（`mcp/` 在不在）

```bash
gh api "repos/sheen945/<repo>/git/trees/HEAD?recursive=1" --jq '.tree[]|select(.type=="blob")|"\(.sha) \(.path)"'
```

**② 指纹逐一比对**（推上去的是不是体检过的那份）—— 这是最硬的证据：

```bash
gh api "repos/sheen945/<repo>/git/trees/HEAD?recursive=1" \
  --jq '.tree[]|select(.type=="blob")|"\(.sha) \(.path)"' | sort > /tmp/remote.txt
git ls-files -s | awk '{print $2" "$4}' | sort > /tmp/local.txt
comm -3 /tmp/remote.txt /tmp/local.txt      # 输出为空 = 完全一致
comm -12 /tmp/remote.txt /tmp/local.txt | wc -l   # = 总文件数
```

`git ls-files -s` 输出里的第二个字段就是 **blob sha**，与线上 tree 的 `sha` 可直接对。
全部一致才敢说"推对了"——比肉眼看文件数可靠得多。

**③ 抽样看内容**：至少 `gh api .../contents/<高风险文件>` 抓一个含 `SECRET` / 路径字样的文件下来看一眼，
确认脱敏和路径泛化真的生效（别只信本地那个已经改过的副本）。

## 常见问题

- **`src refspec main does not match any`**：本地分支还叫 `master`，先 `git branch -M main`
- **git 提交报没有 user.name**：`git config --global user.name/email`，邮箱用 `<id>+<login>@users.noreply.github.com`
- **中文文件名 API 调用失败**：路径要 URL 编码
- **Git Bash 里 `gh api /user/repos` 报 "invalid API endpoint"**：把前导斜杠去掉，写成 `user/repos`
- **`shutil.copy2` 报 `WinError 3 系统找不到指定的路径`**：目标子目录（如 `references/`）还没建，先 `os.makedirs(os.path.dirname(d), exist_ok=True)`
- **更新已有仓库不要用 `gh repo create --source=. --push`**：它会走 https 推送从而失败。正解是
  `git clone git@github.com:... ` → 清空工作区（保留 `.git`）→ 覆盖新内容 → commit → `git push`，
  这样能保住提交历史
