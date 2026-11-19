---
title: Cloudflare 实战：域名托管、Workers / Pages 与证书
date: 2026-11-19 08:00 +0800
categories: [Cloudflare]
tags: [cloudflare, workers, pages, ssl]
---

Cloudflare 把 CDN、DNS、DDoS 防护、WAF 和边缘计算打包在同一个账号里，免费套餐就足以支撑个人建站、跑小接口和做演示项目。
本文按「域名接入 → 跑代码 → 配证书」的顺序，把最常用的几块能力串起来，数字均以 2026 年 11 月的官方文档为准。

## 1. Cloudflare 是什么

一句话：**它是挡在你源站前面的一层全球反向代理**，同时兼任权威 DNS 和安全网关。

请求链路大致是这样：

```text
浏览器 → Cloudflare 边缘节点（就近接入、TLS 终止、缓存/WAF）→ 源站（VPS / 家里的 NAS / 对象存储）
```

几个关键点：

- **Anycast 就近接入**：同一个 IP 在全球上百个机房同时宣告，用户自然落到最近的一个（官方网络页当前列出 348 个城市）。
- **TLS 在边缘终止**：浏览器只和 Cloudflare 握手，源站不需要暴露真实 IP，也不需要能被公网直接访问（这一点在「第 3 节 域名代理」展开）。
- **安全与缓存都是开关**：WAF、限速、缓存规则、重定向都在同一套规则体系里，免费版也有基础能力。
- **免费额度真的够用**：静态站点可以无限带宽，Workers 每天 10 万次请求，Pages 每月 500 次构建。

免费套餐可用的部分（个人项目常用的）：

| 能力 | 免费额度（摘要） |
| --- | --- |
| DNS 托管 + CDN | 不限带宽，任何套餐都不限请求数 |
| Universal SSL 证书 | 免费、自动签发与续期 |
| Workers | 100,000 请求/天，CPU 10 ms/次 |
| Pages | 500 次构建/月，单站点 20,000 个文件 |
| Workers KV | 1 GB 存储，100,000 读/天、1,000 写/天 |

不适合它的场景也要说清楚：需要持久化长连接的自建服务（直接上 Nginx 更省事）、要跑长时间 CPU 计算（Workers 免费版 CPU 只有 10 ms）、以及必须在境内低延迟备案访问的业务（这条要看具体合规要求）。

## 2. 先把域名交给 Cloudflare

后面所有功能的前提：域名在 Cloudflare 上处于 **Active** 状态。

1. 注册 Cloudflare 账号，`Add a domain`（现在叫 Onboard a domain）填根域 `example.com`。
2. 选择套餐（Free 即可），Cloudflare 会自动扫描现有 DNS 记录，扫漏的手工补。
3. 到域名注册商处把 **NS 改成 Cloudflare 分配的两个地址**（形如 `xxx.ns.cloudflare.com`）。
4. 等待激活，一般几分钟到几小时，最长 24 小时。

这套叫 **Full setup（全量接入）**，免费版只能走这条。另一种 **CNAME / Partial setup（部分接入）** 只把某个子域 CNAME 过来，需要 Business 以上套餐或通过合作伙伴，个人项目基本用不到。

> 域名如果在 Cloudflare Registrar 注册，本身就用 Cloudflare 做权威 DNS，可以跳过改 NS 这一步。
{: .prompt-tip }

### 橙云与灰云

DNS 记录右侧那个云朵图标决定了这条记录**是否走代理**：

| | 橙云 Proxied | 灰云 DNS only |
| --- | --- | --- |
| 返回的解析结果 | Cloudflare 的 Anycast IP | 源站真实 IP |
| CDN / 缓存 / WAF / DDoS | 全部生效 | 全都不生效 |
| 真实 IP 是否暴露 | 隐藏 | 暴露 |
| TTL | 固定 Auto（300 秒），不可改 | 自行设置 |

只有 A、AAAA、CNAME 能开代理，MX、TXT 永远是 DNS only。另外注意：**同一个名字下有多条 A 记录时，只要有一条开着橙云，Cloudflare 会把整个名字都按代理处理**。

## 3. 域名代理后的两个副作用

### 源站看到的 IP 不再是访客 IP

