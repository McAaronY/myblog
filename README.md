# Chirpy Starter

[![Gem Version](https://img.shields.io/gem/v/jekyll-theme-chirpy)][gem]&nbsp;
[![GitHub license](https://img.shields.io/github/license/cotes2020/chirpy-starter.svg?color=blue)][mit]

A minimal, ready-to-use template for creating a blog with the [**Chirpy**][chirpy] Jekyll theme. Get up and running in minutes with all critical files pre-configured.

## Why This Starter Exists

When installing Chirpy through [RubyGems.org][gem], Jekyll can only read a subset of theme files (`_data`, `_layouts`, `_includes`, `_sass`, `assets`) and limited `_config.yml` options from the gem. As a result, users cannot enjoy the full out-of-the-box experience that Chirpy offers.

To unlock all features, the following files must be present in your Jekyll site:

```shell
.
├── _config.yml
├── _plugins
├── _tabs
└── index.html
```

This starter bundles those files from the latest **Chirpy** release along with a [CD][CD] workflow, so you can start writing immediately.

## Usage

Check out the [theme's docs](https://github.com/cotes2020/jekyll-theme-chirpy/wiki).

---

## 本站（myblog）说明

### 目录结构

```text
.
├── _config.yml              # 站点配置（lang / timezone / baseurl / avatar ...）
├── _data/                   # contact.yml 侧栏社交图标、share.yml 分享按钮
├── _includes/               # （可选）覆盖主题同名模板片段，本站暂未使用
├── _plugins/                # Jekyll 插件：文章 lastmod 取自 git 历史
├── _posts/                  # 文章，文件名 YYYY-MM-DD-<slug>.md
├── _sass/
│   └── append.scss          # ★ 本站自定义样式（中文排版、.resume 组件、打印）
├── _tabs/                   # 侧栏导航页：categories / tags / archives / about / resume-cn
├── assets/
│   ├── css/jekyll-theme-chirpy.scss  # ★ 样式入口（覆盖主题同名文件，末尾 @use 'append'）
│   ├── img/avatar.svg       # 侧栏头像（占位，待替换真实照片）
│   └── lib/                 # chirpy-static-assets 子模块（仅 assets.self_host.enabled=true 时需要）
├── pages/                   # 不进导航的独立页面（resume-en.md）
└── tools/                   # run.sh 本地预览、test.sh 构建 + htmlproofer
```

### 本地预览

需要 **Ruby 3.1+**（Chirpy 7.6 的 `required_ruby_version` 为 `~> 3.1`）：

```sh
bundle install
bash tools/run.sh          # → http://127.0.0.1:4000/myblog/
bash tools/test.sh         # 生产模式构建 + htmlproofer 检查
```

本机 Ruby 版本不够时，直接用仓库自带的 DevContainer 镜像
`mcr.microsoft.com/devcontainers/jekyll:2-bullseye`，或推送后看 GitHub Actions 构建结果。

### 三个最容易出问题的点

1. **`baseurl` 必须是 `/myblog`**：站点部署在项目页子路径，写空字符串会导致
   `/assets/css/jekyll-theme-chirpy.css` 404，页面完全失去样式。换独立域名时改回 `""` 并新增 `CNAME`。
2. **自定义样式不要写内联 `<style>`**：统一放 `_sass/append.scss`，颜色只用主题的 CSS 变量
   （`--text-color`、`--card-bg`、`--btn-border-color` 等），否则深色模式会失效。
3. **页面内链接用 `| relative_url`**：写死 `/resume/` 这类绝对路径在加 `baseurl` 后会 404。

### 加一篇新文章

```sh
# _posts/2026-09-22-my-title.md 的 front matter 至少包含：
# ---
# title: 标题
# date: 2026-09-22 15:30 +0800
# categories: [Blog, Jekyll]   # 数组表示层级
# tags: [chirpy, jekyll]
# ---
```

提示块用引用 + kramdown 属性列表：

```markdown
> 这是一条提示
{: .prompt-tip }     # 可选：prompt-tip / prompt-info / prompt-warning / prompt-danger
```

## Contributing

This repository is automatically updated with new releases from the theme repository. If you encounter any issues or want to contribute to its improvement, please visit the [theme repository][chirpy] to provide feedback.

## License

This work is published under [MIT][mit] License.

[gem]: https://rubygems.org/gems/jekyll-theme-chirpy
[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/
[CD]: https://en.wikipedia.org/wiki/Continuous_deployment
[mit]: https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE
