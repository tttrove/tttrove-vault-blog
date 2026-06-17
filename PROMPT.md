# tttrove-vault Obsidian → 博客自动发布 项目提示词

## 1. 项目目标

把 Obsidian 笔记仓库自动发布为独立博客站点，朋友通过 `notes.tttrove.qzz.io` 直接访问，无需访问 GitHub。

## 2. 整体架构

```
Obsidian 本地写笔记
    │ git push
    ▼
tttrove-vault（纯笔记仓库） — 只有目录树 + .md 文件
    │ GitHub Actions: notify-blog.yml
    │ → repository_dispatch: notes-updated
    ▼
tttrove-vault-blog（博客代码仓库） — Quartz v5 + 构建部署
    │ GitHub Actions: build-and-deploy.yml
    │   1. checkout 博客代码
    │   2. git clone tttrove-vault → content/
    │   3. rm content/.obsidian content/.github content/.gitignore content/.gitattributes
    │      ⚠️ 必须保留 content/.git（见后文 bug 说明）
    │   4. npm ci
    │   5. npx quartz plugin install
    │   6. npx quartz build
    │   7. wrangler pages deploy public/
    ▼
Cloudflare Pages（项目名: tttrove-vault-blog）
    │
    ▼
notes.tttrove.qzz.io
```

## 3. 仓库信息

| 项目 | 仓库 | 本地路径 |
|---|---|---|
| 笔记仓库 | tttrove/ttrove-vault (public) | `C:\Users\59507\Projects\ttrove-vault` |
| 博客仓库 | tttrove/ttrove-vault-blog (public) | `C:\Users\59507\Projects\ttrove-vault-blog` |

## 4. 技术选型

- **站点生成器**: Quartz v5.0.0（基于 Node.js，天然支持 Obsidian wikilinks、双链、标签）
- **部署**: Cloudflare Pages（通过 wrangler CLI 推送静态文件）
- **触发**: GitHub Actions + repository_dispatch
- **域名**: `notes.ttrove.qzz.io`（`ttrove.qzz.io` 的二级域，托管在 Cloudflare DNS）
- **Node.js**: v22

## 5. GitHub Secrets

### tttrove-vault 仓库（笔记）

| Secret | 值 | 用途 |
|---|---|---|
| `BLOG_REPO_TOKEN` | `<从 GitHub Settings → Developer settings → PAT 获取>` | fine-grained PAT，授权 tttrove-vault-blog，触发 repository_dispatch |

### tttrove-vault-blog 仓库（博客）

| Secret | 值 | 用途 |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | `<从 Cloudflare Dashboard → API Tokens 获取>` | Cloudflare Pages 部署 |
| `CLOUDFLARE_ACCOUNT_ID` | `<从 Cloudflare Dashboard 右下角获取>` | Cloudflare 账户 ID |

## 6. Cloudflare Pages 配置

- 项目名: `ttrove-vault-blog`
- 项目 pages.dev 地址: `https://ttrove-vault-blog.pages.dev/`
- 自定义域名: `notes.ttrove.qzz.io`（状态: active, SSL 正常）
- 部署方式: Direct Upload（通过 wrangler CLI，不绑定 GitHub）

## 7. GitHub Actions 工作流

### tttrove-vault/.github/workflows/notify-blog.yml

```yaml
name: Notify blog rebuild
on:
  push:
    branches: [main]
jobs:
  dispatch:
    runs-on: ubuntu-latest
    steps:
      - uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.BLOG_REPO_TOKEN }}
          repository: tttrove/ttrove-vault-blog
          event-type: notes-updated
          client-payload: |
            {"sha": "${{ github.sha }}", "ref": "${{ github.ref }}"}
```

### tttrove-vault-blog/.github/workflows/build-and-deploy.yml

```yaml
name: Build and deploy
on:
  push:
    branches: [main]
  repository_dispatch:
    types: [notes-updated]
  workflow_dispatch:

concurrency:
  group: pages-deploy
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout blog code
        uses: actions/checkout@v4

      - name: Clone notes content
        run: |
          rm -rf content
          git clone --depth=1 https://github.com/ttrove/ttrove-vault.git content
          # ⚠️ 千万不要删 content/.git！（见 bug #4）
          rm -rf content/.obsidian content/.github content/.gitignore content/.gitattributes

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install dependencies
        run: npm ci

      - name: Install Quartz plugins
        run: npx quartz plugin install

      - name: Build site
        run: npx quartz build

      - name: Deploy to Cloudflare Pages
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy public --project-name=ttrove-vault-blog --branch=main
```

## 8. Quartz v5 配置关键点

### quartz.config.yaml

```yaml
configuration:
  pageTitle: tttrove 笔记
  pageTitleSuffix: ""
  enableSPA: true
  enablePopovers: true
  analytics: null
  locale: zh-CN
  baseUrl: notes.ttrove.qzz.io
  ignorePatterns:
    - private
    - templates
    - .obsidian
    - skills
```

全文见 `C:\Users\59507\Projects\ttrove-vault-blog\quartz.config.yaml`

### 本地构建命令

```bash
cd C:\Users\59507\Projects\ttrove-vault-blog
npx quartz build          # 构建到 public/
npx quartz build --serve  # 构建 + 本地预览（localhost:8080）
```

## 9. 🐛 重要 Bug 记录（已修复）

### Bug 1: wrangler 找不到 CLOUDFLARE_API_TOKEN
- **现象**: `In a non-interactive environment, it's necessary to set a CLOUDFLARE_API_TOKEN`
- **原因**: cloudflare/wrangler-action@v3 传了 apiToken 参数，但 wrangler CLI 实际从环境变量读
- **修复**: 在 workflow step 上加 `env: CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}`

### Bug 2: Quartz CLI 执行权限拒绝
- **现象**: `./quartz/bootstrap-cli.mjs: Permission denied`
- **原因**: `npm run quartz`（即 `./quartz/bootstrap-cli.mjs`）在 CI runner 上没有执行权限
- **修复**: 使用 `npx quartz <command>`（npx 会正确解析本地包）

### Bug 3: repository_dispatch 失败（Bad credentials）
- **现象**: `Resource not accessible by personal access token`
- **原因**: fine-grained PAT 权限不对，repository_dispatch 需要 Contents: Read and Write
- **修复**: 重新创建 PAT，授权 Contents: Read and Write

### ⭐ Bug 4: Quartz 构建时找不到笔记文件（0 files）
- **现象**: content/ 目录有 30 个 .md 文件（find 确认），但 `npx quartz build` 显示 `Found 0 input files`
- **原因**: Quartz v5 的 `quartz/util/glob.ts` 中使用 `globby` 并设置了 `gitignore: true`。
  当我们执行 `rm -rf content/.git` 后，`globby` 无法在 content/ 内找到 .git，于是向上走到
  博客仓库的 .git（`ttrove-vault-blog/.git`），使用博客仓库的 `.gitignore`。
  而博客仓库的 `.gitignore` 中有 `content/` 规则，导致所有笔记文件被忽略。
- **修复**: **必须保留 `content/.git`**。只删 `.obsidian`、`.github`、`.gitignore`、`.gitattributes`。
  这样 globby 使用笔记仓库的 `.gitignore`，不会误杀 .md 文件。
- **验证**: 本地 `git clone --depth=1 [repo] content` 后运行 `rm -rf content/.obsidian content/.github content/.gitignore content/.gitattributes`，
  然后 `npx quartz build` 应显示 `Found 30 input files`。

## 10. 人工操作清单（已执行）

| 步骤 | 状态 |
|---|---|
| 笔记仓库重命名: obsidian-note1 → tttrove-vault | ✅ |
| 删除 CNAME（旧 GitHub Pages 配置） | ✅ |
| 创建博客仓库 tttrove-vault-blog | ✅ |
| 安装 gh CLI（winget install GitHub.cli） | ✅ |
| gh auth login（SSH 协议） | ✅ |
| 创建 Cloudflare API Token（Pages Edit 权限） | ✅ |
| 创建 fine-grained PAT（触发 repository_dispatch） | ✅ |
| 创建 Cloudflare Pages 项目 tttrove-vault-blog | ✅ |
| 绑定自定义域 notes.ttrove.qzz.io | ✅ |
| 创建笔记仓库首页 index.md | ✅ |
| 端到端测试（push → notify → build → deploy → 域名可访问） | ✅ |

## 11. Obsidian 笔记结构

