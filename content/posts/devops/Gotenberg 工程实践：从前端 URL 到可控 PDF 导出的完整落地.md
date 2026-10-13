---
title: "Gotenberg 工程实践：从前端 URL 到可控 PDF 导出的完整落地"
subtitle: ""
date: 2026-09-19T12:06:37+08:00
lastmod: 2026-09-19T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["DevOps"]
tags: ["PDF","Chromium","Gotenberg"]
---

> Gotenberg、Chromium、URL 转 PDF、Vue SPA、ECharts、A4 分页、`waitForExpression`

在很多后台系统中，“导出 PDF”看起来只是一个按钮，真正做起来却很容易从一个简单功能变成一条完整的渲染链路。

尤其当待导出的内容本身就是一个复杂的前端页面：页面需要异步请求数据、动态生成 DOM、渲染 ECharts 图表，还需要按照 A4 纸张分页，同时又要支持几十到几百份页面的批量导出时，传统的前端导出方案就开始暴露局限。

这篇文章记录一次 Gotenberg 的工程实践：从需求和技术选型开始，到 URL 转 PDF、登录态传递、前端打印模式、动态分页、`waitForExpression`，再到 POC、参数调优、批量导出和踩坑经验。

文章不讨论某个具体业务系统，而是把一次真实落地过程中值得复用的思路抽出来。

---

## 先跑一个 POC

如果第一次接触 Gotenberg，最推荐的方式不是先研究后端 SDK，也不是先在业务代码里封装一堆 Service，而是**先直接用 curl 把“URL → PDF”跑通**。

因为这类需求最难验证的不是 HTTP 接口，而是最终 PDF 的实际效果：页面能不能打开、登录态有没有生效、异步数据有没有加载完成、ECharts 有没有显示、A4 分页是否正常、背景色是否保留。

所以可以先准备一个能够被 Gotenberg 访问的前端页面，然后直接执行下面这条 POC 命令：

```bash
curl \
  --request POST \
  http://127.0.0.1:4000/forms/chromium/convert/url \
  --form 'url=http://<page-host>/path/to/page?id=<id>&pdf=1' \
  --form 'extraHttpHeaders={"x-token":"<TOKEN>"}' \
  --form 'preferCssPageSize=true' \
  --form 'omitHeaderFooter=true' \
  --form 'marginTop=0' \
  --form 'marginBottom=0' \
  --form 'marginLeft=0' \
  --form 'marginRight=0' \
  --form 'printBackground=true' \
  --form 'scale=1' \
  --form 'waitDelay=1s' \
  --form 'emulatedMediaType=print' \
  --form 'waitForExpression=window.__PDF_READY__ === true' \
  -o output-a4.pdf
```

先不要急着记住每一个参数。**第一遍的目标只有一个：确认这条链路能够稳定地产生 PDF。**

然后按照下面的顺序逐步观察：

```text
能不能生成 PDF？
      ↓
打开的是目标页面还是登录页？
      ↓
异步数据是否完整？
      ↓
ECharts / Canvas 是否正常？
      ↓
A4 纸张尺寸是否正确？
      ↓
模块有没有被拦腰切断？
      ↓
页面是否存在大片无意义空白？
      ↓
最终再把参数固化到后端代码
```

后面的章节都可以看成是在**解释、调整和验证这条 POC**：

- `extraHttpHeaders` 解决“页面需要登录怎么办”；
- `pdf=1` 解决“屏幕布局和打印布局不同怎么办”；
- `preferCssPageSize=true` 解决“谁来定义 A4 纸张尺寸”；
- `emulatedMediaType=print` 解决“如何触发打印 CSS”；
- `waitForExpression` 解决“什么时候算页面真的准备好了”；
- CSS + JS 解决“模块如何分页、如何避免大片空白”；
- `printBackground=true` 解决“为什么 PDF 里的背景色消失”；
- `scale=1` 解决“为什么页面被缩小后出现白边”；
- 批量导出和并发控制则是在 POC 跑通以后，再把单次转换工程化。


---

## 一、先明确问题：为什么需要后端导出

需求很简单：**把用户在浏览器中看到的一个前端页面，稳定地导出成 PDF。**

最容易想到的是让前端自己完成导出。例如使用 `html2canvas`、`dom-to-image` 等把 DOM 转成图片，再交给 `jsPDF` 生成 PDF。

对于“用户打开一份页面，点击一次下载”的场景，这种方案完全可以工作。但当需求变成批量导出几十甚至几百个页面时，问题就出现了。

| 前端导出方式的局限 | 具体问题 |
| --- | --- |
| 人工操作 | 大量页面需要逐个打开、等待和下载，难以无人值守 |
| 本地资源消耗 | 截图、Canvas 渲染和 PDF 拼接都会消耗浏览器内存与 CPU |
| 位图化 | 页面截图后文字本质上是图片，放大后清晰度和可编辑性较差 |
| 批量能力弱 | 失败重试、并发控制、任务进度、统一打包等能力不适合放在浏览器端 |
| 结果不一致 | 用户浏览器的窗口大小、缩放比例、环境差异都可能影响最终 PDF |

