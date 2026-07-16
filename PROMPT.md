# tttrove-vault：Obsidian 笔记自动发布到博客的项目提示词

## 1. 项目目标

把 Obsidian 笔记仓库自动发布为独立博客站点，访问者可通过 `notes.tttrove.qzz.io` 直接浏览，无需访问 GitHub。

> [!WARNING]
> 笔记仓库虽然是私有仓库，但生成后的博客是公开网站。任何进入 `content/` 且未被工作流删除、未被 Quartz `ignorePatterns` 忽略、也未加密的内容都可能公开。敏感信息、凭证和私人笔记不能只依赖仓库私有性保护。

## 2. 整体架构

```
Obsidian 本地写笔记
    │ git push
    ▼
tttrove-vault（笔记内容仓库） — Markdown、附件、Obsidian 配置及协作配置
    │ GitHub Actions: notify-blog.yml
    │ → repository_dispatch: notes-updated
    ▼
tttrove-vault-blog（博客代码仓库） — Quartz v5 + 构建部署
    │ GitHub Actions: build-and-deploy.yml
    │   1. checkout 博客代码
    │   2. checkout tttrove-vault → content/（获取完整提交历史，用于日期计算）
    │   3. 删除不应发布的仓库元数据和协作配置
    │      ⚠️ 必须保留 content/.git（见后文 bug 说明）
    │   4. npm ci
    │   5. 删除 3 个定制插件的安装目录，再执行 npx quartz plugin install
    │   6. 恢复定制插件源码、安装 devDependencies、重建插件 dist/
    │   7. npx quartz build
    │   8. 验证排序按钮、排序脚本、日期字段已进入 public/
    │   9. wrangler pages deploy public/
    ▼
Cloudflare Pages（项目名: tttrove-vault-blog）
    │
    ▼
notes.tttrove.qzz.io
```

## 3. 仓库信息

| 项目                 | 仓库                         | 本地路径                                         |
| -------------------- | ---------------------------- | ------------------------------------------------ |
| 笔记内容仓库（私有） | `tttrove/tttrove-vault`      | `C:\Users\59507\Projects\obsidian\tttrove-vault` |
| Quartz 博客仓库      | `tttrove/tttrove-vault-blog` | `C:\Users\59507\Projects\tttrove-vault-blog`     |

- `tttrove-vault` 是唯一的笔记内容源，保持私有；博客工作流通过 `NOTES_REPO_TOKEN` 将其检出到 CI runner 的 `content/` 目录。
- `tttrove-vault-blog` 保存 Quartz 配置、构建工作流和部署代码，不直接维护笔记正文。

## 4. 技术选型

- **站点生成器**: Quartz v5 代码基线（根包版本字段当前为 `5.0.0`；实际依赖和插件提交以 `package-lock.json`、`quartz.lock.json` 及仓库提交为准）
- **部署**: Cloudflare Pages（通过 wrangler CLI 推送静态文件）
- **触发**: GitHub Actions + repository_dispatch
- **域名**: `notes.tttrove.qzz.io`（`tttrove.qzz.io` 的二级域，托管在 Cloudflare DNS）
- **Node.js**: v22

## 5. GitHub Secrets

### tttrove-vault 仓库（笔记）

| Secret            | 值                                                     | 用途                                                                |
| ----------------- | ------------------------------------------------------ | ------------------------------------------------------------------- |
| `BLOG_REPO_TOKEN` | `<从 GitHub Settings → Developer settings → PAT 获取>` | fine-grained PAT，授权 tttrove-vault-blog，触发 repository_dispatch |

### tttrove-vault-blog 仓库（博客）

| Secret                  | 值                                                     | 用途                                                                  |
| ----------------------- | ------------------------------------------------------ | --------------------------------------------------------------------- |
| `NOTES_REPO_TOKEN`      | `<从 GitHub Settings → Developer settings → PAT 获取>` | 读取私有仓库 `tttrove/tttrove-vault`，至少授予该仓库 `Contents: Read` |
| `CLOUDFLARE_API_TOKEN`  | `<从 Cloudflare Dashboard → API Tokens 获取>`          | Cloudflare Pages 部署                                                 |
| `CLOUDFLARE_ACCOUNT_ID` | `<从 Cloudflare Dashboard 右下角获取>`                 | Cloudflare 账户 ID                                                    |

`BLOG_REPO_TOKEN` 用于从笔记仓库通知博客仓库，`NOTES_REPO_TOKEN` 用于博客仓库读取私有笔记仓库。两者方向和用途不同，不应混用。

## 6. Cloudflare Pages 配置

- 项目名: `tttrove-vault-blog`
- 项目 pages.dev 地址: `https://tttrove-vault-blog.pages.dev/`
- 自定义域名: `notes.tttrove.qzz.io`（状态: active, SSL 正常）
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
          repository: tttrove/tttrove-vault-blog
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
        uses: actions/checkout@v4
        with:
          repository: tttrove/tttrove-vault
          token: ${{ secrets.NOTES_REPO_TOKEN }}
          path: content
          # 编辑时间来自每个文件的最后一次 Git 提交，必须保留完整历史。
          fetch-depth: 0

      - name: Clean notes metadata
        run: rm -rf content/.obsidian content/.agents content/.opencode content/.github content/.gitignore content/.gitattributes content/AGENTS.md content/opencode.json content/skills content/提示词

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Install dependencies
        run: npm ci

      - name: Install Quartz plugins
        run: |
          # checkout 中只有受版本控制的定制源码，不是完整插件目录。
          # 必须先删目录，让安装器完整克隆 package.json、构建配置和其余源码。
          rm -rf .quartz/plugins/explorer
          rm -rf .quartz/plugins/content-index
          rm -rf .quartz/plugins/note-properties
          npx quartz plugin install

      - name: Apply and build custom plugin patches
        run: |
          # 不要删除这三个插件的 .git。Quartz build 会再次检查 Git HEAD；
          # 没有 .git 时会重新克隆上游版本，覆盖刚恢复的定制代码。
          git restore .quartz/plugins/explorer/src/components/Explorer.tsx
          git restore .quartz/plugins/explorer/src/components/scripts/explorer.inline.ts
          git restore .quartz/plugins/explorer/src/components/styles/explorer.scss
          git restore .quartz/plugins/content-index/src/emitter.ts
          git restore .quartz/plugins/note-properties/src/transformer.ts
          npm --prefix .quartz/plugins/explorer install --include=dev
          npm --prefix .quartz/plugins/content-index install --include=dev
          npm --prefix .quartz/plugins/note-properties install --include=dev
          npm --prefix .quartz/plugins/explorer run build
          npm --prefix .quartz/plugins/content-index run build
          npm --prefix .quartz/plugins/note-properties run build

      - name: Build site
        run: npx quartz build

      - name: Verify customized build artifacts
        run: |
          grep -q 'explorer-sort' public/index.html
          grep -Rq 'explorerSort' public/static
          grep -q '"modified"' public/static/contentIndex.json

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy public --project-name=tttrove-vault-blog --branch=main
```

## 8. Quartz v5 配置关键点

### quartz.config.yaml

```yaml
configuration:
  pageTitle: tttrove 笔记
  pageTitleSuffix: " - tttrove 笔记"
  enableSPA: true
  enablePopovers: true
  analytics: null
  locale: zh-CN
  baseUrl: notes.tttrove.qzz.io
  ignorePatterns:
    - .obsidian
    - .agents
    - .opencode
    - .github
    - AGENTS.md
    - opencode.json
    - skills
    - 提示词
```

当前配置以仓库根目录的 `quartz.config.yaml` 为唯一准确信息源。重点检查 `baseUrl`、`ignorePatterns`、日期插件优先级和定制插件配置。

### 已完成内容同步和插件准备后的构建命令

以下命令使用 PowerShell；全新 clone 的完整初始化流程见第 13 节。

```powershell
Set-Location 'C:\Users\59507\Projects\tttrove-vault-blog'
npx quartz build          # 构建到 public/
npx quartz build --serve  # 构建 + 本地预览（localhost:8080）
```

## 9. 🐛 重要 Bug 记录（已修复）

### Bug 1: wrangler 找不到 CLOUDFLARE_API_TOKEN（历史问题）

- **现象**: `In a non-interactive environment, it's necessary to set a CLOUDFLARE_API_TOKEN`
- **原因**: 早期 workflow 没有把有效凭证传给 wrangler。
- **当前做法**: 使用 `cloudflare/wrangler-action@v3` 的 `apiToken` 和 `accountId` 输入。以当前 workflow 和最近一次成功部署为准，不再额外维护一套过时的 `env` 示例。

