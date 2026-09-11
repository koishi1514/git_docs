# GEO测试内容

在人与LLM对话的过程中. 智能搜索是实时进行的, 我们GEO优化的目的是使得优化后的网页能够被LLM的网页搜索爬虫访问, 侧重点并不是使得网页的内容被向量化后进入LLM的知识库, 所以黑盒测试可以实时进行. (0827更新)
测试重点在于对网站内容做静态审查, 对于LLM回复质量的评估转入监测部分, 本文档可以作为对前端页面进行geo优化的检查文档(0830更新)

LLM对话过程中的触发智能搜索的整个流程如下:
用户输入->查询,意图解析->提取搜索关键词->爬取网页->提取网页内容->文本分块->检索,重排序->向量检索->LLM回复.

检索,重排序->向量检索 过程是黑盒的. 无法测试.

重点测试的环节变成 抓取网页, 提取网页内容, 文本分块 这3个环节. 前两者主要通过审查网页结构进行; 文本分块则先模拟分块, 然后对结果进行审计.

GEO静态测试可以分为三层: **传输与可达性**(爬虫可以到达并获取页面内容), **内容解析与结构**(页面内容以LLM易于提取的方式组织), **内容质量与语义**(LLM认为这个页面的内容可信且高质量)

## L1 传输与可达性

### 站点配置

爬虫发现一个页面，首先依赖robots.txt的可访问性和其中声明的规则。若robots.txt无法访问或配置了 `Disallow: /` 针对特定 AI 爬虫，则整个站点或部分路径可能被跳过。Sitemap 文件为爬虫提供了站点结构信息，可以确保重要页面被收录。`Sitemap` 和`Content-Signal` 都是robots.txt的指令，前者若未在 robots.txt 中声明，或 Sitemap 本身格式错误、缺少目标 URL，都会导致页面不被发现；后者则声明网站对 AI 训练和搜索两种使用场景的偏好。llms.txt是一个经过整理的内容索引文件，可以提高内容被 LLM 检索和引用的概率。

检查项：

- `GET /robots.txt` 应返回状态码 200 且内容不为空。
- `User-agent: GPTBot` / `ClaudeBot` / `PerplexityBot` 的 `Disallow` 规则不应屏蔽目标路径（不出现 `Disallow: /` 或 `Disallow: /path/`）。
- `robots.txt` 中应包含 `Sitemap:` 指令，且对应的 Sitemap URL 可访问。
- Sitemap XML 文件应格式正确，无解析错误。
- 当前页面 URL 应出现在 Sitemap 条目中。
- Sitemap 中的 `<lastmod>` 日期应为 ISO 8601 格式（如 `2026-08-26`），且不超过当前日期。
- `GET /llms.txt` 应返回状态码 200
- `robots.txt` 中的 `Content-Signal:` 指令若声明，应取 `allow` 或 `disallow` 明确值。
- `<meta name="robots">` 不应包含 `noindex` 或 `nofollow`。

### HTTP 传输

爬虫通过 HTTP 请求获取页面内容，响应头的各项参数直接影响爬虫的处理行为。HTTP 请求状态码 200 表示页面正常, HTTPS 加密可以提升LLM对页面的信任。；`Content-Type` 必须为 `text/html` 否则爬虫可能放弃解析；`Content-Encoding` 启用压缩可提升抓取效率；`Cache-Control` 影响爬虫的抓取频率；`X-Robots-Tag` 可以在 HTTP 头层面控制索引行为，优先级高于 `robots.txt`；`Link` 头（RFC 8288）可提供资源发现线索。

检查项：

- GET 目标 URL 应返回状态码 200。
- URL 协议应为 `https://`。
- 响应头 `Content-Type` 应包含 `text/html`。
- 响应头 `Content-Encoding` 建议为 `gzip` 或 `br`（启用压缩为佳，不强制）。
- 响应头 `Cache-Control` 中 `max-age` 建议存在且 ≥ 3600 秒。
- 响应头 `X-Robots-Tag` 不应包含 `noindex` 或 `nofollow`。
- 响应头 `Link` 若存在，应格式符合 RFC 8288 且指向有效资源。

