# Sequential Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the existing single-file study app with plan question types and a monthly calendar, subject/type review blocks with export, and mistake accuracy statistics with visualization.

**Architecture:** Keep all code in `index.html`, following the existing native HTML/CSS/JavaScript structure. Reuse current helpers such as `subjectById()`, `subjectName()`, `populateSubjectSelect()`, `getSubjectTypes()`, `populateQuestionTypeSelect()`, `saveState()`, and `renderAll()` instead of adding new abstractions or files. New data remains in the existing `civilServiceExamReviewPomodoroStateV1` localStorage object and is normalized at read/render boundaries for backward compatibility.

**Tech Stack:** Native HTML, CSS, JavaScript, browser `localStorage`, Chart.js CDN. No backend, build tool, package manager, framework, database, or third-party PDF library.

---

## Scope and File Structure

**Modify only:**
- `index.html`

**Do not create:**
- New app source files
- New dependencies
- Backend/API/database files

**Plan constraints:**
- Do not commit or push automatically. The project instructions require explicit user authorization before git commits or pushes.
- After each module, run JavaScript syntax verification by extracting inline script and checking it with Node `vm.Script`.
- Browser/manual checks are still required before claiming completion.

---

## Task1: Add plan question type support

**Files:**
- Modify: `index.html` plan modal, plan list rendering, state normalization helpers

- [ ] **Step1: Normalize old plan records in `loadState()`**

Add a helper near `normalizeSubject()`:

```js
function normalizePlan(plan) {
 return {
 ...plan,
 questionType: plan.questionType || ''
 };
}
```

Change `loadState()` plans line from:

```js
plans: Array.isArray(parsed.plans) ? parsed.plans : [],
```

to:

```js
plans: Array.isArray(parsed.plans) ? parsed.plans.map(normalizePlan) : [],
```

Expected behavior: old plans without `questionType` still load and display `无`.

- [ ] **Step2: Show plan type tag in `renderPlanList()`**

In each plan card tag row, add a type tag after `subjectPill(plan.subjectId)`:

```js
<span class="tag">题型：${escapeHtml(plan.questionType || '无')}</span>
```

The existing tag row should include subject, question type, cadence, estimated Pomodoros, and due date.

- [ ] **Step3: Add type select to `openPlanModal()`**

Extend the default plan object:

```js
const plan = app.state.plans.find((p) => p.id === planId) || { title: '', subjectId: '', questionType: '', estimatedPomodoros:1, dueDate: todayISO(), completed: false, cadence: 'daily' };
```

Add this field after the subject select:

```html
<div class="field"><label for="planTypeInput">题型</label><select id="planTypeInput"></select></div>
```

After populating `planSubjectInput`, populate the type select:

```js
function refreshPlanTypeSelect(preserveStored = false) {
 populateQuestionTypeSelect($('planTypeInput'), $('planSubjectInput').value, { includeEmpty: true, emptyText: '无' });
 if (preserveStored && plan.questionType && ![...$('planTypeInput').options].some((opt) => opt.value === plan.questionType)) {
 const option = document.createElement('option');
 option.value = plan.questionType;
 option.textContent = `${plan.questionType}（已保留）`;
 $('planTypeInput').appendChild(option);
 }
 if (preserveStored && plan.questionType) $('planTypeInput').value = plan.questionType;
}
refreshPlanTypeSelect(true);
$('planSubjectInput').addEventListener('change', () => refreshPlanTypeSelect(false));
```

- [ ] **Step4: Save plan `questionType`**

Add to the plan payload:

```js
questionType: $('planTypeInput').value,
```

Expected behavior: new and edited plans persist the selected type or empty string.

- [ ] **Step5: Verify Task1**

Run syntax check:

```bash
node -e "const fs=require('fs');const vm=require('vm');const html=fs.readFileSync('index.html','utf8');const script=html.match(/<script>([\s\S]*)<\/script>/)[1];new vm.Script(script);console.log('Inline JavaScript syntax OK');"
```

Expected output:

```text
Inline JavaScript syntax OK
```

Manual browser checks:
- Open `index.html` directly in the browser.
- Create a plan with a subject and type.
- Create a plan with no type and confirm it shows `题型：无`.
- Edit a plan and confirm the saved type is preserved.

---

## Task2: Add monthly plan calendar and selected-date quick toggles