### Bug 2: Quartz CLI 执行权限拒绝

- **现象**: `./quartz/bootstrap-cli.mjs: Permission denied`
- **原因**: `package.json` 中的 `quartz` script 直接调用 `./quartz/bootstrap-cli.mjs`，在 CI 的 Unix 权限环境中可能因文件缺少可执行位失败。
- **修复**: 使用 `npx quartz <command>`（npx 会正确解析本地包）

### Bug 3: repository_dispatch 失败（Bad credentials）

- **现象**: `Resource not accessible by personal access token`
- **原因**: fine-grained PAT 的资源所有者、目标仓库或权限配置不正确。
- **修复**: PAT 的 Repository access 必须包含 `tttrove-vault-blog`，Repository permissions 中 `Contents` 设为 Read and write。

### ⭐ Bug 4: Quartz 构建时找不到笔记文件（0 files）

- **现象**: `content/` 中存在 Markdown 文件，但 `npx quartz build` 显示 `Found 0 input files`。
- **原因**: Quartz v5 的 `quartz/util/glob.ts` 中使用 `globby` 并设置了 `gitignore: true`。
  当我们执行 `rm -rf content/.git` 后，`globby` 无法在 content/ 内找到 .git，于是向上走到
  博客仓库的 .git（`tttrove-vault-blog/.git`），使用博客仓库的 `.gitignore`。
  而博客仓库的 `.gitignore` 中有 `content/` 规则，导致所有笔记文件被忽略。
- **修复**: **必须保留 `content/.git`**。其余不应发布的仓库元数据和协作配置可按 workflow 删除。
  这样 globby 使用笔记仓库的 `.gitignore`，不会误杀 .md 文件。
- **验证**: 构建应发现与当前公开笔记数量相符的非零输入文件，不要把具体数量写死。

### ⭐ Bug 5: 跟踪少量 `.quartz/plugins` 源码导致插件目录不完整

- **现象**: workflow 中当时名为 `Apply custom Explorer sorting` 的步骤报错：
  `Could not read package.json: ENOENT: no such file or directory, open '.quartz/plugins/explorer/package.json'`。
- **原因**: 为了持久化 Explorer 定制，只在博客仓库中跟踪了少量插件源码。checkout 后目录已经存在，
  插件安装器可能把这个不完整目录视为已安装并跳过克隆，但目录中没有 `package.json`、`tsup.config.ts` 和完整源码。
- **修复**: 在 `npx quartz plugin install` 前删除定制插件安装目录：
  `.quartz/plugins/explorer`、`.quartz/plugins/content-index`、`.quartz/plugins/note-properties`。
  安装完成后再用 `git restore` 恢复受博客仓库跟踪的定制文件。

### ⭐ Bug 6: 定制插件重建时 `tsup: not found`

- **现象**: `npm --prefix .quartz/plugins/explorer run build` 报 `sh: 1: tsup: not found`，退出码 127。
- **原因**: Quartz 插件安装器优先使用插件仓库中已有的 `dist/`，不会为这些插件安装完整开发依赖；
  `tsup` 位于 `devDependencies`，恢复源码后不能直接执行 build。
- **修复**: 重建前分别执行：
  `npm --prefix .quartz/plugins/<name> install --include=dev`。

### ⭐ Bug 7: workflow 成功，但发布站点仍使用上游 Explorer

- **现象**: 插件重建、Quartz 构建和 Cloudflare 部署全部显示成功，但线上 HTML 没有 `explorer-sort`。
- **原因**: 定制步骤删除了插件目录中的 `.git`。随后 `npx quartz build` 加载配置时会再次执行插件安装检查；
  因无法解析插件 Git HEAD，安装器删除整个插件目录并重新克隆上游版本，覆盖刚构建的定制 `dist/`。
- **修复**: **恢复定制源码后不要删除插件 `.git`**。安装前可以删除整个目录以强制完整安装，
  但安装完成后必须保留新克隆产生的 `.git`，让 Quartz build 跳过二次安装。