开启代理后，源站日志里的 `REMOTE_ADDR` 是 Cloudflare 节点的地址。真实访客 IP 在请求头里：

- `CF-Connecting-IP`：访客真实 IP（Cloudflare 自己加的，最可靠）
- `X-Forwarded-For`：可能被上游改过，用的时候要谨慎
- `True-Client-IP`：仅 Enterprise 套餐

Nginx 里恢复真实 IP 的最小写法：

```nginx
set_real_ip_from 173.245.48.0/20;   # 按官方 IP 段补全全部网段
set_real_ip_from 103.21.244.0/22;
real_ip_header CF-Connecting-IP;
```

### 源站仍然可能被直连

知道源站 IP 的人可以绕过 Cloudflare 直接打你的机器，缓存、WAF、限速全部失效。常见做法：

1. 源站防火墙只放行 Cloudflare 的 IP 段（可脚本定期同步官方列表）。
2. 源站只监听内网或本机，用 **Cloudflare Tunnel** 出去，根本不开放公网端口。
3. 开 **Authenticated Origin Pulls**，要求回源请求必须带 Cloudflare 客户端证书。

> 顺便一个经典坑：用来做域名归属验证的 CNAME/TXT 记录不要开橙云，开了会验证失败。
{: .prompt-warning }

## 4. Workers：在边缘跑一段 JS

Workers 跑在 V8 isolate 里，不是容器，所以没有冷启动问题；一次请求的代码在离用户最近的机房执行。免费额度：**100,000 请求/天**（UTC 零点重置）、CPU 10 ms/次、内存 128 MB、单次请求 50 个子请求。

### 最小项目

```sh
npm create cloudflare@latest -- demo-api   # 模板选 Worker only + JavaScript
cd demo-api
npx wrangler dev      # 本地 http://localhost:8787
npx wrangler deploy   # 部署到 demo-api.<你的子域>.workers.dev
npx wrangler tail     # 实时日志
```

生成的配置和数据目录：

```text
demo-api/
├── src/index.js          # 入口
├── wrangler.jsonc        # 项目配置（名字、入口、绑定、路由）
└── package.json
```

入口就是一个 `fetch` 处理器，返回 `Response` 即可：

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    if (url.pathname === '/api/health') {
      return Response.json({ ok: true, at: Date.now() });
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

### 绑定资源（KV、D1、R2）

要存数据就加绑定，`env` 上就能直接拿到客户端：

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "demo-api",
  "main": "src/index.js",
  "compatibility_date": "2026-11-19",
  "kv_namespaces": [{ "binding": "CACHE", "id": "<KV_NAMESPACE_ID>" }],
  "routes": [{ "pattern": "api.example.com", "custom_domain": true }]
}
```

```js
export default {
  async fetch(request, env) {
    const hit = await env.CACHE.get('greeting');
    if (hit) return new Response(hit);

    await env.CACHE.put('greeting', 'hello from kv');
    return new Response('cache miss, written');
  },
};
```

KV 最终一致（写入后其它机房可能要几十秒才看到），写同一个 key 免费版限制 1 次/秒，**不要拿它当计数器或库存**，那类场景用 D1 或 Durable Objects。

### 绑自定义域名

`routes` 里写 `custom_domain: true`，Cloudflare 会自动建 DNS 记录并签证书，前提是域名已在该账号下激活。与之相对的 `route`（配合 `zone_name`）只接管某个域名下的部分路径，适合把 `/api/*` 挂到已有站点上。

> Workers 免费版每天 10 万次请求是**账号级**的，跟部署了几个 Worker 无关；超了返回 `Error 1027`。CPU 超了是 `Error 1102`（`Worker exceeded resource limits`），一般只有做图像处理、大 JSON 解析时才会撞到。
{: .prompt-warning }

Workers 适合做什么：轻量 API、鉴权网关、A/B 分流、URL 重写、定时任务（Cron Triggers，免费版 5 个）、给静态站加一层后端。

## 5. Pages：静态站点托管

Pages 解决的是另一件事：**给一个构建产物做全球分发**，顺带可以挂 Functions。

### 走 Git 集成（推荐）

在 Dashboard 里 `Workers & Pages → Create application → Pages → Connect to Git`，选仓库后填三项：

| 配置项 | 本站（Jekyll）的取值 |
| --- | --- |
| Production branch | `main` |
| Build command | `bundle exec jekyll build` |
| Build output directory | `_site` |
| 环境变量 | `RUBY_VERSION = 3.1.3`（对齐 Gemfile 要求） |

之后每次 push 都会自动构建，`main` 之外的分支生成**预览部署**（`<hash>.<project>.pages.dev`），PR 里可以直接看效果。这部分跟本站原来用 GitHub Actions 发布到 GitHub Pages 是同一个思路，只是构建机和 CDN 换成了 Cloudflare。

> Jekyll 站点搬过来要把 `_config.yml` 里的 `baseurl` 改回空字符串、`url` 改成自己的域名，否则还是带着 `/myblog` 前缀，资源路径会 404。
{: .prompt-warning }

### 直接上传（Direct Upload）

不想连 Git 也行，本地构建完推产物：

```sh
bundle exec jekyll build
npx wrangler pages deploy _site --project-name=myblog
```

注意：**Git 集成的项目和 Direct Upload 项目不能互相切换**，选之前想清楚。另外 Pages 文件数上限 20,000，单个文件 25 MiB，构建超时 20 分钟，免费版每月 500 次构建、同时只跑 1 个。

### Pages Functions

在项目根目录建 `functions/`，文件名就是路由：

```text
functions/
├── api/
│   ├── hello.js      → /api/hello
│   └── [id].js       → /api/:id
└── _middleware.js    → 全站中间件
```

```js
// functions/api/hello.js
export async function onRequestGet({ request, env }) {
  return Response.json({ ok: true, ua: request.headers.get('user-agent') });
}
```

函数请求**计入 Workers 的额度**，所以 Pages 免费版实际是「静态无限 + 每天 10 万次函数调用」。

另外两个纯静态文件很有用：`_headers` 配响应头（缓存、安全头），`_redirects` 配 301（最多 2,100 条）。

### Pages 还是 Workers？

两者的边界正在消失：Cloudflare 现在**新项目主推 Workers + Static Assets**（`assets.directory` 指向构建产物），Pages 转为维护状态但仍可用，官方也提供了从 Pages 迁移到 Workers 的指南。

| | Pages | Workers（+ Static Assets） |
| --- | --- | --- |
| 定位 | 静态站点为主，函数为辅 | 应用为主，静态资源是其中一部分 |
| 构建 | 平台托管构建（Git 集成） | 本地/CI 构建后 `wrangler deploy` |
| 请求额度 | 静态不限，函数算 Workers 额度 | 全部算 Workers 额度 |
| 预览环境 | 每个分支自动预览 | 用 Versions / 多 Worker 自行组织 |
| 现状 | 可用，不再加新特性 | 官方推荐的新项目路径 |

结论：**纯静态博客、文档站，Pages 上手最快**；需要路由、绑定、定时任务混在一起的项目，直接用 Workers。

## 6. 证书：链路有两条，别只配一条

```text
浏览器 ──①──> Cloudflare 边缘 ──②──> 源站
        边缘证书              源站证书
```

第 ① 段永远由 Cloudflare 负责，第 ② 段取决于你选的加密模式和源站配置。

### 边缘证书：Universal SSL

域名激活后自动签发，免费、DV（只验证域名所有权）、**有效期 90 天**，到期前 30 天开始自动续期，不需要任何操作。覆盖范围是根域 + 一级子域：

- `example.com` ✅
- `www.example.com` ✅
- `a.b.example.com` ❌（二级子域不在覆盖内）

二级及更深的子域要靠 **Total TLS**（需要 Advanced Certificate Manager，付费）或单张 **Advanced Certificate**，也可以在部分接入（CNAME setup）下由 Cloudflare 单独签发。

证书一直 `Pending` 时按这几点排查：域名是否 Active、源站 80 端口是否可达（HTTP DCV）、`CAA` 记录里是否放行了 Cloudflare 合作的 CA（Let's Encrypt / Google Trust Services / SSL.com）。

### 源站证书与加密模式

| 模式 | 浏览器→CF | CF→源站 | 适用 |
| --- | --- | --- | --- |
| Off | 无 TLS | HTTP | 不要用 |
| Flexible | HTTPS | HTTP 明文 | 源站没法装证书时的临时方案 |
| Full | HTTPS | HTTPS，不校验证书 | 源站用自签证书 |
| Full (strict) | HTTPS | HTTPS，校验有效证书 | **推荐** |
| Strict (SSL-Only Origin Pull) | HTTPS | 仅接受带客户端证书的回源 | 高安全场景 |

源站证书两种选法：

- **Let's Encrypt**：公网可信，90 天，必须配自动续期（certbot / acme.sh）。浏览器直连源站时也正常。
- **Cloudflare Origin CA**：在 Dashboard 一键签发，可签长有效期（最长 15 年），**只有 Cloudflare 信任它**。所以一旦暂停 Cloudflare 或把记录改成灰云，访客就会看到不受信任的证书报错。Cloudflare 也不会发到期提醒，自己记进证书清单。

> 最坑的组合是 `Flexible` + 源站自己强制跳 HTTPS：Cloudflare 用 HTTP 回源、源站 301 到 HTTPS、浏览器再请求 Cloudflare……无限重定向。要么上 `Full (strict)`，要么把源站的强制跳转关掉改由 Cloudflare 的 **Always Use HTTPS** 处理。
{: .prompt-danger }

### 除了证书本身还要开的开关

| 设置 | 建议 |
| --- | --- |
| Always Use HTTPS | 开，统一 301 到 HTTPS |
| Automatic HTTPS Rewrites | 开，自动修正页面里的 `http://` 混链 |
| Minimum TLS Version | TLS 1.2（兼容性与安全折中） |
| HSTS | 确认全站 HTTPS 可用后再开，且先给短 `max-age`；开了以后回退很难 |
| Opportunistic Encryption | 开（HTTP/2 与 `Alt-Svc`） |

`Cloudflare → SSL/TLS → Edge Certificates` 页能看到证书的签发时间、到期时间和覆盖的域名列表，出问题时先看这里。

## 7. 常见报错对照

| 现象 | 八成是这个原因 |
| --- | --- |
| 525 SSL handshake failed | 源站 443 没开或证书没配好 |
| 526 Invalid SSL certificate | 源站是自签证书，但模式选了 Full (strict) |
| 522 Connection timed out | 源站不响应，通常是防火墙挡了 Cloudflare IP 段 |
| 524 A timeout occurred | 源站处理超过 100 秒，把慢任务改成异步 |
| 重定向次数过多 | Flexible 模式 + 源站强制 HTTPS |
| 证书名不匹配（深层子域） | Universal SSL 只覆盖一级子域 |
| 站点一直是 Pending | NS 未生效、CAA 记录拦了 CA、80 端口不通 |
| Error 1027 | Workers 当天请求数超限 |
| Error 1102 | Worker 单次 CPU 超过 10 ms（免费版） |

## 8. 本站要用的组合

把这套落到自己的博客上，最小可用配置是：

1. 域名 NS 指向 Cloudflare（Full setup，Free 套餐）。
2. Jekyll 站点交给 Pages，构建 `bundle exec jekyll build`、输出 `_site`、`RUBY_VERSION=3.1.3`，`baseurl` 改回空。
3. 自定义域用 `www` + apex，橙云开启，Universal SSL 自动签发。
4. 加密模式选 `Full (strict)`；源站如果只是 Pages，这一步由 Cloudflare 内部完成，不用管。
5. 开 Always Use HTTPS，TLS 最低 1.2，稳定一段时间后再上 HSTS。

> 文中的额度与行为核对时间为 2026-11，Cloudflare 调整频繁，动手前建议再扫一眼官方文档的对应页面。
{: .prompt-info }

## 参考资料

- [Cloudflare 官方文档](https://developers.cloudflare.com/){:target="_blank" rel="noopener"}
- [Workers 限额](https://developers.cloudflare.com/workers/platform/limits/){:target="_blank" rel="noopener"}
- [Pages 限额](https://developers.cloudflare.com/pages/platform/limits/){:target="_blank" rel="noopener"}
- [代理状态（橙云 / 灰云）](https://developers.cloudflare.com/dns/proxy-status/){:target="_blank" rel="noopener"}
- [Universal SSL](https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/){:target="_blank" rel="noopener"}
- [加密模式](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/){:target="_blank" rel="noopener"}
- [Cloudflare Origin CA](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/){:target="_blank" rel="noopener"}