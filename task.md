# 任务记录（myblog / Chirpy）

> 记录规范：时间 → 事件内容 → 状态（开始 / 进行中 / 待测试 / 已结束）→ 对应任务
> 状态为「已结束」需用户确认后才可归档。

---

## T-001 按 Chirpy 规范完善项目结构与 CSS 样式

- **时间**：2026-09-22 14:53 CST
- **需求**：这是一个基于 Chirpy 的博客网站，根据 Chirpy 完善项目结构以及 CSS 样式
- **状态**：待测试（代码已提交，等待本地/CI 构建验证）

### 现状排查（对照 chirpy-starter + jekyll-theme-chirpy v7.6.0）

| # | 问题 | 影响 | 处理 |
| --- | --- | --- | --- |
| 1 | `_config.yml` 中 `baseurl: ""`，但站点部署在 `github.com/McAaronY/myblog`（项目页，无 CNAME） | **全站 CSS/JS/图片 404，页面变成无样式裸 HTML**（`/assets/css/jekyll-theme-chirpy.css` 少了 `/myblog` 前缀） | 改为 `baseurl: "/myblog"`，并补 `url` |
| 2 | `lang: en`，内容以中文为主 | UI 文案、日期、字体全部按英文渲染 | 改为 `zh-CN`（主题自带 `_data/locales/zh-CN.yml`）；英文简历页用 `page.lang: en` 单独指定 |
| 3 | `timezone` 为空 | CI（UTC）下文章时间偏移 8 小时 | 设为 `Asia/Shanghai`，Gemfile 补 `tzinfo` |
| 4 | `github.username` / `twitter.username` 仍是模板占位符 | 侧栏社交链接指向不存在账号 | github 改 `McAaronY`，twitter 置空（主题会自动隐藏） |
| 5 | `avatar` 为空 | 侧栏头像空白 | 新增 `assets/img/avatar.svg` 占位头像 |
| 6 | 简历页内联 `<style>`，硬编码 `#eee` / `#eef3f8` 等颜色 | 深色模式下几乎不可读；样式散落在内容里 | 迁移到 `_sass/append.scss`，全部改用 Chirpy CSS 变量 |
| 7 | 简历页写死绝对链接 `/resume/`、`/resume-en/` | 加 baseurl 后链接 404 | 改用 `\|\| relative_url` |
| 8 | `_tabs/resume-cn.md` 缺 `icon` / `order` | 侧栏导航图标为空、排序不可控 | 补 front matter |
| 9 | 无 `assets/css/jekyll-theme-chirpy.scss` 自定义样式入口 | 无法安全扩展主题样式 | 按主题注释「append your custom style below」新增入口 + `_sass/append.scss` |
| 10 | `_posts/` 为空 | 首页 / 归档 / 分类 / 标签全为空壳 | 新增 1 篇中文示例文章 |
| 11 | 缺少本地开发上下文文档 | 反复踩同类坑 | 更新 README，新增本文件 |

### 子任务

- [x] 1. 修正 `_config.yml`（语言 / 时区 / baseurl / url / 社交 / 头像）
- [x] 2. 新建自定义样式入口 `assets/css/jekyll-theme-chirpy.scss`
- [x] 3. 新建 `_sass/append.scss`（中文排版、简历组件、深色模式、打印、减弱动效）
- [x] 4. 重构 `_tabs/resume-cn.md`：去内联样式、补导航 front matter、相对链接
- [x] 5. 重构 `pages/resume-en.md`：同上 + `lang: en`
- [x] 6. 新增 `_posts/2026-09-22-hello-chirpy.md`
- [x] 7. 新增 `assets/img/avatar.svg`
- [x] 8. `Gemfile` 增加 `tzinfo`（Linux/macOS MRI 时区支持）
- [x] 9. 更新 `README.md`（结构说明、本地预览、常见坑）
- [x] 13. 提交 `f612ec9` 并推送到 GitHub（走 SSH，见异常 3），等待 Actions 部署后由用户验证

### 待测试 / 验证

- [ ] 10. **本地构建验证未执行**：机器仅有系统 Ruby 2.6.10，Chirpy 7.6 要求 `ruby ~> 3.1`，
      且未安装 jekyll/bundler 依赖（按项目规范不安装额外软件）。
      需在 DevContainer（`.devcontainer/devcontainer.json` → `mcr.microsoft.com/devcontainers/jekyll:2-bullseye`）
      或 GitHub Actions 上验证：
      - `bash tools/run.sh` 后访问 `http://127.0.0.1:4000/myblog/`，确认样式正常加载
      - 检查深色/浅色模式下的简历页
      - 检查浏览器打印预览（简历导出 PDF）
- [ ] 11. 头像需替换为真实照片（当前为字母占位图）
- [ ] 12. 若将来绑定独立域名：新增 `CNAME`，并把 `baseurl` 改回 `""`、`url` 改为该域名

### 异常记录