### ⭐ Bug 8: 构建成功不能证明定制功能已进入最终产物

- **现象**: CI 和 Cloudflare 都是绿色，但发布结果缺少排序按钮。
- **原因**: 之前只验证命令退出码，没有验证最终 `public/` 的实际内容。
- **修复**: 部署前增加产物门禁：
  - `public/index.html` 必须包含 `explorer-sort`。
  - `public/static` 中必须包含 `explorerSort` 客户端逻辑。
  - `public/static/contentIndex.json` 必须包含 `modified` 日期字段。

  任一检查失败都应阻止 Cloudflare 部署。

### Bug 9: 缺少 frontmatter.modified 时编辑时间等于创建时间

- **现象**: 按编辑时间排序时，文章顺序与创建时间排序相同。
- **原因**: `note-properties` 原先会在没有 `modified` 时执行 `modified = created`，使后续日期插件认为
  frontmatter 已提供编辑时间，不再读取 Git。
- **修复**: 删除这条隐式回退。当前日期来源规则为：
  - 创建时间：`frontmatter.created` → 文件系统创建时间。
  - 编辑时间：`frontmatter.modified` → 文件最后一次 Git 提交时间 → 文件系统修改时间。
  - CI 检出笔记仓库必须使用 `fetch-depth: 0`，否则无法可靠查询旧文件的完整 Git 历史。
- **使用原则**: 笔记只需维护 `created`；一般不要手动维护 `modified`。需要明确覆盖时才写 `modified`。

## 10. 初始部署记录

以下记录仅表示初始部署时已完成，不代表其他电脑当前也已安装或登录相同工具。

| 步骤                                                      | 状态 |
| --------------------------------------------------------- | ---- |
| 笔记仓库重命名: obsidian-note1 → tttrove-vault            | ✅   |
| 删除 CNAME（旧 GitHub Pages 配置）                        | ✅   |
| 创建博客仓库 tttrove-vault-blog                           | ✅   |
| 安装 gh CLI（winget install GitHub.cli）                  | ✅   |
| gh auth login（SSH 协议）                                 | ✅   |
| 创建 Cloudflare API Token（Pages Edit 权限）              | ✅   |
| 创建 fine-grained PAT（触发 repository_dispatch）         | ✅   |
| 创建 Cloudflare Pages 项目 tttrove-vault-blog             | ✅   |
| 绑定自定义域 notes.tttrove.qzz.io                         | ✅   |
| 创建笔记仓库首页 index.md                                 | ✅   |
| 端到端测试（push → notify → build → deploy → 域名可访问） | ✅   |

## 11. Obsidian 内容组织原则

- `tttrove-vault` 是唯一的笔记内容源，实际主题目录和笔记数量会持续变化，不在本说明中维护具体目录树或文章清单。
- `index.md` 是站点首页，负责提供当前有效的分类入口；内容分类应以笔记仓库中的实际目录和首页为准。
- 笔记按主题放入现有目录，附件存放在笔记同级的 `assets/<不含 .md 扩展名的笔记文件名>/` 目录。
- `.agents/skills/` 保存项目级 Agent Skills，`AGENTS.md` 保存仓库协作规则，`opencode.json` 注册 OpenCode 项目配置。这些文件由 Git 同步，但不属于公开站点内容。
- 发布边界由两层机制共同保证：workflow 先删除仓库元数据和协作配置，Quartz 再通过 `ignorePatterns` 忽略残留的非公开路径。准确列表分别以 `.github/workflows/build-and-deploy.yml` 和 `quartz.config.yaml` 为准。
- 内容仓库私有不代表发布站点私有。敏感笔记必须移出发布范围或使用明确的加密机制，不能只依赖 GitHub 仓库权限。
- 内容仓库的目录、分类或笔记数量发生变化时，不需要同步修改本节；只有内容组织原则或发布边界变化时才更新本文档。

## 12. 日常使用流程

1. 在 Obsidian 中写笔记（`C:\Users\59507\Projects\obsidian\tttrove-vault`）
2. 执行 `git commit` 和 `git push`，确认提交已推送到远程 `main`；仅本地保存或 commit 不会触发部署
3. GitHub 收到 push 后，`Notify blog rebuild` 工作流触发博客构建
4. 通常数分钟后 https://notes.tttrove.qzz.io 更新；准确状态以两个仓库的 Actions 和 Cloudflare deployment 为准