**Files:**
- Modify: `index.html` plans section HTML, CSS, calendar helpers, event binding

- [ ] **Step1: Add calendar state to `app`**

In the `app` object, add:

```js
planCalendarMonth: todayISO().slice(0,7),
selectedPlanDate: todayISO(),
```

Expected behavior: calendar defaults to the current month and today is selected.

- [ ] **Step2: Add plan calendar HTML**

In `section-plans`, after the top `学习计划` card and before the today grids, insert:

```html
<div class="card" style="margin-top:16px;">
 <div class="btn-row" style="justify-content:space-between; margin-bottom:12px;">
 <button class="btn" id="planCalendarPrevBtn">上个月</button>
 <h3 style="margin:0;" id="planCalendarTitle">本月计划</h3>
 <button class="btn" id="planCalendarNextBtn">下个月</button>
 </div>
 <div class="calendar-grid calendar-weekdays">
 <div>一</div><div>二</div><div>三</div><div>四</div><div>五</div><div>六</div><div>日</div>
 </div>
 <div class="calendar-grid" id="planCalendarGrid"></div>
 <div class="calendar-detail" id="planCalendarDetail"></div>
</div>
```

- [ ] **Step3: Add calendar CSS**

Near existing layout utilities, add:

```css
.calendar-grid { display: grid; grid-template-columns: repeat(7, minmax(0,1fr)); gap:8px; }
.calendar-weekdays { color: var(--muted); font-size:12px; text-align: center; margin-bottom:8px; }
.calendar-day { min-height:78px; border:1px solid rgba(148,163,184,0.22); border-radius:16px; background: rgba(255,255,255,0.42); padding:8px; cursor: pointer; transition:0.2s ease; }
.calendar-day:hover, .calendar-day.active { border-color: rgba(59,130,246,0.5); box-shadow:010px24px rgba(59,130,246,0.12); }
.calendar-day.muted { opacity:0.45; }
.calendar-date { display: flex; align-items: center; justify-content: space-between; font-weight:700; }
.status-dot { width:8px; height:8px; border-radius:999px; display: inline-block; }
.status-dot.done { background: #22c55e; }
.status-dot.partial { background: #f59e0b; }
.status-dot.pending { background: #ef4444; }
.calendar-count { margin-top:8px; color: var(--muted); font-size:12px; }
.calendar-detail { margin-top:14px; }
```

In existing small-screen media rules, add:

```css
.calendar-day { min-height:58px; padding:6px; }
```

- [ ] **Step4: Add calendar helpers**

Add these functions near `renderPlans()`:

```js
function monthLabel(month) {
 const [year, value] = month.split('-');
 return `${year}年${Number(value)}月`;
}

function shiftMonth(month, delta) {
 const [year, value] = month.split('-').map(Number);
 const date = new Date(year, value -1 + delta,1);
 return `${date.getFullYear()}-${String(date.getMonth() +1).padStart(2, '0')}`;
}

function plansForDate(date) {
 return app.state.plans.filter((plan) => plan.dueDate === date);
}

function planDateStatus(plans) {
 if (!plans.length) return '';
 const completed = plans.filter((plan) => plan.completed).length;
 if (completed === plans.length) return 'done';
 if (completed >0) return 'partial';
 return 'pending';
}
```

- [ ] **Step5: Render calendar and details**

Add:

```js
function renderPlanCalendar() {
 $('planCalendarTitle').textContent = `${monthLabel(app.planCalendarMonth)}计划日历`;
 const [year, month] = app.planCalendarMonth.split('-').map(Number);
 const first = new Date(year, month -1,1);
 const firstWeekday = (first.getDay() +6) %7;
 const daysInMonth = new Date(year, month,0).getDate();
 const cells = [];
 for (let i =0; i < firstWeekday; i +=1) cells.push({ empty: true });
 for (let day =1; day <= daysInMonth; day +=1) {
 const date = `${app.planCalendarMonth}-${String(day).padStart(2, '0')}`;
 const plans = plansForDate(date);
 cells.push({ date, day, plans, status: planDateStatus(plans) });
 }
 $('planCalendarGrid').innerHTML = cells.map((cell) => {
 if (cell.empty) return '<div class="calendar-day muted"></div>';
 const dot = cell.status ? `<span class="status-dot ${cell.status}"></span>` : '';
 return `<button class="calendar-day ${cell.date === app.selectedPlanDate ? 'active' : ''}" data-plan-calendar-date="${cell.date}">
 <span class="calendar-date"><span>${cell.day}</span>${dot}</span>
 <span class="calendar-count">${cell.plans.length ? `${cell.plans.length} 个计划` : '无计划'}</span>
 </button>`;
 }).join('');
 renderPlanCalendarDetail();
}

function renderPlanCalendarDetail() {
 const plans = plansForDate(app.selectedPlanDate);
 const title = `<h3 style="margin:0010px;">${escapeHtml(app.selectedPlanDate)}计划</h3>`;
 if (!plans.length) {
 $('planCalendarDetail').innerHTML = `${title}<div class="muted small">这一天暂无计划</div>`;
 return;
 }
 $('planCalendarDetail').innerHTML = `${title}<div class="list">${plans.map((plan) => `
 <div class="item ${plan.completed ? 'completed' : ''}">
 <div class="item-main">
 <div class="item-title">${escapeHtml(plan.title)}</div>
 <div class="tag-row">${subjectPill(plan.subjectId)}<span class="tag">题型：${escapeHtml(plan.questionType || '无')}</span><span class="tag">${plan.completed ? '已完成' : '未完成'}</span></div>
 </div>
 <div class="item-actions"><label class="check-row small"><input type="checkbox" data-plan-toggle="${plan.id}" ${plan.completed ? 'checked' : ''}/>完成</label></div>
 </div>`).join('')}</div>`;
}
```

Call it from `renderPlans()` after the three plan lists:

```js
renderPlanCalendar();
```

- [ ] **Step6: Bind calendar controls**

In `bindEvents()`, add:

```js
$('planCalendarPrevBtn').addEventListener('click', () => {
 app.planCalendarMonth = shiftMonth(app.planCalendarMonth, -1);
 app.selectedPlanDate = `${app.planCalendarMonth}-01`;
 renderPlans();
});
$('planCalendarNextBtn').addEventListener('click', () => {
 app.planCalendarMonth = shiftMonth(app.planCalendarMonth,1);
 app.selectedPlanDate = `${app.planCalendarMonth}-01`;
 renderPlans();
});
```

Inside the document click handler, add:

```js
const calendarDate = event.target.closest('[data-plan-calendar-date]');
if (calendarDate) {
 app.selectedPlanDate = calendarDate.dataset.planCalendarDate;
 renderPlanCalendar();
}
```

Expected behavior: clicking a date updates the detail list without changing stored data.

- [ ] **Step7: Verify Task2**

Run the syntax check command from Task1.

Manual browser checks:
- Calendar defaults to current month.
- Previous/next month buttons work.
- Dates with no plans are blank.
- Dates with all completed plans show green dot.
- Dates with mixed state show yellow dot.
- Dates with only unfinished plans show red dot.
- Clicking a date shows its plans.
- Toggling completion in calendar detail updates list, dashboard plan summary, and calendar dot.

---

## Task3: Normalize review records for subject/type blocks

**Files:**
- Modify: `index.html` review helpers and `loadState()`

- [ ] **Step1: Add review normalization helpers**

Near `migrateReviewArray()`, add:

```js
function normalizeReview(review, fallbackDate = '', fallbackType = 'daily') {
 if (typeof review === 'string') {
 return { date: fallbackDate, type: fallbackType, done: review, problems: '', actions: '', tomorrow: '', subjectReviews: [] };
 }
 const safe = review && typeof review === 'object' ? review : {};
 return {
 ...safe,
 date: safe.date || fallbackDate,
 type: safe.type || fallbackType,
 done: safe.done || '',
 problems: safe.problems || '',
 actions: safe.actions || '',
 tomorrow: safe.tomorrow || '',
 subjectReviews: Array.isArray(safe.subjectReviews) ? safe.subjectReviews : []
 };
}

function normalizeReviewsMap(reviews) {
 if (!reviews || typeof reviews !== 'object' || Array.isArray(reviews)) return migrateReviewArray(reviews);
 return Object.entries(reviews).reduce((map, [key, review]) => {
 const [date, type = 'daily'] = key.split('_');
 map[key] = normalizeReview(review, date, type);
 return map;
 }, {});
}
```