| 次数 | 时间 | 异常 | 处理 |
| --- | --- | --- | --- |
| 1 | 2026-09-22 | 本机无 Ruby 3.x，无法运行 `bundle exec jekyll build` 做构建校验 | 改为静态校验（YAML front matter / Liquid 变量名 / 与主题 v7.6.0 文件结构逐字比对）+ 云端 CI 验证 |
| 2 | 2026-09-22 | `github.com`、`rubygems.org` 直连超时，仅 `api.github.com` 可用 | 通过 GitHub Contents API 读取主题 v7.6.0 源码做规范比对，不下载/安装任何软件 |
| 3 | 2026-09-22 | **推送失败**：`git push` 报 `Failed to connect to github.com port 443`。原因：`/etc/hosts` 里有 2026-07-20 生成的 GitHub 加速条目，但**缺少 `github.com` 本身**，解析到 20.205.243.166 且 443 端口不互通（已验证 4 个候选 IP 均 000）；`api.github.com`/`codeload.github.com` 因 hosts 里有映射而正常 | 不改系统文件、不关 TLS 校验；改用已验证可用的 SSH 通道一次性推送：`git push git@github.com:McAaronY/myblog.git main:main`（origin 保持 https 不变）。预防措施：让 GitHub 加速工具重新生成 hosts（需用户 sudo），或将 origin 改为 `git@github.com:McAaronY/myblog.git`（需用户确认） |

---

## T-002 完善 Cloudflare 文章（介绍 / Workers & Pages / 域名代理与证书）

- **时间**：2026-11-19 CST
- **需求**：完善 `_posts/2026-11-19-clouflare.md`，覆盖 1) Cloudflare 介绍 2) Workers 与 Pages 的具体使用 3) 域名代理与证书
- **状态**：进行中 → 待测试（内容已写完，构建校验受本机 Ruby 版本限制）

### 做了什么

| # | 内容 | 说明 |
| --- | --- | --- |
| 1 | 重写 front matter | 标题改为「Cloudflare 实战：域名托管、Workers / Pages 与证书」，`categories: [Cloudflare]`，纠正原 `cloudfrare` / `cloudfrare` 拼写错误 |
| 2 | 第 1 节 介绍 | 反向代理 + 权威 DNS + 边缘计算定位、请求链路、免费额度表、不适用场景 |
| 3 | 第 2 节 域名接入 | Full setup 改 NS 流程、Full vs Partial(CNAME) 接入、橙云/灰云行为对比表 |
| 4 | 第 3 节 代理副作用 | `CF-Connecting-IP` 取真实 IP、Nginx `real_ip` 片段、源站防直连三招 |
| 5 | 第 4 节 Workers | C3 脚手架、wrangler 命令、fetch 处理器示例、KV 绑定、custom_domain、免费额度与 1027/1102 报错 |
| 6 | 第 5 节 Pages | Git 集成（本站 Jekyll 构建参数表）、Direct Upload、Functions 路由、`_headers`/`_redirects`、Pages 与 Workers 选型表 |
| 7 | 第 6 节 证书 | 两条链路图、Universal SSL（90 天 / 一级子域 / CAA）、加密模式表、Origin CA vs Let's Encrypt、Always Use HTTPS / HSTS 等开关 |
| 8 | 第 7 节 报错对照 | 525 / 526 / 522 / 524 / 重定向循环 / 深度子域证书 / 1027 / 1102 |
| 9 | 第 8 节 本站落地 | 域名托管 + Pages + Full (strict) 的最小组合清单；附官方文档参考链接 |

### 核对来源（均为 2026-11 抓取的官方文档）

- `workers/platform/limits`、`pages/platform/limits`：请求 / CPU / 构建 / 文件数 / 自定义域上限
- `dns/proxy-status`、`dns/zone-setups/full-setup/setup`：代理状态与接入流程
- `ssl/edge-certificates/universal-ssl`、`ssl/reference/certificate-validity-periods`、`ssl/origin-configuration/ssl-modes`、`ssl/origin-configuration/origin-ca`、`ssl/edge-certificates/additional-options/total-tls`
- `pages/framework-guides/deploy-a-jekyll-site`、`workers/get-started/guide`、`workers/configuration/routing/custom-domains`、`fundamentals/reference/http-headers`、`kv/platform/limits`

### 待测试 / 验证

- [x] 静态校验：front matter 结构、代码围栏配对（22 个标记）、`{: .prompt-* }` 类型合法（tip/warning/danger/info）、无残留 Liquid 语法、无行尾空格与制表符
- [ ] **未执行 `jekyll build`**：本机仅 Ruby 2.6.10 且无 jekyll gem（同 T-001 异常 1），需在 DevContainer / GitHub Actions 上确认页面渲染与表格、提示块样式正常
- [ ] 待用户确认文中「本站落地」一节是否要按实际域名（当前仍是 `mcaarony.github.io/myblog`）调整

### 异常记录

| 次数 | 时间 | 异常 | 处理 |
| --- | --- | --- | --- |
| 1 | 2026-11-19 | 本机无 jekyll / Ruby 3.1，无法构建校验 | 用脚本做 front matter、围栏、prompt 块、Liquid 残留的静态检查，构建交给 CI |

