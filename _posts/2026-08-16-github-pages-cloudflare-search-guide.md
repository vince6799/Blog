---
layout: post
title: "把一个 GitHub Pages 博客真正跑起来：独立域名、Cloudflare 与搜索收录"
description: "以本站的搭建过程为例，完整记录 GitHub Pages 发布、独立域名、Cloudflare DNS、百度与 Google 搜索提交，以及几处容易踩到的坑。"
category: 建站记录
reading_time: 约 27 分钟阅读
---

用 GitHub Pages 建一个能打开的页面并不难。新建仓库，放进 `index.html`，再到设置里打开 Pages，十几分钟就能看到结果。

麻烦通常从后半段开始：怎样让它成为一个可以长期维护的博客；怎样把 `vince6799.github.io/Blog/` 换成自己的域名；Cloudflare 应该开灰云还是橙云；Sitemap、RSS、分享卡片和统计代码分别放在哪里；百度和 Google 又怎样才能发现这些文章。

这篇文章记录的就是本站从空仓库到 `blog.a80s.com` 的完整过程。它不是一套适用于所有项目的模板，里面的域名、仓库名和目录都是真实配置。换成自己的值以后，也可以直接作为一份搭建手册使用。

## 目录

- [先把边界想清楚](#scope)
- [建立仓库与配置 Jekyll](#repository)
- [启用 GitHub Pages](#pages)
- [绑定独立域名](#custom-domain)
- [把仓库检出到本地](#local-checkout)
- [用 Cloudflare 管理 DNS](#cloudflare)
- [按地区限制文章访问](#regional-access)
- [补齐 SEO、RSS、robots.txt 与 Sitemap](#search-basics)
- [向百度提交站点](#baidu)
- [向 Google 提交站点](#google)
- [配置访问统计](#analytics)
- [隐藏暂不公开的文章](#hidden-content)
- [发布后的检查顺序](#post-publish-checks)

## 先把边界想清楚
{: #scope}

本站采用的组合很简单：

- GitHub 保存文章和页面源码；
- GitHub Pages 负责构建和托管静态站点；
- Jekyll 把 Markdown 文章生成 HTML；
- Cloudflare 管理 `a80s.com` 的 DNS，必要时也可以代理流量；
- Google Analytics 统计访问；
- 百度搜索资源平台和 Google Search Console 负责搜索提交与索引观察。

这套方案几乎没有服务器维护成本，文章就是仓库中的文件，备份和版本历史也顺手解决了。代价同样明确：它是静态站点，没有数据库和服务端程序；Pages 页面是公开内容，不适合存放秘密；按国家返回不同页面、登录、评论审核等功能，都需要借助外部服务或边缘代理。

先承认这些限制，后面的选型会省事很多。

## 建仓库时，先别急着挑主题
{: #repository}

仓库名可以任意取。本站使用的是：

```text
vince6799/Blog
```

个人账号并非只能有一个 GitHub Pages 仓库。名为 `USERNAME.github.io` 的用户站点比较特殊，每个账号只有一个；普通项目仓库也能分别启用 Pages。本站属于后者，所以最初的默认地址带有仓库名：

```text
https://vince6799.github.io/Blog/
```

一开始没有使用现成主题，而是先建立最小目录：

```text
Blog/
├── _config.yml
├── _includes/
├── _layouts/
├── _posts/
├── assets/
├── index.html
├── about.html
├── robots.txt
└── CNAME
```

`_posts` 放文章，文件名遵循 `YYYY-MM-DD-title.md`；`_layouts` 放页面骨架；`_includes` 放可以复用的 CSS 或片段；`assets` 保存分享图和静态资源。

本站当前的核心配置如下：

```yaml
title: A80s · 程序员手记
description: 一位拥有 20 多年经验的程序员，分享经验、感悟、测评与生活观察
lang: zh-CN
timezone: Asia/Shanghai

url: "https://blog.a80s.com"
baseurl: ""
permalink: /articles/:title/

markdown: kramdown
theme: null

plugins:
  - jekyll-feed
  - jekyll-seo-tag
  - jekyll-sitemap
```

这里最容易混淆的是 `url` 和 `baseurl`。

使用默认项目地址时，站点在 `/Blog` 下面，`baseurl` 通常要设为 `/Blog`。启用 `blog.a80s.com` 后，博客已经位于这个域名的根目录，继续保留 `/Blog` 反而会生成错误链接，所以本站改成了：

```yaml
url: "https://blog.a80s.com"
baseurl: ""
```

`permalink: /articles/:title/` 则把文章地址统一为：

```text
https://blog.a80s.com/articles/mcp-deep-dive/
```

域名切换完成以后，URL 中不再需要 `/Blog`。原因很直接：独立域名已经成为站点入口，不再经过原来的项目路径。

## GitHub Pages 的发布源
{: #pages}

进入仓库：

```text
Settings → Pages → Build and deployment
```

选择：

```text
Source: Deploy from a branch
Branch: main
Folder: /(root)
```

然后点击 Save。GitHub 官方支持从分支根目录或 `/docs` 目录发布；本站的 Jekyll 文件都在根目录，因此使用 `/(root)`。

这里曾遇到一个很容易误判的问题：分支和目录都显示正确，Save 却是灰色。第一反应往往是怀疑 `_config.yml` 没写对，其实两者没有关系。Save 只负责保存发布源；当界面中的选择与当前配置相同，按钮本来就可能不可点击。判断是否真的启用成功，应该看仓库的 Actions 页面有没有 `pages build and deployment`，以及 Pages 设置中是否给出了站点地址。

修改 `_config.yml` 不会让一个灰色的 Save 按钮突然可用。发布源和 Jekyll 配置是两层事情。

## 给项目地址换上独立域名
{: #custom-domain}

GitHub Pages 设置中的 Custom domain 填写：

```text
blog.a80s.com
```

仓库根目录同时保留一个 `CNAME` 文件，内容只有一行：

```text
blog.a80s.com
```

DNS 中则创建：

| 类型 | 名称 | 目标 |
|---|---|---|
| CNAME | `blog` | `vince6799.github.io` |

目标里没有 `https://`，也没有 `/Blog`。DNS 记录只认识主机名，不认识 URL 路径。GitHub 的文档也明确要求子域名直接指向 `USERNAME.github.io`，不能把仓库名接在后面。

配置顺序最好是先在 GitHub Pages 中登记 Custom domain，再修改 DNS。这样可以减少子域名被其他 Pages 项目占用的风险。DNS 生效后，等待 GitHub 完成域名检查和证书签发，再打开 `Enforce HTTPS`。

第一次绑定域名时，GitHub 需要为它签发 HTTPS 证书。这个过程可能只要几分钟，也可能持续数小时；证书准备好之前，`Enforce HTTPS` 往往不可选。此时不要反复删除 `CNAME`，也不要在 Cloudflare 中用额外的重定向规则强行补 HTTPS。后文会提到，Cloudflare 设为 `Flexible` 才是常见的循环重定向来源。

## 把远程仓库检出到现有目录
{: #local-checkout}

新目录最省事：

```bash
git clone https://github.com/vince6799/Blog.git .
```

如果当前目录已经初始化过 Git，或者里面还留着预览文件，可以先确认没有同名冲突，再连接远程分支：

```bash
git remote add origin https://github.com/vince6799/Blog.git
git fetch origin main
git switch -C main --track origin/main
```

不要为了省事直接清空目录。未跟踪文件不属于仓库，但仍可能是有用的草稿或预览。先看一眼：

```bash
git status --short --branch
```

macOS 还会自动生成 `.DS_Store`。它不应该进入版本库：

```gitignore
.DS_Store
**/.DS_Store
```

如果预览文件统一放在 `outputs/`，又不准备发布，也可以把这个目录加入 `.gitignore`。

## Cloudflare 接管 DNS 以后，先从灰云开始
{: #cloudflare}

在 Cloudflare 中选择 Add a domain，输入 `a80s.com` 并选择套餐。Cloudflare 会尝试扫描现有记录，但扫描结果不能直接当作迁移清单，仍要和原 DNS 控制台逐条核对。网站的 CNAME 很显眼，容易被忽略的反而是邮箱所需的 MX、SPF、DKIM，以及各类 TXT 验证记录。

记录确认无误后，Cloudflare 会给出两台权威 Nameserver。到域名注册商处，把原来的 Nameserver 替换成这两条；这一步是在注册商后台修改域名服务器，不是在 DNS 记录列表里新增两条 NS。等 Cloudflare 中的站点状态变为 Active，再停止使用旧 DNS 服务。传播期间两边的记录保持一致，可以少掉不少偶发解析问题。

本站的博客记录是：

| 类型 | 名称 | 内容 | TTL | 初始状态 |
|---|---|---|---|---|
| CNAME | `blog` | `vince6799.github.io` | Auto | DNS only |

如果以后把博客从 `blog.a80s.com` 移到根域名 `a80s.com`，记录名会从 `blog` 变为 `@`。传统 DNS 规范不允许根域名直接设置 CNAME，因为同一位置还必须存在 SOA、NS 等记录；Cloudflare 会通过 **CNAME Flattening** 处理这个限制，因此可以把 `@` 的 CNAME 目标写成 `vince6799.github.io`。Cloudflare 在对外应答时返回解析后的 IP，而不是暴露一个冲突的根域名 CNAME。

这种写法依赖 DNS 服务商的 Flattening、ALIAS 或 ANAME 能力，并不是所有平台都支持。若将来迁离 Cloudflare，可以改用 GitHub 文档列出的 Pages A/AAAA 记录；不要随手复制某篇旧教程里的 IP，因为 GitHub 可能调整推荐值。

Cloudflare 控制台里的灰云代表 DNS only：Cloudflare 只回答 DNS，不接管 HTTP 流量。橙云代表 Proxied：访问会先经过 Cloudflare，缓存、WAF、流量统计和国家判断才有机会生效。

对一个刚完成域名切换的 GitHub Pages 站点，先使用灰云比较容易排错。确认以下项目都正常后，再决定要不要开橙云：

- `https://blog.a80s.com/` 能打开；
- GitHub Pages 的 DNS check 通过；
- `Enforce HTTPS` 已开启；
- `https://blog.a80s.com/sitemap.xml` 和 `/robots.txt` 返回正常；
- 百度验证文件可以直接访问。

如果只是想使用 Cloudflare 的权威 DNS，保持灰云完全可以。它不会提供 WAF 和边缘缓存，但路径更短，也少一层搜索蜘蛛可能遇到的挑战页面。博客的读者如果主要在中国大陆，橙云也不一定更快，最好用不同运营商的网络实测后再决定。

需要 WAF、缓存或按国家限制访问时，再切橙云。此时 SSL/TLS 模式应使用：

```text
Full (strict)
```

不要选 Flexible。浏览器到 Cloudflare 是 HTTPS、Cloudflare 到 GitHub 却走 HTTP 时，很容易与 GitHub Pages 的 HTTPS 重定向互相打架。GitHub 已经为独立域名签发了可信证书，符合 Full (strict) 的使用条件。

橙云开启后也不要马上叠加一堆“安全优化”。至少应保证这些地址不会遇到验证码或 JavaScript Challenge：

```text
/robots.txt
/sitemap.xml
/baidu_verify_codeva-xawqNP00O2.html
```

否则可能出现一种很隐蔽的故障：普通浏览器打开一切正常，搜索蜘蛛和 GitHub Actions 得到的却是 403 或挑战页。

DNSSEC 也值得开启，但顺序不能反。若原 DNS 服务商已经配置过 DNSSEC，切换 Nameserver 前要先处理旧的 DS 记录；等 Cloudflare 成为权威 DNS 后，再按 Cloudflare 给出的参数在域名注册商处添加新的 DS。旧 DS 与新 DNS 对不上时，结果不是“安全性降低”，而是整个域名解析失败。

## Cloudflare 能做按地区限制，但不是内容保险箱
{: #regional-access}

GitHub Pages 本身无法根据访客 IP 返回不同内容。若确实需要限制某个路径在中国大陆的访问，必须让 `blog` 记录处于橙云状态，再使用 Cloudflare WAF 或 Worker。

WAF 规则可以写成：

```text
(ip.src.country eq "CN" and
 http.request.uri.path eq "/articles/example-review/")
```

动作为 Block。若希望返回普通 404，而不是 Cloudflare 拦截页，可改用 Worker 判断 `request.cf.country` 后自行构造响应。

这只能限制博客入口。仓库是公开的，Markdown 源文件依然可能从 GitHub 被看到；IP 国家识别也存在误差，代理用户显示的是出口 IP。地区规则适合控制访问体验，不应被当作合规审查或保密措施的替代品。

## 搜索引擎需要的基础文件
{: #search-basics}

一个博客至少应让搜索引擎找到三样东西：规范的页面元数据、`robots.txt` 和 Sitemap。本站使用三个 GitHub Pages 支持的 Jekyll 插件：

```yaml
plugins:
  - jekyll-feed
  - jekyll-seo-tag
  - jekyll-sitemap
```

它们分别生成 RSS、常用 SEO 标签和 `sitemap.xml`。默认布局的 `<head>` 中需要调用：

```liquid
{% raw %}
{% seo %}
{% endraw %}
```

RSS 入口则可以直接写进页面头部：

```html
<link rel="alternate"
      type="application/atom+xml"
      title="A80s · 程序员手记"
      href="/feed.xml">
```

`robots.txt` 放在仓库根目录，并通过 Jekyll 输出：

```liquid
---
layout: null
permalink: /robots.txt
---
User-agent: *
Allow: /

Sitemap: {% raw %}{{ '/sitemap.xml' | absolute_url }}{% endraw %}
```

页面内的“一键分享”和分享卡片是两回事。后者是链接被粘贴到社交平台时读取的 Open Graph 图片。本站在 `_config.yml` 中给页面设置默认图片：

```yaml
defaults:
  - scope:
      path: ""
    values:
      image: /assets/share-card.png
```

这张图必须能通过公网 URL 访问，尺寸和文字也要适合缩略显示。平台往往会缓存分享卡片，刚换图片却仍看到旧图，不一定是代码没生效。

Google Analytics 的 Measurement ID 可以写进配置，再由布局加载 `gtag.js`。它解决的是访问统计，不会自动替你完成 Google 搜索提交。Analytics 和 Search Console 是两套产品，这一点很容易混在一起。

## 百度：验证、Sitemap 与 API 推送
{: #baidu}

在百度搜索资源平台添加：

```text
https://blog.a80s.com
```

文件验证最直观。下载百度给出的 HTML 文件后，把文件名和验证字符串保留下来。例如本站使用：

```text
baidu_verify_codeva-xawqNP00O2.html
```

Pages 构建完成后，先直接打开：

```text
https://blog.a80s.com/baidu_verify_codeva-xawqNP00O2.html
```

如果直接把验证文件当作静态文件发布，`jekyll-sitemap` 也可能把它列进 Sitemap。验证地址不是搜索结果候选页，没必要交给搜索引擎，因此本站给文件加了 Front Matter：

```yaml
---
layout: null
permalink: /baidu_verify_codeva-xawqNP00O2.html
sitemap: false
---
ca0457357ae8b0f5bb23e27e26b3f00b
```

显式 `permalink` 很重要：站点若配置了全局永久链接规则，缺少这一行可能把验证文件改成无扩展名路径，原 `.html` 地址会变成 404。Jekyll 构建时会移除 Front Matter，公网文件的正文仍然只有百度给出的验证字符串。Pages 构建完成后，应同时检查两件事：验证 URL 能否返回原字符串，以及 `sitemap.xml` 中是否已经没有这个地址。

验证通过后，在普通收录中提交：

```text
https://blog.a80s.com/sitemap.xml
```

没有 ICP 备案并不等于百度必然拒绝收录。未备案站点的一些平台权益和提交额度会受影响，境外托管也可能让抓取速度不稳定，但最终是否进入索引仍取决于百度能否访问页面以及页面质量。

### 用 GitHub Actions 定时推送

百度普通收录提供 API Token。它应保存为仓库级 Secret：

```text
Settings → Secrets and variables → Actions
Name: BAIDU_PUSH_TOKEN
```

普通工作流使用 Repository secret 即可。Environment secret 只有在 Job 明确绑定某个 Environment 时才会生效。

本站不在每次 push 后立即提交，而是每天北京时间 23:30 运行一次。GitHub Actions 的 Cron 使用 UTC，北京时间 23:30 对应 `15:30 UTC`。

最初的工作流会先下载线上 Sitemap，再提取其中的 URL。后来 Cloudflare 对 GitHub 托管 Runner 返回过 `403`，任务在提交百度之前就中止了。与其继续调低 Cloudflare 的防护，不如取消这项不必要的外部依赖：工作流检出仓库，直接从 `_posts` 生成 URL，并根据 Front Matter 排除隐藏文章。下面是核心部分：

```yaml
name: Submit URLs to Baidu

on:
  workflow_dispatch:
  schedule:
    - cron: "30 15 * * *"

permissions:
  contents: read

jobs:
  submit:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Build public URL list
        shell: bash
        run: |
          set -euo pipefail

          {
            printf '%s\n' \
              'https://blog.a80s.com/' \
              'https://blog.a80s.com/about/'

            while IFS= read -r -d '' post; do
              front_matter=$(awk '
                NR == 1 && $0 == "---" { inside = 1; next }
                inside && $0 == "---" { exit }
                inside { print }
              ' "${post}")

              if printf '%s\n' "${front_matter}" \
                | grep -Eiq '^(hidden:[[:space:]]*true|sitemap:[[:space:]]*false)[[:space:]]*$'; then
                continue
              fi

              permalink=$(printf '%s\n' "${front_matter}" \
                | awk -F':[[:space:]]*' '$1 == "permalink" { print $2; exit }' \
                | tr -d "\"'")

              if [ -z "${permalink}" ]; then
                filename=$(basename "${post}")
                slug=${filename#????-??-??-}
                slug=${slug%.*}
                permalink="/articles/${slug}/"
              fi

              [ "${permalink}" = "/articles/boostnet-review/" ] && continue
              printf 'https://blog.a80s.com%s\n' "${permalink}"
            done < <(find _posts -maxdepth 1 -type f \
              \( -name '*.md' -o -name '*.markdown' -o -name '*.html' \) \
              -print0)
          } | sort -u > urls.txt

      - name: Submit to Baidu
        env:
          BAIDU_PUSH_TOKEN: {% raw %}${{ secrets.BAIDU_PUSH_TOKEN }}{% endraw %}
        run: |
          response=$(curl --fail --silent --show-error \
            -H "Content-Type: text/plain" \
            --data-binary @urls.txt \
            "http://data.zz.baidu.com/urls?site=https://blog.a80s.com&token=${BAIDU_PUSH_TOKEN}")

          echo "${response}" | jq .
          echo "${response}" | jq -e 'has("success")' >/dev/null
```

这里仍然把 BoostNet 路径作为一道显式保险，即使以后误删了文章的 `hidden: true` 或 `sitemap: false`，定时任务也不会把它提交给百度。实际运行时，工作流生成了 6 个地址：首页、关于页和 4 篇公开文章；验证文件与 BoostNet 均不在列表中。

第一次写这个工作流时，提交地址误用了 HTTPS，Actions 返回：

```text
curl: (60) SSL: no alternative certificate subject name
matches target host name 'data.zz.baidu.com'
```

Token 没有参与到这一步，失败发生在 TLS 握手阶段：`data.zz.baidu.com` 的 HTTPS 证书与主机名不匹配。不要用 `curl -k` 关闭证书检查。百度官方示例给出的普通收录端点是：

```text
http://data.zz.baidu.com/urls
```

改回官方端点后，工作流即可正常返回 `success` 和剩余额度。

## Google：Search Console 与 Analytics 不是一回事
{: #google}

Google 的站点提交入口是 Search Console。使用 Cloudflare DNS 时，推荐添加 Domain property：

```text
a80s.com
```

这里不带协议和路径。Domain property 会覆盖 `blog.a80s.com` 以及同一域名下的其他子域名。

Google 会提供一条 TXT 验证记录：

```text
google-site-verification=xxxxxxxxxxxx
```

在 Cloudflare DNS 中添加：

| 类型 | 名称 | 内容 | TTL |
|---|---|---|---|
| TXT | `@` | Google 给出的完整字符串 | Auto |

TXT 记录不存在灰云或橙云。验证成功后不要删除它，否则以后可能失去所有权验证。

进入 Search Console 的“站点地图”，提交：

```text
https://blog.a80s.com/sitemap.xml
```

首页和重要文章还可以使用“网址检查”请求编入索引。这个按钮适合少量重要页面，不适合每次发布后机械地提交整站。Sitemap 也只是发现提示，不是收录承诺；新站从抓取、质量判断到展示，本来就需要时间。

## 访问统计：先确定自己想知道什么
{: #analytics}

搜索平台回答“页面有没有被搜索引擎发现”，访问统计回答的是另一组问题：今天来了多少人、读了哪些文章、从哪里进入、停留了多久。两套数据偶尔能互相印证，却不能相互替代。

个人博客没必要一开始就采集几十个指标。我长期会看的只有几项：文章浏览量、独立访客的大致趋势、入口来源、热门页面，以及移动端和桌面端的比例。数字突然变化时，再去分析具体原因。访问量本来就不大时，自己反复刷新几次便足以扭曲结果，这比少一个高级报表更值得先处理。

### 本站使用 Google Analytics 4

Google Analytics 适合需要分析流量来源、页面路径和自定义事件的站点。配置过程分成两部分：先在 Analytics 后台建立 GA4 媒体资源和 Web 数据流，再把数据流的 Measurement ID 放进网站。这个 ID 以 `G-` 开头，是页面必须公开使用的标识，不是密码，也不需要保存为 GitHub Secret。

本站在 `_config.yml` 中保存当前 ID：

```yaml
google_analytics: G-K00YMYK3FB
```

默认布局 `_layouts/default.html` 的 `<head>` 中读取它：

```liquid
{% raw %}
{% if site.google_analytics %}
<script async src="https://www.googletagmanager.com/gtag/js?id={{ site.google_analytics }}"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag() { dataLayer.push(arguments); }
  gtag('js', new Date());
  gtag('config', '{{ site.google_analytics }}');
</script>
{% endif %}
{% endraw %}
```

把代码放在默认布局，而不是逐篇粘贴，所有页面便会使用同一份配置。发布后打开 GA4 的 Realtime 报告，再用无痕窗口访问一两篇文章；实时报告出现当前访问，才算接入完成。正式报表的处理时间更长，刚部署完没有数据并不一定是故障。

不要同时安装 `gtag.js` 和一套重复触发 GA4 的 Google Tag Manager 配置，否则一次访问可能被计数两次。自己的访问也应排除：固定公网 IP 可以使用 GA4 的 Internal traffic 规则；经常更换网络时，用浏览器拦截脚本或单独的测试环境往往更省事。GA4 的排除过滤一旦启用，会永久丢弃匹配的新数据，先用 Testing 状态观察几天再转为 Active。

Google 的统计依赖访客浏览器成功加载 `googletagmanager.com` 并发送事件。广告拦截器、浏览器隐私设置或网络连通问题都会造成漏记。对于主要面向中国大陆的博客，不能把 GA4 的数字当成服务器日志式的完整访问总量；它更适合观察趋势和来源变化。

### 还有哪些常用选择

| 方案 | 部署方式 | 适合场景 | 需要留意 |
|---|---|---|---|
| Google Analytics 4 | 页面加载 Google Tag | 需要来源归因、事件和较完整的分析能力 | 界面和数据模型较复杂；脚本可能被拦截 |
| Cloudflare Web Analytics | Cloudflare 自动注入或手动添加 Beacon | 已使用 Cloudflare，希望快速查看访问与页面性能 | Web Analytics 的浏览器数据仍可能漏记；边缘请求统计与访客统计不是同一个口径 |
| Umami | 使用其托管服务，或自行部署应用与 PostgreSQL | 希望界面简单、数据由自己掌握 | 自托管意味着另行维护服务、数据库、备份和升级 |
| Plausible | 托管服务或自托管，页面加入轻量脚本 | 只关心页面、来源和目标转化等核心指标 | 托管版收费；高级分析维度少于 GA4 |
| Matomo | 云服务或自托管 | 需要更完整的分析功能，又希望掌握部署和数据 | 自托管成本明显高于静态博客本身 |
| 服务端访问日志 | 从 CDN、反向代理或 Web 服务器汇总请求 | 需要观察爬虫、脚本未执行的访问和状态码 | GitHub Pages 不提供原始 Web 服务器日志；请求数也不能直接等同于真实读者数 |

本站已经把域名放在 Cloudflare 管理，即使 `blog` 记录保持灰云，也可以进入 **Web Analytics → Add a site**，添加 `blog.a80s.com`，再把平台生成的 Beacon 放到 `</body>` 前。若站点切成橙云，Cloudflare 还可以自动注入 Web Analytics，并从边缘侧提供请求统计。这两组数据解决的问题不同：Beacon 更接近浏览器中的真实用户体验，边缘统计看到的是到达 Cloudflare 的 HTTP 请求，其中还包括爬虫和自动程序。

手动接入时应复制控制台为该站点生成的完整代码，不要照抄别人的 Token。结构大致如下：

```html
<script defer
        src="https://static.cloudflareinsights.com/beacon.min.js"
        data-cf-beacon='{"token":"YOUR_SITE_TOKEN"}'>
</script>
```

Umami 的页面接入也很直接。先在 Umami 中添加站点，然后把它生成的脚本放进 `<head>`：

```html
<script defer
        src="https://analytics.example.com/script.js"
        data-website-id="YOUR_WEBSITE_ID">
</script>
```

真正费事的不是这三行代码，而是维护 `analytics.example.com` 背后的应用和数据库。GitHub Pages 只能承载静态页面，无法在同一个 Pages 站点里运行 Umami 服务端；自托管时需要另外准备服务器、容器平台或支持 Node.js 与 PostgreSQL 的托管环境。

### 统计口径不要混着用

GA4、Cloudflare 和 Umami 同时开启并没有技术冲突，但三者给出的数字不会完全一致。独立访客的识别方法、会话超时时间、时区、机器人过滤、广告拦截和数据延迟都不相同。挑一个工具作为长期主口径即可，第二个工具更适合用来检查趋势是否一致，而不是追求两个后台每一天的数字完全相等。

GitHub 仓库的 **Insights → Traffic** 也会显示访问者、热门内容和 Clone，但它统计的是 GitHub 仓库页面，不是 `blog.a80s.com` 的读者。把仓库 Traffic 当成博客 PV，是另一个常见误会。

统计工具会把访问信息发送给第三方，或保存到自己的服务端。启用前应弄清楚它实际采集哪些字段、保存多久、是否使用 Cookie，以及读者所在地区对隐私告知和同意的要求。所谓“无 Cookie”或“隐私友好”可以减少工作量，但不等于在任何地区、任何配置下都自动合规。

## 有些文章不想出现在列表或搜索中
{: #hidden-content}

“不在首页显示”“不出现在 Sitemap”“不让某个搜索引擎抓取”是三件不同的事。

本站给文章增加了可选字段：

```yaml
hidden: true
sitemap: false
```

首页只遍历非隐藏文章：

```liquid
{% raw %}{% assign visible_posts = site.posts
  | where_exp: "post", "post.hidden != true" %}{% endraw %}
```

`hidden: true` 解决列表展示，`sitemap: false` 解决 Sitemap。文章地址本身仍然存在，知道链接的人依旧可以访问。

如果只想阻止百度抓取某个路径，可以在 `robots.txt` 增加：

```text
User-agent: Baiduspider
Disallow: /articles/example-review/

User-agent: *
Allow: /
```

这不会同时阻止 Google。百度已经收录过的旧 URL 也不会立刻消失，索引更新需要时间。

Google 支持 `noindex`。如果页面已经能被 Google 访问，又不希望它出现在结果中，应在页面 `<head>` 输出：

```html
<meta name="robots" content="noindex, nofollow">
```

不要一边用 `robots.txt` 禁止 Googlebot 抓取，一边指望 Google 读取页面里的 `noindex`；爬虫进不了页面，自然看不到这条指令。

## 发布以后，按结果检查，不按按钮猜测
{: #post-publish-checks}

一次正常发布的流程可以很短：

```bash
git status --short
git diff
git add <本次修改的文件>
git commit -m "发布新文章"
git push origin main
```

随后依次检查：

1. GitHub Actions 中 `pages build and deployment` 是否成功；
2. 文章正式 URL 是否返回 200；
3. 首页标题、描述和日期是否正确；
4. `sitemap.xml` 是否包含应公开的文章；
5. `robots.txt` 是否仍然可访问；
6. Cloudflare 是否返回缓存旧页面；
7. 百度定时工作流是否在北京时间 23:30 后成功；
8. Search Console 是否能读取 Sitemap。

真正有用的是这条检查链，而不是反复点击 Save、Verify 或 Request indexing。静态博客的结构并不复杂，故障通常出在几层配置的交界处：GitHub 认为域名没准备好，Cloudflare 已经开始代理；页面已经发布，CDN 仍在返回旧缓存；robots 阻止了爬虫，却又希望它读取 noindex；统计代码正常上报，却以为搜索提交也随之完成。

把每一层的责任分开，问题就容易定位了。

## 这套方案适合什么样的博客

如果文章主要是 Markdown，更新频率不高，又希望保留完整版本历史，GitHub Pages 很合适。独立域名避免站点被仓库路径绑定，Cloudflare 负责 DNS 和可选的边缘能力，搜索平台则提供外部可见性。

它不会把发布变成完全自动的事情，也不会让新文章立刻获得流量。它提供的是一个足够透明的基础：页面怎样生成、域名指向哪里、哪些 URL 交给搜索引擎、一次修改何时上线，都能从配置和提交记录中找到答案。

对于一个准备长期写下去的个人博客，这比“功能很多”更重要。

---

### 相关文档

- [GitHub Pages：配置分支发布源](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)
- [GitHub Pages：管理独立域名](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site)
- [Cloudflare：DNS only 与 Proxied 的区别](https://developers.cloudflare.com/dns/proxy-status/)
- [Cloudflare：Full (strict) SSL 模式](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/)
- [Cloudflare：按国家代码设置 WAF 规则](https://developers.cloudflare.com/waf/custom-rules/use-cases/block-traffic-from-specific-countries/)
- [百度搜索资源平台：API 提交说明](https://ziyuan.baidu.com/college/articleinfo?id=267&page=2)
- [Google Search Console：添加网站资源](https://support.google.com/webmasters/answer/34592)
- [Google：创建并提交 Sitemap](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap)
- [Google：使用 noindex 阻止索引](https://developers.google.com/search/docs/crawling-indexing/block-indexing)
- [Google Analytics：为网站设置 GA4](https://support.google.com/analytics/answer/14183469)
- [Google Analytics：排除内部访问](https://support.google.com/analytics/answer/10104470)
- [Cloudflare Web Analytics：启用与接入](https://developers.cloudflare.com/web-analytics/get-started/)
- [Cloudflare Web Analytics：数据来源与采集方式](https://developers.cloudflare.com/web-analytics/data-metrics/data-origin-and-collection/)
- [Umami：添加统计脚本](https://docs.umami.is/docs/collect-data)
- [Umami：自托管安装说明](https://docs.umami.is/docs/install)
- [Plausible：添加统计脚本](https://plausible.io/docs/plausible-script)
- [GitHub：查看仓库 Traffic](https://docs.github.com/en/repositories/viewing-activity-and-data-for-your-repository/viewing-traffic-to-a-repository)
