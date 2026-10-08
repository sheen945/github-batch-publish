# github-batch-publish

把 `~/.workbuddy/skills/` 下的自研技能批量发布到 GitHub 的 WorkBuddy / Claude Code 技能——一个技能一个独立仓库，含密钥脱敏、配套 MCP 源码归置、仓库描述与话题回填，以及推送后的两级核验。

## 简介

这是一个面向 WorkBuddy 智能体的「技能批量发布」操作手册型技能。它不追求自动化脚本一把梭，而是把一次真实批量发布（2026-10-01 至 2026-10-08 期间多次踩坑、修正、补充）沉淀下来的**网络环境事实、限流规律、脱敏规则、六步标准流程和核验纪律**全部写清楚，让智能体每次执行时都能绕开已知陷阱。

它解决的问题：本地积累了几十个自研技能，需要同步到 GitHub 做备份、分享或版本管理；但直接 `git push` 会撞上 HTTPS 被墙、GitHub 二级限流、技能里藏着的明文 API 令牌这三座大山。本技能把这三座大山各自的绕过方案和顺序都固化了。

适合人群：维护 WorkBuddy / Claude Code 自研技能库、需要定期把技能批量推送到 GitHub 的用户；以及被指派执行该发布任务的 AI 智能体。

## 功能特性

- **四条铁律前置**：先问清范围再动手；脱敏在推送前而非推送后；推送必须走 SSH；绝不修改用户的 git 代理配置（用户配的 `127.0.0.1:7890` 代理没开时用 SSH 绕开，而不是去改它）。
- **三大坑的实测结论与解法**：
  - HTTPS 推送不通：本机实测 `github.com:443` 直连超时/被重置，但 `ssh github.com:22` 正常、`api.github.com` 正常——所以 `gh` CLI 可用、`git push` 走 HTTPS 必败，remote 一律用 `git@github.com:` 形式。已验证 `git -c http.proxy=` 和 `GIT_CONFIG_GLOBAL=` 空文件两种绕过法都无效，明确劝阻勿再尝试。
  - GitHub 二级限流：连续创建约 10 个仓库后账号被临时封禁内容创建，REST 与 GraphQL 共用同一限制池；解法是每 10 分钟重试一次、最多 8 轮的等待重试循环（实测第 2 轮即恢复）。
  - 明文密钥脱敏：给出 5 条必扫正则模式（`sk-` 令牌、`gh[pousr]_` token、Bearer、KEY/SECRET 赋值、手机号），规定替换顺序（先处理代码赋值行改成 `os.environ.get(...)` 保留可运行性，再替换文档残留占位符），并记录 `sk-based-asynchronous-programming` 这类误伤案例及用长度限制规避的方法。
- **六步标准流程**：盘点问范围 → 建工作区复制并排除构建产物 → 脱敏并生成配套文件（README.md 由 SKILL.md 剥掉 frontmatter 生成、.gitignore、按需 .env.example）→ 并入配套 MCP 源码（放 `mcp/` 子目录，不写数据库/向量索引/嵌入模型）→ 建仓库 + SSH 推送（明确禁止 `gh repo create --source=. --push`，须拆成建仓库与推送两步）→ 核验。
- **可见性判断标准**：含真实令牌、平台账号运营类技能（闲鱼/小红书/巨量引擎等）、个人记忆档案 → 私有；通用方法论、无账号凭据 → 公开。
- **已发布 MCP 型仓库的规格档案**：记录 `powerbi-crawler-data-prep`（必须排除 97 MB 的 `adomd/` DLL 并在 mcp/README.md 写明 NuGet 自助获取方式）、`obsidian-notes-rag`、`memory-vault-sync` 三个带 MCP 仓库的排除规则与目录约定，后续更新照抄。
- **推送后两级核验纪律**（2026-10-08 血泪补充）：仓库描述会骗人、文件树不会。必须做 ① 文件树对账（确认 `mcp/` 真的在）、② blob SHA 指纹逐一比对（`git ls-files -s` 对比线上 tree，`comm -3` 为空才算一致）、③ 抽样抓取高风险文件确认脱敏生效。
- **常见问题速查**：`src refspec main does not match any`（分支改名）、git user.name 未配置、中文路径需 URL 编码、Git Bash 下 `gh api` 端点前导斜杠坑、`shutil.copy2` 目标目录未建等六个已知故障的对应解法。

