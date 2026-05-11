---
title: 133《Shift Translator Hover Toggle》调用Chrome自带的离线翻译API，沉浸式翻译的平替
---

沉浸式翻译入侵性越来越强，全是狗皮膏药，甚至还入侵文本框，设置也越来越复杂，很难关，有开发者开发了一款直接调用Chrome自带翻译API的脚本，安装油猴后，通过以下地址安装脚本即可。

脚本开源地址：https://greasyfork.org/en/scripts/577261-shift-translator-hover-toggle-selection-tooltip-chrome-translator-api


使用方法非常简单，选中文本后，按shift即可，速度超块！（虽然谷歌不是一个善良的公司，但谷歌翻译的技术力还是可以的）

使用效果：

![](./133-shift-translator-hover-toggle.assets/0e4cde4592dff16da9c99f5228e3209bae5922bf617751000c4833f5510fb05f.gif)


## 脚本内容

```
// ==UserScript==
// @name         Shift Translator Hover Toggle + Selection Tooltip (Chrome Translator API)
// @namespace    https://example.com/
// @version      1.0.0
// @description  Hover paragraph + modifier key to toggle translation. Select text + modifier key for tooltip translation.
// @match        *://*/*
// @run-at       document-idle
// @grant        GM_getValue
// @grant        GM_setValue
// @license      MIT
// @downloadURL https://update.greasyfork.org/scripts/577261/Shift%20Translator%20Hover%20Toggle%20%2B%20Selection%20Tooltip%20%28Chrome%20Translator%20API%29.user.js
// @updateURL https://update.greasyfork.org/scripts/577261/Shift%20Translator%20Hover%20Toggle%20%2B%20Selection%20Tooltip%20%28Chrome%20Translator%20API%29.meta.js
// ==/UserScript==

(() => {
  'use strict';

  const SOURCE_LANGUAGE = 'en';
  const TARGET_LANGUAGE_CANDIDATES = ['zh-CN', 'zh', 'zh-Hans'];

  const PARAGRAPH_SELECTOR = 'p, li, blockquote';

  const EXCLUDED_SELECTOR = [
    'script',
    'style',
    'noscript',
    'textarea',
    'code',
    'pre',
    'kbd',
    'samp',
    'svg',
    'canvas',
    '[translate="no"]',
    '[data-tm-no-translate="1"]',
  ].join(',');

  const TRANSLATED_COPY_ATTR = 'data-tm-translated-copy';
  const TRANSLATED_FROM_ATTR = 'data-tm-translated-from';
  const SOURCE_ID_ATTR = 'data-tm-source-id';
  const HOVER_CLASS = 'tm-hover-target';

  const MODIFIER_STORAGE_KEY = 'tm_modifier_keys';
  const DEFAULT_MODIFIER_KEYS = ['shift'];

  const translatorCache = new Map();

  let modifierKeys = loadModifierKeys();

  let activeToast = null;
  let toastTimer = null;

  let hoveredParagraph = null;
  let tooltipState = null;

  const style = document.createElement('style');
  style.textContent = `
    @keyframes tm-spin {
      from { transform: rotate(0deg); }
      to { transform: rotate(360deg); }
    }

    .tm-spinner {
      width: 16px;
      height: 16px;
      border-radius: 999px;
      border: 2px solid rgba(0,0,0,.16);
      border-top-color: rgba(0,0,0,.72);
      animation: tm-spin .8s linear infinite;
      flex: 0 0 auto;
    }

    .tm-loading-row {
      display: inline-flex;
      align-items: center;
      gap: 8px;
      padding: 8px 0;
      color: rgba(0,0,0,.62);
      font: 13px/1.4 system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;
    }

    .tm-hover-target {
      outline: 2px solid rgba(60,130,255,.42) !important;
      outline-offset: 3px !important;
      border-radius: 4px;
      background: rgba(60,130,255,.04) !important;
      transition:
        outline-color .12s ease,
        outline-offset .12s ease,
        background-color .12s ease;
    }
  `;
  document.documentElement.appendChild(style);

  function uid() {
    return `tm_${Date.now().toString(36)}_${Math.random().toString(36).slice(2, 8)}`;
  }

  function isElement(value) {
    return value && value.nodeType === Node.ELEMENT_NODE;
  }

  function isParagraphLike(el) {
    return isElement(el) && el.matches(PARAGRAPH_SELECTOR);
  }

  function normalizeModifierKeys(input) {
    if (!input) {
      return [...DEFAULT_MODIFIER_KEYS];
    }

    const allowed = ['shift', 'control', 'command'];

    const parts = input
      .toLowerCase()
      .split('+')
      .map(v => v.trim())
      .filter(Boolean);

    const unique = [...new Set(parts)];
    const valid = unique.filter(v => allowed.includes(v));

    return valid.length ? valid : [...DEFAULT_MODIFIER_KEYS];
  }

  function loadModifierKeys() {
    try {
      const raw = GM_getValue(MODIFIER_STORAGE_KEY, '');
      if (!raw) {
        return [...DEFAULT_MODIFIER_KEYS];
      }
      return normalizeModifierKeys(raw);
    } catch {
      return [...DEFAULT_MODIFIER_KEYS];
    }
  }

  function saveModifierKeys(keys) {
    modifierKeys = normalizeModifierKeys(keys.join('+'));
    GM_setValue(MODIFIER_STORAGE_KEY, modifierKeys.join('+'));
  }

  function isModifierMatch(event) {
    const pressed = [];

    if (event.shiftKey) pressed.push('shift');
    if (event.ctrlKey) pressed.push('control');
    if (event.metaKey) pressed.push('command');

    if (pressed.length !== modifierKeys.length) {
      return false;
    }

    return modifierKeys.every(k => pressed.includes(k));
  }

  function showToast(message, ms = 1600, isError = false) {
    if (!activeToast) {
      activeToast = document.createElement('div');
      activeToast.style.cssText = `
        position:fixed;
        left:16px;
        bottom:16px;
        z-index:2147483647;
        max-width:min(520px,calc(100vw - 32px));
        padding:10px 12px;
        border-radius:10px;
        color:#fff;
        font:13px/1.4 system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;
        box-shadow:0 8px 30px rgba(0,0,0,.28);
        white-space:pre-wrap;
        pointer-events:none;
      `;
      document.documentElement.appendChild(activeToast);
    }

    activeToast.textContent = message;
    activeToast.style.background = isError ? 'rgba(176,0,32,.94)' : 'rgba(20,20,20,.92)';
    activeToast.style.display = 'block';

    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => {
      if (activeToast) {
        activeToast.style.display = 'none';
      }
    }, isError ? 5000 : ms);
  }

  function createLoadingRow(text = 'Translating...') {
    const row = document.createElement('div');
    row.className = 'tm-loading-row';
    row.innerHTML = `
      <span class="tm-spinner"></span>
      <span>${text}</span>
    `;
    return row;
  }

  async function getTranslator() {
    if (!('Translator' in self)) {
      throw new Error('Translator API is not available.');
    }

    for (const targetLanguage of TARGET_LANGUAGE_CANDIDATES) {
      const cacheKey = `${SOURCE_LANGUAGE}->${targetLanguage}`;

      if (translatorCache.has(cacheKey)) {
        return translatorCache.get(cacheKey);
      }

      let availability;
      try {
        availability = await Translator.availability({
          sourceLanguage: SOURCE_LANGUAGE,
          targetLanguage,
        });
      } catch {
        continue;
      }

      if (availability !== 'available' && availability !== 'downloadable') {
        continue;
      }

      const promise = Translator.create({
        sourceLanguage: SOURCE_LANGUAGE,
        targetLanguage,
      });

      translatorCache.set(cacheKey, promise);

      try {
        return await promise;
      } catch {
        translatorCache.delete(cacheKey);
      }
    }

    throw new Error('No supported translator available.');
  }

  async function translatePlainText(text) {
    const translator = await getTranslator();
    return translator.translate(text);
  }

  function getParagraphFromPoint(event) {
    if (typeof document.elementsFromPoint === 'function') {
      const stack = document.elementsFromPoint(event.clientX, event.clientY);

      for (const el of stack) {
        if (!isElement(el)) continue;

        if (el.closest(`[${TRANSLATED_COPY_ATTR}="1"]`)) {
          continue;
        }

        const paragraph = el.closest(PARAGRAPH_SELECTOR);
        if (paragraph) {
          return paragraph;
        }
      }
    }

    return null;
  }

  function setHoveredParagraph(nextParagraph) {
    if (hoveredParagraph === nextParagraph) {
      return;
    }

    if (hoveredParagraph) {
      hoveredParagraph.classList.remove(HOVER_CLASS);
    }

    hoveredParagraph = nextParagraph || null;

    if (hoveredParagraph) {
      hoveredParagraph.classList.add(HOVER_CLASS);
    }
  }

  function stripDuplicateIds(root) {
    if (!root) return;

    if (root.hasAttribute?.('id')) {
      root.removeAttribute('id');
    }

    root.querySelectorAll?.('[id]').forEach(el => el.removeAttribute('id'));
  }

  function collectTranslatableTextNodes(root) {
    const nodes = [];

    const walker = document.createTreeWalker(root, NodeFilter.SHOW_TEXT, {
      acceptNode(node) {
        if (!node?.nodeValue?.trim()) {
          return NodeFilter.FILTER_REJECT;
        }

        const parent = node.parentElement;
        if (!parent) {
          return NodeFilter.FILTER_REJECT;
        }

        if (parent.closest(EXCLUDED_SELECTOR)) {
          return NodeFilter.FILTER_REJECT;
        }

        return NodeFilter.FILTER_ACCEPT;
      },
    });

    let current;
    while ((current = walker.nextNode())) {
      nodes.push(current);
    }

    return nodes;
  }

  async function translateCloneTree(clone) {
    const translator = await getTranslator();
    const textNodes = collectTranslatableTextNodes(clone);

    for (const node of textNodes) {
      const text = node.nodeValue;
      if (!text?.trim()) continue;

      try {
        const translated = await translator.translate(text);
        if (translated) {
          node.nodeValue = translated;
        }
      } catch (err) {
        console.warn(err);
      }
    }
  }

  function findExistingTranslation(original) {
    const sourceId = original.getAttribute(SOURCE_ID_ATTR);
    if (!sourceId) return null;

    const sibling = original.nextElementSibling;

    if (
      sibling &&
      sibling.getAttribute(TRANSLATED_COPY_ATTR) === '1' &&
      sibling.getAttribute(TRANSLATED_FROM_ATTR) === sourceId
    ) {
      return sibling;
    }

    return null;
  }

  async function toggleTranslation(original) {
    let sourceId = original.getAttribute(SOURCE_ID_ATTR);

    if (!sourceId) {
      sourceId = uid();
      original.setAttribute(SOURCE_ID_ATTR, sourceId);
    }

    const existing = findExistingTranslation(original);
    if (existing) {
      existing.remove();
      showToast('Translation hidden.');
      return;
    }

    const loading = createLoadingRow('Translating...');
    original.insertAdjacentElement('afterend', loading);

    const clone = original.cloneNode(true);
    stripDuplicateIds(clone);
    clone.style.opacity = '0.9';
    clone.setAttribute(TRANSLATED_COPY_ATTR, '1');
    clone.setAttribute(TRANSLATED_FROM_ATTR, sourceId);

    try {
      await translateCloneTree(clone);

      if (loading.isConnected) {
        loading.replaceWith(clone);
      }

      showToast('Translation shown.');
    } catch (err) {
      loading.remove();
      showToast(err?.message || 'Translation failed.', 5000, true);
      throw err;
    }
  }

  function getSelectedText() {
    const selection = window.getSelection?.();

    if (selection && !selection.isCollapsed) {
      const text = selection.toString();

      if (text?.trim()) {
        const range = selection.getRangeAt(0);
        return {
          text,
          rect: range.getBoundingClientRect(),
        };
      }
    }

    return null;
  }

  function closeTooltip() {
    if (!tooltipState) return;

    const {
      tooltip,
      onPointerDown,
      onBlur,
      onVisibilityChange,
    } = tooltipState;

    document.removeEventListener('pointerdown', onPointerDown, true);
    window.removeEventListener('blur', onBlur, true);
    document.removeEventListener('visibilitychange', onVisibilityChange, true);

    tooltip.remove();
    tooltipState = null;
  }

  async function openSelectionTooltip(selectionData) {
    if (!selectionData?.text?.trim()) {
      return;
    }

    closeTooltip();

    const tooltip = document.createElement('div');
    tooltip.style.cssText = `
      position:fixed;
      z-index:2147483647;
      max-width:min(520px,calc(100vw - 24px));
      min-width:280px;
      max-height:calc(100vh - 24px);
      overflow:auto;
      background:#fff;
      color:#111;
      border-radius:14px;
      box-shadow:0 16px 48px rgba(0,0,0,.22);
      border:1px solid rgba(0,0,0,.08);
      padding:14px;
      font:14px/1.55 system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;
      word-break:break-word;
      box-sizing:border-box;
      visibility:hidden;
    `;

    const title = document.createElement('div');
    title.style.cssText = `
      font-size:12px;
      font-weight:700;
      text-transform:uppercase;
      letter-spacing:.04em;
      margin-bottom:10px;
      color:rgba(0,0,0,.48);
    `;
    title.textContent = 'Translation';

    const content = document.createElement('div');
    content.style.whiteSpace = 'pre-wrap';
    content.appendChild(createLoadingRow('Translating...'));

    tooltip.appendChild(title);
    tooltip.appendChild(content);
    document.documentElement.appendChild(tooltip);

    const margin = 12;
    const viewportW = window.visualViewport?.width || window.innerWidth;
    const viewportH = window.visualViewport?.height || window.innerHeight;
    const viewportLeft = window.visualViewport?.offsetLeft || 0;
    const viewportTop = window.visualViewport?.offsetTop || 0;

    function positionTooltip() {
      const rect = selectionData.rect;
      const tipRect = tooltip.getBoundingClientRect();

      const tipW = Math.min(tipRect.width, viewportW - margin * 2);
      const tipH = Math.min(tipRect.height, viewportH - margin * 2);

      const spaceBelow = viewportH - (rect.bottom - viewportTop) - margin;
      const spaceAbove = (rect.top - viewportTop) - margin;

      let top;
      if (spaceBelow >= tipH || spaceBelow >= spaceAbove) {
        top = Math.min(rect.bottom + 12, viewportTop + viewportH - tipH - margin);
      } else {
        top = Math.max(viewportTop + margin, rect.top - tipH - 12);
      }

      let left = rect.left - viewportLeft;
      left = Math.min(left, viewportW - tipW - margin);
      left = Math.max(margin, left);

      tooltip.style.left = `${left + viewportLeft}px`;
      tooltip.style.top = `${top}px`;
      tooltip.style.visibility = 'visible';
    }

    requestAnimationFrame(positionTooltip);

    const onPointerDown = (event) => {
      if (!tooltip.contains(event.target)) {
        closeTooltip();
      }
    };

    const onBlur = () => {
      closeTooltip();
    };

    const onVisibilityChange = () => {
      if (document.hidden) {
        closeTooltip();
      }
    };

    document.addEventListener('pointerdown', onPointerDown, true);
    window.addEventListener('blur', onBlur, true);
    document.addEventListener('visibilitychange', onVisibilityChange, true);

    tooltipState = {
      tooltip,
      onPointerDown,
      onBlur,
      onVisibilityChange,
    };

    try {
      const translated = await translatePlainText(selectionData.text);
      if (!tooltipState) return;

      content.textContent = translated || '';

      requestAnimationFrame(() => {
        if (!tooltipState) return;
        positionTooltip();
      });
    } catch (err) {
      console.error(err);
      content.textContent = 'Translation failed.';
      showToast(err?.message || 'Translation failed.', 5000, true);
    }
  }

  function openModifierSettingsModal() {
    const overlay = document.createElement('div');
    overlay.style.cssText = `
      position:fixed;
      inset:0;
      z-index:2147483647;
      background:rgba(0,0,0,.18);
      display:flex;
      align-items:center;
      justify-content:center;
    `;

    const modal = document.createElement('div');
    modal.style.cssText = `
      width:420px;
      background:#fff;
      border-radius:16px;
      padding:20px;
      box-shadow:0 20px 60px rgba(0,0,0,.25);
      font:14px/1.5 system-ui;
    `;

    modal.innerHTML = `
      <div style="font-size:18px;font-weight:700;margin-bottom:12px;">
        Modifier Key Settings
      </div>

      <div style="margin-bottom:12px;color:#666;">
        Allowed: shift / control / command
      </div>

      <input
        id="tm-modifier-input"
        value="${modifierKeys.join('+')}"
        style="
          width:100%;
          padding:10px 12px;
          border-radius:10px;
          border:1px solid rgba(0,0,0,.12);
          box-sizing:border-box;
          font-size:14px;
        "
      />

      <div style="display:flex;justify-content:flex-end;margin-top:16px;gap:8px;">
        <button id="tm-cancel-btn">Cancel</button>
        <button id="tm-save-btn">Save</button>
      </div>
    `;

    overlay.appendChild(modal);
    document.documentElement.appendChild(overlay);

    const close = () => overlay.remove();

    overlay.addEventListener('click', e => {
      if (e.target === overlay) {
        close();
      }
    });

    modal.querySelector('#tm-cancel-btn').addEventListener('click', close);

    modal.querySelector('#tm-save-btn').addEventListener('click', () => {
      const value = modal.querySelector('#tm-modifier-input').value;
      const normalized = normalizeModifierKeys(value);

      saveModifierKeys(normalized);
      showToast(`Modifier updated: ${normalized.join('+')}`);
      close();
    });
  }

  function handlePointerMove(event) {
    const paragraph = getParagraphFromPoint(event);
    setHoveredParagraph(paragraph && isParagraphLike(paragraph) ? paragraph : null);
  }

  async function handleKeyDown(event) {
    const isSettingsShortcut =
      event.code === 'Comma' &&
      ((event.ctrlKey && event.shiftKey) || (event.metaKey && event.shiftKey));

    if (isSettingsShortcut) {
      event.preventDefault();
      event.stopPropagation();
      openModifierSettingsModal();
      return;
    }

    if (!isModifierMatch(event)) {
      return;
    }

    if (event.repeat) {
      return;
    }

    const selectionData = getSelectedText();

    if (selectionData) {
      event.preventDefault();
      event.stopImmediatePropagation();
      await openSelectionTooltip(selectionData);
      return;
    }

    if (!hoveredParagraph || !isParagraphLike(hoveredParagraph)) {
      return;
    }

    event.preventDefault();
    event.stopImmediatePropagation();

    try {
      await toggleTranslation(hoveredParagraph);
    } catch {
      // handled
    }
  }

  document.addEventListener('pointermove', handlePointerMove, true);
  document.addEventListener('keydown', handleKeyDown, true);

  console.log('[TM Translator] Loaded.');
})();
```