### 安全头检查

缺少安全头可能导致浏览器安全警告或混合内容问题，间接降低 LLM 对页面的信任。此外，部分爬虫可能因安全头缺失而降低抓取优先级。

检查项：

- `Strict-Transport-Security` 头应存在且 `max-age` ≥ 31536000。
- `Content-Security-Policy` 头建议存在（至少 `default-src 'self'`）。
- `X-Frame-Options` 头应为 `DENY` 或 `SAMEORIGIN`。
- `X-Content-Type-Options` 头应为 `nosniff`。
- `Referrer-Policy` 头应为 `strict-origin-when-cross-origin` 或更严格（如 `no-referrer`）。
- `Permissions-Policy` 头应存在且未过度限制核心功能。

## L2 内容解析与结构

### HTML 语义化结构

爬虫解析 HTML 时，语义化标签帮助其区分内容的主次。`<main>`/`<article>` 标识核心内容区域，排除导航、页脚等样板。标题标签 `<h1>`–`<h6>` 的层级连续性反映了内容的逻辑结构，跳级会让爬虫难以理解内容层次。`<nav>` 和 `<footer>` 明确导航和页脚，避免这些内容污染主体文本提取。

检查项：

- DOM 中应存在 `<main>` 或 `<article>` 中的至少一个。
- 页面中应有且仅有一个 `<h1>`。
- 标题层级顺序（h1→h2→h3）应连续，没有 h1→h3 等跳跃。
- 导航链接应包裹在 `<nav>` 标签内。
- 页脚内容应包裹在 `<footer>` 标签内。
- `<head>` 中的 render-blocking CSS/JS 建议使用 `async`/`defer` 或 `media="print"` 等优化手段。

### 元数据

元数据可以帮助爬虫和 LLM 了解页面，控制爬虫的行为。`<title>` 是页面的核心标识；`<meta name="description">` 提供摘要；`<link rel="canonical">` 防止重复内容；`<meta name="robots">` 控制爬虫行为，错误配置可能导致页面被排除索引。

检查项：

- `<title>` 标签应存在且非空，长度建议在 10-60 字符之间。
- `<meta name="description">` 应存在，内容长度建议在 50-160 字符之间。
- `<link rel="canonical">` 应存在且指向有效 URL。
- `<meta name="robots">` 不应包含 `noindex` 或 `nofollow`。
- `<meta name="viewport">` 应存在且配置正确（如 `width=device-width, initial-scale=1`）。

### Schema 语法与类型检测

JSON-LD 是 LLM 理解页面实体的关键渠道。语法错误或 `@context`/`@type` 不正确会让爬虫无法解析；框架注入风险（如 React/Vue 客户端渲染）可能导致 JSON-LD 在原始 HTML 中不可见，爬虫无法直接提取。使用已移除或受限的 Schema 类型可能导致 LLM 或搜索引擎无法正确解析。

检查项：

- JSON-LD 内容应能被正常解析为 JSON，无语法错误。
- `@context` 的值应为 `https://schema.org`。
- `@type` 应为 Schema.org 官方定义的有效类型（如 `Article`、`Product`）。
- 若页面使用 React/Vue/Next.js/Nuxt，JSON-LD 应在初始 HTML 中可见，而非由客户端注入。
- `@type` 不应包含 `HowTo`（已移除）。
- `@type` 不应包含 `SpecialAnnouncement`（已弃用）。
- `@type` 若为 `FAQPage`（受限），建议迁移至 `QAPage`。

### 页面类型 Schema 属性要求

不同页面类型应部署对应的 Schema 类型，并包含必要的属性，以便 LLM 准确理解页面实体。

检查项：

