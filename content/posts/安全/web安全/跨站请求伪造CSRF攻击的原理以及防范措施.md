---
title: "跨站请求伪造CSRF攻击的原理以及防范措施"
subtitle: ""
date: 2025-03-13T12:06:37+08:00
lastmod: 2025-06-16T12:06:37+08:00
draft: false
toc:
  enable: true
  auto: true
weight: false
categories: ["安全"]
tags: ["安全","Web安全"]
slug: "csrf-defense"
featuredImage: "/images/security/cover.jpg"
summary: "详解 CSRF 跨站请求伪造攻击的原理、核心要素与常见攻击手法，并深入讲解 CSRF Token、双重提交 Cookie、Origin/Referer 校验等防范措施。"
---

跨站请求伪造（CSRF）是 Web 安全的核心威胁之一。本文详解 CSRF 攻击的原理、核心要素与常见的 GET/POST 请求伪造手法，深入讲解 CSRF Token、双重提交 Cookie 验证、Origin/Referer 头检查等防范措施，并总结实践防御建议。

<!--more-->

### 前言
跨站请求伪造（Cross-Site Request Forgery, CSRF）攻击。它与 XSS 攻击的目标不同，但危害同样巨大，是 Web 安全的核心威胁之一。

### CSRF 攻击原理

CSRF 攻击的核心在于**欺骗用户的浏览器，让其以用户的身份在已认证（登录）的网站上执行攻击者指定的操作**。

1.  攻击者的目标：攻击者诱使已经登录了目标网站（例如银行网站 `bank.com`）的用户，在用户不知情的情况下，向该目标网站发起一个恶意请求（例如转账给攻击者）。
2.  用户已登录：受害者用户之前已经成功登录了目标网站（`bank.com`），浏览器保存了该网站的会话 Cookie（或其他认证凭证）。只要这个 Cookie 未过期，浏览器在向目标网站发送任何请求时都会自动带上它。
3.  攻击者伪造请求：
    *   攻击者精心构造一个指向目标网站关键功能（例如转账接口 `bank.com/transfer`）的 HTTP 请求。这个请求包含了执行恶意操作所需的所有参数（如收款账户、转账金额）。
    *   **关键点：** 构造的请求可以是 `GET`（更容易，比如隐藏在图片链接或链接里）或 `POST`（常用，隐藏在表单里）请求。
4.  诱骗用户触发请求：
    *   攻击者需要设法让已经登录了目标网站的用户访问一个恶意页面（这个页面可以在攻击者控制的网站、论坛帖子、嵌入邮件的页面、甚至被 XSS 攻击的网站）。
    *   当受害者访问这个恶意页面时，该页面会包含**自动触发**或**诱使用户点击**向目标网站发起伪造请求的代码。
5.  请求被发送并执行：
    *   用户的浏览器访问恶意页面，并按照页面指示（自动或用户触发）向目标网站 `bank.com/transfer` 发起伪造的请求。
    *   由于用户之前已登录 `bank.com`，浏览器会自动将用户的 `bank.com` 会话 Cookie 添加到该请求头中。
    *   `bank.com` 服务器收到这个请求：检查 Cookie 有效 -> 验证用户身份通过 -> 执行请求中的操作（转账）-> 返回结果。
6.  攻击完成：攻击者在用户不知情、无意的情况下，以用户的身份和权限成功执行了恶意操作（如转账给攻击者账户）。受害者直到发现自己账户变动时才会察觉到攻击。

### CSRF 攻击的核心要素

1.  受害者用户已登录目标网站 A（身份认证有效）。
2.  目标网站 A 的会话认证机制仅依赖于 Cookie（或其他浏览器自动发送的信息）。
3.  目标网站 A 存在关键操作接口（如转账、改密、改邮箱、发帖），该接口仅通过会话 Cookie 鉴别用户身份，对于请求的来源没有足够验证（只要 Cookie 对就执行）。
4.  攻击者能够构造出该关键操作的完整 HTTP 请求（知道 URL、参数格式）。
5.  攻击者能诱使受害者用户在登录状态下访问包含能触发该请求的恶意页面。

### 常见的 CSRF 攻击实践（构造恶意请求的方式）

1.  GET 请求伪装：
    *   攻击者直接在恶意页面上嵌入指向目标接口的 `GET` 请求（通过 `<img>`， `<script>`， `<iframe>` 等标签的 `src` 属性）。
    *   **例子：** 恶意页面中嵌入：
        `<img src="http://bank.com/transfer?to=attacker_account&amount=10000" width="0" height="0" />`
    *   用户访问该恶意页面时，浏览器会尝试加载图片，自动向 `bank.com` 发送转账的 `GET` 请求（带上用户 Cookie），完成攻击。攻击者账号收到 10000 元。
    *   **问题：** `GET` 请求通常不应该用于修改数据的操作，但现在也能看到（危害更大）。

2.  POST 请求伪装：
    *   攻击者在恶意页面中构造一个隐藏的表单（`<form>`），表单的 `action` 指向目标接口地址（`bank.com/transfer`），`method` 为 `POST`，表单中包含恶意操作所需的参数（隐藏的输入框）。
    *   使用 JS 在页面加载时自动提交表单（`form.submit()`）。
    *   **例子：**
        ```html
        <body onload="document.forms[0].submit()">
            <form action="http://bank.com/transfer" method="POST">
                <input type="hidden" name="to" value="attacker_account" />
                <input type="hidden" name="amount" value="10000" />
                <input type="submit" value="Click for Prize!" />
                <!-- 也可完全隐藏，靠onload提交 -->
            </form>
        </body>
        ```
    *   用户访问恶意页面时，表单被自动提交（即使提交按钮是诱骗性的或隐藏的），浏览器向 `bank.com` 发送包含恶意参数和用户 Cookie 的 `POST` 请求。

3.  **利用其他标签/方法：** 除了图片和表单，攻击者还可以利用 `<link>`（rel=stylesheet 或 prerender/prefetch）、`<video>`、`<object>`、`XMLHttpRequest`、`fetch()`（如果允许跨域）等方法来发送伪造请求。

### CSRF 攻击的后果

*   资金损失：伪造银行转账操作。
*   账户被接管：伪造修改密码、修改密保邮箱操作。
*   数据泄露/篡改：伪造用户资料修改、删除重要数据操作。
*   非授权操作：伪造发送消息、发布帖子、投票、购物车结算等用户权限范围内的任何操作。
*   声誉损害：受害者在不知情的情况下进行的恶意操作可能损害其个人或公司的声誉。

### CSRF 攻击的防范措施（核心思想：增加请求的不可预测性）

1.  CSRF Tokens（最常用、最有效的方案）：
    *   **原理：** 为每个用户会话生成一个**唯一的、随机的、高强度的 Token（令牌）**。将这个 Token 包含在：
        *   服务端生成的表单中（作为隐藏字段 `<input type="hidden" name="csrf_token" value="random_value">`）。
        *   AJAX 请求的请求头中（`X-CSRF-Token: random_value`）。
    *   **服务器验证：** 在处理任何会改变状态的请求（`POST`， `PUT`， `DELETE`， `PATCH`）时，服务器检查请求中是否携带了有效的 CSRF Token，并且该 Token 与当前用户会话中存储的 Token 一致。
    *   **安全性分析：**
        *   攻击者无法在自己的恶意页面中预先得知这个 Token（因为 Token 是随机的且关联用户会话）。
        *   同源策略（Same-Origin Policy）阻止了恶意页面通过 JavaScript 从合法页面读取 Token（因为恶意页面和目标网站不同源）。
        *   因此，攻击者无法在伪造的请求中包含正确的 Token，服务器会拒绝执行请求。
    *   **实现要点：**
        *   Token 应绑定到用户会话（session），并在用户登录后生成。
        *   Token 应具有足够的熵（长度和随机性）以防猜测。
        *   Token 应在表单提交后立即失效（防止表单重复提交）或者在一个会话期内有效（后者更常见，但风险稍高），登出后应销毁。
        *   使用 `POST` 方法保护携带 Token 的请求（避免 Token 泄漏在 URL 上）。