---

## T-003 Cloudflare 文章改为新手向 + 增加图示

- **时间**：2026-11-19 CST
- **需求**：T-002 的文章表述过于生硬、初学者难理解，需要有图片/图示支撑；文末的「参考资料」一节不要
- **状态**：进行中 → 待测试

### 做了什么

| # | 内容 | 说明 |
| --- | --- | --- |
| 1 | 全文改为讲解式 | 每节先给一句「这节解决什么」，再给操作步骤；术语用白话表解释 |
| 2 | 开头建立直觉 | 用「快递驿站 / 小区前台」类比讲清源站、回源、边缘节点、橙云灰云 |
| 3 | 新增 7 张 mermaid 图 | 请求链路、谁在做 DNS、橙云判断、Workers 部署流、Pages 构建流、选型判断、证书两条链路 |
| 4 | 删除文末「参考资料」 | 按用户要求去掉 |
| 5 | 新增「常见问题速查」 | 把报错码、Pending、缓存不更新等新手问题收成一张表 |
| 6 | 标题改为「Cloudflare 上手：域名、Workers / Pages 与证书（图解版）」 | 与内容形态一致 |

### 图示方案说明

沿用本仓库已有做法（`_posts/2026-09-22-AI-agent.md` 已用 mermaid），`_config.yml` 已全局开启 mermaid，
`_includes/js-selector.html` 会按 `page.mermaid` 加载主题自带的 `assets/lib/mermaid/mermaid.min.js`（v11），
不新增本地图片文件、不引入新依赖、不受外部图床失效影响。

### 验证

- [x] 7 个 mermaid 代码块用 mermaid@11（主题同版本）在 Chromium 中 `parse` + `render` 全部通过，并截图核对中文标签与布局；图 1、图 2 已按截图反馈调整文案与方向
- [x] 代码围栏 32 个标记配对、front matter 正常、无残留 Liquid、无行尾空格
- [ ] **仍未执行 `jekyll build`**：本机 Ruby 2.6.10 且无 jekyll gem（同 T-001 异常 1），页面实际样式（提示块、表格、图示容器宽度）需在 CI / DevContainer 确认

### 异常记录

| 次数 | 时间 | 异常 | 处理 |
| --- | --- | --- | --- |
| 1 | 2026-11-19 | 首版图 2 用两个 subgraph 并排，渲染后顺序与方向反直觉、文字拥挤 | 改为「谁来做 DNS」判断图，单列布局 |
| 2 | 2026-11-19 | 图 1 的箭头标签过长，换行后压住箭头 | 精简为「④ 返回响应（已缓存 / 已防护）」 |

### T-003 补充（2026-11-19）

- 需求：介绍部分要给出 Cloudflare 的官方域名/入口
- 处理：开头新增「官方入口」小表（官网、控制台、开发文档、状态页），第 2 节注册步骤直接链接控制台；未恢复文末「参考资料」一节
- 验证：四个域名均实测可访问（`dash.cloudflare.com` 对脚本返回 403 是其自身 WAF 拦截，浏览器正常）；结构检查通过（围栏 32 个配对、7 个 mermaid 块、无 Liquid 残留、无行尾空格）

---

## T-004 移除文章末尾的 CC BY 4.0 授权行

- **时间**：2026-11-19 CST
- **需求**：文章底部出现「本文由作者按照 CC BY 4.0 进行授权」，要求移除
- **状态**：待测试

### 原因

该行由主题 `_layouts/post.html` 的 `post-tail-bottom` 无条件渲染，条件是
`{% if site.data.locales[lang].copyright.license.template %}`，
文案来自 `_data/locales/zh-CN.yml`（2026-09-22 提交 `124f2ce` 加入），与文章内容无关。

已用已部署的老文章核对：`https://yyl202248.ch.eu.org/posts/AI-agent/` 同样存在该行，
说明不是本次新增文章带来的变化。

### 处理

- `_data/locales/zh-CN.yml`：`copyright.license.template` 改为 `false`
- 关键坑：**不能留空、也不能删掉 license 段**。Jekyll `Utils#merge_values` 中
  `if new_val.nil? then old_val`，站点值为 nil 会回退到主题字符串；只有 `false` 能真正覆盖。
- 未动分享图标（`_data/share.yml`），如需一并移除再单独改

### 验证

- [x] `ruby -ryaml` 解析确认 `template` 为 `FalseClass`（Liquid 中 false 为假 → 整块跳过）
- [x] 通读主题 `post.html`：`license.template` 仅出现在该 `{% if %}` 及其内部，`home.html` / `footer.html` 无引用
- [ ] 未执行 `jekyll build`（本机 Ruby 2.6.10 且无 jekyll gem），实际页面需 CI 部署后确认
- [ ] 已知范围限制：本次只覆盖 `zh-CN` locale；若以后有 `lang: en` 的文章，需在 `_data/locales/en.yml` 做同样处理
