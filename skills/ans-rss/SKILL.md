---
name: ani-rss
description: Query and add ANI-RSS subscriptions safely through curl.
version: 0.1.0
platforms:
  - linux
required_environment_variables:
  - name: ANI_RSS_BASE_URL
    prompt: ANI-RSS base URL
    help: Example: https://ani-rss.example.com
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
      - anime
      - homeserver
    category: media
---

# ANI-RSS Skill

This skill operates ANI-RSS through its HTTP API.

Use this skill only for:

1. Querying existing ANI-RSS subscriptions.
2. Creating a new ANI-RSS subscription from an RSS URL.

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

Never ask the user to paste API keys into chat.

Never write API keys to files.

Always pass the API key through the `x-api-key` header.

Base curl pattern:

```bash
curl -sS "$ANI_RSS_BASE_URL/<endpoint>" \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY"
````

## Safety Rules

1. Before adding a subscription, always query existing subscriptions with `/listAni`.
2. Before adding a subscription, always convert and preview the RSS with `/rssToAni` and `/previewAni`.
3. New subscriptions must be created with `enable=false`.
4. Never call delete APIs.
5. Never call config APIs.
6. Never enable or refresh a subscription in this skill.
7. If the RSS preview looks wrong, stop and report the problem instead of adding the subscription.
8. If a matching subscription already exists, do not add a duplicate.
9. If title, season, subtitle group, or RSS source is ambiguous, ask the user for clarification.

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

After querying, summarize the result in a compact format:

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

## Add Subscription from RSS URL

Use this when the user asks:

* 帮我添加这个 RSS
* 帮我追这部番
* 添加自动下载
* add subscription
* subscribe this RSS

Required input:

* RSS URL

Optional input:

* title
* season
* download path
* match rules
* exclude rules

If no RSS URL is provided, ask the user for the RSS URL.

### Step 1: Query Existing Subscriptions

Always run:

```bash
curl -sS "$ANI_RSS_BASE_URL/listAni" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY"
```

Check for an existing subscription with similar:

* title
* season
* RSS URL

If a likely duplicate exists, stop and show the existing subscription.

### Step 2: Convert RSS to ANI-RSS Draft

Run:

```bash
curl -sS "$ANI_RSS_BASE_URL/rssToAni" \
  -X POST \
  -H "content-type: application/json" \
  -H "x-api-key: $ANI_RSS_API_KEY" \
  -d '{"url":"<RSS_URL>"}'
```

Save the returned JSON mentally as the subscription draft.

Do not expose raw API key or secrets.

### Step 3: Force Disabled State

Before previewing or adding, ensure the draft contains:

```json
{
  "enable": false
}
```

If the draft contains `enable:true`, change it to `enable:false`.

This skill must only create disabled subscriptions.

### Step 4: Preview Subscription

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

### Step 5: Add Subscription

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

## Output Format for Add

After adding a subscription, respond like this:

```md
已添加 ANI-RSS 订阅，默认未启用。

- 标题：
- 季：
- RSS：
- 保存路径：
- 启用状态：false
- 解析结果：
- 注意事项：

下一步如果需要开始自动下载，需要手动启用订阅。
```

Do not enable the subscription automatically.

Do not refresh the subscription automatically.

## Error Handling

### Missing API Key

If the API returns authentication error, say:

```md
ANI-RSS API 鉴权失败。请在 Hermes Dashboard 中检查 `ANI_RSS_API_KEY`，更新后重新加载 session。
```

### Connection Failed

If connection fails, say:

```md
无法连接 ANI-RSS。请检查 `ANI_RSS_BASE_URL` 是否正确，以及 Hermes 是否能访问该地址。
```

### Duplicate Subscription

If a duplicate appears to exist, say:

```md
检测到疑似已有订阅，已停止添加，避免重复下载。
```

Then show the matching subscription.

### Bad Preview

If preview result looks wrong, say:

```md
RSS 可以访问，但 ANI-RSS 预览结果不符合预期，已停止添加。
```

Then explain which fields look wrong.
