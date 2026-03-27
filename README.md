# Smart Browser 🧠

OpenClaw 的 browser 工具很强，但大部分 agent 用起来很笨——动不动全页 snapshot、连续截图找同一个按钮、`wait(5000)` 祈祷加载完成。

这个 skill 是 8 个现成的 JS 代码片段，直接解决这些问题。纯指令型，没有脚本没有依赖，agent 读了就能用。

## 8 个技巧

### 🎯 1. 只提取可交互元素

全页 snapshot 返回几百个元素，大部分没用。这段 JS 只返回可点击/可输入的元素，附带坐标和选择器：

```javascript
(() => {
  const els = document.querySelectorAll('a[href], button, input, select, textarea, [role=button], [role=link], [role=tab], [role=menuitem], [onclick], [tabindex]:not([tabindex="-1"])');
  const results = [];
  els.forEach((el, i) => {
    const rect = el.getBoundingClientRect();
    if (rect.width === 0 || rect.height === 0 || rect.bottom < 0 || rect.top > window.innerHeight) return;
    const tag = el.tagName.toLowerCase();
    const type = el.type || '';
    const text = (el.innerText || el.textContent || '').trim().substring(0, 60).replace(/\n/g, ' ');
    const placeholder = el.placeholder || '';
    const ariaLabel = el.getAttribute('aria-label') || '';
    const href = el.href ? el.href.substring(0, 100) : '';
    const id = el.id || '';
    const cls = el.className ? String(el.className).substring(0, 60) : '';
    results.push({ i, tag, type, text: text || placeholder || ariaLabel, href, id, cls, x: Math.round(rect.x + rect.width/2), y: Math.round(rect.y + rect.height/2) });
  });
  return JSON.stringify(results);
})()
```

返回 `[{i, tag, type, text, href, id, cls, x, y}, ...]`，从几百个元素缩到十几个有用的。

### 🔤 2. 按文字查找并点击

知道按钮上写的什么字，一步到位，不需要先 snapshot：

```javascript
(() => {
  const target = '按钮文字';
  const el = [...document.querySelectorAll('a, button, [role=button], input[type=submit]')]
    .find(e => e.innerText?.trim().includes(target) || e.value?.includes(target) || e.getAttribute('aria-label')?.includes(target));
  if (el) { el.click(); return 'clicked: ' + (el.innerText || el.value || '').trim().substring(0, 50); }
  return 'not found: ' + target;
})()
```

### 📐 3. 限定区域 snapshot

页面太大时，先找到目标区域再截图：

```javascript
(() => {
  const sections = document.querySelectorAll('section, form, [role=dialog], [role=main], main, .modal, .popup, .dropdown');
  return [...sections].map((s, i) => ({
    i,
    tag: s.tagName.toLowerCase(),
    id: s.id,
    cls: String(s.className).substring(0, 60),
    text: s.innerText?.substring(0, 100).replace(/\n/g, ' ')
  }));
})()
```

找到目标 section 后，用 `browser snapshot` 的 `ref` 参数限定范围。

### ⏳ 4. 条件等待

别用固定 `wait(5000)`，等目标元素真正出现：

```javascript
await new Promise((resolve) => {
  const target = '目标文字';
  let tries = 0;
  const timer = setInterval(() => {
    if (document.body.innerText.includes(target) || ++tries > 20) { clearInterval(timer); resolve(); }
  }, 500);
});
document.body.innerText.includes('目标文字') ? 'found' : 'timeout'
```

每 500ms 检查一次，最多等 10 秒。比固定等待靠谱得多。

### 🚫 5. 清除弹窗遮罩

Cookie 提示、登录弹窗、隐私同意框，一键清掉：

```javascript
(() => {
  const overlays = document.querySelectorAll('[class*=overlay], [class*=modal], [class*=popup], [class*=cookie], [class*=consent], [role=dialog]');
  let removed = 0;
  overlays.forEach(el => {
    if (el.offsetHeight > 200 || getComputedStyle(el).position === 'fixed') {
      el.remove();
      removed++;
    }
  });
  return `removed ${removed} overlays`;
})()
```

### 📍 6. 精确坐标点击

ref 点不准时的备用方案——先获取坐标，再用鼠标点：

```javascript
(() => {
  const el = document.querySelector('你的CSS选择器');
  if (!el) return 'not found';
  const rect = el.getBoundingClientRect();
  return JSON.stringify({ x: Math.round(rect.x + rect.width/2), y: Math.round(rect.y + rect.height/2), w: Math.round(rect.width), h: Math.round(rect.height) });
})()
```

拿到坐标后用 `browser act` 的 click 操作。

### 📝 7. 表单批量填写

一次填多个字段，不用来回 snapshot：

```javascript
(() => {
  const data = {
    '#name': '值1',
    '#email': '值2',
    '#phone': '值3'
  };
  const results = [];
  for (const [sel, val] of Object.entries(data)) {
    const el = document.querySelector(sel);
    if (el) {
      el.focus();
      el.value = val;
      el.dispatchEvent(new Event('input', {bubbles: true}));
      el.dispatchEvent(new Event('change', {bubbles: true}));
      results.push(`✅ ${sel}`);
    } else {
      results.push(`❌ ${sel} not found`);
    }
  }
  return results.join(', ');
})()
```

注意 `dispatchEvent` 触发 input 和 change 事件，不然 React/Vue 等框架的表单不会响应。

### 📖 8. 提取核心文本

只需要页面文字内容时，不用截图：

```javascript
(() => {
  const main = document.querySelector('main, article, [role=main], #content, .content, .article-content, #js_content');
  const target = main || document.body;
  return target.innerText.substring(0, 8000);
})()
```

自动找 `<main>`、`<article>` 等主内容区域，找不到就 fallback 到 body。

## 💡 推荐操作流程

```
1. 打开页面      → browser open
2. 快速扫描      → 技巧 1（只看可交互元素）
3. 精准操作      → 技巧 2（按文字点击）或 snapshot + ref
4. 遇到遮挡      → 技巧 5（清弹窗）+ 技巧 6（坐标点击）
5. 读取内容      → 技巧 8（提取核心文本）
```

别这样做：
- ❌ 一上来就全页 snapshot
- ❌ 连续多次 snapshot 找同一个元素（用 evaluate 一步定位）
- ❌ 固定时间 wait

## 📦 安装

```bash
# ClawHub
clawhub install smart-browser

# 手动
git clone https://github.com/withnomeaning/smart-browser.git ~/.openclaw/skills/smart-browser
```

## 📁 文件结构

```
smart-browser/
└── SKILL.md    # 8 个技巧 + 完整 JS 代码
```

没有脚本、没有依赖、没有运行时。就一个 SKILL.md，Python 3 都不需要。

## License

MIT — [@withnomeaning](https://github.com/withnomeaning)
