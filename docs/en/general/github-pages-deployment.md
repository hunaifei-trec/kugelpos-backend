# GitHub Pages 部署方式对比：gh-pages 分支 vs GitHub Actions

## 核心原理

无论使用哪种方式，**MkDocs 构建的产物完全相同**。都是把 `.md` 源文件编译成静态 HTML 站点后交给 GitHub Pages 托管。两种方式的唯一区别是：**编译产物存放在哪里**。

```
源文件（.md）→ MkDocs 构建 → 静态 HTML 产物 → GitHub Pages 对外服务
                                     ↑
                          这一步存放位置不同
```

---

## 方式一：gh-pages 分支

### 原理

MkDocs 构建完成后，将产物 **推送到仓库的 `gh-pages` 分支**，GitHub Pages 从该分支读取文件对外服务。

```
main 分支
  └─ 源文件（.md, mkdocs.yml）
       │
       │  mkdocs gh-deploy --force
       ▼
gh-pages 分支（可见 Git 分支）
  ├─ index.html
  ├─ assets/        ← CSS、JS、字体
  ├─ en/            ← 编译后的英文文档
  ├─ ja/            ← 编译后的日文文档
  ├─ search/        ← 搜索索引
  ├─ 404.html
  ├─ sitemap.xml
  └─ .nojekyll
       │
       │  GitHub Pages 从此分支服务
       ▼
  https://user.github.io/repo/
```

### GitHub Actions Workflow

```yaml
name: Deploy MkDocs (gh-pages branch)

on:
  push:
    branches: [main]

permissions:
  contents: write   # 需要写权限来推送 gh-pages 分支

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.x'
      - run: pip install mkdocs-material
      - run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          mkdocs gh-deploy --force   # 构建 + 推送到 gh-pages 分支
```

### GitHub Pages Settings 配置

```
Build and deployment
  Source: Deploy from a branch
  Branch: gh-pages / (root)
```

### 特点

| 项目 | 说明 |
|------|------|
| **产物可见性** | 在仓库里可以看到 `gh-pages` 分支及所有 HTML 文件 |
| **Git 历史** | 每次部署都会在 `gh-pages` 分支新增一个 commit |
| **仓库存储** | 构建产物占用 Git 仓库存储空间 |
| **调试** | 可直接在 GitHub 上浏览编译后的 HTML 文件 |
| **适用场景** | 需要直接检查编译产物、或旧项目兼容 |

---

## 方式二：GitHub Actions artifact（推荐）

### 原理

MkDocs 构建完成后，将产物作为 **Pages artifact 上传到 GitHub 的 Pages 专用存储空间**，不以任何 Git 分支形式存在，GitHub Pages 直接从该存储空间对外服务。

```
main 分支
  └─ 源文件（.md, mkdocs.yml）
       │
       │  mkdocs build → site/
       ▼
GitHub Pages 专用存储（不可见，非 Git 分支）
  ├─ index.html
  ├─ assets/
  ├─ en/
  ├─ ja/
  ├─ search/
  └─ ...
       │
       │  GitHub Pages 从内部存储服务
       ▼
  https://user.github.io/repo/
```

> 产物和方式一**完全相同**，只是不存在可见的 `gh-pages` 分支。

### GitHub Actions Workflow

```yaml
name: Deploy MkDocs (GitHub Actions artifact)

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read    # 只需要读权限
  pages: write      # 写入 Pages 存储
  id-token: write   # OIDC 身份验证

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.x'
      - run: pip install mkdocs-material
      - run: mkdocs build --strict        # 只构建，不推送分支
      - uses: actions/upload-pages-artifact@v3
        with:
          path: site/                     # 上传构建产物

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/deploy-pages@v4     # 部署到 Pages 存储
```

### GitHub Pages Settings 配置

```
Build and deployment
  Source: GitHub Actions   ← 选这个，无需指定分支
```

### 特点

| 项目 | 说明 |
|------|------|
| **产物可见性** | 不在仓库中可见，存于 GitHub Pages 内部存储 |
| **Git 历史** | 不产生任何额外 commit，仓库历史干净 |
| **仓库存储** | 不占用 Git 仓库存储空间 |
| **调试** | 可在 Actions 运行记录中下载 artifact 查看 |
| **适用场景** | 新项目推荐，GitHub 官方推荐的现代方式 |

---

## 完整对比表

| 对比维度 | gh-pages 分支 | GitHub Actions artifact |
|----------|--------------|------------------------|
| **本质** | 产物存入 Git 分支 | 产物存入 Pages 专用存储 |
| **仓库分支数** | main + gh-pages（2个） | 只有 main（1个） |
| **Git 存储占用** | ✗ 占用（每次部署 ~几MB） | ✅ 不占用 |
| **产物可见性** | ✅ 可在 GitHub 上浏览 | ✗ 不可见（可下载 artifact） |
| **workflow 权限** | `contents: write` | `pages: write` + `id-token: write` |
| **Settings 配置** | Branch: gh-pages | Source: GitHub Actions |
| **构建命令** | `mkdocs gh-deploy --force` | `mkdocs build` + upload artifact |
| **最终 URL** | 完全相同 | 完全相同 |
| **页面内容** | 完全相同 | 完全相同 |
| **GitHub 官方推荐** | 旧方式 | ✅ 新方式（推荐） |

---

## 常见问题

**Q: 改成 GitHub Actions 方式后，之前的 gh-pages 分支需要删除吗？**

A: 需要手动删除，否则会存在残留分支（但不影响实际部署）。

```bash
git push origin --delete gh-pages
```

**Q: 两种方式可以随时切换吗？**

A: 可以。切换时只需修改 workflow 文件，并在 Settings 里对应修改 Source 设置即可。

**Q: GitHub Actions artifact 方式的产物保留多久？**

A: GitHub Pages 的部署 artifact 持续保留（不会过期），与普通 Actions artifact（默认 90 天过期）不同。