- [ ] **Step2: Use review normalization in `loadState()`**

Change reviews line to:

```js
reviews: normalizeReviewsMap(parsed.reviews),
```

Expected behavior: old review objects and old string reviews load into the existing form and have `subjectReviews: []`.

- [ ] **Step3: Update `migrateReviewArray()` to normalize entries**

Change assignment inside `migrateReviewArray()` to:

```js
map[reviewKey(review.date, type)] = normalizeReview({ ...review, type }, review.date, type);
```

- [ ] **Step4: Verify Task3**

Run the syntax check command from Task1.

Manual browser checks:
- Existing daily/weekly review form loads without errors.
- Saving a normal review still works.
- History still shows existing review entries.

---

## Task4: Add review subject/type blocks and save behavior

**Files:**
- Modify: `index.html` review section HTML, review rendering, event binding

- [ ] **Step1: Add subject review container to HTML**

In the review editor, after `reviewTomorrow` textarea and before the button row, add:

```html
<div class="subject-review-panel">
 <div class="btn-row" style="justify-content:space-between; margin-top:12px;">
 <h3 style="margin:0;">分科 /题型复盘</h3>
 <button class="btn" id="addSubjectReviewBtn" type="button">添加科目复盘</button>
 </div>
 <div class="list" id="subjectReviewList" style="margin-top:10px;"></div>
</div>
```

- [ ] **Step2: Add subject review render helper**

Add near review functions:

```js
function currentReview() {
 return normalizeReview(app.state.reviews[currentReviewKey()] || {}, $('reviewDate').value || todayISO(), $('reviewType').value);
}

function renderSubjectReviewBlocks(review) {
 const blocks = review.subjectReviews || [];
 if (!blocks.length) {
 $('subjectReviewList').innerHTML = '<div class="muted small">暂无分科复盘，可按科目或题型补充专项反思。</div>';
 return;
 }
 $('subjectReviewList').innerHTML = blocks.map((block) => `
 <div class="item subject-review-block" data-subject-review-id="${escapeHtml(block.id)}">
 <div class="item-main">
 <div class="form-row">
 <div class="field"><label>科目</label><select data-subject-review-subject="${escapeHtml(block.id)}"></select></div>
 <div class="field"><label>题型</label><select data-subject-review-type="${escapeHtml(block.id)}"></select></div>
 </div>
 <div class="field"><label>专项反思 / 错题总结</label><textarea data-subject-review-content="${escapeHtml(block.id)}">${escapeHtml(block.content || '')}</textarea></div>
 </div>
 <div class="item-actions"><button class="btn danger" data-subject-review-delete="${escapeHtml(block.id)}">删除</button></div>
 </div>
 `).join('');
 blocks.forEach((block) => {
 const subjectSelect = document.querySelector(`[data-subject-review-subject="${CSS.escape(block.id)}"]`);
 const typeSelect = document.querySelector(`[data-subject-review-type="${CSS.escape(block.id)}"]`);
 populateSubjectSelect(subjectSelect, { includeEmpty: true });
 subjectSelect.value = block.subjectId || '';
 populateQuestionTypeSelect(typeSelect, subjectSelect.value, { includeEmpty: true, emptyText: '无' });
 if (block.questionType && ![...typeSelect.options].some((opt) => opt.value === block.questionType)) {
 const option = document.createElement('option');
 option.value = block.questionType;
 option.textContent = `${block.questionType}（已保留）`;
 typeSelect.appendChild(option);
 }
 typeSelect.value = block.questionType || '';
 });
}
```

- [ ] **Step3: Load subject review blocks into form**

At the end of `loadReviewIntoForm()`, before setting `app.reviewFormDirty = false`, call:

```js
renderSubjectReviewBlocks(normalizeReview(review, $('reviewDate').value || todayISO(), $('reviewType').value));
```

- [ ] **Step4: Collect subject review form values**

Add:

```js
function collectSubjectReviewBlocks() {
 return [...document.querySelectorAll('[data-subject-review-id]')].map((element) => {
 const id = element.dataset.subjectReviewId;
 return {
 id,
 subjectId: document.querySelector(`[data-subject-review-subject="${CSS.escape(id)}"]`).value,
 questionType: document.querySelector(`[data-subject-review-type="${CSS.escape(id)}"]`).value,
 content: document.querySelector(`[data-subject-review-content="${CSS.escape(id)}"]`).value.trim()
 };
 }).filter((block) => block.subjectId || block.questionType || block.content);
}
```

