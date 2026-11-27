# AE互联能力核查：能否完成自有账号取数需求

## 结论

🟢 **能，而且是本会话目前最短路径。** `ae_douyin_fetch_creator_item_list` 一次性给「自有账号 + 全量作品列表 + 逐条指标」，不需要开放平台、不需要小程序、不需要备案、不需要授权弹窗、不需要自己起 Chrome CDP。唯一前提：**本人创作者后台（creator.douyin.com）Cookie**。

🔴 前面几轮「必须走经营矩阵」的判断，前提限定在「官方 OpenAPI / 小程序授权」两条路。AE 走的是第三条：复用创作者后台登录态的第三方封装，与已审阅的 `douyin-data-mcp` 同源，但不用自建采集脚本。前面文档结论不变，只是被绕过了。

需求口径：按本会话主线＝「自动收集账号数据，进行系统分析，制定下一步运营方案」（用户 2026-11-26 原话，来源 A）。若指对标账号数据，AE 另有公开数据工具（sec_uid + 作品 ID 链路），是另一套判定，不在本文结论内。

---

## 一、决定性工具（B，GetToolDetail 原文）

`ae_douyin_fetch_creator_item_list`

| 项 | 内容 |
|---|---|
| 用途 | 「Retrieve a page of published items for the Douyin creator account **authenticated by the supplied Cookie**」 |
| 必填 | `cookie`（Douyin Creator Platform Cookie）、`start_time`、`end_time`（毫秒时间戳） |
| 分页 | `count` 默认 10、最多 100/页；`cursor` 翻页；`has_more` 判停 |
| 指标 | 播放、点赞、评论、分享、收藏（实时）；2s 跳出率、5s / 完整完播率、封面点击率、平均播放时长、粉丝增量（每小时）；主页访问量（每天） |
| 排序 | `order_by` 1–26，覆盖上述全部指标升降序 |
| 字段 | `fields` 默认 `metrics,review,visibility`（含审核状态、可见性） |
| 价格 | **[数值] 积分/次**（成功调用即计费，无结果也计费） |
| 限制原文 | 「Do not use for arbitrary creators **without their authenticated Cookie**」「This is a list query, **not a bulk export**」「Avoid excessively broad time ranges because requests may time out」「response schema leaves data **unstructured**」 |

链路：

```
创作者后台 Cookie → ae_douyin_fetch_creator_item_list（按时间范围分页）
  → 每页 ≤100 条作品 + 逐条指标 → 本地汇总 → 系统分析 → 运营方案
```

多账号＝每个账号一份 Cookie 分别调，无需扫码授权。账号级补充可用 `ae_douyin_billboard_fetch_hot_account_trends_list`（新增点赞/作品/评论/分享趋势）、`ae_douyin_billboard_fetch_hot_account_item_analysis_list`（账号内容分析），各 [数值] 积分/次。

## 二、无 Cookie 时的备选（公开数据，B）

| 步 | 工具 | 价格 | 口径 |
|---|---|---|---|
| 主页链接 → sec_user_id | `ae_douyin_get_all_sec_user_id`（≤10 链接/次） | 2.07 | 批量转 ID |
| 列账号作品 | `ae_douyin_fetch_user_post_videos`（≤20/页）或 `ae_douyin_fetch_douplus_user_posts` | 2.07 | 原文「web-based retrieval **may be unstable**」，可选网页 Cookie |
| 逐条统计 | `ae_douyin_fetch_multi_video_statistics`（≤50 ID/次） | 51.64 | 只有 `digg_count` / `download_count` / `play_count` / `share_count`，**无评论数** |
| 逐条统计（含评论） | `ae_douyin_xingtu_multi_get_item_stats`（≤20 ID/次） | 82.62 | `watch_cnt`（含付费）、点赞/评论/分享/收藏、互动率、平均观看时长 |
| 单条明细 | `ae_douyin_xingtu_get_item_play_count` | 4.13 | `watch_cnt` 含付费推广，非自然播放 |

评论内容另有 `ae_douyin_fetch_video_comments`、`ae_douyin_fetch_video_comment_replies`。

---

## 三、边界（未核实项）

| 级别 | 项 | 依据 |
|---|---|---|
| 🔴 | Cookie 是硬前提，且会过期 | 工具必填 `cookie`；原文「if authentication fails, check whether the Cookie has expired」 |
| 🟠 | Cookie 作为请求参数发给 AE 服务端 | 登录态凭据外发；`douyin-data-mcp` 是纯本地、不回传登录态。两者安全边界不同，需自行取舍 |
| 🟠 | 返回结构未固定 | 原文「response schema leaves data unstructured」，实际字段名要实测一次才确定 |
| 🟠 | 只能查有 Cookie 的自有账号 | 原文「Do not use for arbitrary creators」；对标账号需走公开数据链路 |
| 🟡 | 播放量口径 | 公开链路 `play_count` / `watch_cnt` 的来源与口径未核实；`watch_cnt` 明确「includes paid promotion」，不等于自然播放 |
| 🟡 | 时间范围易超时 | 原文提示过宽时间范围会 timeout，需分段 |
| 🟡 | 稳定性 | `ae_douyin_fetch_user_post_videos` 原文标注 web-based retrieval may be unstable |

---

## 四、成本口径（B，2026-11-27）

| 项 | 值 |
|---|---|
| 账户余额 | [数值] 积分（`GetAccountOverview`，key `[账户标识]`） |
| 单账号 1000 条作品 | ≈ 10 次 × 2.07 ＝ **[数值] 积分**（口径：每页满 100 条、一次成功、无重试） |
| 10 个账号一轮 | ≈ [数值] 积分 |
| 计费规则 | 成功调用即计费，返回空结果也计费 |

---

## 五、核查范围与来源

- B 工具实测（2026-11-27）：`ListTools` 全平台第 1 页 100/2748；`keyword=douyin` 三页共 **259 个抖音工具全部覆盖**；`keyword=作品列表`（total=1）、`keyword=授权`（total=3，均无关）、`keyword=创作者后台`（total=0）；`RecommendTools` 两轮（30 + 32 候选）；`GetToolDetail(ae_douyin_fetch_creator_item_list)`；`GetAccountOverview`。
- 全平台 2748 个非抖音工具未逐页遍历，结论不涉及它们。
- A 用户材料：需求原话。C 联网检索：前几轮官方 OpenAPI / 小程序公告核实。
- 本轮只读核查：未调用任何 `ExecuteTool`，未消耗积分取数，未改 skill，未动控制台。

## 六、下一步需确认

1. 提供本人创作者后台 Cookie（敏感凭据，建议只在本地配置，不明文外发）。
2. 实测一次 `ae_douyin_fetch_creator_item_list`，确认真实返回字段名、指标完整性、Cookie 是否有效。
3. 确认后再定是否替换前面几条官方路径。