2.  双重提交 Cookie 验证：
    *   **原理：**
        *   用户登录后，服务器除了设置会话 Cookie（`Set-Cookie: SessionId=...; ...`），还会在客户端设置一个单独的 CSRF Token Cookie（通常不能标记为 `HttpOnly`），如 `Set-Cookie: CSRF-Token=random_value; ...`。
        *   客户端在发起敏感请求（通常是非 `GET`）时，除了浏览器自动在 `Cookie` 头中带上会话 Cookie 和 CSRF-Token Cookie，还需要在请求头（如 `X-CSRF-Token`）或者请求体（如参数 `csrf_token`）中也带上这个 CSRF Token 值。
        *   服务器在验证请求时：
            *   从请求头或请求体中取出 CSRF Token 值（Token 值）。
            *   从请求头中的 Cookie 里取出 CSRF Token Cookie 值。
            *   比较这两个 Token 值是否一致。一致则执行操作。
    *   **安全性分析：**
        *   攻击者可以通过构造恶意请求让浏览器带上合法的会话 Cookie（因为浏览器自动做），**但无法让浏览器带上合法的 CSRF Token Cookie 发送到攻击者控制的服务器**（攻击者拿不到 Cookie）。
        *   攻击者在伪造请求的 `X-CSRF-Token` 头或请求体中放入什么值？他不知道用户真实的 CSRF Token 值（因为 Cookie 不可被恶意 JS 读取），也无法猜测。
        *   因此，服务器收到的伪造请求中，请求头/体中的 Token 值和 Cookie 中的 Token 值不一致（甚至缺失），验证失败。
    *   **优缺点：** 相比单纯使用 Token 在表单中，此方法 JS 更容易获取 Token（从 Cookie 读取）。需要额外注意防止子域泄露问题（设置 Cookie 的 `Domain` 和 `Path` 属性严格控制范围）。现代框架常用此方式结合 Token 使用。

3.  检查 `Origin` / `Referer` HTTP 请求头：
    *   **原理：**
        *   服务器在处理敏感请求时，检查 HTTP 请求头 `Origin`（更可靠）或 `Referer`（存在隐私问题和可被篡改风险，但旧浏览器兼容好）。
        *   **`Origin` 头：** 表示**请求发起的源（协议+域名+端口）**。不能由 JS 设置。浏览器在跨域请求和同站 `POST` 请求中通常会发送。
        *   **`Referer` 头：** 表示**请求来源的完整页面 URL**。可能会被一些防火墙或浏览器插件屏蔽或隐私保护策略移除（空），也可能被篡改（不安全）。
    *   **服务器验证：** 如果请求头中包含 `Origin`，服务器检查它是否是**预期**的合法源（一个或多个白名单源，通常是网站自己的源地址）。如果请求头中没有 `Origin`（旧浏览器或简单请求如 GET），可以检查 `Referer`（如果存在），并确保其来源域是合法的。
    *   **优缺点：**
        *   实现简单。
        *   `Origin` 头比 `Referer` 可靠且安全。
        *   不能完全依赖作为唯一防线。`Referer` 头可能被禁用/篡改/为空。在特定配置的嵌套页面（如 `file:` 协议或 HTTPS 跳 HTTP）下可能存在问题。攻击者在某些中间人攻击（如 Wifi 热点）或浏览器插件漏洞场景下可能绕过。
        *   **建议：** 作为**深度防御**措施，与 CSRF Token 等方法**结合**使用，增加攻击难度。

4.  要求自定义请求头（配合 CORS）：
    *   **原理：**
        *   服务端要求处理敏感操作的请求（通常是 `POST`， `PUT`， `DELETE` 等）必须携带一个特定的、非标准的安全自定义 HTTP 请求头（如 `X-Requested-With: XMLHttpRequest` 或自定义的 `X-Custom-Header: Value`）。
        *   攻击者构造的 CSRF 攻击通常使用 `<form>` 提交，浏览器默认不会（也无法）自动设置自定义的请求头（只会在同源请求中自动设置 Cookie）。
        *   **CORS 策略限制：** 如果服务器设置了严格的 CORS（跨域资源共享）策略，只允许特定的源（自身）发起请求，那么恶意页面发起的跨域 AJAX 请求：
            *   需要预检请求（`OPTIONS`）。
            *   由于请求携带了自定义头（即使 AJAX 设置了 `X-Requested-With` 等头），这个预检请求会被浏览器发送以检查是否被目标服务器允许。
            *   **关键：** 除非目标服务器的 CORS 策略明确允许了攻击者恶意网站的源访问并携带该自定义头，否则预检请求会被服务器拒绝。浏览器就不会发送后续真正的敏感请求。即使恶意网站绕过 AJAX 使用 `<form>`，也无法添加自定义头。
        *   因此，服务器收到没有这个自定义请求头的请求（例如通过 `<form>` 提交的 CSRF 攻击请求），可以拒绝处理。
    *   **优缺点：**
        *   实现相对简单。
        *   利用浏览器安全机制。
        *   **依赖于 CORS 策略的有效配置**，需要严格限制允许的源（`Access-Control-Allow-Origin`）、允许的自定义头（`Access-Control-Allow-Headers`）。错误的 CORS 配置会引入其他严重漏洞（如泄露 API 数据）。
        *   只对 `XMLHttpRequest` / `fetch()` 请求有效，无法防御通过 `<form>` 提交的简单 CSRF 攻击（但 `<form>` 提交无法携带自定义头，服务器会拒绝此类没带头的请求）。
        *   对于使用 JS 库发起 JSONP 请求的老旧应用无效。
    *   **建议：** 对于主要使用 API（AJAX）进行交互的应用，可以作为 CSRF Token 的有力补充。对于传统多表单提交的应用，效果受限。

