---
title: "Hugo 双端同步部署：GitHub Pages + srht.site 完整实操"
date: 2026-06-14
draft: false
tags: ["Hugo", "GitHub Pages", "srht.site", "hut", "部署", "双端同步"]
---

![Dual Deploy](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&h=600&fit=crop)

**问题背景**：同一套 Hugo 源码，想同时部署到：
- **源站**：`https://ningfeiyu.github.io` (GitHub Pages)
- **镜像站**：`https://ninq.srht.site` (sr.ht Pages)

核心痛点：**`baseURL` 冲突** —— 构建时写死哪个域名，另一端的所有绝对链接（导航、canonical、sitemap、资源引用）就会指错。

---

## 根因分析

Hugo 在构建时会把 `baseURL` 注入到：
- 导航菜单 `<a href="https://baseURL/posts/">`
- Canonical 标签 `<link rel="canonical" href="https://baseURL/...">`
- Sitemap `<loc>https://baseURL/...</loc>`
- 部分主题内置的绝对资源链接

若只用一个 `hugo.toml`，双端必有一端链接错指。

---

## 解决方案：双配置 + 独立构建目录

### 1. 目录结构

```
ningfeiyu.github.io/
├── hugo.toml              # 主配置：baseURL = "https://ningfeiyu.github.io/"
├── hugo.srht.toml         # 镜像配置：baseURL = "https://ninq.srht.site/" + relativeURLs = true
├── content/
├── themes/PaperMod/       # 已内置主题（非 submodule）
├── public/                # GitHub Pages 产物
└── public_srht/           # srht.site 产物
```

### 2. 两份配置文件

**`hugo.toml`（GitHub Pages 版）**
```toml
baseURL = "https://ningfeiyu.github.io/"
languageCode = "zh-cn"
title = "Ningfeiyu"
theme = "PaperMod"
# 其余参数保持默认
```

**`hugo.srht.toml`（srht.site 版）**
```toml
baseURL = "https://ninq.srht.site/"
languageCode = "zh-cn"
title = "Ningfeiyu"
theme = "PaperMod"
relativeURLs = true   # 关键：资源链接用相对路径 /assets/... 而非绝对路径
```

> `relativeURLs = true` 让 CSS/JS/图片等静态资源输出相对路径，避免跨域加载失败。

### 3. 双端构建脚本 `deploy-dual.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO_DIR="/tmp/ningfeiyu.github.io"
cd "$REPO_DIR"

echo "🔄 拉取最新代码..."
git pull origin master
git submodule update --init --recursive 2>/dev/null || true

# ========== 1. GitHub Pages 版 ==========
echo "📦 构建 GitHub Pages 版..."
hugo -c hugo.toml -d public --minify --gc

# GitHub Actions 会自动部署 public/ 到 gh-pages 分支
# 如需手动推送：
# cd public && git init && git add . && git commit -m "deploy $(date)" && git push -f git@github.com:ningfeiyu/ningfeiyu.github.io.git master:gh-pages

# ========== 2. srht.site 版 ==========
echo "📦 构建 srht.site 版..."
hugo -c hugo.srht.toml -d public_srht --minify --gc

echo "📤 部署到 srht.site..."
cd public_srht
tar -czf /tmp/site_srht.tar.gz .
hut pages publish -d ninq.srht.site /tmp/site_srht.tar.gz

# ========== 3. 验证双端同步 ==========
echo "🔍 验证双端同步..."
verify_sync() {
    local url="$1"
    local label="$2"
    local code=$(curl -s -o /dev/null -w "%{http_code}" "$url" --max-time 10)
    local content=$(curl -s "$url" --max-time 10)
    local opc3_count=$(echo "$content" | grep -c "OPC 3.0" || echo 0)
    
    if [[ "$code" == "200" && "$opc3_count" -gt 0 ]]; then
        echo "  ✅ $label: HTTP $code, 内容正常 (OPC 3.0 × $opc3_count)"
        return 0
    else
        echo "  ❌ $label: HTTP $code, 内容异常 (OPC 3.0 × $opc3_count)"
        return 1
    fi
}

verify_sync "https://ningfeiyu.github.io/posts/opc-3-advanced-2026-guide/" "GitHub Pages"
verify_sync "https://ninq.srht.site/posts/opc-3-advanced-2026-guide/" "srht.site"