因此，需求最终从：

```text
“前端把当前页面下载成 PDF”
```

变成了：

```text
“后端能够访问一个 URL，把完整页面渲染成 PDF，并支持批量执行。”
```

这也是选择 Gotenberg 的起点。

---

## 二、为什么选择 Gotenberg，而不是继续使用 alpine-chrome

在选择 Gotenberg 之前，曾经使用过 `zenika/alpine-chrome` 配合浏览器自动化完成页面导出。

当时的思路比较直接：

```text
后端
  ↓
Chromium
  ↓
打开前端页面
  ↓
等待页面渲染
  ↓
截图整页
  ↓
把长图放进 PDF
```

这种方式能够解决“后端生成 PDF”的问题，但它实际上更接近“网页截图”，而不是“网页打印”。

主要问题有几个。

### 1. 截图方案天然更像图片

整页截图之后再生成 PDF，文字、图表和页面元素都被转换成了位图。

这会带来：

- 放大后文字不够清晰；
- 无法像原生 PDF 一样自然地选择文字；
- 页面越长，图片越大；
- A4 多页分页需要后端自己处理。

### 2. A4 分页需要自己实现

截图得到的是一张长图，而真正的打印需求通常是：

```text
页面
 ↓
A4
 ↓
第 1 页
第 2 页
第 3 页
...
```

如果继续沿用截图方案，就需要自己解决图片切割、页高计算、比例换算等问题。

### 3. 固定等待不可靠

早期方案中比较容易出现：

```text
Sleep(8s)
```

页面快的时候，8 秒是在浪费时间；页面慢的时候，8 秒又可能不够。

### 4. 项目维护状态也是技术选型的重要因素

`zenika/alpine-chrome` 已经不再作为活跃项目持续维护。对于依赖 Chromium 的生产链路而言，底层浏览器版本、安全更新和兼容性都属于长期风险。

因此，最终放弃继续围绕 alpine-chrome 自己拼装整套导出链路。

---

## 三、Gotenberg 真正解决的是什么

Gotenberg 的核心能力可以简单理解为：

> **给它一个 URL，它使用 Chromium 打开页面，再通过浏览器原生打印能力生成 PDF。**

最核心的调用形式就是：

```text
URL
 ↓
Chromium
 ↓
printToPDF
 ↓
PDF
```

这和截图方案的思路完全不同。

截图方案是：

```text
网页 → 图片 → PDF
```

Gotenberg 更接近：

```text
网页 → 浏览器打印 → PDF
```

因此，页面本身可以继续使用 Vue、React 等前端框架，也可以包含普通 DOM、CSS、Canvas、ECharts 等内容。

除了 URL 转 PDF，Gotenberg 还提供 HTML、Markdown、Office 文档转换，以及 PDF 合并、拆分、旋转、水印等能力。本文主要关注其中最适合前端页面导出的 Chromium URL → PDF 能力。

Gotenberg 的优势并不只是“能生成 PDF”，更重要的是它把 Chromium 的打印能力封装成了一个 HTTP 服务。后端不需要自己维护浏览器自动化逻辑，只需要调用 HTTP API。

---

## 四、真正的难点不是“生成 PDF”，而是“稳定地打印一个前端页面”

选择 Gotenberg 以后，很快会发现一个事实：

> **Gotenberg 负责打印，但它不会替你把前端页面设计成一个适合打印的页面。**

实际页面通常不是一张静态 HTML：

- 数据来自异步 API；
- Vue 根据数据动态生成 DOM；
- 页面包含 ECharts 等 Canvas 图表；
- 某些模块通过条件渲染决定是否出现；
- 页面高度会随着数据量变化；
- 浏览器屏幕布局和 A4 打印布局并不完全相同。

因此，真正需要解决的是两个问题：

### 问题一：什么时候可以开始打印？

如果页面还没加载完数据就开始转换，可能得到：

```text
图表空白
模块高度为 0
部分数据缺失
DOM 尚未生成
```

### 问题二：在哪里分页？

A4 的高度是固定的，而页面内容高度是动态的。

如果直接让浏览器随意分页，可能出现：

```text
一个卡片
──────────────
切页线
──────────────
卡片剩余内容
```

对于报告、数据看板、统计页面来说，这种结果通常不可接受。

所以这次实践最终形成了一个非常明确的分工：

```text
Gotenberg / Chromium
    负责“把网页打印出来”

前端 CSS
    负责“页面应该如何打印”

前端 JS
    负责“什么时候准备好，以及动态分页”
```

---

## 五、先用 curl 做 POC，再开始后端编码

这次实践中，一个比较重要的研发方法是：

> **先验证 Gotenberg 的能力边界，再写后端代码。**

