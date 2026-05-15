---
name: ani-rss
description: Query ANI-RSS subscriptions and add new subscriptions through ANI-RSS built-in Mikan proxy APIs.
version: 0.2.0
platforms:
  - linux
required_environment_variables:
  - name: ANI_RSS_BASE_URL
    prompt: ANI-RSS API base URL
    help: Example: https://ani-rss.example.com/api. Include /api if your ANI-RSS deployment uses it.
    required_for: calling ANI-RSS API
  - name: ANI_RSS_API_KEY
    prompt: ANI-RSS API Key
    help: Configure this in Hermes Dashboard. Do not paste it into chat.
    required_for: calling protected ANI-RSS API
metadata:
  hermes:
    tags:
      - ani-rss
      - rss
      - mikan
      - anime
      - homeserver
    category: media
    requires_toolsets:
      - terminal
---

# ANI-RSS Skill

This skill operates ANI-RSS through its HTTP API.

Use this skill only for:

1. Querying existing ANI-RSS subscriptions.
2. Searching Mikan through ANI-RSS built-in proxy APIs.
3. Creating a new disabled ANI-RSS subscription from a Mikan RSS source.

Do not use this skill for:

- deleting subscriptions
- enabling or disabling subscriptions
- refreshing subscriptions
- changing ANI-RSS global config
- changing downloader config
- changing proxy config
- deleting downloaded files

## Runtime Variables

The following environment variables must be configured in Hermes Dashboard:

- `ANI_RSS_BASE_URL`
- `ANI_RSS_API_KEY`

`ANI_RSS_BASE_URL` should be the API base URL.

Examples:

```env
ANI_RSS_BASE_URL=http://ani-rss.apps.svc.cluster.local:7789/api
ANI_RSS_BASE_URL=https://ani-rss.example.com/api
````

Never ask the user to paste API keys into chat.

Never write API keys to files.

Always pass the API key through the `x-api-key` header.

Base curl pattern:

```bash
curl -sS "$ANI_RSS_BASE_URL/<endpoint>" \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY"
```

If a request returns 404, check whether `ANI_RSS_BASE_URL` incorrectly includes or omits `/api`.

## Safety Rules

1. The Hermes pod does not need direct access to Mikan, DMHY, or other external RSS sites.
2. To search Mikan, always call ANI-RSS `/mikan` and `/mikanGroup`.
3. Do not ask the user for a Mikan RSS URL until ANI-RSS Mikan proxy search has failed.
4. Before adding a subscription, always query existing subscriptions with `/listAni`.
5. Before adding a subscription, always convert and preview the RSS with `/rssToAni` and `/previewAni`.
6. New subscriptions must be created with `enable=false`.
7. Never call delete APIs.
8. Never call config APIs.
9. Never enable or refresh a subscription in this skill.
10. If the Mikan search returns multiple plausible candidates, show candidates and ask the user to choose.
11. If multiple subtitle groups are available, prefer common high-quality groups only if the user has an obvious preference; otherwise show candidates.
12. If a matching subscription already exists, do not add a duplicate.
13. If preview looks wrong, stop and report the problem instead of adding the subscription.

## Query Existing Subscriptions

Use this when the user asks:

* 当前有哪些订阅
* 查一下 ANI-RSS
* 有没有已经订阅某部番
* list subscriptions
* show subscriptions

Command:

```bash
curl -sS "$ANI_RSS_BASE_URL/listAni" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY"
```

After querying, summarize the result compactly:

```md
当前 ANI-RSS 订阅：

- 标题：
  - ID:
  - 季：
  - 启用：
  - RSS：
  - 保存路径：
```

If the response is large, only show the most relevant entries and mention that more entries exist.

## Search Mikan Through ANI-RSS

Use this when the user asks:

* 帮我追某部番
* 搜一下某部番
* 添加自动下载
* 从 Mikan 找 RSS
* subscribe anime by title

Do not use external web search, browser, curl to Mikan directly, or direct access to DMHY.

Always search through ANI-RSS.

### Search by title

```bash
curl -sS "$ANI_RSS_BASE_URL/mikan?text=<URL_ENCODED_TITLE>" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY" \
  -d '{}'