```
ttrove-vault/
├── index.md                    # 博客首页
├── Linux笔记/                  # Linux 运维、编译等
│   ├── Ubuntu Netplan 静态 IP 配置最佳实践.md
│   ├── Nginx 学习与高级配置指南.md
│   ├── Linux 服务器入侵排查手册.md
│   └── ... (共 15 篇)
├── 漏洞/                       # 内核漏洞分析
│   ├── Dirty Pipe: Linux 内核管道 Page Cache 覆写提权漏洞 (CVE-2022-0847).md
│   ├── Dirty Cow: Linux内核写时复制竞态条件提权漏洞 (CVE-2016-5195).md
│   └── ... (共 9 篇)
├── vmware笔记/                  # VMware 配置
│   └── vmware开机自启虚拟机后台运行.md
├── 提示词/                     # 提示词工程
│   └── Linux服务器入侵排查手册生成提示词.md
├── skills/                     # ⚠️ 不发布（在 ignorePatterns 中）
└── assets/                     # 附件图片（Obsidian 自定义附件路径 `./assets/${noteFileName}`）
```

配置了 ignorePatterns: [private, templates, .obsidian, skills]，`skills/` 目录不会被发布

## 12. 日常使用流程

1. 在 Obsidian 中写笔记（`C:\Users\59507\Projects\ttrove-vault`）
2. git commit && git push
3. GitHub Actions 自动触发
4. 约 2 分钟后，https://notes.ttrove.qzz.io 自动更新

## 13. 继续开发的建议

如果需要在另一台电脑继续：

1. 安装环境: Git, Node.js v22, npm, gh CLI
2. 登录 gh: `gh auth login`（SSH 协议，使用现有密钥对）
3. Clone 两个仓库:
   ```bash
   git clone git@github.com:ttrove/ttrove-vault.git
   git clone git@github.com:ttrove/ttrove-vault-blog.git
   ```
4. 博客仓库本地开发:
   ```bash
   cd tttrove-vault-blog
   npm ci
   npx quartz plugin install
   npx quartz build --serve  # 预览 localhost:8080
   ```
5. 如果需要本地有笔记内容，手动 clone 到 content/:
   ```bash
   rm -rf content
   git clone --depth=1 git@github.com:ttrove/ttrove-vault.git content
   ```
6. 修改博客主题/配置后，直接 push，CI 会自动部署

## 14. Cloudflare 凭证（⚠️ 敏感，勿外泄）

| 项目 | 值 |
|---|---|
| API Token | `<从 Cloudflare Dashboard → API Tokens 获取>` |
| Account ID | `<从 Cloudflare Dashboard 右下角获取>` |
| Pages 项目名 | `ttrove-vault-blog` |
| pages.dev URL | `ttrove-vault-blog.pages.dev` |
| 自定义域 | `notes.ttrove.qzz.io` |
| Zone ID (ttrove.qzz.io) | `33dd00b73f47b6d5365699ad11b940c2` |

## 15. GitHub PAT（⚠️ 敏感，勿外泄）

| 项目 | 值 |
|---|---|
| 名称 | tttrove-vault-notify-blog |
| Token | `<从 GitHub Settings → Developer settings → PAT 获取>` |
| 授权仓库 | tttrove-vault-blog |
| 权限 | Contents: Read and Write |

## 16. i18n 支持

Quartz v5 支持多语言，配置中使用 `locale: zh-CN`。i18n 翻译文件位于:
`quartz/i18n/locales/zh-CN.ts`

如需修改界面文本（如 "Backlinks" → "反向链接"），编辑此文件。

## 17. 故障排查速查

| 问题 | 检查点 |
|---|---|
| 站点 404 | Cloudflare Pages Dashboard 查看最新 deployment status |
| 笔记不更新 | 检查 tttrove-vault 的 Actions 中 notify 是否成功；检查 PAT 是否过期 |
| 构建失败 | 看 tttrove-vault-blog Actions 日志具体错误步 |
| 还是 0 files | 确认 content/.git 没有被删除 |
| 域名访问不了 | Cloudflare DNS 中 notes CNAME 是否指向 tttrove-vault-blog.pages.dev |
| 本地构建问题 | 确保 `npm ci` 成功，`npx quartz plugin install` 完成 |