不要一开始就在 Go 服务里封装一堆参数，然后通过完整业务链路反复调试 PDF。

因为 PDF 是一个非常典型的“效果导向”问题：参数是否正确，最终还是要打开 PDF 看排版。

因此更高效的方式是直接使用 curl：

```text
Gotenberg
   ↓
curl
   ↓
生成 PDF
   ↓
打开 PDF 检查效果
   ↓
修改参数 / 修改前端 CSS
   ↓
再次生成
```

整个过程可以拆成几个阶段。

### 第一阶段：先确认“能生成”

只传 URL，让 Gotenberg 能够打开页面并生成 PDF。

这一步不追求排版漂亮，只确认基础链路可用。

### 第二阶段：解决登录和数据问题

如果页面需要登录，则加入 HTTP Header 或 Cookie，让 Chromium 获得合法的登录态。

### 第三阶段：解决动态渲染问题

引入打印模式和 `waitForExpression`，让前端明确告诉 Gotenberg：页面已经准备完成。

### 第四阶段：解决分页问题

调整 `@page`、`@media print`、`break-inside` 等 CSS，同时配合 JS 动态计算页面高度。

### 第五阶段：固化参数

最终把已经通过 POC 验证的 Form 参数放进后端代码。

这样做的好处是：

> **POC 解决“能不能稳定打印”，后端代码解决“如何把它工程化”。**

---

## 六、需要登录的 URL 怎么处理

实际项目中的前端页面通常不是公开页面，因此 Gotenberg 打开 URL 时也需要携带登录态。

比较常见的方案有两类。

### 方案一：Authorization / Token

如果系统使用 JWT、Bearer Token 或自定义 Header，可以使用 `extraHttpHeaders`：

```bash
--form 'extraHttpHeaders={"x-token":"<TOKEN>"}'
```

或者：

```bash
--form 'extraHttpHeaders={"Authorization":"Bearer <TOKEN>"}'
```

Gotenberg 会将这些 Header 交给 Chromium，使页面导航以及页面后续的 HTTP 请求能够携带相应 Header。

这种方式特别适合前后端分离、API 使用 Token 鉴权的 SPA。

需要注意 Token 的有效期。批量导出几百个页面时，整个任务可能持续较长时间，如果 Token 很快过期，就可能出现前面正常、后面大量 401 的情况。

### 方案二：Cookie / Session

如果系统使用传统 Session，也可以把 Cookie 传给 Chromium：

```bash
--form 'cookies=@cookies.json;type=application/json'
```

例如：

```json
[
  {
    "name": "session_id",
    "value": "<SESSION_ID>",
    "domain": "example.com",
    "path": "/",
    "secure": true,
    "httpOnly": true,
    "sameSite": "Lax"
  }
]
```

两种方案的选择很简单：

| 方案 | 更适合 |
| --- | --- |
| `extraHttpHeaders` | JWT、Bearer Token、自定义请求头鉴权 |
| `cookies` | Cookie + Session 鉴权 |

核心思想都是一样的：

> **把登录态转换成 HTTP 层可以携带的 Header 或 Cookie，再交给 Chromium。**

另外需要注意 SPA 的前端路由守卫。如果页面发现浏览器本地没有 Token 就强制跳转登录页，那么即使 Gotenberg 已经通过 Header 携带了合法 Token，页面仍然可能被前端自己重定向。

这也是后面为什么需要 `pdf=1` 的原因之一。

---

## 七、为什么额外增加 `pdf=1`：给页面增加一个“打印模式”

同一个前端页面通常存在两种完全不同的使用场景：

```text
屏幕浏览
    ↓
响应式缩放、交互按钮、导航栏

打印 PDF
    ↓
固定画布、打印 CSS、分页、图表重绘
```

屏幕布局和打印布局并不是一回事。

例如屏幕模式可能根据浏览器窗口宽度执行：

```text
transform: scale(...)
```

把一个固定宽度的页面缩放到当前窗口中。

但打印模式如果继续使用这种缩放，DOM 的实际布局尺寸就可能和纸张尺寸不一致，从而影响分页计算。

因此增加：

```text
?pdf=1
```

作为一个非常明确的打印模式开关。

前端可以根据：

```js
const isPrintMode = route.query.pdf === '1'
```

决定是否：

- 关闭屏幕态缩放；
- 执行专门的打印准备流程；
- 隐藏下载按钮等交互元素；
- 放行打印模式下的鉴权逻辑；
- 进行 ECharts 重绘；
- 执行动态分页；
- 最终设置 `window.__PDF_READY__ = true`。

因此 `pdf=1` 并不是 Gotenberg 的特殊参数，而是**前端和打印链路之间约定的一个模式开关**。

它表达的是：

> “这个页面不是给人交互使用的，而是正在被打印引擎渲染。”

---

## 八、A4 页面尺寸：为什么最终选择 `preferCssPageSize=true`