- [ ] **Step5: Save subject reviews with overall review**

In `saveReview()`, add:

```js
subjectReviews: collectSubjectReviewBlocks(),
```

Expected behavior: one review can store multiple subject/type blocks.

- [ ] **Step6: Bind add/delete/change/input events**

In `bindEvents()`, add:

```js
$('addSubjectReviewBtn').addEventListener('click', () => {
 const review = currentReview();
 review.subjectReviews = collectSubjectReviewBlocks();
 review.subjectReviews.push({ id: uid('revsub'), subjectId: '', questionType: '', content: '' });
 app.state.reviews[currentReviewKey()] = review;
 app.reviewFormDirty = true;
 renderSubjectReviewBlocks(review);
});
```

In the document click handler, add:

```js
const subjectReviewDelete = event.target.closest('[data-subject-review-delete]');
if (subjectReviewDelete) {
 const review = currentReview();
 review.subjectReviews = collectSubjectReviewBlocks().filter((block) => block.id !== subjectReviewDelete.dataset.subjectReviewDelete);
 app.state.reviews[currentReviewKey()] = review;
 app.reviewFormDirty = true;
 renderSubjectReviewBlocks(review);
}
```

In the document change handler, add:

```js
const subjectReviewSubject = event.target.closest('[data-subject-review-subject]');
if (subjectReviewSubject) {
 const id = subjectReviewSubject.dataset.subjectReviewSubject;
 const typeSelect = document.querySelector(`[data-subject-review-type="${CSS.escape(id)}"]`);
 populateQuestionTypeSelect(typeSelect, subjectReviewSubject.value, { includeEmpty: true, emptyText: '无' });
 app.reviewFormDirty = true;
}
if (event.target.closest('[data-subject-review-type]')) app.reviewFormDirty = true;
```

Add input listener through event delegation:

```js
document.addEventListener('input', (event) => {
 if (event.target.closest('[data-subject-review-content]')) app.reviewFormDirty = true;
});
```

- [ ] **Step7: Verify Task4**

Run the syntax check command from Task1.

Manual browser checks:
- Add a subject review block.
- Select subject and type.
- Enter content and save.
- Reload page and confirm the block persists.
- Delete a subject review block and save.

---

## Task5: Add review history expansion and TXT/PDF export

**Files:**
- Modify: `index.html` review history rendering, export helpers, print CSS, event binding

- [ ] **Step1: Add export/print CSS**

Add:

```css
.export-preview { white-space: pre-wrap; line-height:1.8; }
@media print {
 body { background: #fff; }
 .sidebar, .topbar, .nav, .btn, .item-actions, .tabs { display: none !important; }
 .app-shell, .main, .section, .card { display: block !important; box-shadow: none !important; background: #fff !important; padding:0 !important; }
 .section:not(#section-reviews) { display: none !important; }
 #reviewHistory .item { break-inside: avoid; border:1px solid #ddd; margin-bottom:12px; }
}
```

- [ ] **Step2: Add review formatting helpers**

Add near review functions:

```js
function reviewTypeName(type) {
 return type === 'weekly' ? '每周复盘' : '每日复盘';
}

function reviewText(review) {
 const blocks = review.subjectReviews || [];
 const lines = [
 `${review.date}${reviewTypeName(review.type)}`,
 '====================',
 '',
 '【整体复盘】',
 `今日完成情况：${review.done || '未填写'}`,
 `困难与不足：${review.problems || '未填写'}`,
 `改进措施：${review.actions || '未填写'}`,
 `明日重点：${review.tomorrow || '未填写'}`,
 '',
 '【分科/题型复盘】'
 ];
 if (!blocks.length) lines.push('暂无分科/题型复盘');
 blocks.forEach((block) => {
 lines.push(`--- ${subjectName(block.subjectId)} / ${block.questionType || '无'} ---`);
 lines.push(block.content || '未填写');
 lines.push('');
 });
 return lines.join('\n');
}

function downloadTextFile(filename, content) {
 const blob = new Blob([content], { type: 'text/plain;charset=utf-8' });
 const url = URL.createObjectURL(blob);
 const link = document.createElement('a');
 link.href = url;
 link.download = filename;
 link.click();
 URL.revokeObjectURL(url);
}
```

