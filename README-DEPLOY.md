# Hexo + xnew 主题部署说明

## 已完成

- Hexo 8.1.2 站点已建好，主题 `XadillaX/hexo-xnew` 已接入 `themes/xnew`
- 依赖已安装，`hexo generate` 构建通过（35 个文件）
- 已修复主题原生的 3 个问题（见下文）
- 产出在 `public/`，可直接部署

## ⚠️ 重要环境限制：本机 `/sdcard` 无法安装依赖

这个工作区挂载在 Android 的 FUSE 文件系统（`fuseblk`）上，**不允许创建符号链接**：

```
npm error EACCES: permission denied, symlink '../@babel/parser/bin/babel-parser.js'
```

npm 安装依赖时必须在 `node_modules/.bin/` 建软链接，因此在 `/sdcard` 下**必然失败**。

**解法**：依赖装在 `/root/build/blog`（那里支持软链接），源码同步回来。

```bash
# 首次准备
cp -r /sdcard/dsha/workplace/blog /root/build/blog
cd /root/build/blog
npm install

# 构建
npx hexo generate
# 产出在 /root/build/blog/public/，再拷回 /sdcard/dsha/workplace/blog/public
```

在正常电脑（或 Cloudflare CI）上没有这个问题，直接 `npm install && npm run build` 即可。

## 部署到 Cloudflare Pages

| 配置项 | 值 |
|---|---|
| Framework preset | `Hexo` |
| Build command | `npm run build` |
| Build output directory | `public` |
| `NODE_VERSION` | `22` |

`package.json` 的 `scripts.build` 已经是 `hexo generate`。

## 已修复的主题问题

### 1. favicon 路径双斜杠

`layout/_head.pug` 写的是 `config.root + theme.favicon`，而 `config.root` 是 `/`。
如果主题配置里 `favicon: /favicon.ico`（带前导斜杠），输出会变成 `//favicon.ico`。

**已改**：`themes/xnew/_config.yml` → `favicon: favicon.ico`（不带斜杠）。

注意 `logo` 的用法**不同**：`layout/_header.pug` 直接输出 `theme.logo`，不拼 root，
所以 `logo` 必须带前导斜杠（`/images/avatar.gif`）。两个字段风格相反，改动时别混。

### 2. 硬编码的旧路由 `/pigeonhole`

`layout/_content.pug` 开头把访问 `/archives/` 的请求重定向到 `/pigeonhole`
（作者早期的路由命名），且拼接方式会产生 `//pigeonhole`。

**已改**：删除这段重定向。归档页在模板下方本身有完整渲染逻辑，不需要跳转。

### 3. 缺失的配置字段

主题引用了两个示例配置里没有的字段：

| 字段 | 引用位置 | 处理 |
|---|---|---|
| `theme.favicon` | `layout/_head.pug` | 已在 `themes/xnew/_config.yml` 补上 |
| `config.feed.path` | `layout/_head.pug` | 已在站点 `_config.yml` 的 `feed:` 段补上 |

不补的话，页面会输出 `href="undefined"`。

## 站点配置位置

- **站点级**：`_config.yml`（标题、url、主题选择、feed、代码高亮）
- **主题级**：`themes/xnew/_config.yml`（logo、favicon、菜单、社交图标、disqus）

## 部署前需要改的地方

1. `_config.yml` 的 `url` —— 改成你的实际域名
2. `_config.yml` 的 `title` / `author` / `description`
3. `themes/xnew/_config.yml` 的 `github` 等社交账号

## 已知隐患

### 代码高亮可能失效

主题用一段 jQuery（`layout/_scripts.pug`）重写 Hexo 输出的 `<figure class="highlight">` 结构，
依赖 **jQuery 3.1.1 + highlight.js 9.9.0**（2016-2017 年的版本），从 `cdn.staticfile.net` 加载。

风险点：Hexo 8 的代码块 HTML 结构与这几行选择器（`td.gutter`、`figure.highlight`）可能已不匹配。
**这个失败是静默的** —— 不报错，只是代码块没样式。建议部署后实际打开一篇带代码的文章确认。

站点 `_config.yml` 里已把 `highlight.wrap: true`、`hljs: false` 配成主题期望的形态。

### CDN 在国内

`_head.pug` 和 `_scripts.pug` 引用了 `cdn.staticfile.net`（七牛云）三个资源。
如果面向海外读者，加载会慢。建议换成 jsDelivr 或自托管到 `source/`。

### Disqus 国内不可用

`layout/_comment.pug` 用 Disqus 做评论。中国大陆基本无法访问。
`themes/xnew/_config.yml` 里 `disqus.short_name` 已留空（评论区不显示）。

## 主题技术栈说明

- 模板：**Pug**（`layout/*.pug`），需 `hexo-renderer-pug`
- 样式：**Stylus**（`source/styles/*.styl`），需 `hexo-renderer-stylus`
- 这两个渲染器缺一不可，`package.json` 里已包含