一开始为了支持 A4，最直接的做法是把 A4 尺寸直接写进 Gotenberg：

```bash
--form 'paperWidth=8.27' \
--form 'paperHeight=11.69'
```

这当然可以工作。

但问题在于：页面的排版实际上已经由前端 CSS 决定。如果后端再用英寸定义一次纸张尺寸，就出现了两套尺寸体系：

```text
后端
paperWidth / paperHeight
        ↓
物理纸张尺寸

前端
CSS / DOM / @page
        ↓
页面布局尺寸
```

这两套体系之间需要进行转换，复杂页面中很容易出现细微的比例、边距和分页差异。

因此最终选择：

```bash
--form 'preferCssPageSize=true'
```

Gotenberg 的这个参数会优先使用 CSS `@page` 定义的页面尺寸，而不是 Form 参数中的 `paperWidth` / `paperHeight`。

这样页面和纸张使用同一套布局定义。

### 一个重要的 `@page` 示例

```css
/* 无头打印纸张：与 1920px 宽 DOM 按 A4 比例对齐 */
@page {
  size: 1920px 2715.43px;
  margin: 0;
}

@media print {
  html,
  body {
    margin: 0 !important;
    padding: 0 !important;
    background: #f2f2f2 !important;
  }
}
```

其中：

```text
1920 × 297 / 210 ≈ 2715.43
```

也就是说，用一个固定宽度的 DOM 画布，再按照 A4 的 `210:297` 比例计算页面高度。

这样就可以建立一个比较稳定的关系：

```text
一个 PDF 页面
        ↕
一个固定高度的 DOM 画布
```

前端的分页计算就有了明确基准。

### 为什么不再同时设置两套纸张尺寸？

实践中的经验是：

> **纸张尺寸最好只让一个地方负责。**

如果决定让 CSS 定义打印纸张，那么 Gotenberg 使用：

```bash
preferCssPageSize=true
```

并让 `@page` 成为页面尺寸的唯一来源。

---

## 九、Gotenberg 关键 Form 参数怎么配

POC 最终形成了一套比较稳定的参数组合：

```bash
curl \
  --request POST \
  http://127.0.0.1:4000/forms/chromium/convert/url \
  --form 'url=http://<page-host>/path/to/page?id=<id>&pdf=1' \
  --form 'extraHttpHeaders={"x-token":"<TOKEN>"}' \
  --form 'preferCssPageSize=true' \
  --form 'omitHeaderFooter=true' \
  --form 'marginTop=0' \
  --form 'marginBottom=0' \
  --form 'marginLeft=0' \
  --form 'marginRight=0' \
  --form 'printBackground=true' \
  --form 'scale=1' \
  --form 'waitDelay=1s' \
  --form 'emulatedMediaType=print' \
  --form 'waitForExpression=window.__PDF_READY__ === true' \
  -o output.pdf
```

几个关键参数可以这样理解：

| 参数 | 作用 |
| --- | --- |
| `url` | 要打印的前端 URL，通常增加 `pdf=1` |
| `extraHttpHeaders` | 向 Chromium 页面请求注入鉴权 Header |
| `preferCssPageSize=true` | 优先采用 CSS `@page` 的纸张尺寸 |
| `omitHeaderFooter=true` | 去掉浏览器自动生成的页眉、页脚 |
| `margin*` | 控制打印页边距，本方案由 CSS `@page` 统一管理 |
| `printBackground=true` | 打印背景色、卡片背景、图表背景等 |
| `scale=1` | 保持原始打印比例，不额外缩放 |
| `emulatedMediaType=print` | 使用 print 媒体环境，触发 `@media print` |
| `waitForExpression` | 等待页面主动报告“已经准备完成” |
| `waitDelay` | 固定等待时间，本方案只作为兜底 |

其中真正影响排版的参数，建议不要孤立地调整，而要和前端 CSS 一起理解。

---

## 十、`waitForExpression`：从“盲等”变成“页面主动报告完成”

这是整个实践中最值得保留下来的设计之一。

最开始容易想到的是：

```bash
--form 'waitDelay=5s'
```

逻辑很简单：

```text
打开页面
 ↓
等 5 秒
 ↓
生成 PDF
```

问题也很明显。

页面如果 2 秒就完成：

```text
浪费 3 秒
```

页面如果 7 秒才完成：

```text
5 秒后就开始打印
 ↓
拿到半成品
```

所以更合理的思路是让页面自己告诉 Gotenberg：

```text
“我准备好了。”
```

Gotenberg 提供了 `waitForExpression`：

```bash
--form 'waitForExpression=window.__PDF_READY__ === true'
```

前端则维护：

```js
window.__PDF_READY__ = false
```

在所有打印准备工作完成以后：

```js
window.__PDF_READY__ = true
```

一个简化后的准备流程可以是：

