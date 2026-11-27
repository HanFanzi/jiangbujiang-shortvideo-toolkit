# 抖音自有账号取数：git 开源方案调研

## 结论

git 上有现成解法，且正好补上官方开放平台下线后的缺口：一批 2026 年新建的开源项目，用「复用本人创作者后台登录态 + Playwright/Chrome」只读抓取自有账号数据，输出本地 CSV/JSON/SQLite，再通过 MCP 或 Skill 接给 Agent 做复盘。对应上一轮调研里的 C（后台导出）和 D（登录态自建采集），不用付费、不用第三方数据服务、不用再等官方恢复。

本机环境全部齐备：Node v22.22.2、npm 10.9.7、Python 3.13.12、Google Chrome、ffmpeg 9.0.2；只缺 Playwright（一条 `npx playwright install chromium` 或 `pip install playwright` 补齐）。

三处硬约束：全部非官方、复用登录态，可能违反平台条款或触发风控；创作者后台内部接口会变，要跟维护；这批项目 star 都低、作者自述「单元测试不代表真实账号端到端验收」。

按本轮口径，skill 未改动，也未安装任何项目。

---

## 一、项目清单

仓库元数据为 GitHub API 实测（2026-11-27），来源级别 B；功能与声明来自各仓库 README，来源级别 C。

| 色标 | 仓库 | ★/fork | 技术栈 | 许可 | 最近推送 | 定位 | 账号数 |
|---|---|---|---|---|---|---|---|
| 🟢 | `wytnzs/douyin-data-mcp` | 0 / 0 | Node 22+，零第三方依赖，手写 MCP | MIT | 2026-10-06 | 创作者后台数据抓成本地总表，接给 Agent | 单账号 |
| 🟢 | `zzyong24/moonlit-douyin-extractor` | 2 / 0 | Node 22+，Playwright，可装成 Skill | MIT | 2026-11-18 | 多账号按别名隔离登录态、指标、评论、索引 | 多账号 |
| 🟡 | `Kuhakucai/douyin-mcp` | 95 / 14 | Python 3.11+，FastMCP，Playwright，SQLite | AGPL-3.0 | 2026-07-29（更新 11-18） | 指标+音轨文案，数据可追溯，导出 JSON/CSV | 单账号 |
| 🟡 | `wenyg/douyin-creator-tools` | 172 / 36 | JavaScript，Playwright | 未见 | 2026-07-15 | 创作者中心操作工具（作品/评论/发文） | 单份登录态 |
| 🟠 | `qtest3/douyin-creator-tools` | — | Node，Playwright | — | — | 作品列表、未回复评论导出与回复、发文章 | 单份登录态 |
| 🟠 | `0xsline/creatorhub`（CreatorHub） | 2.4k–2.85k（二手口径） | Python 3.10+，Web 面板 | 未见 LICENSE | 2026-10 活跃 | 多平台面板，账号独立 Profile + 代理隔离 | 多账号 |
| ⚪ | `NanmiCoder/MediaCrawler` | 71848 / 13513 | Python，Playwright | NOASSERTION（声明仅供学习） | 2026-11-26 | 7 平台公开数据/关键词/评论爬虫 | 非自有后台口径 |
| ⚪ | `Evil0ctal/Douyin_TikTok_Download_API` | 21584 / 3000 | Python，Docker，PostgreSQL，REST+MCP | Apache-2.0 | 2026-11-20 | 自部署解析/下载/搜索 API | 公开数据 |
| ⚪ | `Gitee: votouch/TikTokDownloader`（DouK-Downloader） | — | Python | — | — | 下载 + 作品详细数据、CSV/XLSX/SQLite | 公开数据 |

`douyin-mcp in:name stars:>20` 全量结果 7 个，其余为无水印解析、上传、私信方向（`yzfly/douyin-mcp-server` ★1288 已 archived、`WJZ-P/douyin-upload-mcp-skill` ★93 等），不解决自有账号指标取数。

---

## 二、四个对口项目的能力对照

| 项 | douyin-data-mcp | moonlit-douyin-extractor | douyin-mcp | douyin-creator-tools |
|---|---|---|---|---|
| 采集范围 | 只读自己账号创作者后台 | 只读自己有权访问的账号 | 只读本人账号真实可见数据 | 创作者中心读写 |
| 指标 | 播放/点赞/评论/分享/收藏/时长/发布日 + 5秒完播率、2秒跳出率、平均播放秒 + 账号级近7天总览 | 播放/点赞/评论/分享/收藏 + 评论 | 播放/点赞/评论/分享/收藏/完播率/5秒完播率/平均观看时长/曝光/涨粉 + 图文新版块指标 | 作品列表、评论（含未回复） |
| 按天累积/历史 | 主表按「抓取日期+作品ID」累积，带日期导出副本 | 每次采集保留指标历史，按评论 ID 合并 | 快照入库，可导出全部历史快照 | 无 |
| 缺值处理 | 单条整体完播率、单条涨粉拿不到，留空不估算 | 缺字段不编造，失败记录原因 | 页面未显示存 null，不用 0 或猜测值填充；附字段覆盖率、缓存新鲜度 | — |
| 输出 | 中文表头 CSV + 总览 md + 状态 md | 按账号分目录 JSON | SQLite，导出 JSON/CSV | JSON |
| 接入 Agent | 原生 MCP，6 个工具（status/check/fetch/merge/daily/data），README 明写 WorkBuddy 在插件面板 → MCP 添加 | 可整仓装为 Skill，或直接跑脚本 | FastMCP，需先 `acknowledge-platform-risk --yes` | 命令行 |
| 登录 | 专用浏览器窗口扫码一次，Chrome 136+ 起独立 profile（`~/.douyin-data/browser-profile`） | 临时 Playwright profile 扫码，读到稳定身份才绑定别名 | 专用 Chrome 扫码，登录态留本地不回传 Agent | 复用 `.playwright/douyin-profile` |
| 系统 | 未限定（提供 启动.bat / 启动.command） | 需桌面环境 | macOS / Windows | 未限定 |
| 维护点 | 接口变了只改 `collect.js`；已列 3 个坑（未登录返回 HTTP 200 + `status_code:8`、`item_id` int64 丢精度、`involved_vertical` 必填） | 页面变化或身份不匹配时应停止处理 | 改版需跟 selector（已在跟创作者中心新版数据块） | 页面结构变化先用 `npm run view` 人工确认 |