# 验证导航链接域名正确性
echo "🔗 验证导航链接..."
gh_nav=$(curl -s https://ningfeiyu.github.io/ | grep -o 'href="https://ningfeiyu.github.io/[^"]*"' | head -1)
srht_nav=$(curl -s https://ninq.srht.site/ | grep -o 'href="https://ninq.srht.site/[^"]*"' | head -1)

[[ "$gh_nav" == *'ningfeiyu.github.io'* ]] && echo "  ✅ GitHub Pages 导航指向正确" || echo "  ❌ GitHub Pages 导航错误: $gh_nav"
[[ "$srht_nav" == *'ninq.srht.site'* ]] && echo "  ✅ srht.site 导航指向正确" || echo "  ❌ srht.site 导航错误: $srht_nav"

echo "🎉 双端同步部署完成！"
```

### 4. 给脚本执行权限并运行

```bash
chmod +x deploy-dual.sh
./deploy-dual.sh
```

---

## 关键技术点总结

| 环节 | GitHub Pages | srht.site | 说明 |
|------|--------------|-----------|------|
| **配置文件** | `hugo.toml` | `hugo.srht.toml` | 独立 `baseURL` |
| **相对链接** | 不需要 | `relativeURLs = true` | 避免静态资源跨域 |
| **构建目录** | `public/` | `public_srht/` | 互不干扰 |
| **部署工具** | GitHub Actions (`actions/deploy-pages`) | `hut pages publish` | 各自原生方式 |
| **域名验证** | 自动 | `hut meta show` 确认用户 | sr.ht 需配置 token |

---

## 自动化：GitHub Actions 双端触发

在 `.github/workflows/deploy-dual.yml` 中添加 srht.site 部署步骤：

```yaml
name: Deploy to GitHub Pages + srht.site

on:
  push:
    branches: [master]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.147.0'
          extended: true
      
      - name: Build GitHub Pages
        run: hugo -c hugo.toml -d public --minify
      
      - name: Build srht.site
        run: |
          hugo -c hugo.srht.toml -d public_srht --minify
          cd public_srht && tar -czf /tmp/site_srht.tar.gz .
      
      - name: Deploy to srht.site
        if: ${{ secrets.HUT_TOKEN != '' }}
        run: |
          echo "instance \"srht.site\" { access-token \"${{ secrets.HUT_TOKEN }}\" }" > ~/.config/hut/config
          hut pages publish -d ninq.srht.site /tmp/site_srht.tar.gz

  deploy-github:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: github-pages
          path: public
      - uses: actions/deploy-pages@v4
        id: deployment
```

> 只需在 GitHub 仓库 Settings → Secrets 添加 `HUT_TOKEN` 即可自动双端部署。

---

## 常见坑 & 避坑指南

| 现象 | 原因 | 修复 |
|------|------|------|
| srht.site 导航指向 github.io | 只用一份配置，`baseURL` 写死 | 双配置文件 + 独立构建 |
| CSS/JS 404 | `relativeURLs` 未开启，绝对路径跨域 | `hugo.srht.toml` 加 `relativeURLs = true` |
| 文章不显示 | 未加 `--buildFuture` 或日期在未来 | 文章 front matter `date` ≤ 今天，或加 `-F` |
| 子模块为空 | `git submodule update --init` 未跑 | CI 里 `submodules: recursive` |
| hut 认证失败 | 配置文件格式不对 | 用 `instance "srht.site" { access-token "..." }` 格式 |

---

## 一键同步命令速查

```bash
# 手动双端同步（本地跑）
cd /tmp/ningfeiyu.github.io && ./deploy-dual.sh

# 仅同步 srht.site（文章改动后快速同步）
cd /tmp/ningfeiyu.github.io && hugo -c hugo.srht.toml -d public_srht && cd public_srht && tar -czf /tmp/s.tar.gz . && hut pages publish -d ninq.srht.site /tmp/s.tar.gz

# 验证双端
curl -s https://ningfeiyu.github.io/posts/opc-3-advanced-2026-guide/ | grep -c "OPC 3.0"
curl -s https://ninq.srht.site/posts/opc-3-advanced-2026-guide/ | grep -c "OPC 3.0"
```

---

## 结语

**核心原则**：一份源码，两份配置，两个构建目录，两条部署链路 —— 互不干扰，各自正确。

这样既保留了 GitHub Pages 的免费 CDN + 自动 HTTPS，又拥有了 sr.ht 的极客友好、去中心化镜像站。双端同步，抗审查、抗单点故障，真正实现「内容所有权归你」。

---

*📝 源码仓库：`github.com/ningfeiyu/ningfeiyu.github.io`*  
*🔗 双端地址：`ningfeiyu.github.io` / `ninq.srht.site`*  
*🛠 工具链：Hugo 0.147 + PaperMod + GitHub Actions + hut 0.6*