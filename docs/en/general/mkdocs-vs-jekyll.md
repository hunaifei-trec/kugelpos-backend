# MkDocs vs Jekyll：GitHub Pages 文档站方案对比

> 基于 `kugelpos-backend` 项目的实际部署实验结论（2026-03）

---

## 背景

项目 `docs/` 目录下有 **49 个 Markdown 文件**，分 EN/JA 双语，文件名为 `api-specification.md`、`model-specification.md` 等。  
目标：通过 GitHub Pages 公开部署为文档网站，**不修改任何已有 MD 文件内容**。

---

## 核心差异一览

| 对比维度 | MkDocs Material | Jekyll + just-the-docs |
|----------|:--------------:|:---------------------:|
| **是否需改 MD 文件** | ✅ 完全不需要 | ⚠️ 部分需要（见下方说明） |
| **导航配置方式** | 在 `mkdocs.yml` 里手写 nav 树 | MD 文件 front matter 或 `_config.yml` defaults |
| **视觉效果** | ⭐⭐⭐⭐⭐ Material Design | ⭐⭐⭐⭐ just-the-docs 暗色 |
| **搜索** | ✅ 内置全文搜索 | ✅ 内置搜索 |
| **代码高亮 + 复制** | ✅ 完整支持 | ✅ 支持 |
| **构建语言** | Python | Ruby |
| **部署配置复杂度** | 低（1 个配置文件） | 中（需 Gemfile + _config.yml） |
| **GitHub Pages 原生支持** | 需 GitHub Actions | ✅ 原生支持（也可用 Actions） |
| **构建速度** | ⭐⭐⭐⭐⭐ 快（~16s） | ⭐⭐⭐ 慢（安装 Ruby gems 耗时） |

---

## Jekyll 的根本限制

### `defaults` 对无 front matter 的文件无效

Jekyll 提供 `_config.yml` 的 `defaults` 机制，理论上可按路径批量设置 `layout`、`parent` 等属性，**无需在 MD 文件中写 front matter**：

```yaml
# _config.yml 中的 defaults 方案
defaults:
  - scope: { path: "" }
    values: { layout: "default" }     # ← 理论上应用到所有文件

  - scope: { path: "en/cart" }
    values: { parent: "English" }     # ← en/cart/ 下的文件设为子项
```

**实际问题：**

> Jekyll 只对含有 front matter（`---` 标记）的文件执行模板处理。  
> **没有 `---` 的文件被视为「静态资产」，直接原样输出，`defaults` 完全无效。**

### 实验结果

| 文件 | front matter | 访问 `.html` URL | 说明 |
|------|:-----------:|:---------------:|------|
| `docs/index.md` | ✅ 有（手动添加） | ✅ 正常渲染 | 唯一生效的首页 |
| `en/cart/api-specification.md` | ❌ 无 | ❌ 404 | defaults 不生效 |
| `ja/README.md` | ❌ 无 | 返回原始 markdown | 被当作静态文件 |

### 结论

**Jekyll「完全不改 MD 文件」做不到。** 要让 Jekyll 处理文件，每个文件必须有 `---` front matter。  
若批量在 48 个文件头部添加 front matter，会影响文件在 GitHub 上的 Markdown 原始预览可读性。

---

## MkDocs 方案

### 工作原理

MkDocs 直接读取 MD 原文，**不依赖 front matter**，导航完全由 `mkdocs.yml` 中的 `nav:` 配置定义：

```yaml
# mkdocs.yml - 完整控制导航，无需改任何 MD 文件
nav:
  - Home: index.md
  - English:
    - Overview: en/README.md
    - Cart Service:
      - API Specification: en/cart/api-specification.md
      - Model Specification: en/cart/model-specification.md
    # ...
  - 日本語:
    - 概要: ja/README.md
    - カートサービス:
      - API仕様書: ja/cart/api-specification.md
```

### 实验结果

所有 49 个 MD 文件（含无 front matter 的 `README.md`）全部正常渲染，左侧导航层级正确，搜索可用，代码高亮完整。

---

## 部署配置对比

### MkDocs Material（当前 `main` 分支方案）

**所需文件（2 个）：**

```
mkdocs.yml                        ← 唯一配置文件（主题 + 导航）
.github/workflows/docs.yml        ← CI/CD（push to main 自动触发）
```

**GitHub Pages Settings：**
```
Source: GitHub Actions
```

**workflow 核心步骤：**
```yaml
- run: pip install mkdocs-material
- run: mkdocs build --strict
- uses: actions/upload-pages-artifact@v3
  with: { path: site/ }
- uses: actions/deploy-pages@v4
```

---

### Jekyll + just-the-docs（`jekyll-docs` 分支的实验方案）

**所需文件（3 个）：**

```
docs/_config.yml                   ← Jekyll 配置（主题 + defaults）
docs/Gemfile                       ← Ruby 依赖声明
.github/workflows/jekyll-docs.yml  ← CI/CD
```

**必须在 Gemfile 中声明的依赖（just-the-docs 需要全部）：**
```ruby
gem "jekyll"
gem "jekyll-remote-theme"
gem "jekyll-seo-tag"         # ← 缺失会构建失败
gem "jekyll-sitemap"         # ← 缺失会构建失败
gem "jekyll-include-cache"   # ← 缺失会构建失败
```

**workflow 核心步骤：**
```yaml
- uses: ruby/setup-ruby@v1
  with: { ruby-version: '3.1' }
- run: cd docs && bundle install
- run: cd docs && bundle exec jekyll build --destination ../site
- uses: actions/upload-pages-artifact@v3
  with: { path: site/ }
```

---

## 本项目的选择：MkDocs Material ✅

当前 `main` 分支使用 MkDocs Material，`jekyll-docs` 分支保留 Jekyll 实验版本。

**选择 MkDocs 的理由：**

1. **零改动 MD 文件** — 49 个文档文件完全不变
2. **导航完全可控** — `mkdocs.yml` 里定义完整的中文/英文两级导航树
3. **构建简单可靠** — Python 单包 `mkdocs-material`，无依赖地狱
4. **视觉效果更好** — Material Design，暗/亮主题切换，代码复制按钮

**访问地址：** https://hunaifei-trec.github.io/kugelpos-backend/
