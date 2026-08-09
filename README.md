# GitHub Skill Publisher

本地 Agent / Skill / 工具包想发到 GitHub 公开仓库？不想每次踩一遍 `.git/config` 明文 PAT 泄漏、auto_init README 覆盖、中文路径 404、private topic 残留这些坑？跑这一套，10 分钟发完 6 文件 + topics + release。

## 适合谁用

- 经常把本地 skill / agent / 工具包开源化的开发者
- 想给多个 GitHub 仓库做统一 description / topics 同步的人
- 需要把私有仓改公开、又怕误发个人信息和凭据的协作者
- 做账号级整理（系列 topics 分类、套件索引、归档治理）的人

## 快速开始

### 方式 A：gh/git 主流程（推荐）

```bash
# 1. 准备发布目录（必须剔除 .git/__pycache__/.DS_Store，防 PAT 泄漏）
mkdir -p /tmp/release-<name> && cd <源目录> && tar --exclude=.git --exclude=__pycache__ --exclude=.DS_Store -cf - . | tar -xf - -C /tmp/release-<name>/

# 2. 建仓推送
cd /tmp/release-<name> && git init -b main && git add -A
git -c user.name="Shi Yan" -c user.email="shiyan521@users.noreply.github.com" commit -m "feat: 发布 <name>"
gh repo create <owner>/<repo-name> --public --source=. --push

# 3. topics（三系列 + 领域）+ release
gh repo edit <owner>/<repo-name> --add-topic series-content-workflow
gh release create v0.1.0 --title "v0.1.0" --notes "## 首发版本\n..." 
```

### 方式 B：publish.py 纯 API（github.com:443 不通时的降级）

```bash
export GH_TOKEN='github_pat_xxx'  # 从 https://github.com/settings/tokens 生成,选 Contents 读写权限

python3 scripts/publish.py <owner> <repo-name> <本地目录> "口播博主卡开头？5 套钩子公式，30 秒出脚本"
```

脚本会自动：脱敏检查（含 URL 内嵌凭据）→ 元数据检查（LICENSE 署名 / frontmatter name）→ 建空仓 → 逐文件上传 → 打印完成链接。

### 单独调用 API 函数

```python
import sys, os
sys.path.insert(0, 'scripts')
from publish import privacy_scan, check_metadata, create_repo, upload_dir, set_topics, create_release, set_visibility

# 脱敏 + 元数据检查
issues = privacy_scan('/path/to/skill')
meta = check_metadata('/path/to/skill', 'my-skill')
if not issues and not meta:
    create_repo(token, 'my-skill', '一句话描述', private=False, topics=['chinese', 'tool', 'github'])
    upload_dir(token, '<owner>', 'my-skill', '/path/to/skill')
    create_release(token, '<owner>', 'my-skill', 'v0.1.0', 'v0.1.0', '首发版本说明...')
```

## 文件说明

```
github-skill-publisher/
├── SKILL.md                    # 完整工作流定义（脱敏/合规/元数据标准、踩坑清单、挑刺项）
├── README.md                   # 本文件,使用文档
├── LICENSE                     # MIT
├── .gitignore                  # Python / 编辑器 / 运行产物
├── scripts/
│   └── publish.py              # API 降级脚本,8 个封装函数
└── references/
    └── cheatsheet.md           # 5 分钟快查表
```

## 核心 API 函数

| 函数 | 用途 |
|------|------|
| `privacy_scan(local_dir, ignore_files)` | 脱敏检查，跳过 `__pycache__/.DS_Store`，**不跳过 `.git`**（专扫 `.git/config` 内嵌凭据），扫绝对路径/URL 内嵌凭据/邮箱/手机号/密钥 |
| `check_metadata(local_dir, repo_name)` | 元数据检查：LICENSE 署名统一 + frontmatter name 与仓库名一致 |
| `create_repo(token, name, desc, private, topics)` | 建空仓 |
| `upload_dir(token, owner, repo, local_dir, file_order)` | 递归上传整个目录 |
| `set_topics(token, owner, repo, topics)` | 改 topics |
| `set_visibility(token, owner, repo, private)` | 改公开/私有 |
| `create_release(token, owner, repo, tag, name, body)` | 发 release |

## 推荐流程

1. **剔除 .git**（绝对不能跳）→ 防止 `.git/config` 内嵌 PAT 泄漏
2. **脱敏检查** → `privacy_scan()`
3. **合规审查** → 涉及第三方平台 API 必须加 Compliance 声明，禁逆向/绕风控
4. **元数据检查** → `check_metadata()`（LICENSE 署名 / frontmatter name）
5. **建仓推送** → gh CLI 优先，publish.py 降级
6. **补 topics** → 三系列（series-content-workflow / series-agent-engineering / series-system-governance）+ 领域
7. **发 release** → 新包 v0.1.0 / 迭代 v1.x.0，notes 含脱敏保证
8. **验证** → 本地语法检查（sh/py/ps1）+ 引用完整性 + 远端脱敏复扫 + WebFetch 主页
9. **凭据清理** → `unset GH_TOKEN` + 临时目录不留痕

## 踩过的坑（已封装,不用再踩）

- **`.git/config` 明文 PAT 泄漏**（最严重）：remote URL 内嵌 `https://user:PAT@github.com/...`，发布包带 .git 目录即中招，靠 GitHub secret scanning 自动 revoke 兜底，绝不能依赖
- LICENSE 署名不一致（空署名 / shiyan / Shiyan 混用）→ 统一 `Copyright (c) 2026 Shi Yan`
- frontmatter name 与仓库名不一致 → 保持 name = 目录名 = 仓库名
- `auto_init` 创建的 README 覆盖必须先 GET 拿 sha 再 PUT,否则 422
- 中文路径 URL 要 `urllib.parse.quote(path, safe='/')`,否则 404
- 私有改公开后 `private` topic 残留,要立即重设
- 清理绝对路径时容易把 `[text](path)` 链接语法误删,渲染成空白
- zsh 提交含中文逗号 commit message 显示"nothing to commit" 是误报,实际成功
- description 拼接中英混排会显得"缝合",必须纯中文
- 归档仓库是只读的，改可见性/内容须先 unarchive，完事再 archive 回去
- 非交互命令（archive/delete）不带 `--yes` 会报错

## description 改写规则

**四块信息必到**（谁用 / 卡在啥时 / 怎么干 / 拿到啥）：

- 问号起头抛出卡点
- 数字要实在（5 套、30 秒、1 分钟、100+）
- 句尾给具体物（脚本 / 清单 / HTML / Markdown）
- 禁忌词：开源/灵活/可扩展/智能/一站式

## 反馈

用着有问题？去 GitHub Issues 提。