```js
window.__PDF_READY__ = false

await dataReady
await document.fonts.ready
await nextTick()

// 等待浏览器完成一轮布局
await new Promise(resolve => {
  requestAnimationFrame(() => {
    requestAnimationFrame(resolve)
  })
})

// ECharts 等图表重新计算尺寸
resizeCharts()

// 动态分页
paginateForPdf()

// 所有打印准备工作结束
window.__PDF_READY__ = true
```

这里有一个很重要的细节：

```text
window.__PDF_READY__ = true
```

是**赋值**；

而 Gotenberg 中：

```text
window.__PDF_READY__ === true
```

是**判断**。

两者职责不同。

`waitForExpression` 的价值在于，它把等待条件从：

```text
“我猜 5 秒应该够了”
```

变成：

```text
“页面完成所有准备工作后，再通知打印引擎。”
```

对于包含异步 API、ECharts、动态 DOM 的 SPA，这种方式明显更可靠。

---

## 十一、前端动态分页：为什么 JS 也必须参与

仅仅设置 `@page` 并不能保证复杂页面得到理想分页。

因为页面高度是动态的。

例如一个模块可能因为数据量不同，从：

```text
400px
```

变成：

```text
720px
```

如果提前写死分页位置，很容易产生空白或者切断。

因此实践中采用了“测量真实高度 + 按模块分页 + 动态补偿模块间距”的方式。这里的 JS 参与分页，其实有**两个目的**，而不只是防止模块被切断。

### 1. 第一目的：避免模块被拦腰切断

核心思路是：

```text
页面宽度固定
        ↓
计算一页对应的 DOM 高度
        ↓
逐个测量模块真实高度
        ↓
判断当前模块能否完整放入当前页
        ↓
放不下 → 整体移动到下一页
        ↓
放得下 → 继续放置
```

例如：

```text
当前页剩余 500px

下一个模块高度 700px

        ↓

放不下

        ↓

整个模块移动到下一页
```

而不是：

```text
模块 700px
────────────
前 500px 留在本页
──────────── ← 切页线
后 200px 到下一页
```

这就是动态分页最重要的原则：

> **分页单位应该尽量是“业务模块”，而不是任意一个 DOM 高度。**

### 2. 第二目的：解决“模块都挤在页面顶部、底部却大片空白”的视觉问题

这点非常容易被忽略。

假设一个 A4 页面中有多个独立的 `div` 模块：

```text
┌──────────────────────┐
│ 模块 A                │
│                      │
│ 模块 B                │
│                      │
│ 模块 C                │
│                      │
│ 模块 D                │
│                      │
│                      │
│                      │
│        大片空白       │
│                      │
└──────────────────────┘
```

如果完全依赖普通文档流，浏览器只会按照 DOM 顺序把模块从上往下排列。**它并不知道这些模块在视觉上应该均匀地分布在这一页中。**

结果就是：模块全部集中在页面上方，下面留下很大的空白。技术上 PDF 是正确的，但视觉上会显得“挤在一起”，缺少报告页面应有的段落感。

因此，这里的 JS 分页还有一个很实际的职责：**计算当前页剩余的可用高度，然后把一部分剩余空间动态分配到模块之间的间距中。**

例如原本：

```text
模块 A
  ↓ 35px
模块 B
  ↓ 35px
模块 C
  ↓ 35px
模块 D

剩余 600px
```

可以把这部分剩余空间均匀补偿到模块之间：

```text
模块 A
  ↓ 180px
模块 B
  ↓ 180px
模块 C
  ↓ 180px
模块 D

页面刚好填满
```

这里的 180px 只是示意，实际值应该根据当前页面剩余空间、模块数量、最小间距等条件动态计算。核心原则是：**只增加模块之间的间距，不压缩模块自身高度。**

这样做有三个直接收益：

1. **页面更均匀**：多个模块不会全部堆在页面顶部；
2. **视觉上更有段落感**：模块之间有自然呼吸空间，更接近人工排版后的报告；
3. **切页线更干净**：把剩余空间放在模块之间，切页线更容易落在空白区域，而不是内容中间。

因此，这里的 JS 并不是简单地“代替 CSS 分页”，而是在做浏览器普通流式布局做不到的**版面均衡**。

可以把整个过程理解成：

```text
CSS
  ↓
告诉浏览器“哪些地方不能拆、哪里可以换页”

JS
  ↓
测量真实高度
  ↓
决定模块属于哪一页
  ↓
计算当前页剩余空间
  ↓
把剩余空间均匀分配到模块间距

Chromium
  ↓
按照最终布局执行原生打印
```

### 为什么需要实时测量

不要简单地把每个模块的理论高度相加。

更可靠的方式是使用：

```js
getBoundingClientRect()
```

读取浏览器实际布局结果。

因为真实高度还会受到：

- 字体加载；
- ECharts 尺寸；
- margin；
- padding；
- 条件渲染；
- 浏览器布局计算；
- 内容换行。

等因素影响。

因此动态分页本质上是在做一件事：

