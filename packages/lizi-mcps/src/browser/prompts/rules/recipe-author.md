### 为站点编写配方（recipe authoring）

当某个站点会被反复操作、值得沉淀为确定性流程时，使用以下流程为其编写一份**配方**（declarative recipe）。之后，通过 `action: "recipe"` 一步即可完成，几乎无需模型对 DOM 进行盲目探索。配方是数据而非代码；编写完成后，使用 `action: "saveRecipe"` 将其保存到本地层（L2），内置的同名配方将被您的版本覆盖，删除本地文件即可恢复内置配方。

#### 作者流程（recon → 发现 → 抽取 → 验证 → 沉淀）

1. **侦察**：首先使用 `action: "siteguide"` 和 `site` 检查是否存在现成的指南或配方；如果没有，则 `navigate` 到目标页面，并使用**限定范围的 `snapshot`**（带 `selector` 收窄范围）来了解结构，同时使用 `action: "requests"`（带**具体**的 `filter`）查看页面发送了哪些 XHR/fetch 请求。
2. **优先发现接口**：许多列表/详情页面背后都是 JSON API。找到接口 URL 后，将可变部分（如关键词、页码、subreddit 等）抽象为 `inputs` 变量，并在 URL 中使用 `{{var}}` 占位。**读取 GET JSON 接口最可靠的方式是：`navigate` 到该 URL，然后使用 `extract` 并设置 `{ fields: { body: "body" } }` 来读取 JSON**（不要使用 responseBody 来读取历史响应，因为它会等待下一个匹配的响应）。
3. **鉴权**：判断是否需要登录。如果需要，在 siteguide 的 `auth` 字段中明确说明，并在配方中将与登录状态相关的步骤设置为 `optional: true` 以进行探测；实际登录由用户在持久浏览器中完成一次。
4. **抽取**：使用 `extract` 提取 DOM 字段——**先通过限定范围的 `snapshot` 确认选择器在页面上真实存在，然后再编写**（不要凭空猜测选择器，也不要将属性拼接到选择器中）。`fields` 中的字符串是纯 CSS 选择器（获取 textContent），获取属性使用 `{selector, attr}`，获取链接使用 `{selector, type:"href"}`。列表使用 `from` 和 `multiple`。
5. **交互步骤**：`click` / `type` / `select` 直接使用**稳定的 CSS `selector`**（运行时由 act 解析，不硬编码 snapshot ref，跨会话不会失效）。`type` 配合 `submit: true` 提交表单。
6. **验证**：编写完成后**立即使用 `action: "recipe"` 运行一遍**，确认 `output` 非空且字段正确；如有问题，返回第 4 步修正选择器。
7. **沉淀**：使用 `action: "saveRecipe"`，传入 `site`（站点 host）、`recipeDraft`（配方对象）以及可选的 `siteGuideDraft`。草稿会先按 schema 校验，不合法会返回错误信息供您修正。

#### 取数策略（先判断站点类型，再选择步骤）

绝大多数列表/详情背后都是 JSON 接口，根据是否需要登录态分为两类：

- **公开接口（public）**：`navigate` 到接口 URL，然后使用 `extract { fields: { body: "body" } }` 读取完整的 JSON/XML。适用于无需 cookie 的公开 GET 接口（如 hn.algolia.com、api.stackexchange.com、dev.to/api、export.arxiv.org、/w/api.php）。简单，无需构造同源请求。
- **带 cookie / 反爬（cookie）**：某些站点（如 Reddit、多数社交站）对**直接 navigate 到 .json** 会返回 SPA HTML 以阻止爬取，或者结果依赖登录状态。正确做法是**先 `navigate` 到站点主域，然后使用 `evaluate` 步骤在页面内进行 `fetch`**：
  ```json
  { "action": "navigate", "url": "https://www.reddit.com" },
  { "action": "wait", "loadState": "load" },
  { "action": "evaluate", "as": "posts",
    "fn": "async () => { const r = await fetch('/r/{{subreddit|js}}/hot.json?limit=25&raw_json=1', { credentials: 'include' }); const d = await r.json(); return (d.data?.children||[]).map(c => ({ title: c.data.title, score: c.data.score, url: 'https://www.reddit.com'+c.data.permalink })); }" }
  ```
  `evaluate` 的 `fn` 是一段**函数表达式源码**（可以是 `async`，会被 await），在页面上下文中运行：由于已在目标域上，`fetch(相对路径, { credentials: 'include' })` 会自动携带该站的 cookie（登录状态），并且是同源请求，不会触发反爬机制。fn 的返回值通过 `as` 存储，`output` 引用它（runner 会自动解包，见下文）。**登录本身由用户在持久浏览器中完成一次**，配方只负责携带 cookie 取数。

#### recipe schema（编写 `recipeDraft` 时参考此结构）

```json
{
  "id": "唯一 id（覆盖内置配方时使用相同 id）",
  "match": ["host 子串（可选，用于提示）"],
  "description": "一句话说明",
  "inputs": { "query": { "required": true } },
  "steps": [
    { "action": "navigate", "url": "https://site/api?q={{query|url}}" },
    { "action": "wait", "loadState": "load" },
    { "action": "type", "selector": "input[name=q]", "value": "{{query}}", "submit": true },
    { "action": "click", "selector": "button.go" },
    { "action": "extract", "as": "items",
      "extract": { "from": ".row", "multiple": true,
        "fields": { "title": ".t", "url": { "selector": "a", "type": "href" } } } }
  ],
  "output": "{{items}}"
}
```

- 步骤的 `action` 可选值：`navigate` / `click` / `type` / `select` / `wait` / `extract` / `evaluate` / `requests` / `responseBody`。
- `evaluate` 运行一段页面内 JavaScript（函数表达式源码，可为 async），`{{var}}` 会插入到源码中——用于上述 cookie/反爬站点的同源 fetch。
- **所有被 `{{var}}` 引用的输入都必须声明 `required: true`**：引擎不提供任何默认值，引用未传入的变量会抛出 `missing recipe variable` 错误。
- **`evaluate`/`extract` 步骤的页面内返回值会被 runner 自动解包**：`as` 存储的变量以及 `output: "{{var}}"` 直接就是函数返回的数组/对象（runner 已移除 `{ result: ... }` 外壳），消费方可直接使用，无需再读取 `.result`。
- **插值修饰符**：`{{q|url}}` 对值进行 `encodeURIComponent` 编码（用于 URL 查询参数），`{{q|js}}` 将值转义为安全的 JavaScript 字符串字面量形式（用于 `evaluate` 的 `fn` 源码中，并用引号包裹），避免包含空格、引号或 `&` 的值破坏 URL 或源码。
- `as` 将步骤结果存储为变量；`{{var}}` 可在后续步骤和 `output` 中插值；`output` 写 `{{单个变量}}` 会原样返回该变量（结构化数据）。
- 任一非 `optional` 步骤失败即停止，并返回 `failedStep`——此时可回退到手动 `snapshot` + `extract`。

#### 取舍

- **分页**：第一版将页码/cursor 设为 `inputs`，一次配方运行一页，翻页时多次调用即可。
- 配方是**加速器而非唯一路径**：如果站点改版导致某一步失败，请停下来使用 snapshot+ref 手动完成，并考虑使用 `saveRecipe` 更新配方。