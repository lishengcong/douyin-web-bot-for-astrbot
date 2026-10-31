# 用浏览器把抖音接进 AstrBot

> 一条养在 AstrBot 里的数字生命：会私信聊天、会自己刷抖音、会读评论区、会评论并 @ 你。
>
> 全部基于**真实 Chrome + 网页版**，不碰抖音的 IM 协议，因此没有登录风控问题，
> 而且顺带拿到了「刷视频 / 读评论 / 发评论」这些协议做不到的能力。

---

## 目录

- [它是什么](#它是什么)
- [架构](#架构)
- [环境准备](#环境准备)
- [第一部分：私信通道](#第一部分私信通道)
- [第二部分：抖音工具插件](#第二部分抖音工具插件)
- [第三部分：额度感知](#第三部分额度感知)
- [配置文件](#配置文件)
- [能力清单](#能力清单)
- [性能取舍](#性能取舍)
- [已知限制](#已知限制)
- [可以继续做的](#可以继续做的)

---

## 它是什么

一套让 AstrBot 里的角色「活在抖音上」的方案，由三部分组成：

```
真实 Chrome（已登录的抖音网页版）
       │
       ├── 私信通道插件  → 把网页私信变成 AstrBot 的消息来源
       ├── 抖音工具插件  → 给她刷视频、读评论、发评论的能力
       └── 额度插件      → 让她知道自己的 token 还剩多少
```

**核心取舍：用秒级延迟，换掉协议登录的风控风险，并拿到内容能力。**

---

## 架构

```
抖音网页版（www.douyin.com）
   │
   ├── 私信  /chat
   ├── 推荐  /jingxuan
   ├── 视频  /video/<id>
   ├── 主页  /user/self
   └── 通知  顶部悬浮面板
          │
          ▼
   真实 Chrome（独立 profile，带窗口）
          │  Playwright 驱动 / CDP 调试
          ▼
   AstrBot 插件层
   ├── astrbot_plugin_douyinchat     私信平台适配器
   ├── astrbot_plugin_douyin_browse  抖音工具（8 个）
   └── astrbot_plugin_fish_budget    token 额度注入
          │
          ▼
   AstrBot 大脑（人格 / 记忆 / 自我进化 / 模型）
```

**一个 Chrome，三个插件共用。** 私信和刷视频在同一个登录态下工作。

---

## 环境准备

### 1. 独立 Chrome profile

不要用你日常的浏览器 profile。

```powershell
$profile = 'D:\qqfish\dchat-profile'
New-Item -ItemType Directory -Path $profile -Force

& 'C:\Program Files\Google\Chrome\Application\chrome.exe' `
  "--user-data-dir=$profile" `
  --no-first-run --no-default-browser-check `
  --remote-debugging-port=9222
```

在弹出的窗口里扫码登录一次，之后 profile 就带着登录态了。

### 2. ⚠️ 必须带窗口，不能用无头

无头模式会被抖音识别，直接返回验证码页。

如果不想让它占屏幕，把窗口挪到屏幕外：

```
--window-position=-2000,0
```

### 3. 依赖

| 依赖 | 说明 |
|---|---|
| Python | AstrBot 自带环境即可 |
| Playwright | `pip install playwright`（**不需要** `playwright install`，我们用系统 Chrome） |
| Chrome | 已安装即可 |

---

## 第一部分：私信通道

用插件 `astrbot_plugin_douyinchat`，它把抖音网页私信接成 AstrBot 的平台适配器。

装好后在 `data/cmd_config.json` 的 `platform` 里加一条：

```jsonc
{
  "id": "douyin",
  "type": "douyin",
  "enable": true,
  "browser_profile_dir": "D:\\qqfish\\dchat-profile",
  "cookies": "",
  "headless": false,                 // 必须 false
  "watched_conversations": [],       // 空 = 自动发现
  "group_conversations": [],         // 群聊名字写这里
  "poll_interval": 3,
  "login_timeout": 300,
  "self_uid": "",
  "conversation_aliases": []
}
```

### 需要改的四个地方

原插件能跑，但有几处要调整才能用得舒服。

**① 用系统 Chrome，不下载 Playwright 内核**

`core/client.py`：

```diff
         self.context = await self.pw.chromium.launch_persistent_context(
             str(self.profile_dir),
+            executable_path=r"C:\Program Files\Google\Chrome\Application\chrome.exe",
             headless=self.headless,
```

**② 白名单改为可选**

`core/adapter.py`：

```diff
         if not watched:
-            raise ValueError("douyin adapter requires non-empty watched_conversations")
+            logger.warning("[douyin] 未配置白名单，将自动发现并监听所有私聊会话")
```

**③ 空名单时自动发现会话**

原版空名单时主循环什么都不轮询，只打一句「将监听所有会话」的日志。
改成自动拉取会话列表，并定期刷新：

```python
        discovered: list[str] = list(watched)
        last_discover = 0.0

        while not self._shutdown.is_set():
            if not watched:
                now = time.monotonic()
                if not discovered or (now - last_discover) > 60:
                    try:
                        convs = await self.client.list_conversations()
                        names = [
                            (c.get("nickname") or c.get("name") or "").strip()
                            for c in convs
                            if (c.get("nickname") or c.get("name"))
                        ]
                        # 只轮询最上面几个：列表按最近消息排序，新消息会置顶
                        discovered = names[:5]
                    except Exception as exc:
                        logger.warning(f"[douyin] 会话发现失败: {exc}")
                    last_discover = now

            for conv_key in (watched or discovered):
                ...
```

**④ 按 uid 的白名单 + 主人**

昵称会变，uid 不会。事件的 `sender.user_id` 就是发送者 uid。

```python
    WHITELIST_PATH = r"D:\qqfish\douyin-whitelist.json"

    def _allow_sender(self, abm: AstrBotMessage) -> bool:
        cfg = self._load_whitelist()
        if str(cfg.get("mode") or "all").strip().lower() != "whitelist":
            return True

        sender = str(abm.sender.user_id or "")
        master = str(cfg.get("master") or "").strip()
        allowed = {str(x) for x in (cfg.get("allowed_uids") or [])}
        if master:
            allowed.add(master)
        if sender in allowed:
            return True

        names = {str(x) for x in (cfg.get("allowed_names") or [])}
        if names and str(abm.sender.nickname or "").strip() in names:
            return True
        return False
```

---

## 第二部分：抖音工具插件

`astrbot_plugin_douyin_browse` —— 给她八个工具。

### 工具清单

| 工具 | 作用 | 读/写 |
|---|---|---|
| `douyin_feed` | 看推荐流（标题/作者/点赞/时长/id） | 读 |
| `douyin_next` | 往下翻一屏 | 读 |
| `douyin_search` | 搜关键词 | 读 |
| `douyin_comments` | 读评论区（带序号） | 读 |
| `douyin_notifications` | 看通知：@我 / 回复我 / 赞我 / 关注我 | 读 |
| `douyin_comment` | 发评论，支持 @ 某人 | 写 |
| `douyin_reply_comment` | 回复某条评论 | 写 |
| `douyin_edit_bio` | 改自己的简介 | 写 |

### 关键技术选择：读 DOM，不截图

```
20 条视频的完整信息 ≈ 1500 token
同样内容用截图   ≈ 30000 token
```

差 20 倍，而且不会看错字。

**推荐流的锚点**（稳定）：

```js
document.querySelectorAll('.discover-video-card-item')   // 卡片
  .map(el => ({
    id: el.getAttribute('data-aweme-id'),                // 视频 id
    text: el.innerText,
  }))
```

卡片文本形如：`01:17 | 13.0万 | 标题 #话题 | @作者 | · 23小时前`

**评论区的锚点**：

```js
document.querySelectorAll('[data-e2e="comment-item"]')
```

### 评论区面板

评论输入框默认不显示，要点评论图标才展开：

```python
await page.evaluate(
    "() => { const e = document.querySelector('[data-e2e=\"feed-comment-icon\"]'); "
    "if (e) e.click(); }"
)
```

输入框是 `contenteditable`，靠 placeholder 文案定位（抖音把 placeholder 写在显示文本里，不是属性）：

```js
const phNode = [...document.querySelectorAll('div,span')]
  .find(e => e.children.length === 0 &&
        (e.textContent || '').trim() === '留下你的精彩评论吧');
// 然后向上找可编辑祖先
```

### 评论里 @ 人

**用键盘，不用点按钮** —— 抖音的 @ 是输入触发的：

```python
await page.keyboard.type("@")
await page.wait_for_timeout(900)
await page.keyboard.type(nickname)
await page.wait_for_timeout(1500)
await page.keyboard.press("Enter")
```

### 通知面板（这里有个细节）

**`通知` 是悬停浮层，不是点击。**

鼠标移上去才展开，移开就消失。所以流程必须是：

```python
pos = await page.evaluate(HOVER_POS_JS)      # 找到「通知」的屏幕坐标
await page.mouse.move(pos["x"], pos["y"])
await page.wait_for_timeout(3000)            # 等浮层出现
panel = await page.evaluate(READ_PANEL_JS)   # 立刻读
await page.mouse.move(10, 400)               # 移开，别一直挡着
```

面板内容形如：

```
[nick] @某人 在评论中提到了你  08-22
[nick] 回复了你的评论          08-23
[nick] 赞了你的评论            09-03
[nick] 关注了你                10-23
```

**「在评论中提到了你」就是有人 @ 她。**

### 写操作的三重保护

```
① 只能对「当前打开的视频」操作 —— 不能乱发
② 拿到成功信号才算发出；看不到就报告「不确定」，不重试
③ 独立开关 + 每日次数上限
```

---

## 第三部分：额度感知

`astrbot_plugin_fish_budget` —— 每轮请求前注入一行：

```xml
<system_reminder>你今天还剩约 44 万 token 预算（已用 12%，还很充足）。</system_reminder>
```

数据来自 AstrBot 自带的 `provider_stats` 表：

```sql
SELECT SUM(token_input_other) + SUM(token_input_cached),
       SUM(token_output)
  FROM provider_stats
 WHERE date(created_at) = ?
```

**是真实用量，不是编的。** 挂载点是 `@filter.on_llm_request()`：

```python
@filter.on_llm_request()
async def inject_budget(self, event: AstrMessageEvent, req: ProviderRequest):
    ...
    req.extra_user_content_parts.append(TextPart(text=line).mark_as_temp())
```

---

## 配置文件

| 文件 | 作用 |
|---|---|
| `D:\qqfish\douyin-whitelist.json` | 谁能收到回复 |
| `D:\qqfish\douyin-actions.json` | 写操作开关与上限 |
| `D:\qqfish\fish-budget.json` | 每日 token 预算 |
| `data\cmd_config.json` | 平台 / 人格 / 模型 / admins |

### 白名单

```json
{
  "mode": "whitelist",
  "master": "336910714929712",
  "allowed_uids": ["336910714929712"],
  "allowed_names": []
}
```

| 字段 | 说明 |
|---|---|
| `mode` | `all` 全回 / `whitelist` 只回名单 |
| `master` | 主人 uid，自动放行 |
| `allowed_uids` | 按 uid 放行（最稳，改名不失效） |
| `allowed_names` | 按昵称放行（方便，但对方改名就失效） |

**改完立刻生效，不用重启。**

### 写操作开关

```json
{
  "comment_enabled": true,
  "bio_enabled": false,
  "daily_comment_limit": 5,
  "daily_bio_limit": 1
}
```

### 预算

```json
{
  "daily_input_tokens": 500000,
  "daily_output_tokens": 60000
}
```

---

## 能力清单

| 能力 | 实现方式 |
|---|---|
| 私信收发 | 平台适配器轮询 + 页面输入 |
| 读推荐流 | DOM 抓 `.discover-video-card-item` |
| 刷下一个 | 鼠标滚轮 |
| 读评论区 | DOM 抓 `[data-e2e="comment-item"]` |
| 看通知 / @ | 悬停顶部「通知」读浮层 |
| 发评论 | 点评论图标 → 填输入框 → 点发送 |
| 评论 @ 人 | 键盘打 `@` 触发下拉 → 打昵称 → 回车 |
| 回复评论 | 点该条评论下的「回复」→ 填 → 发 |
| 额度感知 | 查 `provider_stats` + 注入提醒 |

---

## 性能取舍

适配器需要「点开对话」才能读到消息，所以**每轮点击次数直接决定延迟**。

```
全量轮询 29 个会话 → 一轮几十秒，肉眼能看到窗口一直在跳
只轮询前 5 个       → 一轮约 10 秒
```

**为什么前 5 个够**：会话列表按最近消息排序，谁发消息谁置顶。

可调参数：

```jsonc
"poll_interval": 3      // 轮询间隔（秒），调小更快但更耗资源
```

---

## 已知限制

| 限制 | 说明 |
|---|---|
| **回复慢** | 秒级，不是毫秒 |
| **必须带窗口** | 无头会被识别成验证码页 |
| **浏览器要常开** | 关了就不收消息 |
| **网页改版就坏** | 选择器要跟着改 |
| **没有互关判断** | 网页版不暴露互关状态，只能用 uid/昵称白名单 |
| **陌生人只能发一条** | 抖音自身限制，主动私信要避开陌生人 |
| **群聊 @ 需验证** | 需要确认群消息里的 @ 段能否解析 |

---

## 可以继续做的

| 想法 | 价值 |
|---|---|
| **发作品** | 上传视频到创作者中心 —— 她有内容，评论和 @ 才有意义 |
| **群聊 @ 支持** | 先验证群消息的 @ 段 |
| **改头像** | 文件上传，风险最高 |
| **只轮询有未读的会话** | 现在点 5 次，理想是点 0~2 次 |
| **看过的内容写进长期记忆** | 现在能读，但读到的未必都记住 |

---

## 一句话总结

**用真实浏览器当眼睛和手 —— 私信、刷视频、读评论统一在一个登录态里，
工具读 DOM 而不是截图，人格里写清她有哪些工具。**

功能是骨架，人格才是她。