> **让 JavaScript 先把页面“排好”，再让 Chromium 把排好的页面“打印出来”。**

---

## 十二、CSS 如何告诉浏览器应该在哪里分页

JS 负责精确分页，但 CSS 本身也提供了一套打印分页规则。

最常见的是：

```css
break-inside
break-before
break-after
```

以及兼容性更广的旧属性：

```css
page-break-inside
page-break-before
page-break-after
```

### `break-inside`

控制元素内部是否允许分页。

例如：

```css
.module {
  break-inside: avoid;
  page-break-inside: avoid;
}
```

含义是：

> 尽量不要把这个模块从中间切开。

对于卡片、统计模块、图表块等非常有用。

### `break-before`

控制元素前面是否分页：

```css
.section {
  break-before: page;
  page-break-before: always;
}
```

可以让一个重要章节从新页开始。

### `break-after`

控制元素后面是否分页：

```css
.cover {
  break-after: page;
  page-break-after: always;
}
```

适合封面、章节分隔等场景。

### `page-break-*` 与 `break-*` 有什么区别？

可以把它们理解成新旧两代 API：

| 旧属性 | 新属性 | 含义 |
| --- | --- | --- |
| `page-break-inside` | `break-inside` | 控制元素内部是否允许分页 |
| `page-break-before` | `break-before` | 控制元素之前是否分页 |
| `page-break-after` | `break-after` | 控制元素之后是否分页 |

`page-break-*` 是早期专门面向打印分页的属性；`break-*` 属于更现代的 CSS Fragmentation 体系，语义更通用。

工程实践中，如果需要兼顾打印引擎兼容性，可以同时写：

```css
.module {
  break-inside: avoid;
  page-break-inside: avoid;
}
```

这样 CSS 负责表达“分页规则”，JS 负责处理“动态高度下的精确分页”。

---

## 十三、ECharts 为什么需要特殊处理

复杂前端页面中，ECharts 是 PDF 导出比较容易出问题的一类元素。

因为 ECharts 通常通过 Canvas 绘制，而 Canvas 的最终图像依赖容器尺寸。

页面在浏览器中正常显示，并不意味着 Chromium 开始打印时 Canvas 一定已经处于正确尺寸。

因此打印准备阶段需要主动让图表重新计算：

```js
const instance = echarts.getInstanceByDom(el)
if (instance) {
  instance.resize()
}
```

这一步解决的是：

```text
页面尺寸发生变化
        ↓
ECharts 仍使用旧尺寸
        ↓
打印时 Canvas 内容异常
```

重新 `resize()` 后：

```text
重新计算尺寸
        ↓
重新绘制 Canvas
        ↓
再设置 __PDF_READY__
        ↓
Gotenberg 开始打印
```

因此对于 ECharts、Chart.js、AntV 等 Canvas 图表，一个通用原则是：

> **图表重绘完成之后，再发送 PDF_READY 信号。**

---

## 十四、参数踩坑实录：从“能生成”到“排版可控”

真正做过 PDF 导出以后，会发现参数本身并不复杂，难的是理解每个参数到底影响哪一层。

### 1. `paperWidth` / `paperHeight`：一开始直接写死 A4

最开始使用：

```bash
--form 'paperWidth=8.27' \
--form 'paperHeight=11.69'
```

思路没有错：A4 就是 `210mm × 297mm`。

但实际页面已经有自己的 `@page` 和 DOM 尺寸，这时后端再定义一套纸张尺寸，容易出现布局和分页上的细微差异。

最终改为：

```bash
--form 'preferCssPageSize=true'
```

让 CSS `@page` 成为纸张尺寸的主要来源。

**经验：纸张尺寸最好只由一处负责。**

### 2. `scale=0.6`：白边是怎么来的

最初为了让页面“塞进 A4”，把：

```bash
--form 'scale=0.6'
```

作为尝试。

结果内容整体缩小，四周出现明显白边。

后来才明确：`scale` 是打印缩放因子，不是一个“页面塞不下时自动优化分页”的参数。

如果页面本身已经按照纸张尺寸设计，再把整体缩放到 `0.6`，相当于又缩了一遍。

最终恢复：

```bash
--form 'scale=1'
```

**经验：排版错乱时，优先检查页面尺寸、CSS 和分页，不要第一时间靠 `scale` 救场。**

### 3. `singlePage`：看起来完整，实际上不适合打印

还尝试过：

```bash
--form 'singlePage=true'
```

它的思路是把整个内容变成一张超长页面。

这样看起来不会被切开，但如果最终还要打印到 A4，就会产生整体缩小的问题。

它更适合“整页海报”之类的场景，而不是标准 A4 文档。

因此最终取消。

### 4. `waitDelay=5s`：60 个页面多浪费 5 分钟

这是一次非常典型的工程教训。

最初为了保险，把：

```bash
--form 'waitDelay=5s'
```

作为固定等待。

单份 PDF 感觉不明显，但批量导出 60 个页面以后：