5.  SameSite Cookie 属性：
    *   **原理：** 在设置 Cookie 时（特别是会话 Cookie），添加 `SameSite` 属性来控制 Cookie 在跨站请求时是否被发送。
        *   **`SameSite=Strict`：** 最严格。浏览器只会在同站请求（即请求来源和目标网站同源）中发送此 Cookie。这意味着用户点击外部链接进入目标网站时，最开始访问的页面是没有会话 Cookie 的（相当于用户还没登录状态），需要重新登录，然后再执行操作（如从邮件中的链接进入银行）。**安全但用户体验差。**
        *   **`SameSite=Lax`：** **推荐的平衡方案（现代浏览器默认行为）。** 在跨站请求中，对于“安全的 HTTP 方法”（如 `GET`， `HEAD`），且该请求会导致浏览器在顶层导航栏切换页面（如 `<a>` 链接点击），浏览器会发送 Cookie。但对于非安全的请求（如 `POST`， `PUT`， `DELETE`）、或通过 `<img>`， `<script>`， `<form>`（没有导致顶层导航改变）发起的跨站请求，浏览器则不会发送 Cookie。这意味着攻击者伪造的 `POST` 转账请求不会携带用户的 `SameSite=Lax` 会话 Cookie，服务器认为用户未登录（或会话无效），攻击失败。攻击者伪造的 `GET` 请求（如转账链接，设置 Strict 或 Lax 后）虽然可能发送 Cookie，但要求是顶层导航跳转（用户会看到页面跳转到银行并转账成功页面，更容易察觉），并且安全方法通常不用来做数据修改操作（银行也不该用 GET 做转账）。
        *   **`SameSite=None; Secure`：** 最宽松。浏览器会在跨站请求中发送 Cookie，但必须标记为 `Secure`（要求 HTTPS）。只有特殊场景（如跨多个域名的登录系统，嵌入的第三方 iframe 内容需要认证）才需要。容易受到 CSRF 攻击。
    *   **优缺点：**
        *   **浏览器内置支持，服务端仅需在设置 Cookie 时添加属性（如 `Set-Cookie: SessionId=...; Secure; HttpOnly; SameSite=Lax; ...`）。** 实现成本最低。
        *   `SameSite=Lax` 能有效防御绝大多数基于 `<form>` POST 的 CSRF 攻击，且不影响用户正常浏览体验（点击链接登录状态有效）。
        *   `SameSite=Strict` 防御最强但体验差。
        *   存在一些兼容性问题（旧版本浏览器支持度有限），但现在主流浏览器支持良好。
        *   **不能作为唯一防御措施。** 浏览器兼容性问题、应用本身存在低危 CSRF 漏洞（如用 GET 做状态修改）、以及 `SameSite=None` 的设置不当都会削弱效果。
    *   **建议：** **强烈推荐为所有会话 Cookie 设置 `Secure; HttpOnly; SameSite=Lax`（或更严格的 Strict）。** 作为**基础防御层**，并与其他措施（如 CSRF Token）结合提供深度防御。这是现代框架的推荐做法。

### 实践防御建议总结

*   黄金标准：
    *   **核心防御：对所有敏感操作（非 `GET`）强制使用 CSRF Tokens + 验证（包括表单和 AJAX）。**
    *   **基础加固：为会话 Cookie 设置 `Secure`， `HttpOnly`， `SameSite=Lax`（或 Strict）。**
*   深度防御：
    *   启用合理的 CORS 策略，并考虑**对于 API 请求要求使用自定义安全头**（配合 CORS）。
    *   检查关键请求的 `Origin` 头（作为补充验证）。
    *   避免使用 `GET` 请求进行任何有状态或敏感的操作修改（应使用 `POST`， `PUT`， `DELETE`）。
    *   实施最小权限原则。
    *   使用现代安全框架，它们通常内置 CSRF 防护（如 Django 的 CSRF 中间件，Spring Security 的 CSRF 防护）。
*   用户体验与安全的平衡： `SameSite=Lax` 通常是会话 Cookie 的最佳设置。

通过综合运用这些措施，尤其是 Token + `SameSite=Lax`，可以非常有效地抵御绝大多数的 CSRF 攻击，保护用户账户和数据的安全。安全是一个多层次的过程，纵深防御是关键。
