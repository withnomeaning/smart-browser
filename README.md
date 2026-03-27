# Smart Browser 🧠

Your agent opens a page, takes a full snapshot, stares at 500 elements, finds one button, clicks it, takes another full snapshot. Repeat.

Slow, expensive (tokens add up), and half the time the snapshot is too large to reason about properly.

This skill is 8 ready-to-use JS snippets that fix this. Pure instructions — no scripts, no dependencies, no runtime. Agent reads the techniques, applies them immediately.

## ⚡ Quick Start

```bash
# ClawHub
clawhub install smart-browser

# Manual
git clone https://github.com/withnomeaning/smart-browser.git ~/.openclaw/skills/smart-browser
```

That's it. One file, zero setup:

```
smart-browser/
└── SKILL.md    # 8 techniques with complete JS code
```

## 🎯 The 8 Techniques

### 1. Extract Interactive Elements Only

Instead of snapshotting the entire page, one JS call returns only clickable/typeable elements with coordinates:

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

Returns `[{i, tag, type, text, href, id, cls, x, y}, ...]` — from hundreds of elements down to the ~15 that matter. Filters out invisible and off-screen elements automatically.

### 2. Find & Click by Text

Know the button label? Skip the snapshot entirely:

```javascript
(() => {
  const target = 'Submit';
  const el = [...document.querySelectorAll('a, button, [role=button], input[type=submit]')]
    .find(e => e.innerText?.trim().includes(target) || e.value?.includes(target) || e.getAttribute('aria-label')?.includes(target));
  if (el) { el.click(); return 'clicked: ' + (el.innerText || el.value || '').trim().substring(0, 50); }
  return 'not found: ' + target;
})()
```

One `evaluate` call. No snapshot round-trip.

### 3. Scoped Snapshot

Page too large? Find the target section first, then snapshot only that area:

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

Identify the section, then use `browser snapshot` with a `ref` to limit scope.

### 4. Conditional Wait

Stop using `wait(5000)` and hoping for the best. Wait for the actual element:

```javascript
await new Promise((resolve) => {
  const target = 'Success';
  let tries = 0;
  const timer = setInterval(() => {
    if (document.body.innerText.includes(target) || ++tries > 20) { clearInterval(timer); resolve(); }
  }, 500);
});
document.body.innerText.includes('Success') ? 'found' : 'timeout'
```

Polls every 500ms, gives up after 10s. Way more reliable than fixed delays.

### 5. Remove Overlays

Cookie banners, login popups, consent dialogs — kill them all:

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

### 6. Coordinate Click

When ref-based clicking misses, get exact coordinates and click directly:

```javascript
(() => {
  const el = document.querySelector('.your-selector');
  if (!el) return 'not found';
  const rect = el.getBoundingClientRect();
  return JSON.stringify({ x: Math.round(rect.x + rect.width/2), y: Math.round(rect.y + rect.height/2), w: Math.round(rect.width), h: Math.round(rect.height) });
})()
```

Then use `browser act` click with the returned coordinates.

### 7. Batch Form Fill

Fill multiple fields in one call instead of snapshot → type → snapshot → type:

```javascript
(() => {
  const data = {
    '#name': 'Alice',
    '#email': 'alice@example.com',
    '#phone': '555-0100'
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

The `dispatchEvent` calls are critical — without them, React/Vue/Angular forms won't register the change.

### 8. Extract Core Text

Just need the text? Don't screenshot:

```javascript
(() => {
  const main = document.querySelector('main, article, [role=main], #content, .content, .article-content, #js_content');
  const target = main || document.body;
  return target.innerText.substring(0, 8000);
})()
```

Auto-detects `<main>`, `<article>`, or common content containers. Falls back to `document.body`.

## 💡 Recommended Flow

```
1. Open page      → browser open
2. Quick scan     → Technique 1 (interactive elements only)
3. Take action    → Technique 2 (click by text) or snapshot + ref
4. Blocked?       → Technique 5 (clear overlays) + 6 (coordinate click)
5. Need content?  → Technique 8 (extract text)
```

What NOT to do:
- ❌ Full-page snapshot as your first move
- ❌ Multiple snapshots hunting for the same element
- ❌ Fixed-time waits

## License

MIT — [@withnomeaning](https://github.com/withnomeaning)
