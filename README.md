# collapse-long-chatgpt-messages-brave-scriptlet
<img width="779" height="229" alt="image" src="https://github.com/user-attachments/assets/a205bf71-df63-48f8-858f-762a38b7eea1" />

Brave custom scriptlet that automatically collapses long ChatGPT messages you send, and adds a small expand/collapse button at the top-right — Gemini-like. Local-only, no network, no telemetry. Built with ChatGPT.

## What it does
- Targets only user messages.
- Collapses long text.
- Adds a minimal top-right toggle button.

## Install (Brave Custom Scriptlet)
1. Open `brave://settings/shields/filters`.
2. Enable **Developer mode**.
3. Click **Add new scriptlet**, name it exactly `collapse-long-chatgpt-messages-brave-scriptlet`, and paste the script below.
4. Save. Brave stores it as **`user-collapse-long-chatgpt-messages-brave-scriptlet.js`**.
5. In **Custom rules**, add:
```javascript
chatgpt.com##+js(user-collapse-long-chatgpt-messages-brave-scriptlet.js)
chat.openai.com##+js(user-collapse-long-chatgpt-messages-brave-scriptlet.js)
```
7. Reload the page.


## Script
```javascript
(() => {
  'use strict';

  // -------- Config (adjust if needed) --------
  const MAX_LINES = 6;                 // collapsed view lines
  const HEIGHT_FALLBACK = 320;         // px threshold if line-clamp not effective
  const MIN_CHAR_TO_COLLAPSE = 280;    // ignore very short messages

  // -------- Class/attr names (scoped) --------
  const STYLE_ID = 'cu-style';
  const ATTR_PROCESSED = 'data-cu-processed';
  const CLS_COLLAPSIBLE = 'cu-collapsible';
  const CLS_COLLAPSED = 'cu-collapsed';
  const CLS_USE_CLAMP = 'cu-use-clamp';
  const CLS_TOGGLE = 'cu-toggle';

  // -------- Minimal, isolated CSS --------
  const css = `
    .${CLS_COLLAPSIBLE} { position: relative; padding-right: 28px; }
    .${CLS_COLLAPSIBLE}.${CLS_USE_CLAMP}.${CLS_COLLAPSED} {
      display: -webkit-box;
      -webkit-line-clamp: ${MAX_LINES};
      -webkit-box-orient: vertical;
      overflow: hidden !important;
    }
    .${CLS_COLLAPSIBLE}.${CLS_COLLAPSED}:not(.${CLS_USE_CLAMP}) {
      max-height: ${HEIGHT_FALLBACK}px;
      overflow: hidden !important;
    }
    .${CLS_COLLAPSIBLE} .${CLS_TOGGLE} {
      position: absolute; top: 6px; right: 6px;
      width: 24px; height: 24px; border: 1px solid #5555;
      border-radius: 50%; background: transparent;
      display: inline-flex; align-items: center; justify-content: center;
      cursor: pointer; padding: 0; margin: 0;
    }
    .${CLS_COLLAPSIBLE} .${CLS_TOGGLE}:hover { background: #0000000f; }
    .${CLS_COLLAPSIBLE} .${CLS_TOGGLE} svg { width: 14px; height: 14px; }
    .${CLS_COLLAPSIBLE}.${CLS_COLLAPSED} .icon-down { display: inline; }
    .${CLS_COLLAPSIBLE}.${CLS_COLLAPSED} .icon-up { display: none; }
    .${CLS_COLLAPSIBLE}:not(.${CLS_COLLAPSED}) .icon-down { display: none; }
    .${CLS_COLLAPSIBLE}:not(.${CLS_COLLAPSED}) .icon-up { display: inline; }
  `;

  function injectStyle() {
    if (document.getElementById(STYLE_ID)) return;
    const s = document.createElement('style');
    s.id = STYLE_ID;
    s.textContent = css;
    document.documentElement.appendChild(s);
  }

  // Heuristic: is this a user message container?
  function isUserContainer(el) {
    if (!(el instanceof HTMLElement)) return false;
    if (el.tagName === 'ARTICLE' && el.getAttribute('data-turn') === 'user') return true;
    return !!el.querySelector('[data-message-author-role="user"]');
  }

  // Find the best node to clamp (stable and text-rich)
  function findContentNode(container) {
    const prefer = container.querySelector('.whitespace-pre-wrap');
    if (prefer) return prefer;
    const role = container.querySelector('[data-message-author-role="user"]');
    if (role) {
      const firstTextEl = Array.from(role.querySelectorAll('*'))
        .find(e => Array.from(e.childNodes || []).some(n => n.nodeType === 3 && n.textContent.trim().length > 0));
      return firstTextEl || role;
    }
    return container;
  }

  // Decide whether we should collapse this content
  function shouldCollapse(el) {
    const text = (el.innerText || '').trim();
    if (text.length < MIN_CHAR_TO_COLLAPSE) return false;

    // Temporarily measure natural height
    const prev = { display: el.style.display, maxHeight: el.style.maxHeight, overflow: el.style.overflow };
    el.style.display = ''; el.style.maxHeight = ''; el.style.overflow = '';
    const h = el.scrollHeight;
    el.style.display = prev.display; el.style.maxHeight = prev.maxHeight; el.style.overflow = prev.overflow;

    return h > HEIGHT_FALLBACK;
  }

  function addToggle(contentEl) {
    if (contentEl.querySelector(`.${CLS_TOGGLE}`)) return;

    const btn = document.createElement('button');
    btn.type = 'button';
    btn.className = CLS_TOGGLE;
    btn.setAttribute('aria-label', 'Expand text');
    btn.setAttribute('title', 'Expand text');
    btn.innerHTML = `
      <svg class="icon-down" viewBox="0 0 24 24" fill="currentColor"><path d="M7 10l5 5 5-5"/></svg>
      <svg class="icon-up"   viewBox="0 0 24 24" fill="currentColor"><path d="M7 14l5-5 5 5"/></svg>
    `;
    btn.addEventListener('click', (ev) => {
      ev.stopPropagation();
      const collapsed = contentEl.classList.toggle(CLS_COLLAPSED);
      btn.setAttribute('aria-label', collapsed ? 'Expand text' : 'Collapse text');
      btn.setAttribute('title', collapsed ? 'Expand text' : 'Collapse text');
    }, { passive: true });

    contentEl.appendChild(btn);
  }

  function prepareContent(contentEl) {
    contentEl.classList.add(CLS_COLLAPSIBLE);
    contentEl.classList.add(CLS_USE_CLAMP); // prefer clamp; fallback below
    if (shouldCollapse(contentEl)) {
      contentEl.classList.add(CLS_COLLAPSED);
      addToggle(contentEl);
      return true;
    }
    return false;
  }

  function processContainer(container) {
    if (container.getAttribute(ATTR_PROCESSED) === '1') return;
    const contentEl = findContentNode(container);
    const collapsed = prepareContent(contentEl);
    if (collapsed) container.setAttribute(ATTR_PROCESSED, '1');
  }

  function scan() {
    // cover both article containers and nested role nodes
    const nodes = new Set();
    document.querySelectorAll('article, [data-message-author-role="user"]').forEach(n => {
      const c = n.closest('article') || n;
      if (isUserContainer(c)) nodes.add(c);
    });
    nodes.forEach(processContainer);
  }

  // Throttled observer
  let scheduled = false;
  function scheduleScan() {
    if (scheduled) return;
    scheduled = true;
    requestAnimationFrame(() => { scheduled = false; scan(); });
  }

  function onReady(fn) {
    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', fn, { once: true });
    } else fn();
  }

  onReady(() => {
    injectStyle();
    scan();
    const mo = new MutationObserver(scheduleScan);
    mo.observe(document.documentElement, { subtree: true, childList: true });
  });
})();
```
