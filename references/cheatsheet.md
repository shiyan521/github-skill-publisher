# GitHub Skill Publisher Cheatsheet

## 5 分钟快查

### 标准发布流程（gh/git 优先）

```bash
# 1. 剔除 .git（防 PAT 泄漏，绝对不能跳）
mkdir -p /tmp/release-<name> && cd <源目录> && tar --exclude=.git --exclude=__pycache__ --exclude=.DS_Store -cf - . | tar -xf - -C /tmp/release-<name>/

# 2. 脱敏检查
python3 <repo>/scripts/publish.py --scan-only <local-dir> 2>/dev/null || \
python3 -c "
import sys; sys.path.insert(0, '<repo>/scripts')
from publish import privacy_scan, check_metadata
print(privacy_scan('<local-dir>'))
print(check_metadata('<local-dir>', '<repo-name>'))
"

# 3. 建仓推送
cd /tmp/release-<name> && git init -b main && git add -A
git -c user.name="Shi Yan" -c user.email="shiyan521@users.noreply.github.com" commit -m "feat: 发布 <name>"
gh repo create <owner>/<repo-name> --public --source=. --push

# 4. 补 topics（三系列选一 + 领域/类型）
gh repo edit <owner>/<repo-name> --add-topic series-content-workflow --add-topic chinese

# 5. 发 release（新包 v0.1.0 / 迭代 v1.x.0）
gh release create v0.1.0 --title "v0.1.0" --notes "## 首发版本\n[一句话描述]\n### 脱敏保证\n- ..."
```

### 降级流程（github.com:443 不通，api.github.com 可用）

```bash
# publish.py 全量上传
export GH_TOKEN='...'
python3 scripts/publish.py <owner> <repo> <本地目录> "<description>"

# 单文件更新（README 等）
gh api -X PUT repos/<owner>/<repo>/contents/<file> \
  -f message="update: <说明>" \
  -f content="$(base64 -i <file> | tr -d '\n')" \
  -f sha="$(gh api repos/<owner>/<repo>/contents/<file> -q '.sha')"
# 注意：此方式会使本地 git 历史发散，后续 git fetch && git reset --soft origin/main 对齐
```

### 归档 / 私有化

```bash
gh repo archive shiyan521/<repo> --yes        # 归档（只读；visibility 不变，公开仍可见）
gh repo unarchive shiyan521/<repo> --yes      # 恢复
gh repo edit shiyan521/<repo> --visibility private --accept-visibility-change-consequences  # 私有化（归档仓库须先 unarchive）
```

## 必查项

| 检查 | 工具 |
|------|------|
| `.git` / `__pycache__` / `.DS_Store` 混入 | `ls -d .git` / `unzip -l x.zip \| grep -E '\.git/|__pycache__'` |
| URL 内嵌凭据 / 绝对路径 / 邮箱 / 手机号 / 密钥 | `privacy_scan()` |
| LICENSE 署名统一 | `check_metadata()` / `grep -i copyright LICENSE` |
| frontmatter name = 仓库名 | `check_metadata()` / `grep '^name:' SKILL.md` |
| 跨平台脚本实机验证标注 | README 检查（ps1 等须标"尚未实机复测"） |
| 语法检查 | `.sh`: `bash -n` / `.py`: `py_compile` / `.ps1`: PowerShell Parser |
| README 链接完整性 | 引用仓库 `gh repo view` 核实 + WebFetch 主页 |
| 远端脱敏复扫 | `curl raw.githubusercontent.com` 后跑 grep |
| topic 残留 | 看 `topics` 列表 |
| description 一致性 | description / topics / README 三者对照 |

## 合规红线

**不爬平台、不绕风控、不自动发布。只处理"用户已有"的内容。**

- 抓取他人账号 / 逆向私有 API / 绕风控 → 不上传
- 官方公开 API → 加 Compliance 声明 + 默认限速（≥1.0s，对齐服务端配额）
- 用户自己的内容 → README 注明「仅限本人或已获授权内容」

Compliance 模板：
```
> ⚠️ **合规使用声明**：仅限处理本人账号或已获授权的内容，请遵守平台协议与适用法律，勿二次分发。
```

## description 改写模板

```
[人群][卡点场景]?[具体方式],[具体结果]。
```

例：
- 口播博主每条视频卡开头 10 分钟？5 套钩子公式轮换，30 秒出能拍的脚本。
- 囤了一堆素材不知道先拍哪条？5 维评分表，1 分钟排出 A/B/C 优先级。

禁忌词：开源/灵活/可扩展/智能/什么场景/一站式/全方位

## 三系列 topics

按系列选一 + 领域/类型配：

- `series-content-workflow` —— 内容生产管线（选题/写作/发布/复盘）
- `series-agent-engineering` —— Agent / Skill 工程（方法论、审计、交互）
- `series-system-governance` —— 系统治理（存储/性能/网络/安全审计）

例：`chinese, series-content-workflow, hook-formula, script-writing, short-video, sop`

## 套件索引仓模式

多包成套件时建索引仓（如 system-governance-suite），成员包 README 顶部标注：

```
> 本包属于 **<suite-name>**（<一句话描述>）的一部分，可与套件内其他包组合使用：<成员列表>。
```

## release notes 必含

```
## 首发版本

[一句话描述]

### 包含内容
- ...

### 脱敏保证
- ...
```

## 不修（收益低）

- `chinese` topic 多仓重复
- 缺 Issues 模板
- commit message 风格统一
