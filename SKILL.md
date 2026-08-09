---
name: github-skill-publisher
description: |-
  把本地 Agent / Skill / 工具包标准化发布到 GitHub 公开仓库。当用户要把本地文件转成开源 skill 上传时触发。
  完整工作流：.git 剔除 → 脱敏检查（含 URL 内嵌凭据）→ 合规审查 → 元数据检查（LICENSE 署名 / frontmatter name）→
  gh/git 建仓推送（API 降级）→ topics（三系列）/ description / release → 语法与远端验证 → 交接回执。
  封装了全部踩坑：.git/config 明文 PAT 泄漏、auto_init README 覆盖、URL 转码、私改公开、topic 清理。
agent_created: true
---

# GitHub Skill Publisher

## Purpose

把本地任意 Agent / Skill / 工具包按统一标准发布到 GitHub 公开仓库。

触发条件：
- 用户说"把这个上传到 GitHub / 发布到 GitHub / 推到我的 GitHub"
- 任何本地 skill / agent / 工具包要开源化
- 批量发布 / 账号级整理（归类、归档、改 topics）

## 标准化输出

每个发布的仓库必须满足（38 仓实操验证过的标准）：

| 维度 | 要求 |
|------|------|
| 文件结构 | `SKILL.md` + `README.md` + `LICENSE`（MIT）+ `.gitignore` + `references/` + `assets/` |
| **不得包含** | `.git/` 目录、`__pycache__/`、`.DS_Store`（打包/发布前必须剔除） |
| LICENSE 署名 | 统一 `Copyright (c) 2026 Shi Yan`（不允许空署名 / 混用其他名字） |
| frontmatter | `name` 必须 = 目录名 = 仓库名（如 `local-material-batch-skill` 目录里 name 不能是 `local-material-batch`） |
| 语言 | **中文优先**，5 段结构（项目说明 / 适合谁用 / 快速开始 / 文件说明 / 推荐流程） |
| description | 用「卡点+方案+结果」口语化结构，**不写书名号**、**不写"X 的人"死板定义** |
| topics | 三系列 topics + 领域/类型 topics（见「系列与套件规范」） |
| release | **新包 v0.1.0 / 迭代升级 v1.x.0**，notes 写明脱敏保证 |
| 合规 | 涉及第三方平台 API 的包必须带 Compliance 声明（见「合规审查」） |

**description 改写规则**：
- 四块信息必到：谁用 + 卡在啥时 + 怎么干 + 拿到啥
- 问号起头、逗号接方案、句尾具体物
- 数字实在（5 套、30 秒、1 分钟、100+）
- 不写：开源/灵活/可扩展/智能/什么场景/一站式

## 标准工作流

### Step 0: 准备（.git 剔除，**绝对不能跳过**）

**最严重事故：发布包把 `.git/config` 一起带上，remote URL 内嵌明文 PAT 泄漏到公开仓库。** 本地源目录是 git 仓库时：

```bash
# 方式 A：干净导出（推荐，保留 .git 在源目录）
cd <源目录> && git clean -ndf && git stash list  # 先确认工作区干净
mkdir -p /tmp/release-<name> && cd <源目录> && tar --exclude=.git --exclude=__pycache__ --exclude=.DS_Store -cf - . | tar -xf - -C /tmp/release-<name>/

# 方式 B：直接发布本地目录时，确认源目录无 .git（`ls -d .git` 无输出）
# 检查 zip 是否含 .git：unzip -l <pkg.zip> | grep '\.git/config'
```

**反向检查**：解压别人给的 zip 发布时，先 `unzip -l x.zip | grep -E '\.git/|__pycache__|\.DS_Store'`，有则先剔除再发。

### Step 1: 脱敏检查（**绝对不能跳过**）

发布前**逐文件扫描**以下高风险模式（`scripts/publish.py` 的 `privacy_scan()` 已封装）：

```bash
patterns=(
  '<USER_DIR>/'           # 本地绝对路径（用户根目录 = 个人身份）
  'https?://[^/\s:]+:[^@\s]+@'  # URL 内嵌凭据（https://user:PAT@github.com/...）
  '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}'  # 邮箱
  '1[3-9][0-9]{9}'   # 手机号
  'sk-[A-Za-z0-9]{20,}'  # OpenAI key
  'ghp_[A-Za-z0-9]{20,}' / 'github_pat_[A-Za-z0-9]{20,}' / 'gho_'  # GitHub PAT
  'api[_-]?key|secret|password|token'  # 凭据关键词
  '<YOUR_NAME>|<OTHER_NAME>'  # 真实人名（按需调整）
)
```

**注意**：`privacy_scan()` 自动跳过规则定义自身（publish.py / SKILL.md）和 `__pycache__` / `.DS_Store`；**`.git` 不跳过**——`.git/config` 的内嵌凭据是重点扫描目标，保证 Step 0 遗漏时能兜底拦截。