```

Example:

```bash
curl -sS "$ANI_RSS_BASE_URL/mikan?text=%E6%B7%A1%E5%B3%B6%E7%99%BE%E6%99%AF" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY" \
  -d '{}'
```

Inspect the response:

* `data.totalItem`
* `data.weeks[].weekLabel`
* `data.weeks[].items[].title`
* `data.weeks[].items[].url`
* `data.weeks[].items[].exists`
* `data.weeks[].items[].score`

If there are multiple candidates, present them to the user.

Do not guess if multiple candidates look plausible.

### Search by Mikan bangumiId

If the user provides a Mikan bangumi ID, or if a previous result contains a clear bangumi ID, use:

```bash
curl -sS "$ANI_RSS_BASE_URL/mikan?text=bangumiId:%20<BANGUMI_ID>" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY" \
  -d '{}'
```

This should return exactly one candidate when the ID is valid.

## Get Mikan Subtitle Groups

After selecting a Mikan candidate, call `/mikanGroup` with the candidate `url`.

```bash
curl -sS "$ANI_RSS_BASE_URL/mikanGroup?url=<URL_ENCODED_MIKAN_BANGUMI_URL>" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY"
```

Inspect the response:

* `data[].label`
* `data[].rss`
* `data[].bgmUrl`
* `data[].updateDay`
* `data[].items[].title`
* `data[].items[].formatSize`
* `data[].items[].createdAt`

Pick a subtitle group only when the choice is obvious.

If multiple groups are plausible, show candidates:

```md
找到多个字幕组：

1. 字幕组：
   - RSS:
   - 最近更新：
   - 样例条目：
2. 字幕组：
   - RSS:
   - 最近更新：
   - 样例条目：

请选择要订阅的字幕组。
```

## Add Subscription from Mikan Search

Use this when the user asks to follow or auto-download an anime by title.

### Step 1: Query Existing Subscriptions

Always run:

```bash
curl -sS "$ANI_RSS_BASE_URL/listAni" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY"
```

Check for existing subscriptions with similar:

* title
* season
* RSS URL
* Mikan title

If a likely duplicate exists, stop and show the existing subscription.

### Step 2: Search Mikan

Search through ANI-RSS:

```bash
curl -sS "$ANI_RSS_BASE_URL/mikan?text=<URL_ENCODED_TITLE>" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY" \
  -d '{}'
```

If no result is found, say:

```md
ANI-RSS 的 Mikan 代理没有找到匹配番剧。此时才需要用户提供 RSS URL 或 Mikan bangumiId。
```

Do not claim that Hermes cannot search the web until this ANI-RSS proxy search has been tried.

### Step 3: Select Candidate

From `data.weeks[].items[]`, choose the candidate only if unambiguous.

Fields to inspect:

* `title`
* `url`
* `exists`
* `score`

If `exists=true`, warn that ANI-RSS may already have this Mikan bangumi subscribed.

### Step 4: Get Subtitle Groups

Call:

```bash
curl -sS "$ANI_RSS_BASE_URL/mikanGroup?url=<URL_ENCODED_MIKAN_BANGUMI_URL>" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY"
```

Select a group.

Use the selected group fields:

* `rss`
* `label`
* `bgmUrl`

### Step 5: Convert RSS to ANI-RSS Draft

Use the selected group data:

```bash
curl -sS "$ANI_RSS_BASE_URL/rssToAni" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY" \
  -d '{
    "url": "<GROUP_RSS_URL>",
    "type": "mikan",
    "bgmUrl": "<GROUP_BGM_URL>",
    "subgroup": "<GROUP_LABEL>"
  }'
```

The response should contain an ANI-RSS subscription draft.

### Step 6: Force Disabled State

Before previewing or adding, ensure the draft contains:

```json
{
  "enable": false
}
```

If the draft contains `enable:true`, change it to `enable:false`.

This skill must only create disabled subscriptions.

### Step 7: Preview Subscription

Use the draft JSON as the request body:

```bash
cat > /tmp/ani-rss-draft.json <<'EOF'
<PASTE_DRAFT_JSON_HERE>
EOF

curl -sS "$ANI_RSS_BASE_URL/previewAni" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY" \
  -d @/tmp/ani-rss-draft.json
