---
title: 用 Chirpy 搭建博客的几处关键配置
date: 2026-09-22 08:00 +0800
categories: [Blog, Jekyll]
tags: [chirpy, jekyll, github-pages]
---

本站基于 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy){:target="_blank" rel="noopener"}
主题（Jekyll 4 + 内置搜索、分类、标签、PWA）。这里记录几处最容易踩坑的配置，方便以后回看。

## 1. baseurl 决定样式能否加载

主题通过 `/assets/css/jekyll-theme-chirpy.css` 引入样式表，这个路径由 `baseurl` 拼接而来。
部署在 GitHub 项目页（`https://<user>.github.io/<repo>/`）时，必须写明仓库名：

```yaml
# _config.yml
url: "https://mcaarony.github.io"
baseurl: "/myblog"
`baseurl` 写成空字符串时，站点仍然部署在 `/myblog/` 路径下，但所有静态资源指向根路径，
结果是 **页面能打开、样式全部 404**。

> 若将来绑定独立域名（新增 `CNAME` 文件），记得同步把 `baseurl` 改回 `""`。
{: .prompt-warning }

## 2. 中文排版与时区

```yaml
lang: zh-CN            # 主题自带 _data/locales/zh-CN.yml，界面文案随语言切换
timezone: Asia/Shanghai # 需要 tzinfo gem，否则 CI（UTC）下文章时间会差 8 小时
```

页面语言还可以单独覆盖：`pages/resume-en.md` 的 front matter 里写了 `lang: en`，
于是这一页的界面文案回英文，正文语言属性也跟着变。

## 3. 自定义样式放在哪里

主题的样式入口是 `assets/css/jekyll-theme-chirpy.scss`。Jekyll 4 中「站点文件优先于主题 gem 文件」，
所以在仓库里放一个同名文件即可接管样式表；站点自己的样式拆到 `_sass/append.scss`，
由入口文件在主题之后 `@use` 引入：

```text
assets/css/jekyll-theme-chirpy.scss  →  @use 'main.bundle'（主题） → @use 'append'（本站）
```

> 自定义样式只用主题暴露的 CSS 变量（如 `--text-color`、`--btn-border-color`、`--card-shadow`），
> 浅色 / 深色模式与主题切换按钮就能自动生效，不需要为 `[data-bs-theme]` 单独写分支。
{: .prompt-tip }

## 4. 本地预览

```sh
# 需要 Ruby 3.1+，官方镜像最省事：.devcontainer/devcontainer.json
bundle install
bash tools/run.sh        # http://127.0.0.1:4000/myblog/
```

## 5. 目录约定

| 路径 | 作用 |
| --- | --- |
| `_config.yml` | 站点配置 |
| `_tabs/` | 顶部/侧栏导航页（`order` 控制顺序，`icon` 用 Font Awesome 类名） |
| `pages/` | 不进导航的独立页面 |
| `_posts/` | 文章，文件名 `YYYY-MM-DD-title.md` |
| `_data/` | 社交图标（contact.yml）、分享按钮（share.yml）、界面文案覆写 |
| `_sass/` | 站点自定义样式 |
| `assets/img/` | 头像等自有图片 |