## 13. 继续开发的建议

如果需要在另一台电脑继续：

1. 安装环境：Git、Node.js 22、npm 10.9.2 或更高版本、gh CLI
2. 登录 gh: `gh auth login`（SSH 协议，使用现有密钥对）
3. Clone 博客仓库：
   ```bash
   git clone git@github.com:tttrove/tttrove-vault-blog.git
   cd tttrove-vault-blog
   ```
4. 将笔记仓库完整 clone 到博客仓库的 `content/`。以下命令会删除现有 `content/`；执行前先确认其中没有未提交修改：
   ```bash
   git -C content status 2>/dev/null || true
   rm -rf content
   # 不要使用 --depth=1，编辑时间排序依赖完整 Git 历史。
   git clone git@github.com:tttrove/tttrove-vault.git content
   ```
5. 准备依赖和定制插件。以下命令在 Git Bash 中执行：
   ```bash
   npm ci
   rm -rf .quartz/plugins/explorer .quartz/plugins/content-index .quartz/plugins/note-properties
   npx quartz plugin install
   git restore .quartz/plugins/explorer/src/components/Explorer.tsx
   git restore .quartz/plugins/explorer/src/components/scripts/explorer.inline.ts
   git restore .quartz/plugins/explorer/src/components/styles/explorer.scss
   git restore .quartz/plugins/content-index/src/emitter.ts
   git restore .quartz/plugins/note-properties/src/transformer.ts
   npm --prefix .quartz/plugins/explorer install --include=dev
   npm --prefix .quartz/plugins/content-index install --include=dev
   npm --prefix .quartz/plugins/note-properties install --include=dev
   npm --prefix .quartz/plugins/explorer run build
   npm --prefix .quartz/plugins/content-index run build
   npm --prefix .quartz/plugins/note-properties run build
   ```
6. 构建并预览：
   ```bash
   npx quartz build --serve  # 预览 localhost:8080
   ```
7. 修改博客主题或配置后提交并 push，CI 会自动部署。

## 14. Cloudflare 项目标识

| 项目          | 值                             |
| ------------- | ------------------------------ |
| Pages 项目名  | `tttrove-vault-blog`           |
| pages.dev URL | `tttrove-vault-blog.pages.dev` |
| 自定义域      | `notes.tttrove.qzz.io`         |

API Token、Account ID 和 GitHub PAT 的配置统一见第 5 节。真实 Secret 不得写入本文件或提交到仓库。

## 15. i18n 支持

Quartz v5 支持多语言，配置中使用 `locale: zh-CN`。i18n 翻译文件位于:
`quartz/i18n/locales/zh-CN.ts`

如需修改界面文本（如 "Backlinks" → "反向链接"），编辑此文件。

这是对 Quartz 核心源码的直接修改，升级或同步上游前必须检查相关 diff，避免被覆盖。

## 16. 故障排查速查

| 问题                 | 检查点                                                                                    |
| -------------------- | ----------------------------------------------------------------------------------------- |
| 站点 404             | 查看 Cloudflare Pages 最新部署状态                                                        |
| 笔记不更新           | 查看内容仓库的 `Notify blog rebuild` 是否成功，并检查相关 PAT 是否过期                    |
| 私有笔记仓库检出失败 | 检查博客仓库的 `NOTES_REPO_TOKEN` 是否包含 `tttrove-vault`，并至少具有 `Contents: Read`   |
| 构建失败             | 查看博客仓库 `Build and deploy` 工作流中具体失败的步骤和日志                              |
| 仍然是 0 files       | 确认 `content/.git` 没有被删除                                                            |
| 排序按钮在线上消失   | 检查 `Verify customized build artifacts` 是否通过，并确认插件安装后的 `.git` 未被删除     |
| 域名无法访问         | 检查 Cloudflare DNS 中 `notes` CNAME 是否指向 `tttrove-vault-blog.pages.dev`              |
| 本地构建问题         | 确保 `npm ci`、插件完整安装、定制源码恢复、devDependencies 安装和三个定制插件重建均已成功 |