## 工作原理 / 技术栈

本技能是纯方法论手册（playbook），不含可执行脚本。执行时由智能体按手册逐步调用系统工具完成：

- **GitHub CLI（`gh`）**：建仓库、加 SSH 公钥、查仓库、回填描述与话题、调用 API 做文件树对账（走 `api.github.com`，本机直连正常）。
- **git over SSH**：所有推送走 `git@github.com:` 的 22 端口，绕开被墙的 443。
- **正则脱敏**：用 Python 执行扫描与替换（明确说明本机 Git Bash 的 `grep --include` 有坑，勿用 grep 递归）。
- **环境前提**：Windows 本机环境、`~/.workbuddy/skills/` 技能目录、已登录的 `gh`、已配置的 SSH 密钥（`~/.ssh/id_ed25519`）。

## 安装与使用

将本仓库目录复制到 WorkBuddy 技能目录即可：

```
~/.workbuddy/skills/github-batch-publish/
```

或项目级 `.workbuddy/skills/` 目录；Claude Code 用户放 `~/.claude/skills/`。

**触发方式**：对智能体说「把技能上传到 GitHub」「技能发到 GH」「批量发布技能」「技能同步到 GitHub」「把对应的 MCP 也传上去」等，即会加载本手册执行。

**执行流程概要**：

1. 盘点：`ls ~/.workbuddy/skills/`、`gh repo list` 查重名、`gh auth status` 确认登录；带 `__skillhub` 后缀的技能商店下载版默认排除（但需向用户确认）。
2. 向用户问清四件事：发布范围、公开/私有、脱敏策略、MCP 怎么放。
3. 在 `<项目>/gh-publish/` 建工作区，复制技能并排除 `build`、`dist`、`__pycache__`、`.git`、`node_modules`、`.venv`、`chroma`、`logs`、`models` 及 `.pyc/.log/.db/.exe/.dll/.zip`、`.env` 等。
4. 脱敏 + 生成 README.md / .gitignore / .env.example。
5. 有配套 MCP 的把源码归置到 `mcp/` 子目录并写 `mcp/README.md`。
6. `gh repo create` 建远端 → `git init -b main` → commit → SSH remote → push；已有仓库则 clone 后清空工作区（留 .git）覆盖内容再推，保住提交历史。
7. 两级核验：文件树对账 + blob SHA 指纹比对 + 高风险文件抽样，全过才算完成。

## 项目结构

```
github-batch-publish/
├── SKILL.md       # 技能定义（frontmatter + 完整操作手册，智能体加载入口）
├── README.md      # 本文件（中文说明）
├── README_EN.md   # 英文说明
└── .gitignore     # 常规排除规则
```

仓库不含脚本与 references——所有知识都内嵌在 SKILL.md / README 正文中，执行时由智能体直接调用系统命令完成。

## 注意事项

- **本技能包含大量本机环境特有的实测结论**（如 443 被墙、SSH 22 通、代理地址 `127.0.0.1:7890`），换网络环境后需重新验证网络章节再套用。
- **脱敏是不可回头的红线**：密钥一旦推上 GitHub，即使后续删除也会留在历史和缓存里，必须先扫后传，推送后还要在线上再扫一遍。
- **二级限流期间不要反复猛试**：REST 和 GraphQL 共用限制池，换接口无用，按 10 分钟间隔重试即可。
- **不要自作主张改用户的 git/代理配置**；也不要为了省事用 `gh repo create --source=. --push`（内部走 HTTPS，必败）。
- 核验时**仓库描述不可信**——必须以线上文件树和 blob 指纹为准，曾发生描述写着「含 MCP 增强包」但 `mcp/` 实际未推送成功、漏了近 40 天的事故。
- 文中示例中的用户名 `sheen945` 为作者账号，使用时替换为自己的 GitHub 用户名。

## License

MIT

## 作者

sheen945
