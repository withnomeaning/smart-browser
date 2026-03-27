---
name: smart-browser
description: >
  高效浏览器操作技巧：快速识别页面可交互元素、精准定位目标、减少无效 snapshot。
  Use when: 操作网页、点击按钮、填写表单、浏览器自动化、网页交互。
  NOT for: 纯内容抓取（用 web_fetch）、不需要交互的页面阅读。
---

# Smart Browser — 高效浏览器操作

## 核心原则

**先扫描再操作，不要盲目 snapshot 整个页面。**

## 技巧 1：只提取可交互元素

在操作页面前，先用 evaluate 跑这段 JS，只返回可点击/可输入的元素：

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

返回格式：`[{i, tag, type, text, href, id, cls, x, y}, ...]`
- `text`: 元素显示文字（或 placeholder / aria-label）
- `x, y`: 元素中心坐标（可用于精确点击）
- `id, cls`: 用于 CSS 选择器定位

## 技巧 2：按文字查找并点击

如果知道要点的按钮文字，一步到位：

```javascript
(() => {
  const target = '按钮文字';
  const el = [...document.querySelectorAll('a, button, [role=button], input[type=submit]')]
    .find(e => e.innerText?.trim().includes(target) || e.value?.includes(target) || e.getAttribute('aria-label')?.includes(target));
  if (el) { el.click(); return 'clicked: ' + (el.innerText || el.value || '').trim().substring(0, 50); }
  return 'not found: ' + target;
})()
```

## 技巧 3：限定区域 snapshot

snapshot 整个页面太大时，先用 evaluate 找到目标区域，再对该区域做 snapshot：

```javascript
// 找到包含特定文字的区域
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

然后用 `browser snapshot` 的 `ref` 参数限定范围。

## 技巧 4：等待特定元素出现

页面加载慢时，不要用固定 wait，用条件等待：

```javascript
// 等待某个文字出现（最多等 10 秒）
await new Promise((resolve) => {
  const target = '目标文字';
  let tries = 0;
  const timer = setInterval(() => {
    if (document.body.innerText.includes(target) || ++tries > 20) { clearInterval(timer); resolve(); }
  }, 500);
});
document.body.innerText.includes('目标文字') ? 'found' : 'timeout'
```

## 技巧 5：处理弹窗/遮罩

很多网站有 cookie 提示、登录弹窗等遮挡操作，先清掉：

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

## 技巧 6：精确坐标点击（备用方案）

当 ref 点不准时，用 evaluate 获取坐标后直接鼠标点击：

```javascript
(() => {
  const el = document.querySelector('你的CSS选择器');
  if (!el) return 'not found';
  const rect = el.getBoundingClientRect();
  return JSON.stringify({ x: Math.round(rect.x + rect.width/2), y: Math.round(rect.y + rect.height/2), w: Math.round(rect.width), h: Math.round(rect.height) });
})()
```

然后用 `browser act` 的 `click` + 坐标操作。

## 技巧 7：表单批量填写

一次性填多个字段，减少来回：

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

## 技巧 8：获取页面核心文本内容

不需要完整 snapshot 时，快速提取页面主要文字：

```javascript
(() => {
  const main = document.querySelector('main, article, [role=main], #content, .content, .article-content, #js_content');
  const target = main || document.body;
  return target.innerText.substring(0, 8000);
})()
```

## 操作流程建议

1. **打开页面** → `browser open`
2. **快速扫描** → evaluate 技巧 1（只看可交互元素）
3. **精准操作** → evaluate 技巧 2（按文字点击）或 snapshot + ref
4. **遇到问题** → 技巧 5 清弹窗，技巧 6 坐标点击
5. **需要读内容** → 技巧 8 提取核心文本

避免：
- ❌ 一上来就完整 snapshot（太慢太大）
- ❌ 连续多次 snapshot 找同一个元素（用 evaluate 一步定位）
- ❌ 固定时间 wait（用条件等待）
