# 历史复盘批量导出 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在"历史复盘"区域新增批量选择与导出功能，支持 TXT / HTML / 打印为PDF 三种格式，打包为 ZIP 下载。

**Architecture:** 所有改动集中在 `index.html` 单文件内。新增 JSZip CDN 脚本；在 app 对象加入选中状态 Set；把现有导出函数的内容生成逻辑抽取为独立 builder 函数；`renderReviewHistory()` 增加复选框渲染并调用新增的 `updateBatchBar()`；新增 `batchExportZip()` 异步函数；在已有事件绑定区补充新 UI 的事件监听。

**Tech Stack:** 原生 JS、HTML、JSZip 3.10.1（CDN）

---

## 文件变动一览

- Modify: `index.html:20` — 紧后插入 JSZip CDN script 标签
- Modify: `index.html:848` — app 对象加 `_selectedReviews: new Set()`
- Modify: `index.html:754-762` — 历史复盘卡片 HTML，加入批量操作栏
- Modify: `index.html:2015-2060` — 重写 `renderReviewHistory()`，加复选框、调用 `updateBatchBar`
- Modify: `index.html:2062-2095` — 重写 `exportReviewTxt()`，委托给 `buildReviewTxt()`
- Modify: `index.html:2097-2127` — 重写 `exportReviewPdf()`，委托给 `buildReviewHtml()`
- Modify: `index.html:2478-2484` — filter tab 点击时清空选中 Set
- Modify: `index.html:2628-2638` — `document.addEventListener('change', ...)` 增加复选框分支
- Insert before line 2062: `buildReviewTxt(review)` 函数
- Insert before line 2062: `buildReviewHtml(review)` 函数
- Insert before line 2062: `updateBatchBar(entries)` 函数
- Insert before line 2062: `batchExportZip()` 异步函数
- Insert in `bindEvents()`: `batchExportZipBtn`、`selectAllReviews`、格式 radio 的事件绑定

---

### Task 1: 引入 JSZip CDN

**Files:**
- Modify: `index.html:20`

- [ ] **Step 1: 在 Chart.js 内联脚本之后（第 20 行 `</script>` 后）插入 JSZip script 标签**

在 `index.html` 第 20 行找到：
```
</script>
```
在它的**下一行**（即第 21 行）插入：
```html
<script src="https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js"></script>
```

- [ ] **Step 2: 验证插入位置正确**

在终端运行：
```bash
node -e "const fs=require('fs');const lines=fs.readFileSync('index.html','utf8').split('\n');console.log(lines.slice(18,24).join('\n'));"
```
预期输出第 21 行包含 `jszip`。

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add JSZip CDN for batch review export"
```

---

### Task 2: 初始化选中状态 `_selectedReviews`

**Files:**
- Modify: `index.html:848`

- [ ] **Step 1: 在 app 对象中加入 `_selectedReviews`**

找到 `index.html` 中 app 对象定义（约第 848 行）：
```js
      _expandedReviewKey: '',
      _reviewFilter: 'all',
```
替换为：
```js
      _expandedReviewKey: '',
      _reviewFilter: 'all',
      _selectedReviews: new Set(),
```

- [ ] **Step 2: 语法检查**

```bash
node -e "
const fs = require('fs');
const src = fs.readFileSync('index.html', 'utf8');
const m = src.match(/<script>([\s\S]*?)<\/script>\s*<script src/);
if (!m) { console.log('script block not found'); process.exit(1); }
try { new (require('vm').Script)(m[1]); console.log('OK'); } catch(e) { console.error(e.message); process.exit(1); }
"
```
预期输出：`OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: init _selectedReviews Set on app object"
```

---

### Task 3: 更新历史复盘卡片 HTML，增加批量操作栏

**Files:**
- Modify: `index.html:754-762`

- [ ] **Step 1: 替换历史复盘卡片 HTML**

找到（约第 754-762 行）：
```html
          <div class="card">
            <h3>历史复盘</h3>
            <div class="tabs" id="reviewFilterTabs">
              <button class="tab-chip active" data-review-filter="all">全部</button>
              <button class="tab-chip" data-review-filter="daily">每日复盘</button>
              <button class="tab-chip" data-review-filter="weekly">每周复盘</button>
            </div>
            <div class="list" id="reviewHistory"></div>
          </div>