---

## 三、数据口径边界

| 边界 | 内容 |
|---|---|
| 指标时效 | 指标截至 = 抓取日 − 1，几点抓都一样；新作品当天无完播率，后台次日补齐 |
| 播放量 | 累计值，不是当日增量；当日增量 = 今天这行 − 上一天同一条 |
| 拿不到的字段 | 单条「整体完播率」「单条涨粉」（后台只开放到账号级）；图文作品无视频深度指标 |
| 建议抓取时间 | 下午 1 点之后（深度指标次日补齐） |
| 单账号上限 | 上述 🟢/🟡 四个里，只有 moonlit 明确做多账号；其余单账号，多号需多份 profile 或自行扩展 |

---

## 四、合规与风险

| 色标 | 项 | 来源 |
|---|---|---|
| 🔴 | 全部为非官方社区工具，未获抖音授权/认可/背书；复用登录态操作浏览器，即使只读本人账号真实可见数据，也可能违反平台条款或触发账号风控 | `Kuhakucai/douyin-mcp` README 与 PLATFORM_COMPLIANCE.md 原文；himcp 收录页同口径 |
| 🔴 | 通用爬虫类（MediaCrawler、DouK-Downloader、Douyin_TikTok_Download_API）抓的是公开数据/关键词/竞品，不是自有账号后台结算口径，混合使用会把两个数据域串味 | 各仓库 README、免责声明 |
| 🟠 | 创作者后台内部接口无 SLA，页面或接口变更即需改代码；这批项目都是 2026 年新建，社区验证薄 | 各仓库 README / commit 记录 |
| 🟠 | `moonlit-douyin-extractor` 自述「单元测试不代表已通过真实账号登录或平台端到端验收，请在自己的账号和本机环境验证后使用」 | 仓库 README 原文 |
| 🟠 | CreatorHub 带发布、私信、取关等写操作，且做 UA/WebGL/地理位置指纹伪造与代理隔离，超出「只读收集」范围 | 仓库 README 与二手评测（腾讯云、掘金、头条，来源级别 C 二手） |
| 🟡 | 许可证差异：`douyin-mcp` 为 AGPL-3.0（有传染性），`MediaCrawler` 为 NOASSERTION 且声明仅供学习，其余多为 MIT / Apache-2.0 | GitHub API license 字段 |
| 🟢 | 登录态与数据全部留本机专用 profile，不上传、不回传 Agent；缺值不填充、结论附采集时间与字段覆盖率 | 四个对口项目 README |

---

## 五、推荐路径

1. 🟢 先用 `wytnzs/douyin-data-mcp` 跑通单账号全链路：零第三方依赖、原生 MCP、README 直接给 WorkBuddy 配置、按天累积主表、`douyin_daily` 一条命令走完抓取→校验→并表→总览→日志、中文 CSV 可直接进分析。缺 Playwright 一条命令补齐。
2. 🟢 多账号扩展切 `zzyong24/moonlit-douyin-extractor`：唯一明确做多账号别名隔离 + 登录态复核 + 指标历史 + 评论按 ID 合并的 MIT 项目，且可装成 Skill，与用户「我有多个账号」直接对上。
3. 🟡 若要更全指标（涨粉、曝光、图文新版块）或要音轨文案做内容复盘，再评估 `Kuhakucai/douyin-mcp`：★95 最成熟、数据可追溯最严谨，但 AGPL-3.0、单账号、需先执行风险确认、首次安装流程最重。
4. ⚪ 对标数据继续走现有红狐 HUB，不上 MediaCrawler / 下载类项目。
5. ⚪ CreatorHub 不纳入：含写操作与指纹伪造，超出只读范围。

三步都能在本机直接跑，无需新增付费、无需新建应用、无需平台审批；唯一人工动作是每个账号本人扫码一次。

---

## 六、当前状态

| 项 | 状态 |
|---|---|
| skill 文件 | 未改动，`SKILL.md` 与 `douyin_data.py` 保留原接口清单和 `URL_VIDEO_LIST` |
| 安装动作 | 未执行，本轮只做检索与核实 |
| 本机环境 | Node 22.22.2 / npm 10.9.7 / Python 3.13.12 / Chrome / ffmpeg 9.0.2 齐备；Playwright 未安装 |
| 已交付 | `research/抖音数据接口下线_官方公告与解决方案调研.md`、`research/抖音自有账号取数_git开源方案调研.md` |
| 待拍板 | 是否装 `douyin-data-mcp` 跑通单账号；是否按多账号需求直接上 `moonlit-douyin-extractor`；其余账号标签 + 抖音号；skill 是否按结论改造 |