- [ ] **Step3: Expand review history entries**

Replace each history item body in `renderReviewHistory()` with:

```js
<div class="item-main">
 <div class="item-title">${escapeHtml(r.date)} · ${reviewTypeName(r.type)}</div>
 <div class="small muted">${escapeHtml((r.done || '未填写完成情况').slice(0,70))}</div>
 <details style="margin-top:10px;">
 <summary class="small">展开详情</summary>
 <div class="export-preview">${escapeHtml(reviewText(r))}</div>
 </details>
</div>
<div class="item-actions">
 <button class="btn" data-review-load="${escapeHtml(r.date)}" data-review-type="${escapeHtml(r.type)}">查看</button>
 <button class="btn" data-review-export-txt="${escapeHtml(reviewKey(r.date, r.type))}">导出TXT</button>
 <button class="btn primary" data-review-export-pdf="${escapeHtml(reviewKey(r.date, r.type))}">导出PDF</button>
</div>
```

Expected behavior: history can expand and export actions are visible per review.

- [ ] **Step4: Bind export actions**

In document click handler, add:

```js
const reviewExportTxt = event.target.closest('[data-review-export-txt]');
const reviewExportPdf = event.target.closest('[data-review-export-pdf]');
if (reviewExportTxt) {
 const review = app.state.reviews[reviewExportTxt.dataset.reviewExportTxt];
 if (review) downloadTextFile(`${review.date}-${review.type}-review.txt`, reviewText(review));
}
if (reviewExportPdf) {
 const review = app.state.reviews[reviewExportPdf.dataset.reviewExportPdf];
 if (review) {
 $('reviewDate').value = review.date;
 $('reviewType').value = review.type;
 loadReviewIntoForm();
 window.print();
 }
}
```

- [ ] **Step5: Verify Task5**

Run the syntax check command from Task1.

Manual browser checks:
- History entries expand and show overall + subject/type blocks.
- TXT export downloads a readable `.txt` file.
- PDF export opens browser print flow.
- Print preview hides navigation and noisy controls.

---

## Task6: Add mistake accuracy fields and per-record display

**Files:**
- Modify: `index.html` mistake modal, mistake list, state normalization

- [ ] **Step1: Normalize old mistake records**

Add helper near `normalizePlan()`:

```js
function normalizeMistake(mistake) {
 return {
 ...mistake,
 totalQuestions: Number(mistake.totalQuestions) ||0,
 wrongQuestions: Number(mistake.wrongQuestions) ||0
 };
}
```

Change `loadState()` mistakes line to:

```js
mistakes: Array.isArray(parsed.mistakes) ? parsed.mistakes.map(normalizeMistake) : [],
```

- [ ] **Step2: Add accuracy helpers**

Near mistake functions, add:

```js
function validAccuracyRecord(record) {
 const total = Number(record.totalQuestions) ||0;
 const wrong = Number(record.wrongQuestions) ||0;
 return total >0 && wrong >=0 && wrong <= total;
}

function accuracyText(record) {
 if (!validAccuracyRecord(record)) return '正确率：未记录';
 const total = Number(record.totalQuestions);
 const wrong = Number(record.wrongQuestions);
 const correct = total - wrong;
 return `正确率：${correct}/${total}，${Math.round((correct / total) *100)}%`;
}
```

- [ ] **Step3: Add inputs to mistake modal**

In `openMistakeModal()`, extend the default mistake object:

```js
const mistake = app.state.mistakes.find((m) => m.id === mistakeId) || { subjectId: '', questionType: '', summary: '', reason: '', answer: '', tags: [], date: todayISO(), totalQuestions:0, wrongQuestions:0 };
```

Add after the date row:

```html
<div class="form-row">
 <div class="field"><label for="mistakeTotalInput">本次做题总数</label><input type="number" id="mistakeTotalInput" min="0" value="${Number(mistake.totalQuestions) ||0}" /></div>
 <div class="field"><label for="mistakeWrongInput">本次错题数</label><input type="number" id="mistakeWrongInput" min="0" value="${Number(mistake.wrongQuestions) ||0}" /></div>
</div>
```