```

Inspect the preview result.

Check:

* whether the title is correct
* whether the season is correct
* whether episode parsing looks sane
* whether download path looks sane
* whether too many items are skipped
* whether the RSS source appears to match the intended anime

If the preview looks wrong, stop and explain what looks wrong.

### Step 8: Add Subscription

Only after preview succeeds, add the disabled subscription:

```bash
curl -sS "$ANI_RSS_BASE_URL/addAni" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY" \
  -d @/tmp/ani-rss-draft.json
```

After adding, query `/listAni` again to verify that the subscription exists:

```bash
curl -sS "$ANI_RSS_BASE_URL/listAni" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY"
```

## Add Subscription from Direct RSS URL

Use this only when:

* the user directly provides an RSS URL
* ANI-RSS Mikan proxy search failed
* the source is not Mikan

Required input:

* RSS URL

Optional input:

* type: `mikan`, `ani-bt`, or `other`
* bgmUrl
* subgroup

If the RSS URL is a Mikan RSS URL, use:

```json
{
  "type": "mikan"
}
```

If the source is unknown, use:

```json
{
  "type": "other"
}
```

Then follow the same flow:

1. `/listAni`
2. `/rssToAni`
3. force `enable=false`
4. `/previewAni`
5. `/addAni`
6. `/listAni`

## Output Format for Mikan Search

When candidates are found:

```md
ANI-RSS 通过 Mikan 代理找到了这些候选：

1. 标题：
   - URL:
   - 评分：
   - 已存在：

2. 标题：
   - URL:
   - 评分：
   - 已存在：
```

## Output Format for Subtitle Groups

When groups are found:

```md
找到这些字幕组：

1. 字幕组：
   - RSS:
   - BGM:
   - 最近更新：
   - 样例条目：

2. 字幕组：
   - RSS:
   - BGM:
   - 最近更新：
   - 样例条目：
```

## Output Format for Add

After adding a subscription, respond like this:

```md
已添加 ANI-RSS 订阅，默认未启用。

- 标题：
- 季：
- Mikan 标题：
- 字幕组：
- RSS：
- BGM：
- 保存路径：
- 启用状态：false
- 解析到的条目：
- 被跳过的条目：
- 注意事项：

下一步如果需要开始自动下载，需要手动启用订阅。
```

Do not enable the subscription automatically.

Do not refresh the subscription automatically.

## Error Handling

### Mikan Proxy Failed

If `/mikan` fails:

```md
ANI-RSS 的 Mikan 代理搜索失败。Hermes 不应直接访问 Mikan；请检查 ANI-RSS 容器是否能访问 Mikan，以及 ANI-RSS 的 Mikan host / proxy 配置。
```

### No Mikan Result

If `/mikan` returns no candidates:

```md
ANI-RSS 的 Mikan 代理没有找到匹配番剧。可以提供 Mikan bangumiId 或 RSS URL 继续添加。
```

### No Subtitle Group

If `/mikanGroup` returns no groups:

```md
找到了 Mikan 番剧页，但没有拿到字幕组 RSS。请检查 ANI-RSS 是否能访问该 Mikan 页面，或手动提供 RSS URL。
```

### Missing API Key

If the API returns authentication error:

```md
ANI-RSS API 鉴权失败。请在 Hermes Dashboard 中检查 `ANI_RSS_API_KEY`，更新后重新加载 session。
```

### Connection Failed

If connection fails:

```md
无法连接 ANI-RSS。请检查 `ANI_RSS_BASE_URL` 是否正确，以及 Hermes 是否能访问该地址。
```

### 404

If all ANI-RSS API calls return 404:

```md
ANI-RSS API 返回 404。请检查 `ANI_RSS_BASE_URL` 是否需要包含 `/api`。例如应配置为 `https://ani-rss.example.com/api`，而不是 `https://ani-rss.example.com`。
```

### Duplicate Subscription

If a duplicate appears to exist:

```md
检测到疑似已有订阅，已停止添加，避免重复下载。
```

Then show the matching subscription.

### Bad Preview

If preview result looks wrong:

```md
RSS 可以访问，但 ANI-RSS 预览结果不符合预期，已停止添加。
```

Then explain which fields look wrong.