**判断标准**：
- 任何 API key / token / secret / 邮箱 / 手机号 / 真实人名 / 直播间 ID → 标记为不能公开
- 通用脚本代码 + 通用 README + 配置文件（无密钥） → 可公开
- 内部编号（S01-S23 之类）→ 改成占位符（`{material_id}`）
- 硬性排除规则（如"排除施言"）→ 改成可配置参数（`speaker_filter`）
- 引流句（如"关注施言"）→ 改成参数（`{creator_name}`）

**报告 → 用户拍板 → 再执行**。绝不自己拿主意开公开。

### Step 2: 合规审查（涉及第三方平台时必做）

账号红线：**不爬平台、不绕风控、不自动发布。所有 Skill 只处理"用户已有"的内容。**

发布前对照检查：

| 场景 | 处理 |
|------|------|
| 抓取他人账号 / 绕过风控 / 逆向私有 API | **不上传**（douyin-video-transcript 事故教训：TikHub 逆向接口，靠 GitHub secret scanning 才兜底） |
| 调用官方公开 API（如 Discourse API） | 保留但加 Compliance 声明 + 默认限速对齐服务端配额（60 次/分钟 → delay ≥ 1.0s） |
| 处理"用户自己的内容"（本机文件 / 自己的账号） | 正常，README 注明「仅限本人或已获授权内容」 |
| 描述里出现「绕过」「逆向」「抓取他人」等词 | 中性化改写，强调合规边界 |

Compliance 声明模板（README 顶部）：

```
> ⚠️ **合规使用声明**：仅限处理本人账号或已获授权的内容，请遵守平台协议与适用法律，勿二次分发。
```

### Step 3: 文件结构对齐 + 元数据检查

最小集：

```
<repo-name>/
├── README.md           # 中文,5 段结构
├── SKILL.md            # 可复用的技能说明,frontmatter name = 目录名
├── LICENSE             # MIT,署名 "Copyright (c) 2026 Shi Yan"
├── .gitignore          # Python/编辑器/运行产物
├── references/         # 深度文档
│   └── *.md
└── assets/             # 模板/示例
    └── *.md
```

发布前三查（38 仓整理实际踩中过的）：
1. **LICENSE 署名**：`grep -i "copyright" LICENSE` → 必须为 `Copyright (c) 2026 Shi Yan`（原包出现空署名 / shiyan / Shiyan 共 6 种不一致）
2. **frontmatter name**：`grep '^name:' SKILL.md` → 必须与目录名、仓库名一致
3. **跨平台脚本标注**：未在对应平台实机复测的脚本（如 Windows `.ps1`）必须在 README 标注「××侧为适配初版，尚未实机复测」，避免用户误用

### Step 4: GitHub 操作（gh/git 优先，API 降级）

**主流程（gh CLI + git，保留完整提交历史，最快）**：

```bash
# 1. 建仓并推送（自动 --public）
cd <发布目录> && git init -b main
git -c user.name="Shi Yan" -c user.email="shiyan521@users.noreply.github.com" add -A
git -c user.name="Shi Yan" -c user.email="shiyan521@users.noreply.github.com" commit -m "feat: 发布 <name>"
gh repo create shiyan521/<repo-name> --public --source=. --push

# 2. 补 topics（三系列 + 领域）
gh repo edit shiyan521/<repo-name> --add-topic series-content-workflow --add-topic chinese

# 3. 发 release（新包 v0.1.0 / 迭代 v1.x.0）
gh release create v0.1.0 --title "v0.1.0" --notes "## 首发版本
[一句话描述]
### 包含内容
- ...
### 脱敏保证
- ..."
```

**降级流程（github.com:443 不通但 api.github.com 可用时）**：

```bash
# 方式 A：publish.py 纯 API 上传（scripts/publish.py，已封装全部 API 坑）
export GH_TOKEN='...'
python3 scripts/publish.py <owner> <repo> <本地目录> "<description>"

# 方式 B：单文件更新（README 等），绕过 git 协议
gh api -X PUT repos/<owner>/<repo>/contents/<file> \
  -f message="update: <说明>" \
  -f content="$(base64 -i <file> | tr -d '\n')" \
  -f sha="$(gh api repos/<owner>/<repo>/contents/<file> -q '.sha')"
```

**注意**：方式 B 会与本地 git 历史发散，后续如需本地 git 管理，先 `git fetch && git reset --soft origin/main` 对齐。

**归档 / 恢复公开**（非交互必须带 `--yes`）：

```bash
gh repo archive shiyan521/<repo> --yes        # 归档（只读，visibility 不变）
gh repo unarchive shiyan521/<repo> --yes      # 恢复
gh repo edit shiyan521/<repo> --visibility private --accept-visibility-change-consequences  # 私有化（归档仓库须先 unarchive）
```

**红线**：不删除仓库（归档/私有即可）；删除需用户明确要求且不可逆。

