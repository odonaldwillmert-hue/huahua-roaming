---
name: huahua-roaming
description: "制作花花漫游国风名片，提供赏曲、时事、书籍、自由探索四个方向及带真实来源的漫游来信。用户说派花花去旅行、去漫游、推荐音乐并带回名片、音乐/新闻/书籍明信片时使用。赏曲名片可选联动已安装的 Hana Audio Player；不可用时保持只读。"
profile: card-skill
---

# 花花漫游 · 山水来信

本配方负责推荐名片和来信，真实检索由 Agent 完成。音乐播放交给 **Yuexiye 的「音频播放器」Hana App，ID `hanako-audio-player`**；不是 `hanako-simple-player`，不在本卡实现音频播放器。

## 不变的边界

- 保留赏曲、时事、书籍、自由探索四个方向，以及已有青绿山水、朱砂印章和 Hana 控件风格。
- 图片、印章、样式随包携带；字体跟随 Hana／系统。图片是 GPT 意境插画，不是新闻现场、音乐演出现场或书籍封面。
- 不在卡片里设置 `audio.src`、抓音频直链、嵌入第三方 App 页面或绕过宿主权限。
- 不自动播放，不自动安装播放器，不修改 App、登录信息或音乐库文件。不因安装配方开启定时任务。
- 未取得来源时不编造发现；未实际听取或通读时不冒充已听、已读。

## 选方向卡

模板：`huahua-roaming/assets/roam.card.html`（最终名称以管理器安装结果为准）。

```json
{
  "title": "花花漫游 · 山水来信",
  "template": "huahua-roaming/assets/roam.card.html",
  "state": {
    "uiLanguage": "zh",
    "trip": {
      "selected": 0,
      "direction": "赏曲",
      "topicId": "music",
      "status": "ready",
      "art": "gpt-landscape-and-seal"
    }
  }
}
```

四个方向依次为 `0/music/赏曲`、`1/news/时事`、`2/books/书籍`、`3/explore/自由探索`。

用户点出发后发出 `trip.request`，载荷包含 `topicId`、`direction` 和本次任务说明。Agent 根据该方向真实检索，再用结果模板寄回来信。事件受理不是任务完成；检索失败要如实说明。

## 赏曲名片：有 App 就联动，没有就只读

1. 先核实本次推荐的曲名、演奏者／歌手、版本和资料来源。
2. **铸卡前**检查当前会话可用工具：优先 `audio_find_play`，或对已核实音源使用 `audio_play`。可通过工具目录发现，必要时用 `extension_manager inspect` 确认 `app:hanako-audio-player` 对当前助手已启用。
3. 只有这个 App 已启用、工具确实对事件将返回的当前会话可用，才传 `playback.playerAvailable: true`。不要把“其他会话有工具”“安装过 App”当作当前可用证据。
4. 没有 App、工具不可用或无法确认时，传 `playback: null`（也可以省略字段）。名片仍完整展示介绍、画作和来源，不显示播放按钮，不催用户安装。
5. 用户点名片里的「播放」，卡片发出 `roaming.play`，Agent 负责调用 App；卡片不直接调用工具，因此不会把工具返回的播放卡留在不可见的调用结果里。

可选状态示例（必须填写本次推荐，不固定为示例曲目）：

```json
{
  "playback": {
    "playerAvailable": true,
    "track": "流水",
    "artist": "管平湖",
    "query": "流水 管平湖",
    "server": "bilibili"
  }
}
```

`server` 仅允许 `netease`、`tencent`、`bilibili`，缺省为 `netease`。`query` 只含具体曲名、作者／演奏者等关键词，不传整句聊天内容。

### 收到 `roaming.play`

- 按事件中的精确 `cardInstanceId` 检查来源，不替换成另一张同名卡。
- 重新确认 `hanako-audio-player` 的播放工具仍可用；若不可用就简短说明，本卡仍可读，不自动装 App 或跨会话绕行。
- 首选调用 `audio_find_play({query, server, playNow:true})`；若已有同版本的真实可播 source，可用 `audio_play({source, title})`，不要拿歌名冒充音频地址。
- 让 App 工具返回的 `details.card` 通过正常工具返回通道交付播放器；不自行拼 App 内部 URL，不复制 App 实现。
- 检查工具的实际返回曲目。如果检索首条与推荐不匹配，要指出差异，不声称播放了原推荐。
- “已入列”“已请求切歌”“已开始播放”是三种不同状态。工具只回 `queued` 或 `playRequested` 时，只说已交给播放器。首次播放可能要在 App 内再点一次播放／继续播放；这是 App 的正常行为，不伪装已出声。
- 闲聊也遵守同一能力检查：若闲聊未开放这些工具，默认只读，不偷偷指定工作会话作为事件去向。

## 来信模板

模板：`huahua-roaming/assets/letter.card.html`。`state` 整体替换默认示例，必须携带完整内容：

| 字段 | 用途 |
|---|---|
| `uiLanguage` | 当前为 `zh`，文案固定中文 |
| `topicId` / `direction` | 四个方向之一，选择对应本地画作 |
| `issue` | 本次序号或短标签；不要虚构累计次数 |
| `title` / `byline` | 本次标题与摘要 |
| `imageAlt` / `artNote` | 意境画说明，不能伪称现场照片 |
| `letter` | 2–6 段短来信，事实与联想分开 |
| `facts` | 1–6 项 `{k, v}` 已核实信息 |
| `notes` | 1–4 项 `{title, body}` 解释或欣赏线索 |
| `sources` | 1–4 项 `{label, url}`，使用实际查阅的 HTTPS 来源 |
| `footnote` | 核查范围、未试听／未通读、版本差异等限制 |
| `playback` | 赏曲可选；默认 `null`，其他方向不显示播放入口 |
| `view` | 可选 `{page:"letter"}`；阅读页由卡片分别保存 |

时事核对事件与报道日期；书籍核对书名、作者、出版信息，不捏造引文；自由探索选择范围适中的真实主题。结果里的外部链接仅是来源，音频文件不随配方分发。

## 降级与持久化

- 无播放 App：只读；无宿主 `emit`：也不显示播放入口。
- 已点击播放的状态写入该卡的 `playerRequest`，只表示请求送达，不表示音乐正在播放。
- 两页来信共用同一个高度；切换不会重新检索。
- 独立打开仍可阅读。来源链接按浏览器规则打开；宿主能力不可用时不承诺后台任务。
