# Smart Browser 🧠

**Stop snapshot-spamming. Start surgical strikes.** 8 battle-tested techniques to make your agent actually good at browser automation.

**别再无脑截全页了。精准打击才是王道。** 8 个实战技巧，让你的 Agent 浏览器操作效率翻倍。

---

## The Problem / 问题

Most agents use the browser tool like this:
1. Open page → full snapshot → stare at 500 elements → find the one button → click → full snapshot again → repeat

That's slow, expensive (tokens!), and often fails because the snapshot is too large to reason about.

大部分 Agent 操作浏览器的画风是这样的：
1. 打开页面 → 全页截图 → 盯着 500 个元素发呆 → 找到那个按钮 → 点击 → 再截一次全页 → 循环

又慢又费 token，而且经常因为 snapshot 太大导致推理出错。

**Smart Browser fixes this with 8 targeted techniques.**

**Smart Browser 用 8 个技巧解决这个问题。**

---

## Techniques / 技巧一览

### 1. 🎯 Extract Interactive Elements Only / 只提取可交互元素

Instead of snapshotting everything, run one JS to get only clickable/typeable elements with their coordinates:

不截全页，一段 JS 返回所有可点击/可输入元素：

```javascript
// Returns: [{tag, text, href, id, cls, x, y}, ...]
// Filters out invisible and off-screen elements automatically
```

> Go from 500 elements to the 15 that matter.
> 从 500 个元素缩减到真正有用的 15 个。

### 2. 🔤 Find & Click by Text / 按文字查找并点击

Know the button text? Skip the snapshot entirely:

知道按钮写的啥？直接跳过 snapshot：

```javascript
// "发布" → finds it → clicks it → done
// One evaluate call. No snapshot needed.
```

### 3. 📐 Scoped Snapshot / 限定区域截图

Page too big? Find the target section first, then snapshot only that area.

页面太大？先定位目标区域，只截那一块。

### 4. ⏳ Smart Wait / 条件等待

Don't `wait(5000)` and pray. Wait for the actual element to appear:

别 `wait(5000)` 然后祈祷。等目标元素真正出现：

```javascript
// Polls every 500ms, gives up after 10s
// Way better than fixed delays
```

### 5. 🚫 Remove Overlays / 清除弹窗遮罩

Cookie banners, login popups, consent dialogs — nuke them all in one shot:

Cookie 提示、登录弹窗、隐私同意框 — 一键全清：

```javascript
// Auto-detects fixed overlays, modals, popups → removes them
```

### 6. 📍 Coordinate Click / 精确坐标点击

When ref-based clicking misses, get exact coordinates and click directly.

当 ref 点不准时，获取精确坐标直接点。

### 7. 📝 Batch Form Fill / 表单批量填写

Fill multiple fields in one call instead of snapshot → type → snapshot → type → ...

一次填多个字段，不用来回 snapshot：

```javascript
// {'#name': 'Alice', '#email': 'a@b.com', '#phone': '123'}
// All fields filled + events dispatched in one evaluate
```

### 8. 📖 Extract Core Text / 提取核心文本

Just need the text content? Skip the snapshot, grab it directly:

只需要文字内容？直接提取，不用截图：

```javascript
// Finds <main>, <article>, or #content → returns text
// 8000 chars max, clean and fast
```

---

## Recommended Flow / 推荐操作流程

```
1. Open page         → browser open
2. Quick scan         → Technique 1 (interactive elements only)
3. Take action        → Technique 2 (click by text) or snapshot + ref
4. Trouble?           → Technique 5 (clear overlays) + 6 (coordinate click)
5. Need content?      → Technique 8 (extract text)
```

**Avoid / 避免：**
- ❌ Full-page snapshot as first action / 一上来就全页截图
- ❌ Multiple snapshots to find the same element / 连续截图找同一个元素
- ❌ Fixed-time waits / 固定时间等待

---

## Install / 安装

### Via ClawHub

```bash
clawhub install smart-browser
```

### Manual / 手动

```bash
git clone https://github.com/withnomeaning/smart-browser.git ~/.openclaw/skills/smart-browser
```

---

## Details / 技术细节

This is a **pure instruction skill** — just a SKILL.md with ready-to-use JS snippets. No scripts, no dependencies, no runtime. Your agent reads the techniques and applies them during browser operations.

这是一个**纯指令型 Skill** — 只有一个 SKILL.md，里面是现成可用的 JS 代码片段。没有脚本，没有依赖，没有运行时。Agent 读完就能用。

```
smart-browser/
└── SKILL.md    # 8 techniques with copy-paste JS snippets
```

## Requirements / 依赖

- OpenClaw with browser tool enabled
- That's literally it / 真的就这些

## License

MIT

## Author

[@withnomeaning](https://github.com/withnomeaning)