### Step 5: 验证（发布后必做）

1. **本地语法检查**：
   - `.sh`：`bash -n <file>`
   - `.py`：`python3 -m py_compile <file>`
   - `.ps1`：`pwsh -NoProfile -Command "[System.Management.Automation.Language.Parser]::ParseFile('<file>', [ref]$null, [ref]$null)"`
2. **引用完整性**：README / SKILL.md 里引用的仓库、文件、链接必须真实存在（`gh repo view` 核实或 `curl -sI` 验 200）
3. **远端脱敏复扫**：curl 拉取 raw 文件，用 Step 1 的 grep 再扫一遍
4. **WebFetch 主页**：确认 README 渲染正常、无空链接
5. **交接回执**：发布结果写回交接状态卡（仓库 URL / 分支 / tag / 验证结论）

### Step 6: 收尾

1. 账号级同步：新增仓库同步进 profile README（系列分区导航，对标花叔模式）
2. `unset GH_TOKEN` —— 凭据立即从环境清空
3. 临时文件 `rm -rf /tmp/xxx-release` —— 不留痕
4. 写 memory：今天推了几个仓、commit hash、踩了什么坑

## 系列与套件规范

### 三系列 topics（账号级分类体系）

每个仓库按内容挂一个系列 topic，另加领域/类型 topics：

- `series-content-workflow` —— 内容生产管线（选题/写作/发布/复盘）
- `series-agent-engineering` —— Agent / Skill 工程（方法论、审计、交互）
- `series-system-governance` —— 系统治理（存储/性能/网络/安全审计）

例：`podcast-chat-prep` → `series-content-workflow`；`system-storage-safe-auditor` → `series-system-governance`

### 套件索引仓模式

多包组成套件时（如 6 个系统治理包），建一个索引仓（如 `system-governance-suite`）做路由，各成员包 README 顶部互相标注归属：

```
> 本包属于 **<suite-name>**（<一句话描述>）的一部分，可与套件内其他包组合使用：<成员列表>。
```

索引仓不替代独立仓库，成员仍是独立可安装的包。

## 踩过的坑（必看）

| 坑 | 现象 | 解决 |
|----|------|------|
| **`.git/config` 明文 PAT 泄漏**（最严重） | 发布包带 .git 目录，remote URL 内嵌 `https://user:PAT@github.com/...`，公开仓库泄露凭据 | Step 0 强制剔除；GitHub secret scanning 会自动 revoke（401 验证），但绝不能依赖它兜底 |
| `auto_init` README 覆盖 | PUT 返回 422 "sha wasn't supplied" | 先 GET 拿 sha 再 PUT |
| 中文路径 URL | `urllib.error.HTTPError 404` | `urllib.parse.quote(path, safe='/')` |
| 清理绝对路径时误删链接语法 | 渲染后 5 个目录链接空白 | 重写 README,补完整 `[text](path)` |
| 私有仓改公开 | `private` topic 残留 | 改完 visibility 立即重设 topics |
| 描述拼接中英混排 | 仓库主页"中英缝合感" | 全部用纯中文一句话 |
| commit message 中文逗号 | zsh 显示"nothing to commit" 误报 | 实际成功,看 git log 确认 |
| 凭据来源失效 | Keychain PAT 读不到、gh CLI 未登录 | 必问用户贴 PAT,只用一次立即清空 |
| 归档仓库操作失败 | PATCH/PUT 报 "archived so was read-only" | 先 unarchive 再操作，完事再 archive 回去 |
| 非交互命令缺参数 | `gh repo archive` 报 "--yes required" | 归档/删除类命令一律带 `--yes` |

## 客观挑刺清单（必做项）

| 类型 | 常见问题 | 必查 |
|------|---------|------|
| 隐私 | `.git` / `__pycache__` / `.DS_Store` 混入发布 | 必查 |
| 隐私 | URL 内嵌凭据、绝对路径、邮箱、手机号、硬编码密钥 | 必查 |
| 隐私 | 旧版本 `private` / `wip` / `archived` topic 残留 | 改完 visibility 必查 |
| 元数据 | LICENSE 署名不是统一格式 | 必查 |
| 元数据 | frontmatter name ≠ 目录名 ≠ 仓库名 | 必查 |
| 元数据 | description / topics / release 三者一致 | 必查 |
| 合规 | 涉及抓取/逆向/绕风控但无 Compliance 声明 | 必查 |
| 链接 | README 清理时把 `[text](path)` 语法误删 | 必查 |
| 元数据 | `chinese` topic 多仓重复 | 收益低,不修 |
| 流程 | commit message / Issues 模板 | 收益低,不修 |

## 适用场景

- 本地 Agent / Skill / 工具包的开源化
- 私有仓改公开（必带逐文件脱敏审查）
- 旧仓库中文化 / 重写
- 批量仓库的 description / topics 同步
- 账号级整理（系列 topics 分类、套件索引、归档治理）