- **首页 Organization schema**：应包含 `name`、`url`、`knowsAbout`、`hasOfferCatalog`。
- **服务页 Service schema**：应包含 `provider`、`areaServed`、`hasOfferCatalog`、`description`。
- **术语页 DefinedTerm schema**：应包含 `name`、`description`、`inDefinedTermSet`。
- **解决方案页 ScholarlyArticle schema**：应包含 `author`、`datePublished`、`about`。
- **案例页 CaseStudy schema**：应包含 `author`、`about`（含 Project 的 industry/technicalFeature/serviceType）、`result`。
- **FAQ 块 FAQPage schema**：应对每个 Q/A 独立标记 `Question`/`Answer`。
- **全局导航 ContactPoint schema**：应包含联系信息（电话、邮箱等）。

### 身份标记（作者与机构）

作者署名、机构信息和发布日期让内容可追溯，是 LLM 评估内容可信度的重要信号。完整的 Schema 标记让 LLM 直接获取来源信息，无需从正文中推断。

检查项：

- 应存在 `<meta name="author">` 或 Person schema 中的作者信息，至少一种。
- 应存在 `<time>` 标签或 `datePublished` 属性，且为 ISO 8601 格式。
- `author` 应为 Person 对象（非字符串），且包含 `name` 和 `url`。
- 建议存在 Organization/LocalBusiness schema，且包含 `name` 和 `url`。
- Article schema 应包含 `headline`、`author`、`datePublished` 三个属性。

## L3 内容质量与语义

### 内容可提取性

爬虫获取 HTML 后，会提取纯文本内容用于后续处理。若核心内容依赖 JavaScript 渲染（如 SPA），则爬虫可能获取不到有效文本。同时，Markdown 内容协商（`Accept: text/markdown`）可让爬虫直接获得结构化文本，提高提取效率。

检查项：

- 禁用 JS 渲染后，`<body>` 内应有实质文本。
- GET 请求带 `Accept: text/markdown` 头时，返回 Markdown 格式为佳（可选）。

### 内容分块适应性

RAG在处理网页内容时，会按照一定的逻辑进行文本分块。例如开源框架的LangChain 的 HTMLHeaderTextSplitter 和 LlamaIndex 的 SimpleWebPageReader会利用 `<h1>`–`<h6>` 作为语义边界，并按段落/句子递归切割。

检查项：

- 使用 `HTMLHeaderTextSplitter` 模拟分块时，`<h1>`–`<h6>` 应能被识别为有效边界，至少识别出 3 个层级。
- 按 `["\n\n", "\n", ".", " ", ""]` 递归切割后，每块应包含完整语义单元，字数 ≥ 50 且不截断句子。
- 分块结果中应排除 `<script>`、`<style>`、导航和页脚内容。

### 答案块质量（规则）

论文《GEO: Generative Engine Optimization》认为：最优 AI 引用段落的长度区间为 134–167 词，自包含性，信息密度和事实丰富度都会影响LLM的引用决策。

检查项：

- 每块字数建议落在 134-167 词区间（可放宽至 100-200）。
- 代词（he/she/it/they/this）占总词数的比例应 < 2%。
- 专有名词数量（通过 NER 或正则匹配大写词组）应 ≥ 3 个。
- 平均句长（总词数 ÷ 总句数）应在 10-20 词之间。
- 段落数（空行分隔）应 ≥ 3 段。

### 答案块质量（LLM 评估）

以下 3 个检查方向交由 LLM 进行语义层面的评估，弥补规则检查无法覆盖的语义判断：

- **自包含性（Self-Containment）**：将段落输入 LLM，判断该段落脱离上下文后是否仍能被独立理解。检查代词指代是否明确、是否依赖外部信息才能理解核心含义。
- **信息密度（Information Density）**：将段落输入 LLM，判断单位篇幅内承载的有效信息量。检查是否存在冗余表述、空泛描述，或信息重复。
- **事实丰富度（Factual Richness）**：将段落输入 LLM，判断是否包含足够的事实支撑。检查数据、引用、具体案例是否充分支撑段落中的核心论点。

### 联系信息

联系信息增加透明度，是内容信息完整度的体现。

检查项：

- 页面中应出现 email / phone / address 等联系方式，至少一种。