```text
60 × 5s
= 300s
= 5 分钟
```

这 5 分钟全部来自固定等待。

而页面有的可能 1 秒就准备好了，有的可能需要更久。

最终改成：

```text
waitForExpression
        ↓
window.__PDF_READY__ === true
```

让页面自己报告完成状态。

`waitDelay` 则退回到兜底角色。

**经验：批量场景下，任何“每页固定等待”的参数都会被页面数量放大。**

---

## 十五、Gotenberg 的并发与批量导出：不要把单实例当成无限并发引擎

批量导出几百个页面时，另一个非常重要的问题是吞吐量。

这里需要纠正一个容易产生的理解：

> **Gotenberg 不是“一个请求处理完，再处理下一个”的绝对串行模型，也不是可以无限并发的模型。它内部的 Chromium 有并发上限和请求队列。**

Gotenberg 的 Chromium 模块会管理转换请求，超过可用并发能力的请求会进入队列；并发过高时，队列、超时和资源消耗都会成为瓶颈。

因此真正需要关注的是：

```text
单实例
  ↓
有限的 Chromium 并发能力
  ↓
请求进入队列
  ↓
CPU / 内存 / 页面复杂度成为吞吐瓶颈
```

对于几十到几百个页面，不建议简单地启动几百个 goroutine，然后全部同时请求同一个 Gotenberg。

更合理的做法是：

### 第一层：后端控制并发

例如使用 worker pool 或信号量，把同时进行的 PDF 转换数量控制在一个经过压测验证的范围。

### 第二层：观察实际资源

重点观察：

- CPU；
- 内存；
- 单页平均耗时；
- P95/P99 耗时；
- 失败率；
- 503；
- Chromium 重启情况；
- 队列等待时间。

### 第三层：需要更高吞吐时横向扩展

如果单个 Gotenberg 实例达到瓶颈，更合理的方式是：

```text
                ┌─ Gotenberg #1
后端 → LB ──────┼─ Gotenberg #2
                ├─ Gotenberg #3
                └─ Gotenberg #4
```

多个实例共同承担批量任务。

因此，真正的优化方向不是简单地“把一个实例的并发调到最大”，而是：

> **控制单实例并发 + 观察资源 + 必要时水平扩容。**

具体并发数需要结合 Gotenberg 版本、CPU、内存和页面复杂度压测确定，而不是直接套一个固定数字。

---

## 十六、生产部署与调试：普通日志为什么看不到页面 Network

生产环境可以使用相对简单的 Docker 启动方式：

```bash
docker run -d \
  --name gotenberg \
  --restart unless-stopped \
  --memory="4g" \
  --memory-swap="4g" \
  -p "4000:3000" \
  gotenberg/gotenberg:8
```

调试时，可以临时提高日志级别：

```bash
docker run -d \
  --name gotenberg-debug \
  --memory="4g" \
  --memory-swap="4g" \
  -p "4000:3000" \
  gotenberg/gotenberg:8 \
  gotenberg \
  --log-level=debug \
  --log-format=text \
  --chromium-auto-start \
  --chromium-clear-cache \
  --chromium-clear-cookies
```

然后：

```bash
docker logs -f gotenberg-debug
```

### 为什么 debug 日志仍然看不到完整的浏览器 Network？

这是一个很容易误解的地方。

Gotenberg 的 debug 日志主要是 Gotenberg 自己的运行日志，它不是 Chrome DevTools 的 Network 面板。

也就是说，你不能期待：

```text
Chrome DevTools
    ↓
Network
    ↓
GET xxx
POST xxx
404
401
500
```

完整地变成：

```text
docker logs gotenberg-debug
```

如果需要定位页面内部 API 是否失败，更有效的方法通常是：

1. 前端增加临时 API 请求日志；
2. 前端在打印模式下暴露关键状态；
3. 服务端查看对应 API 的访问日志；
4. 对关键 API 增加错误状态记录；
5. 必要时再使用 Chromium/CDP 级别的网络调试。

Gotenberg 的 debug 日志更适合回答：

```text
Gotenberg 有没有收到请求？
Chromium 是否正常？
转换是否失败？
转换耗时多久？
```

而不适合单独承担：

```text
页面内部所有 XHR/fetch 到底请求了什么？
```

另外，调试容器不要和生产容器同时绑定同一个宿主机端口。

---

## 十七、一次完整的前后端协作链路

经过前面的实践，可以把整个过程浓缩成下面这条链路：

```text
后端提交导出
      ↓
构造目标 URL + pdf=1
      ↓
附带 Token / Cookie
      ↓
请求 Gotenberg
      ↓
Chromium 打开前端页面
      ↓
Vue 加载异步数据
      ↓
DOM 动态生成
      ↓
ECharts 渲染
      ↓
字体 / 布局稳定
      ↓
JS 动态计算分页
      ↓
CSS print 规则生效
      ↓
window.__PDF_READY__ = true
      ↓
waitForExpression 命中
      ↓
Chromium printToPDF
      ↓
得到 PDF
```