```
替换为：
```html
          <div class="card">
            <h3>历史复盘</h3>
            <div class="tabs" id="reviewFilterTabs">
              <button class="tab-chip active" data-review-filter="all">全部</button>
              <button class="tab-chip" data-review-filter="daily">每日复盘</button>
              <button class="tab-chip" data-review-filter="weekly">每周复盘</button>
            </div>
            <div id="batchExportBar" style="display:flex;align-items:flex-start;justify-content:space-between;gap:8px;margin:10px 0 4px;flex-wrap:wrap;">
              <div style="display:flex;align-items:center;gap:8px;">
                <label style="display:flex;align-items:center;gap:6px;cursor:pointer;font-size:14px;">
                  <input type="checkbox" id="selectAllReviews" />全选
                </label>
                <span id="selectedReviewCount" class="small muted">已选 0 条</span>
              </div>
              <div style="display:flex;flex-direction:column;gap:4px;align-items:flex-end;">
                <div style="display:flex;gap:10px;align-items:center;flex-wrap:wrap;">
                  <label style="font-size:13px;cursor:pointer;"><input type="radio" name="exportFormat" value="txt" checked /> TXT</label>
                  <label style="font-size:13px;cursor:pointer;"><input type="radio" name="exportFormat" value="html" /> HTML</label>
                  <label style="font-size:13px;cursor:pointer;"><input type="radio" name="exportFormat" value="pdf" /> 打印为PDF</label>
                  <button class="btn" id="batchExportZipBtn" disabled>导出 ZIP</button>
                </div>
                <span id="pdfExportNote" class="small muted" style="display:none;max-width:300px;text-align:right;">
                  选择「打印为PDF」时，将导出带打印样式的 HTML 文件，用浏览器打开后按 Ctrl+P（Mac: ⌘P）另存为 PDF。
                </span>
              </div>
            </div>
            <div class="list" id="reviewHistory"></div>
          </div>
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add batch export toolbar to review history card"
```

---

### Task 4: 抽取内容生成函数 `buildReviewTxt` 和 `buildReviewHtml`

**Files:**
- Insert before `exportReviewTxt` (约第 2062 行)

- [ ] **Step 1: 在 `exportReviewTxt` 函数定义之前插入两个 builder 函数**

找到（约第 2062 行）：
```js
    function exportReviewTxt() {
```
在它的**上方**插入：
```js
    function buildReviewTxt(review) {
      const date = review.date || todayISO();
      const typeLabel = review.type === 'weekly' ? '每周复盘' : '每日复盘';
      const sr = review.subjectReviews || [];
      let text = `========================================\n`;
      text += `  ${date} ${typeLabel}\n`;
      text += `========================================\n\n`;
      text += `【完成情况】\n${review.done || '无'}\n\n`;
      text += `【困难与不足】\n${review.problems || '无'}\n\n`;
      text += `【改进措施】\n${review.actions || '无'}\n\n`;
      text += `【明日重点】\n${review.tomorrow || '无'}\n\n`;
      if (sr.length) {
        text += `----------------------------------------\n`;
        text += `  科目复盘\n`;
        text += `----------------------------------------\n\n`;
        sr.forEach((srb) => {
          text += `【${subjectName(srb.subjectId)}${srb.questionType ? ' - ' + srb.questionType : ''}】\n`;
          text += `${srb.content || '无内容'}\n\n`;
        });
      }
      text += `========================================\n`;
      text += `  导出时间：${new Date().toLocaleString()}\n`;
      text += `========================================\n`;
      return text;
    }

    function buildReviewHtml(review) {
      const date = review.date || todayISO();
      const typeLabel = review.type === 'weekly' ? '每周复盘' : '每日复盘';
      const sr = review.subjectReviews || [];
      let body = `<div style="font-family:sans-serif;max-width:800px;margin:0 auto;padding:20px;">
        <h2 style="text-align:center;">${escapeHtml(date)} ${typeLabel}</h2>
        <hr/>
        <h3>完成情况</h3><p>${escapeHtml(review.done || '无')}</p>
        <h3>困难与不足</h3><p>${escapeHtml(review.problems || '无')}</p>
        <h3>改进措施</h3><p>${escapeHtml(review.actions || '无')}</p>
        <h3>明日重点</h3><p>${escapeHtml(review.tomorrow || '无')}</p>`;
      if (sr.length) {
        body += `<hr/><h2 style="text-align:center;">科目复盘</h2>`;
        sr.forEach((srb) => {
          body += `<h3>${escapeHtml(subjectName(srb.subjectId))}${srb.questionType ? ' - ' + escapeHtml(srb.questionType) : ''}</h3>
            <p>${escapeHtml(srb.content || '无内容')}</p>`;
        });
      }
      body += `<hr/><p style="text-align:center;color:#999;font-size:12px;">导出时间：${new Date().toLocaleString()}</p></div>`;
      return `<!DOCTYPE html><html><head><meta charset="utf-8"/><title>复盘_${escapeHtml(date)}</title><style>
        body { font-family: -apple-system, "PingFang SC", sans-serif; max-width: 800px; margin: 20px auto; padding: 20px; color: #1e293b; line-height: 1.7; }
        h2 { color: #326ca9; } h3 { color: #4f8fd8; } hr { border: none; border-top: 1px solid #e2e8f0; margin: 20px 0; }
      </style></head><body>${body}</body></html>`;
    }

```

- [ ] **Step 2: 重写 `exportReviewTxt()` 委托给 `buildReviewTxt`**

找到并完整替换 `exportReviewTxt` 函数：
```js
    function exportReviewTxt() {
      const key = currentReviewKey();
      const review = app.state.reviews[key];
      if (!review) { alert('请先保存当前复盘再导出。'); return; }
      const date = review.date || todayISO();
      const typeLabel = review.type === 'weekly' ? '每周复盘' : '每日复盘';
      const sr = review.subjectReviews || [];
      let text = `========================================\n`;
      text += `  ${date} ${typeLabel}\n`;
      text += `========================================\n\n`;
      text += `【完成情况】\n${review.done || '无'}\n\n`;
      text += `【困难与不足】\n${review.problems || '无'}\n\n`;
      text += `【改进措施】\n${review.actions || '无'}\n\n`;
      text += `【明日重点】\n${review.tomorrow || '无'}\n\n`;
      if (sr.length) {
        text += `----------------------------------------\n`;
        text += `  科目复盘\n`;
        text += `----------------------------------------\n\n`;
        sr.forEach((srb) => {
          text += `【${subjectName(srb.subjectId)}${srb.questionType ? ' - ' + srb.questionType : ''}】\n`;
          text += `${srb.content || '无内容'}\n\n`;
        });
      }
      text += `========================================\n`;
      text += `  导出时间：${new Date().toLocaleString()}\n`;
      text += `========================================\n`;
      const blob = new Blob([text], { type: 'text/plain;charset=utf-8' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `复盘_${date}_${review.type}.txt`;
      a.click();
      URL.revokeObjectURL(url);
    }
```
替换为：
```js
    function exportReviewTxt() {
      const key = currentReviewKey();
      const review = app.state.reviews[key];
      if (!review) { alert('请先保存当前复盘再导出。'); return; }
      const text = buildReviewTxt(review);
      const blob = new Blob([text], { type: 'text/plain;charset=utf-8' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `复盘_${review.date || todayISO()}_${review.type}.txt`;
      a.click();
      URL.revokeObjectURL(url);
    }
```

- [ ] **Step 3: 重写 `exportReviewPdf()` 委托给 `buildReviewHtml`**

找到并完整替换 `exportReviewPdf` 函数：
```js
    function exportReviewPdf() {
      const key = currentReviewKey();
      const review = app.state.reviews[key];
      if (!review) { alert('请先保存当前复盘再导出。'); return; }
      const date = review.date || todayISO();
      const typeLabel = review.type === 'weekly' ? '每周复盘' : '每日复盘';
      const sr = review.subjectReviews || [];
      let html = `<div style="font-family:sans-serif;max-width:800px;margin:0 auto;padding:20px;">
        <h2 style="text-align:center;">${date} ${typeLabel}</h2>
        <hr/>
        <h3>完成情况</h3><p>${escapeHtml(review.done || '无')}</p>
        <h3>困难与不足</h3><p>${escapeHtml(review.problems || '无')}</p>
        <h3>改进措施</h3><p>${escapeHtml(review.actions || '无')}</p>
        <h3>明日重点</h3><p>${escapeHtml(review.tomorrow || '无')}</p>`;
      if (sr.length) {
        html += `<hr/><h2 style="text-align:center;">科目复盘</h2>`;
        sr.forEach((srb) => {
          html += `<h3>${escapeHtml(subjectName(srb.subjectId))}${srb.questionType ? ' - ' + escapeHtml(srb.questionType) : ''}</h3>
            <p>${escapeHtml(srb.content || '无内容')}</p>`;
        });
      }
      html += `<hr/><p style="text-align:center;color:#999;font-size:12px;">导出时间：${new Date().toLocaleString()}</p></div>`;
      const win = window.open('', '_blank');
      if (!win) { alert('请允许弹出窗口以导出 PDF。'); return; }
      win.document.write(`<!DOCTYPE html><html><head><meta charset="utf-8"/><title>复盘_${date}</title><style>
        body { font-family: -apple-system, "PingFang SC", sans-serif; max-width: 800px; margin: 20px auto; padding: 20px; color: #1e293b; line-height: 1.7; }
        h2 { color: #326ca9; } h3 { color: #4f8fd8; } hr { border: none; border-top: 1px solid #e2e8f0; margin: 20px 0; }
      </style></head><body>${html}</body></html>`);
      win.document.close();
      setTimeout(() => { win.print(); }, 300);
    }
```
替换为：
```js
    function exportReviewPdf() {
      const key = currentReviewKey();
      const review = app.state.reviews[key];
      if (!review) { alert('请先保存当前复盘再导出。'); return; }
      const html = buildReviewHtml(review);
      const win = window.open('', '_blank');
      if (!win) { alert('请允许弹出窗口以导出 PDF。'); return; }
      win.document.write(html);
      win.document.close();
      setTimeout(() => { win.print(); }, 300);
    }
```

- [ ] **Step 4: 语法检查**

```bash
node -e "
const fs = require('fs');
const src = fs.readFileSync('index.html', 'utf8');
const scripts = [...src.matchAll(/<script>([\s\S]*?)<\/script>/g)];
const mainScript = scripts.find(m => m[1].includes('function buildReviewTxt'));
if (!mainScript) { console.error('main script not found'); process.exit(1); }
try { new (require('vm').Script)(mainScript[1]); console.log('OK'); } catch(e) { console.error(e.message); process.exit(1); }
"
```
预期输出：`OK`

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "refactor: extract buildReviewTxt/buildReviewHtml, delegate from exportReviewTxt/Pdf"
```

---

### Task 5: 新增 `updateBatchBar` 和 `batchExportZip` 函数

**Files:**
- Insert before `exportReviewTxt` (after Task 4 inserts, 即在两个 builder 函数下方)

- [ ] **Step 1: 在 `buildReviewHtml` 函数之后、`exportReviewTxt` 之前插入 `updateBatchBar` 和 `batchExportZip`**

找到紧接在 `buildReviewHtml` 函数结束 `}` 后、`exportReviewTxt` 函数开始前的空行，插入：
```js
    function updateBatchBar(entries) {
      const sel = app._selectedReviews;
      const keys = entries.map((r) => reviewKey(r.date, r.type));
      const selectedInView = keys.filter((k) => sel.has(k));
      $('selectedReviewCount').textContent = `已选 ${sel.size} 条`;
      $('batchExportZipBtn').disabled = sel.size === 0;
      const cb = $('selectAllReviews');
      if (!keys.length) {
        cb.checked = false;
        cb.indeterminate = false;
      } else if (selectedInView.length === keys.length) {
        cb.checked = true;
        cb.indeterminate = false;
      } else if (selectedInView.length > 0) {
        cb.checked = false;
        cb.indeterminate = true;
      } else {
        cb.checked = false;
        cb.indeterminate = false;
      }
    }

    async function batchExportZip() {
      if (typeof JSZip === 'undefined') {
        alert('JSZip 加载失败，请检查网络连接后重试。');
        return;
      }
      const keys = [...app._selectedReviews];
      if (!keys.length) return;
      const format = document.querySelector('input[name="exportFormat"]:checked')?.value || 'txt';
      const zip = new JSZip();
      keys.forEach((key) => {
        const review = app.state.reviews[key];
        if (!review) return;
        const date = review.date || todayISO();
        const baseName = `复盘_${date}_${review.type}`;
        if (format === 'txt') {
          zip.file(`${baseName}.txt`, buildReviewTxt(review));
        } else if (format === 'html') {
          zip.file(`${baseName}.html`, buildReviewHtml(review));
        } else {
          zip.file(`pdf/${baseName}.html`, buildReviewHtml(review));
        }
      });
      const dateStr = todayISO().replace(/-/g, '');
      const blob = await zip.generateAsync({ type: 'blob' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `复盘导出_${keys.length}条_${dateStr}.zip`;
      a.click();
      URL.revokeObjectURL(url);
    }

```

- [ ] **Step 2: 语法检查**

```bash
node -e "
const fs = require('fs');
const src = fs.readFileSync('index.html', 'utf8');
const scripts = [...src.matchAll(/<script>([\s\S]*?)<\/script>/g)];
const mainScript = scripts.find(m => m[1].includes('function batchExportZip'));
if (!mainScript) { console.error('batchExportZip not found'); process.exit(1); }
try { new (require('vm').Script)(mainScript[1]); console.log('OK'); } catch(e) { console.error(e.message); process.exit(1); }
"
```
预期输出：`OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add updateBatchBar and batchExportZip functions"
```

---

### Task 6: 更新 `renderReviewHistory()` 加入复选框

**Files:**
- Modify: `index.html` — `renderReviewHistory` 函数

- [ ] **Step 1: 完整替换 `renderReviewHistory` 函数**

找到并替换整个 `renderReviewHistory` 函数（从 `function renderReviewHistory() {` 到其对应的闭合 `}`）：

旧函数（从 `function renderReviewHistory()` 开始到第一个独立 `}` 结束，约 45 行）：
```js
    function renderReviewHistory() {
      const filter = app._reviewFilter || 'all';
      const entries = Object.values(app.state.reviews)
        .filter((r) => filter === 'all' || r.type === filter)
        .sort((a, b) => `${b.date}_${b.type}`.localeCompare(`${a.date}_${a.type}`));
      const container = $('reviewHistory');
      if (!entries.length) {
        container.innerHTML = '<div class="muted small">暂无历史复盘</div>';
        return;
      }
      const expandedKey = app._expandedReviewKey;
      container.innerHTML = entries.map((r) => {
        const key = reviewKey(r.date, r.type);
        const isExpanded = expandedKey === key;
        const sr = r.subjectReviews || [];
        return `
          <div class="item" style="margin-bottom:8px;">
            <div class="item-main">
              <div class="item-title">${escapeHtml(r.date)} · ${r.type === 'weekly' ? '每周复盘' : '每日复盘'}</div>
              <div class="small muted">${escapeHtml((r.done || '未填写完成情况').slice(0, 70))}</div>
              ${sr.length ? `<div class="small muted">${sr.length} 个科目复盘<\div>` : ''}
            </div>
            <div class="item-actions">
              <button class="btn" data-review-load="${escapeHtml(r.date)}" data-review-type="${escapeHtml(r.type)}">查看</button>
              <button class="btn" data-review-expand="${escapeHtml(key)}">${isExpanded ? '收起' : '展开'}</button>
              <button class="btn danger" data-review-delete="${escapeHtml(key)}">删除</button>
            </div>
            ${isExpanded ? `
              <div style="margin-top:8px;padding:8px;background:rgba(255,255,255,0.4);border-radius:12px;width:100%;">
                <div class="small" style="margin-bottom:4px;"><strong>完成情况：</strong>${escapeHtml(r.done || '无')}</div>
                <div class="small" style="margin-bottom:4px;"><strong>困难不足：</strong>${escapeHtml(r.problems || '无')}</div>
                <div class="small" style="margin-bottom:4px;"><strong>改进措施：</strong>${escapeHtml(r.actions || '无')}</div>
                <div class="small" style="margin-bottom:4px;"><strong>明日重点：</strong>${escapeHtml(r.tomorrow || '无')}</div>
                ${sr.length ? '<div style="margin-top:6px;"><strong>科目复盘：</strong></div>' : ''}
                ${sr.map((srb) => `
                  <div class="subject-review-block" style="margin-top:4px;">
                    <div class="small"><strong>${escapeHtml(subjectName(srb.subjectId))}${srb.questionType ? ' - ' + escapeHtml(srb.questionType) : ''}</strong></div>
                    <div class="small">${escapeHtml(srb.content || '无内容')}</div>
                  </div>
                `).join('')}
              </div>
            ` : ''}
          </div>
        `;
      }).join('');
    }
```
替换为（注意旧代码中 `<\div>` 是错误写法，新代码同时修正）：
```js
    function renderReviewHistory() {
      const filter = app._reviewFilter || 'all';
      const entries = Object.values(app.state.reviews)
        .filter((r) => filter === 'all' || r.type === filter)
        .sort((a, b) => `${b.date}_${b.type}`.localeCompare(`${a.date}_${a.type}`));
      const container = $('reviewHistory');
      if (!entries.length) {
        container.innerHTML = '<div class="muted small">暂无历史复盘</div>';
        updateBatchBar([]);
        return;
      }
      const expandedKey = app._expandedReviewKey;
      container.innerHTML = entries.map((r) => {
        const key = reviewKey(r.date, r.type);
        const isExpanded = expandedKey === key;
        const sr = r.subjectReviews || [];
        const checked = app._selectedReviews.has(key) ? 'checked' : '';
        return `
          <div class="item" style="margin-bottom:8px;">
            <input type="checkbox" class="review-select-cb" data-review-key="${escapeHtml(key)}" ${checked} style="flex-shrink:0;align-self:center;margin-right:4px;" />
            <div class="item-main">
              <div class="item-title">${escapeHtml(r.date)} · ${r.type === 'weekly' ? '每周复盘' : '每日复盘'}</div>
              <div class="small muted">${escapeHtml((r.done || '未填写完成情况').slice(0, 70))}</div>
              ${sr.length ? `<div class="small muted">${sr.length} 个科目复盘</div>` : ''}
            </div>
            <div class="item-actions">
              <button class="btn" data-review-load="${escapeHtml(r.date)}" data-review-type="${escapeHtml(r.type)}">查看</button>
              <button class="btn" data-review-expand="${escapeHtml(key)}">${isExpanded ? '收起' : '展开'}</button>
              <button class="btn danger" data-review-delete="${escapeHtml(key)}">删除</button>
            </div>
            ${isExpanded ? `
              <div style="margin-top:8px;padding:8px;background:rgba(255,255,255,0.4);border-radius:12px;width:100%;">
                <div class="small" style="margin-bottom:4px;"><strong>完成情况：</strong>${escapeHtml(r.done || '无')}</div>
                <div class="small" style="margin-bottom:4px;"><strong>困难不足：</strong>${escapeHtml(r.problems || '无')}</div>
                <div class="small" style="margin-bottom:4px;"><strong>改进措施：</strong>${escapeHtml(r.actions || '无')}</div>
                <div class="small" style="margin-bottom:4px;"><strong>明日重点：</strong>${escapeHtml(r.tomorrow || '无')}</div>
                ${sr.length ? '<div style="margin-top:6px;"><strong>科目复盘：</strong></div>' : ''}
                ${sr.map((srb) => `
                  <div class="subject-review-block" style="margin-top:4px;">
                    <div class="small"><strong>${escapeHtml(subjectName(srb.subjectId))}${srb.questionType ? ' - ' + escapeHtml(srb.questionType) : ''}</strong></div>
                    <div class="small">${escapeHtml(srb.content || '无内容')}</div>
                  </div>
                `).join('')}
              </div>
            ` : ''}
          </div>
        `;
      }).join('');
      updateBatchBar(entries);
    }
```

- [ ] **Step 2: 语法检查**

```bash
node -e "
const fs = require('fs');
const src = fs.readFileSync('index.html', 'utf8');
const scripts = [...src.matchAll(/<script>([\s\S]*?)<\/script>/g)];
const mainScript = scripts.find(m => m[1].includes('review-select-cb'));
if (!mainScript) { console.error('review-select-cb not found'); process.exit(1); }
try { new (require('vm').Script)(mainScript[1]); console.log('OK'); } catch(e) { console.error(e.message); process.exit(1); }
"
```
预期：`OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add checkboxes to review history items"
```

---

### Task 7: 绑定所有新增事件

**Files:**
- Modify: `index.html` — `bindEvents()` 函数内的事件绑定区，以及 `document.addEventListener('change', ...)` 处理器

- [ ] **Step 1: filter tab 点击时清空选中状态**

找到（约 2478 行）：
```js
      $('reviewFilterTabs').addEventListener('click', (event) => {
        const btn = event.target.closest('[data-review-filter]');
        if (!btn) return;
        $('reviewFilterTabs').querySelectorAll('.tab-chip').forEach((b) => b.classList.toggle('active', b === btn));
        app._reviewFilter = btn.dataset.reviewFilter;
        renderReviewHistory();
      });
```
替换为：
```js
      $('reviewFilterTabs').addEventListener('click', (event) => {
        const btn = event.target.closest('[data-review-filter]');
        if (!btn) return;
        $('reviewFilterTabs').querySelectorAll('.tab-chip').forEach((b) => b.classList.toggle('active', b === btn));
        app._reviewFilter = btn.dataset.reviewFilter;
        app._selectedReviews.clear();
        renderReviewHistory();
      });
```

- [ ] **Step 2: 在 `bindEvents()` 中绑定批量操作区事件**

找到（约 2485 行）：
```js
      $('saveSubjectBtn').addEventListener('click', saveSubject);
```
在它**上方**插入：
```js
      $('batchExportZipBtn').addEventListener('click', batchExportZip);
      $('selectAllReviews').addEventListener('change', () => {
        const filter = app._reviewFilter || 'all';
        const entries = Object.values(app.state.reviews).filter((r) => filter === 'all' || r.type === filter);
        if ($('selectAllReviews').checked) {
          entries.forEach((r) => app._selectedReviews.add(reviewKey(r.date, r.type)));
        } else {
          entries.forEach((r) => app._selectedReviews.delete(reviewKey(r.date, r.type)));
        }
        renderReviewHistory();
      });
      document.querySelectorAll('input[name="exportFormat"]').forEach((radio) => {
        radio.addEventListener('change', () => {
          const v = document.querySelector('input[name="exportFormat"]:checked')?.value;
          $('pdfExportNote').style.display = v === 'pdf' ? 'block' : 'none';
        });
      });
```

- [ ] **Step 3: 在 `document.addEventListener('change', ...)` 处理器中加入复选框分支**

找到（约 2628 行）：
```js
      document.addEventListener('change', (event) => {
        const toggle = event.target.closest('[data-plan-toggle]');
```
替换为：
```js
      document.addEventListener('change', (event) => {
        const reviewCb = event.target.closest('.review-select-cb');
        if (reviewCb) {
          const key = reviewCb.dataset.reviewKey;
          if (reviewCb.checked) app._selectedReviews.add(key);
          else app._selectedReviews.delete(key);
          const filter = app._reviewFilter || 'all';
          const entries = Object.values(app.state.reviews)
            .filter((r) => filter === 'all' || r.type === filter)
            .sort((a, b) => `${b.date}_${b.type}`.localeCompare(`${a.date}_${a.type}`));
          updateBatchBar(entries);
          return;
        }
        const toggle = event.target.closest('[data-plan-toggle]');
```

- [ ] **Step 4: 语法检查**

```bash
node -e "
const fs = require('fs');
const src = fs.readFileSync('index.html', 'utf8');
const scripts = [...src.matchAll(/<script>([\s\S]*?)<\/script>/g)];
const mainScript = scripts.find(m => m[1].includes('batchExportZip'));
if (!mainScript) { console.error('not found'); process.exit(1); }
try { new (require('vm').Script)(mainScript[1]); console.log('OK'); } catch(e) { console.error(e.message); process.exit(1); }
"
```
预期：`OK`

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: wire up batch export events (select-all, checkboxes, format radio, export button)"
```

---

### Task 8: 手动验证

- [ ] **Step 1: 用浏览器打开 `index.html`，切换到"复盘"页**

确认"历史复盘"卡片中：
- 筛选 tab 正常（全部 / 每日复盘 / 每周复盘）
- 批量操作栏出现：全选 checkbox、已选 0 条、TXT/HTML/打印为PDF radio、导出 ZIP 按钮（灰色）

- [ ] **Step 2: 验证选中逻辑**

1. 勾选一条记录 → "已选 N 条" 数字更新，导出 ZIP 按钮变为可点击
2. 点击"全选" → 所有记录被勾选
3. 切换筛选 tab → 选中状态清零，导出 ZIP 按钮变灰

- [ ] **Step 3: 验证 PDF 说明小字**

选中"打印为PDF" radio → 出现说明小字；切换回 TXT/HTML → 小字隐藏

- [ ] **Step 4: 导出 TXT ZIP**

勾选若干记录，选 TXT，点击"导出 ZIP"，确认下载的 ZIP 中包含对应 `.txt` 文件，文件名格式为 `复盘_YYYY-MM-DD_daily.txt`

- [ ] **Step 5: 导出 HTML ZIP**

选 HTML 格式导出，打开 ZIP 中的 `.html` 文件，确认内容格式正确

- [ ] **Step 6: 导出打印为PDF ZIP**

选"打印为PDF"导出，确认 ZIP 中有 `pdf/复盘_*.html` 文件，打开后 Ctrl+P 可正常打印

- [ ] **Step 7: 验证原有单条导出功能未受影响**

在复盘表单区点击"导出 TXT"和"导出 PDF"，确认仍正常工作

- [ ] **Step 8: 确认控制台无新增错误**

打开浏览器开发者工具 Console，确认无报错

- [ ] **Step 9: 最终 commit**

```bash
git add index.html
git commit -m "test: verify batch export feature works end-to-end"
```

---

## 注意事项

- JSZip 需要网络加载，离线时"打印为PDF"和 HTML 格式仍正常（纯文本生成），TXT 同样正常；只有点击"导出 ZIP"按钮时才依赖 JSZip，失败有 alert 兜底。
- `_selectedReviews` 是 `Set`（引用类型），不随 `saveState()` 持久化，刷新页面后选中状态自动清零，符合预期。
- 旧代码中 `<\div>` （第 2035 行）是一处 bug，新代码已修正为 `</div>`。
