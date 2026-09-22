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