这里最重要的认识是：

> **Gotenberg 并没有替代前端，而是把浏览器的打印能力搬到了后端。**

所以最终的 PDF 质量，本质上取决于：

```text
前端页面是否“可打印”
        ×
Gotenberg 参数是否正确
        ×
页面是否在正确的时机进入打印
```

---

## 十八、职责划分：到底谁负责什么

经过这次实践，可以把职责边界总结成下面这张表：

| 问题 | 谁负责 |
| --- | --- |
| A4 / 页面尺寸选择 | Gotenberg + CSS `@page` 协同 |
| 页边距 | Gotenberg / CSS `@page`，二者需要统一规划 |
| 是否打印背景 | Gotenberg |
| 是否使用 print 媒体 | Gotenberg |
| 页面宽度 | CSS |
| 模块内部布局 | HTML / CSS |
| 元素是否允许拆分 | CSS |
| 哪个模块从新页开始 | CSS / JS |
| 页面高度如何变化 | HTML / CSS / JS |
| 动态分页 | JS + Chromium |
| 最终页面分片 | Chromium |
| PDF 文件生成 | Gotenberg |
| 批量任务编排 | 后端 |
| 并发控制 | 后端 + Gotenberg 实例配置 |

这张表背后其实只有一句话：

> **Gotenberg 负责打印能力，前端负责可打印的页面，后端负责工程化编排。**

---

## 十九、这次实践最终沉淀下来的几个原则

### 原则一：后端导出，不代表前端不用管

后端只是把“用户按 Ctrl+P”这件事自动化了。

真正的打印排版仍然属于前端页面本身。

### 原则二：固定等待只能兜底，不能作为主要同步机制

对于动态页面：

```text
waitDelay
```

是简单但粗糙的方案。

更可靠的是：

```text
waitForExpression
        ↓
window.__PDF_READY__
```

### 原则三：纸张尺寸最好只有一个权威来源

如果页面已经通过 CSS `@page` 定义纸张，就使用：

```text
preferCssPageSize=true
```

不要让后端和前端同时维护两套尺寸。

### 原则四：不要用 `scale` 解决真正的排版问题

`scale` 是缩放，不是分页算法。

页面排版不正确时，优先检查：

```text
@page
@media print
DOM 宽度
margin
分页算法
```

### 原则五：复杂页面需要“CSS + JS”双保险

CSS 负责声明：

```text
哪里允许分页
哪里不允许分页
哪里必须换页
```

JS 负责处理：

```text
真实高度
动态模块
图表尺寸
分页位置
```

### 原则六：批量导出要从一开始考虑吞吐量

一个页面能导出来只是 POC。

真正上线还需要考虑：

```text
并发控制
超时
重试
资源限制
失败率
多实例扩容
```

### 原则七：先 POC，再工程化

最值得保留的研发习惯可能不是某个参数，而是：

```text
curl 验证
   ↓
调整 Gotenberg 参数
   ↓
调整前端 CSS / JS
   ↓
PDF 效果确认
   ↓
参数固化
   ↓
后端代码化
```

这样能够把最不确定的“PDF 排版问题”前置解决。

---

## 二十、结语：Gotenberg 的价值不是“生成 PDF”，而是把浏览器打印能力服务化

如果只是问：

> “Gotenberg 能不能把一个 URL 转成 PDF？”

答案其实非常简单。

真正有工程价值的问题是：

> **一个需要登录、依赖异步数据、包含 ECharts、页面高度动态变化的现代 SPA，能不能稳定地被批量打印成排版可控的 A4 PDF？**

这次实践给出的答案是可以，但需要明确职责：

```text
Gotenberg
    ↓
提供 Chromium 打印能力

前端 CSS
    ↓
定义可打印页面

前端 JS
    ↓
准备数据、重绘图表、动态分页、发送 READY

后端
    ↓
鉴权、任务编排、并发控制、重试、文件管理
```

最终，Gotenberg 并不是一个“把 HTML 变成 PDF 的黑盒子”。

更准确地说，它是一个**把浏览器原生打印能力服务化的基础设施组件**。

一旦接受这个定位，很多问题就会变得清晰：

- PDF 排版问题 → 回到 CSS / DOM；
- 动态渲染问题 → 回到页面生命周期；
- 等待问题 → 用显式 Ready 信号解决；
- 批量性能问题 → 用并发控制和水平扩展解决；
- 登录问题 → 把登录态转换成 Header / Cookie；
- 参数问题 → 先用 curl 做 POC，再固化到后端。

这也是这次工程实践中最大的收获：**不要试图让 Gotenberg 替你解决前端排版，而应该让它忠实地执行一套已经准备好的“可打印页面”。**

---

## 参考资料

- Gotenberg 官方文档：
  https://gotenberg.dev/docs/getting-started/introduction