## 《Shift Translator Hover Toggle + Selection Tooltip (Chrome Translator API)》 下载链接

<table style="table-layout: fixed;">
<tbody>
<tr>
<td><div style="text-align: center;"><div style="font-weight: bold">Chrome</div><br/><div style="text-align: center;"><img  style="width:50px; height:auto;" src="https://v2fy.com/asset/0i/ChromeAppHeroes/page/001_markdown_here.assets/chromeappheroes-chrome-icon.png"/></div></div></td>
<td><div style="text-align: center;" ><div style="font-weight: bold">Edge</div><br/><div><img style="width:50px; height:auto;" src="https://v2fy.com/asset/0i/ChromeAppHeroes/page/001_markdown_here.assets/chromeappheroes-edge-icon.png"/></div></div></td>
<td><div style="text-align: center;" ><div style="font-weight: bold">FireFox</div><br/><div style="text-align: center;"><img  style="width:50px; height:auto;" src="https://v2fy.com/asset/0i/ChromeAppHeroes/page/001_markdown_here.assets/chromeappheroes-firefox-icon.png"/></div></div></td>
<td><div style="text-align: center;" ><div style="font-weight: bold">离线安装包</div><br/><div style="text-align: center;"><img  style="width:50px; height:auto;" src="https://v2fy.com/asset/0i/ChromeAppHeroes/page/001_markdown_here.assets/chromeappheroes-github-download.png"/></div></div></td>
</tr>
<tr>
<td>
<div style="text-align: center;"><a  href="https://greasyfork.org/zh-CN/scripts/577261-shift-translator-hover-toggle-selection-tooltip-chrome-translator-api">下载链接 / Download link</a></div>
</td>
<td>
<div style="text-align: center;"><a  href="https://greasyfork.org/zh-CN/scripts/577261-shift-translator-hover-toggle-selection-tooltip-chrome-translator-api">下载链接 / Download link</a></div>
</td>
<td>
<div style="text-align: center;"><a  href="https://greasyfork.org/zh-CN/scripts/577261-shift-translator-hover-toggle-selection-tooltip-chrome-translator-api">下载链接 / Download link</a></div>
</td>
<td>
<div style="text-align: center;"><a  href="https://cdn.jsdelivr.net/gh/zhaoolee/ChromeAppHeroes/backup/133-shift-translator-hover-toggle.zip">下载链接 / Download link</a></div>
</td>
</tbody>
</table>



## 小结

大模型时代，垃圾信息满天飞，假新闻，假图片的制作成本越来越低，作为一个开发者，能从海外新闻找信息来源，读流畅阅读英文原版文档，可以让我们的认知更符合现实，不让道心蒙尘。



## 写在最后(我需要你的支持) / At the end (I need your support)

- 本文属于**Chrome插件英雄榜** 项目的一部分, 项目Github地址: [https://github.com/zhaoolee/ChromeAppHeroes](https://github.com/zhaoolee/ChromeAppHeroes)


- This article is part of the **ChromeAppHeroes** project. Github link : [https://github.com/zhaoolee/ChromeAppHeroes](https://github.com/zhaoolee/ChromeAppHeroes) 

- **Chrome插件英雄榜**, 为优秀的Chrome插件写一本中文说明书, 让Chrome插件英雄们造福人类, 如果你喜欢这个项目, 希望你能为本项目添加一颗 🌟星.

- ChromeAppHeroes, Write a Chinese manual for the excellent Chrome plugin, let the Chrome plugin heroes benefit the human, If you like this project, I hope you can add a star 🌟 to this project.