- [ ] **Step4: Validate and save accuracy fields**

Before constructing payload in save handler:

```js
const totalQuestions = Math.max(0, Math.floor(Number($('mistakeTotalInput').value) ||0));
const wrongQuestions = Math.max(0, Math.floor(Number($('mistakeWrongInput').value) ||0));
if ((totalQuestions >0 || wrongQuestions >0) && totalQuestions <=0) { alert('请输入大于0 的做题总数。'); return; }
if (wrongQuestions > totalQuestions) { alert('错题数不能大于做题总数。'); return; }
```

Add to payload:

```js
totalQuestions,
wrongQuestions,
```

- [ ] **Step5: Display accuracy on each mistake card**

In `renderMistakes()`, add a muted line after the tag row:

```js
<div class="small muted" style="margin-top:8px;">${accuracyText(m)}</div>
```

Expected behavior: cards show either calculated accuracy or `正确率：未记录`.

- [ ] **Step6: Verify Task6**

Run the syntax check command from Task1.

Manual browser checks:
- Save mistake without accuracy fields.
- Save mistake with total20 and wrong2; card shows `18/20，90%`.
- Try wrong21 with total20; save is blocked.

---

## Task7: Add aggregate accuracy statistics and chart

**Files:**
- Modify: `index.html` dashboard HTML, chart rendering, event binding

- [ ] **Step1: Add accuracy chart HTML**

In dashboard chart grid, add a new card near mistake charts:

```html
<div class="card">
 <div class="btn-row" style="justify-content:space-between; margin-bottom:12px;">
 <h3 style="margin:0;">正确率统计</h3>
 <div class="btn-row">
 <select id="accuracyGranularityFilter" style="min-width:130px;"><option value="subject">按科目</option><option value="type">按题型</option></select>
 <select id="accuracySubjectFilter" style="min-width:150px;"><option value="all">全部科目</option></select>
 </div>
 </div>
 <div class="notice small" id="accuracySummary">暂无正确率数据</div>
 <div class="small muted chart-empty" id="accuracyEmpty">暂无正确率数据</div>
 <div class="chart-box"><canvas id="accuracyChart"></canvas></div>
</div>
```

- [ ] **Step2: Sync new subject filter**

In `syncSubjectDropdowns()`, add:

```js
populateSubjectSelect($('accuracySubjectFilter'), { includeAll: true });
if (!$('accuracySubjectFilter').value) $('accuracySubjectFilter').value = 'all';
```

- [ ] **Step3: Add aggregate helpers**

Add near chart helpers:

```js
function aggregateAccuracy(records) {
 return records.filter(validAccuracyRecord).reduce((sum, record) => {
 const total = Number(record.totalQuestions);
 const wrong = Number(record.wrongQuestions);
 sum.total += total;
 sum.wrong += wrong;
 sum.correct += total - wrong;
 return sum;
 }, { total:0, wrong:0, correct:0 });
}

function accuracyPercent(summary) {
 return summary.total ? Math.round((summary.correct / summary.total) *100) :0;
}

function accuracyChartData() {
 const granularity = $('accuracyGranularityFilter').value || 'subject';
 const subjectFilter = $('accuracySubjectFilter').value || 'all';
 const valid = app.state.mistakes.filter(validAccuracyRecord).filter((m) => subjectFilter === 'all' || m.subjectId === subjectFilter);
 if (granularity === 'type' && subjectFilter !== 'all') {
 const types = getSubjectTypes(subjectFilter);
 const labels = [...types, '未标注'];
 return labels.map((label) => {
 const records = valid.filter((m) => label === '未标注' ? !m.questionType : m.questionType === label);
 const summary = aggregateAccuracy(records);
 return { label, summary, value: accuracyPercent(summary) };
 }).filter((item) => item.summary.total >0);
 }
 return app.state.subjects.map((subject) => {
 const records = valid.filter((m) => m.subjectId === subject.id);
 const summary = aggregateAccuracy(records);
 return { label: subject.name, summary, value: accuracyPercent(summary) };
 }).filter((item) => item.summary.total >0);
}
```

- [ ] **Step4: Render accuracy chart**

Inside `renderCharts()`, after existing chart renders, add:

```js
const accuracyItems = accuracyChartData();
const totalAccuracy = aggregateAccuracy(app.state.mistakes.filter((m) => ($('accuracySubjectFilter').value || 'all') === 'all' || m.subjectId === $('accuracySubjectFilter').value));
$('accuracySummary').textContent = totalAccuracy.total ? `累计做题 ${totalAccuracy.total} 道，错题 ${totalAccuracy.wrong} 道，正确 ${totalAccuracy.correct} 道，整体正确率 ${accuracyPercent(totalAccuracy)}%` : '暂无正确率数据';
$('accuracyEmpty').classList.toggle('active', !accuracyItems.length);
if (app.charts.accuracy) app.charts.accuracy.destroy();
app.charts.accuracy = new Chart($('accuracyChart'), {
 type: 'bar',
 data: {
 labels: accuracyItems.map((item) => item.label),
 datasets: [{
 label: '正确率 %',
 data: accuracyItems.map((item) => item.value),
 backgroundColor: 'rgba(96,165,250,0.65)',
 borderColor: '#60a5fa',
 borderWidth:1,
 borderRadius:10
 }]
 },
 options: {
 responsive: true,
 maintainAspectRatio: false,
 scales: { y: { min:0, max:100, ticks: { callback: (value) => `${value}%` } } },
 plugins: { legend: { display: false } }
 }
});
```

- [ ] **Step5: Bind accuracy filters**

In `bindEvents()`, add:

```js
$('accuracyGranularityFilter').addEventListener('change', renderCharts);
$('accuracySubjectFilter').addEventListener('change', renderCharts);
```

If granularity is `type` and subject is `all`, the chart should fall back to subject comparison because type comparison requires one selected subject.

- [ ] **Step6: Verify Task7**

Run the syntax check command from Task1.

Manual browser checks:
- With no accuracy data, empty state appears.
- Add records across subjects; subject chart shows one bar per subject.
- Select one subject and `按题型`; chart shows current types plus `未标注` when data exists.
- Summary totals match saved mistake records.

---

## Task8: Final regression verification

**Files:**
- Verify: `index.html`

- [ ] **Step1: Run JavaScript syntax verification**

Run:

```bash
node -e "const fs=require('fs');const vm=require('vm');const html=fs.readFileSync('index.html','utf8');const script=html.match(/<script>([\s\S]*)<\/script>/)[1];new vm.Script(script);console.log('Inline JavaScript syntax OK');"
```

Expected output:

```text
Inline JavaScript syntax OK
```

- [ ] **Step2: Manual browser regression checklist**

Open `index.html` directly in the browser and check:
- Dashboard cards render.
- Study trend chart still updates for7/30 days and subject filter.
- Mistake trend chart still filters by subject/type.
- Pomodoro countdown starts, pauses, resets, and records focus sessions.
- Pomodoro count-up starts and can be ended/recorded.
- Subject management still adds/edits/deletes subjects.
- Subject type management still adds/renames/deletes types.
- Mistake modal still supports dynamic `+ 添加新题型`.
- Plan module supports type selection and calendar quick toggles.
- Review module supports subject/type blocks and TXT/PDF export.
- Accuracy module validates inputs, displays per-record accuracy, and renders aggregate chart.
- AI settings remain local-only and unchanged.

- [ ] **Step3: Inspect working tree before any git action**

Run only if the user asks about git status or asks to commit:

```bash
git status --short
```

Expected: modified `index.html` and this plan/spec docs as applicable.

Do not run `git commit` or `git push` unless the user explicitly asks.

---

## Self-Review

**Spec coverage:**
- Plan type selection: Task1.
- Plan list type tags: Task1.
- Plan monthly calendar, status dots, selected-date details, quick toggles: Task2.
- Review subject/type data normalization: Task3.
- Review subject/type editor blocks: Task4.
- Review history expansion and TXT/PDF export: Task5.
- Mistake total/wrong fields, validation, per-record display: Task6.
- Aggregate accuracy statistics and chart: Task7.
- Regression and syntax verification: Task8.

**Placeholder scan:** No TBD/TODO/later placeholders. Each implementation step names the file and concrete code or command.

**Type consistency:** Uses existing `subjectId`, `questionType`, `date`, `type`, `done`, `problems`, `actions`, `tomorrow`, `subjectReviews`, `totalQuestions`, and `wrongQuestions` property names consistently with the approved spec and current app structure.
