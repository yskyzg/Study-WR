# Civil Service Study App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a complete single-file HTML app for civil service exam study planning, Pomodoro tracking, mistake review, analytics, retrospectives, subject management, and optional AI assistance.

**Architecture:** Create one `index.html` containing semantic markup, cold minimalist glassmorphism CSS, and modular vanilla JavaScript. Persist one state object in `localStorage`; derive dashboard metrics and Chart.js datasets from that state on each render. Keep all mutation paths behind small helpers that save state and refresh affected views.

**Tech Stack:** HTML5, CSS3, vanilla JavaScript, localStorage, Chart.js CDN, OpenAI-compatible `/chat/completions` HTTP API.

---

## File Structure

- Create: `index.html` — the complete application: markup, styles, state model, storage, rendering, Pomodoro timer, CRUD handlers, charts, review logic, AI calls.
- Existing docs remain unchanged:
  - `docs/superpowers/specs/2026-06-06-civil-service-study-app-design.md`
  - `docs/superpowers/plans/2026-06-06-civil-service-study-app.md`

Because the user explicitly requested a pure single-file app, do not split CSS or JavaScript into separate files.

---

### Task 1: Create single-file shell, theme, navigation, and storage model

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create the HTML document shell**

Create `index.html` with this structure:

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>公务员考试备考智能复盘与番茄钟工具</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    :root {
      --bg-a: #e0f0ff;
      --bg-b: #d4e4ff;
      --bg-c: #e8e0f0;
      --text: #243447;
      --muted: #6c7a89;
      --line: rgba(90, 120, 160, 0.18);
      --card: rgba(255, 255, 255, 0.58);
      --card-strong: rgba(255, 255, 255, 0.78);
      --primary: #4f8fcf;
      --primary-dark: #376fa8;
      --danger: #d76a7c;
      --ok: #5aa897;
      --shadow: 0 18px 45px rgba(74, 104, 143, 0.16);
      --radius: 22px;
    }
    * { box-sizing: border-box; }
    body {
      margin: 0;
      min-height: 100vh;
      color: var(--text);
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Microsoft YaHei", sans-serif;
      background: linear-gradient(135deg, var(--bg-a), var(--bg-b), var(--bg-c));
    }
    button, input, select, textarea { font: inherit; }
  </style>
</head>
<body>
  <div class="app-shell">
    <aside class="sidebar" aria-label="主导航"></aside>
    <main class="main" id="app"></main>
  </div>
  <script>
    const STORAGE_KEY = 'civilServiceStudyApp.v1';
    const todayISO = () => new Date().toISOString().slice(0, 10);
    const uid = (prefix) => `${prefix}_${Date.now()}_${Math.random().toString(16).slice(2)}`;
  </script>
</body>
</html>
```

- [ ] **Step 2: Add layout and component CSS**

Add CSS for `.app-shell`, `.sidebar`, `.nav-btn`, `.main`, `.card`, `.grid`, `.stat-card`, forms, buttons, tables/lists, chart cards, responsive behavior. Use the approved cold gradient and glass cards. Include mobile behavior where the sidebar becomes a horizontal top strip.

Expected CSS selectors to include:

```css
.app-shell { display: grid; grid-template-columns: 260px 1fr; gap: 24px; min-height: 100vh; padding: 24px; }
.sidebar { position: sticky; top: 24px; height: calc(100vh - 48px); padding: 22px; border: 1px solid var(--line); border-radius: var(--radius); background: var(--card); backdrop-filter: blur(18px); box-shadow: var(--shadow); }
.nav-btn.active { background: linear-gradient(135deg, rgba(79,143,207,.18), rgba(119,139,218,.18)); color: var(--primary-dark); }
.card { border: 1px solid var(--line); border-radius: var(--radius); background: var(--card); backdrop-filter: blur(18px); box-shadow: var(--shadow); padding: 22px; }
.btn.primary { color: white; background: linear-gradient(135deg, #4f8fcf, #7b8cda); }
@media (max-width: 900px) { .app-shell { grid-template-columns: 1fr; padding: 14px; } .sidebar { position: static; height: auto; } }
```

- [ ] **Step 3: Define default application state**

Add this JavaScript state shape:

```js
const DEFAULT_SUBJECTS = [
  { id: 'xingce_yanyu', name: '言语理解与表达', color: '#5B9BD5' },
  { id: 'xingce_shuliang', name: '数量关系', color: '#6AAED6' },
  { id: 'xingce_panduan', name: '判断推理', color: '#7B8CDA' },
  { id: 'xingce_ziliao', name: '资料分析', color: '#59B7C7' },
  { id: 'xingce_changshi', name: '常识判断', color: '#8CA6DB' },
  { id: 'shenlun', name: '申论', color: '#7AA7C7' },
  { id: 'gongji', name: '公共基础知识', color: '#90B8D8' },
  { id: 'mianshi', name: '面试', color: '#A0A7DC' }
];

function defaultState() {
  const today = todayISO();
  return {
    subjects: DEFAULT_SUBJECTS,
    plans: [
      { id: uid('plan'), title: '资料分析专项练习', subjectId: 'xingce_ziliao', pomodoros: 2, dueDate: today, scope: 'daily', completed: false, createdAt: new Date().toISOString() },
      { id: uid('plan'), title: '申论小题复盘', subjectId: 'shenlun', pomodoros: 1, dueDate: today, scope: 'daily', completed: true, createdAt: new Date().toISOString() }
    ],
    mistakes: [
      { id: uid('mistake'), subjectId: 'xingce_panduan', summary: '图形推理旋转规律判断失误', reason: '只观察局部，未比较整体方向变化', answer: '先判断元素数量，再比较旋转角度', tags: '图形推理,规律', date: today },
      { id: uid('mistake'), subjectId: 'shenlun', summary: '概括题要点遗漏', reason: '材料段落层次划分不清', answer: '逐段标记主体、问题、对策', tags: '申论,概括', date: today }
    ],
    sessions: [
      { id: uid('session'), subjectId: 'xingce_ziliao', minutes: 50, type: 'focus', startedAt: `${today}T08:30:00`, endedAt: `${today}T09:20:00` },
      { id: uid('session'), subjectId: 'shenlun', minutes: 25, type: 'focus', startedAt: `${today}T10:00:00`, endedAt: `${today}T10:25:00` }
    ],
    reviews: [],
    settings: { focusMinutes: 25, breakMinutes: 5, apiKey: '', baseUrl: 'https://api.openai.com/v1', model: 'gpt-4o-mini' }
  };
}
```

- [ ] **Step 4: Add storage helpers**

Implement:

```js
let state = loadState();
let currentView = 'dashboard';

function loadState() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return defaultState();
    const parsed = JSON.parse(raw);
    return { ...defaultState(), ...parsed, settings: { ...defaultState().settings, ...(parsed.settings || {}) } };
  } catch (error) {
    console.warn('读取本地数据失败，使用默认数据', error);
    return defaultState();
  }
}

function saveState() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
}

function subjectById(id) {
  return state.subjects.find(subject => subject.id === id) || { id: '', name: '未分类', color: '#9fb3c8' };
}
```

- [ ] **Step 5: Verify shell manually**

Open `index.html` in a browser. Expected: no console syntax errors; page background renders with cold gradient; app root exists but views may still be incomplete.

---

### Task 2: Implement shared render utilities, navigation, dashboard metrics, and charts

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add shared formatting and aggregation helpers**

Implement helpers:

```js
function minutesToText(minutes) {
  const h = Math.floor(minutes / 60);
  const m = minutes % 60;
  return h ? `${h}小时${m}分钟` : `${m}分钟`;
}

function sameDate(isoLike, date) {
  return String(isoLike || '').slice(0, 10) === date;
}

function sessionsInRange(days) {
  const start = new Date();
  start.setHours(0, 0, 0, 0);
  start.setDate(start.getDate() - days + 1);
  return state.sessions.filter(session => new Date(session.endedAt || session.startedAt) >= start);
}

function totalMinutesForDate(date) {
  return state.sessions.filter(session => sameDate(session.endedAt, date)).reduce((sum, session) => sum + Number(session.minutes || 0), 0);
}

function mistakesForDate(date) {
  return state.mistakes.filter(mistake => mistake.date === date);
}
```

- [ ] **Step 2: Add navigation rendering**

Create the sidebar using:

```js
const NAV_ITEMS = [
  { id: 'dashboard', icon: '📊', label: '仪表盘' },
  { id: 'pomodoro', icon: '⏱️', label: '番茄钟' },
  { id: 'plans', icon: '🗓️', label: '计划' },
  { id: 'mistakes', icon: '📘', label: '错题' },
  { id: 'reviews', icon: '📝', label: '复盘' },
  { id: 'settings', icon: '⚙️', label: '设置' }
];

function renderShell() {
  document.querySelector('.sidebar').innerHTML = `
    <div class="brand"><span class="brand-mark">公</span><div><strong>公考复盘</strong><small>专注 · 记录 · 改进</small></div></div>
    <nav>${NAV_ITEMS.map(item => `<button class="nav-btn ${currentView === item.id ? 'active' : ''}" data-view="${item.id}"><span>${item.icon}</span>${item.label}</button>`).join('')}</nav>
  `;
  document.querySelectorAll('.nav-btn').forEach(button => {
    button.addEventListener('click', () => { currentView = button.dataset.view; renderApp(); });
  });
}
```

- [ ] **Step 3: Add dashboard render function**

Implement `renderDashboard()` with cards for today minutes, today Pomodoro count, incomplete plans, today mistakes, AI reminder card, and three chart canvases:

```html
<canvas id="trendChart"></canvas>
<canvas id="subjectPieChart"></canvas>
<canvas id="mistakeBarChart"></canvas>
```

Include a range selector:

```html
<select id="trendRange"><option value="7">近7天</option><option value="30">近30天</option></select>
```

- [ ] **Step 4: Add Chart.js instance management**

Implement:

```js
const chartRefs = {};
function destroyChart(id) {
  if (chartRefs[id]) { chartRefs[id].destroy(); delete chartRefs[id]; }
}
function subjectLabels() { return state.subjects.map(subject => subject.name); }
function subjectColors() { return state.subjects.map(subject => subject.color); }
```

Add functions:

- `renderTrendChart(days)` — labels each day, sums sessions per day in hours.
- `renderSubjectPieChart()` — sums all session minutes by subject.
- `renderMistakeBarChart()` — counts mistakes by subject.

Each function should call `destroyChart(canvasId)` before creating a new `Chart`.

- [ ] **Step 5: Wire root render**

Implement:

```js
function renderApp() {
  renderShell();
  const app = document.getElementById('app');
  if (currentView === 'dashboard') app.innerHTML = renderDashboard();
  if (currentView === 'pomodoro') app.innerHTML = renderPomodoro();
  if (currentView === 'plans') app.innerHTML = renderPlans();
  if (currentView === 'mistakes') app.innerHTML = renderMistakes();
  if (currentView === 'reviews') app.innerHTML = renderReviews();
  if (currentView === 'settings') app.innerHTML = renderSettings();
  bindViewEvents();
}

document.addEventListener('DOMContentLoaded', renderApp);
```

- [ ] **Step 6: Verify dashboard manually**

Open `index.html`. Expected: sidebar navigation works; dashboard shows sample metrics; Chart.js renders line, pie, and bar charts without console errors.

---

### Task 3: Implement Pomodoro timer

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add timer state**

Add:

```js
let timer = {
  mode: 'focus',
  status: 'idle',
  remainingSeconds: 0,
  totalSeconds: 0,
  intervalId: null,
  subjectId: '',
  startedAt: null
};
```

- [ ] **Step 2: Add Pomodoro view markup**

Implement `renderPomodoro()` returning a card with:

- Subject select `#timerSubject`
- Focus minutes input `#focusMinutes`
- Break minutes input `#breakMinutes`
- SVG circular progress ring using `#timerCircle`
- Time display `#timerDisplay`
- Buttons `#startTimer`, `#pauseTimer`, `#resetTimer`

Use the current settings and timer state for initial values.

- [ ] **Step 3: Add timer functions**

Implement:

```js
function prepareTimerIfNeeded() {
  if (timer.remainingSeconds > 0) return;
  const minutes = timer.mode === 'focus' ? Number(state.settings.focusMinutes) : Number(state.settings.breakMinutes);
  timer.totalSeconds = Math.max(1, minutes) * 60;
  timer.remainingSeconds = timer.totalSeconds;
}

function startTimer() {
  const selectedSubject = document.getElementById('timerSubject')?.value || timer.subjectId;
  if (timer.mode === 'focus' && !selectedSubject) { alert('开始前请选择当前学习科目'); return; }
  timer.subjectId = selectedSubject;
  state.settings.focusMinutes = Number(document.getElementById('focusMinutes').value || 25);
  state.settings.breakMinutes = Number(document.getElementById('breakMinutes').value || 5);
  saveState();
  prepareTimerIfNeeded();
  timer.status = 'running';
  timer.startedAt = timer.startedAt || new Date().toISOString();
  clearInterval(timer.intervalId);
  timer.intervalId = setInterval(tickTimer, 1000);
  updateTimerUI();
}

function pauseTimer() {
  timer.status = 'paused';
  clearInterval(timer.intervalId);
  updateTimerUI();
}

function resetTimer() {
  if (timer.status === 'running' && !confirm('确定重置当前计时吗？')) return;
  clearInterval(timer.intervalId);
  timer = { ...timer, status: 'idle', remainingSeconds: 0, totalSeconds: 0, startedAt: null };
  updateTimerUI();
}
```

- [ ] **Step 4: Add completion behavior and sound**

Implement:

```js
function tickTimer() {
  timer.remainingSeconds -= 1;
  if (timer.remainingSeconds <= 0) finishTimerSegment();
  updateTimerUI();
}

function playDoneSound() {
  const audio = new AudioContext();
  const oscillator = audio.createOscillator();
  const gain = audio.createGain();
  oscillator.connect(gain);
  gain.connect(audio.destination);
  oscillator.frequency.value = 680;
  gain.gain.setValueAtTime(0.001, audio.currentTime);
  gain.gain.exponentialRampToValueAtTime(0.18, audio.currentTime + 0.02);
  gain.gain.exponentialRampToValueAtTime(0.001, audio.currentTime + 0.8);
  oscillator.start();
  oscillator.stop(audio.currentTime + 0.85);
}

function finishTimerSegment() {
  clearInterval(timer.intervalId);
  playDoneSound();
  if (timer.mode === 'focus') {
    const minutes = Number(state.settings.focusMinutes || 25);
    state.sessions.push({ id: uid('session'), subjectId: timer.subjectId, minutes, type: 'focus', startedAt: timer.startedAt || new Date().toISOString(), endedAt: new Date().toISOString() });
    saveState();
    alert('专注结束，已记录学习时长。准备休息一下吧。');
    timer.mode = 'break';
  } else {
    alert('休息结束，可以开始下一轮专注。');
    timer.mode = 'focus';
  }
  timer.status = 'idle';
  timer.remainingSeconds = 0;
  timer.totalSeconds = 0;
  timer.startedAt = null;
  renderApp();
}
```

- [ ] **Step 5: Bind Pomodoro events**

In `bindViewEvents()`, when `currentView === 'pomodoro'`, bind the three buttons and input changes. Expected IDs: `startTimer`, `pauseTimer`, `resetTimer`, `timerSubject`, `focusMinutes`, `breakMinutes`.

- [ ] **Step 6: Verify timer manually**

Set focus minutes to `1`, select a subject, start timer, pause/resume, then let it complete. Expected: sound plays, session is recorded, dashboard today total increases after returning to dashboard.

---

### Task 4: Implement learning plans CRUD

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add plan render function**

Implement `renderPlans()` with a form containing:

- Hidden `#planId`
- `#planTitle`
- `#planSubject`
- `#planPomodoros`
- `#planDueDate`
- `#planScope` with daily/weekly
- Submit button

Then render today's plans grouped by incomplete and complete:

```js
const todayPlans = state.plans.filter(plan => plan.dueDate === todayISO());
const incomplete = todayPlans.filter(plan => !plan.completed);
const complete = todayPlans.filter(plan => plan.completed);
```

Also render all future/other plans in a secondary list so weekly tasks remain visible.

- [ ] **Step 2: Add plan mutation functions**

Implement:

```js
function savePlanFromForm(event) {
  event.preventDefault();
  const id = document.getElementById('planId').value;
  const title = document.getElementById('planTitle').value.trim();
  if (!title) { alert('请输入任务名称'); return; }
  const payload = {
    title,
    subjectId: document.getElementById('planSubject').value,
    pomodoros: Number(document.getElementById('planPomodoros').value || 1),
    dueDate: document.getElementById('planDueDate').value || todayISO(),
    scope: document.getElementById('planScope').value,
    completed: false
  };
  if (id) {
    const existing = state.plans.find(plan => plan.id === id);
    Object.assign(existing, payload, { completed: existing.completed });
  } else {
    state.plans.push({ id: uid('plan'), ...payload, createdAt: new Date().toISOString() });
  }
  saveState();
  renderApp();
}

function togglePlan(id) {
  const plan = state.plans.find(item => item.id === id);
  if (plan) { plan.completed = !plan.completed; saveState(); renderApp(); }
}

function deletePlan(id) {
  if (!confirm('确定删除这个计划任务吗？')) return;
  state.plans = state.plans.filter(plan => plan.id !== id);
  saveState();
  renderApp();
}
```

- [ ] **Step 3: Add edit behavior**

Implement `editPlan(id)` that fills the form values from the selected plan and scrolls to the form.

- [ ] **Step 4: Bind plan events**

In `bindViewEvents()`, when `currentView === 'plans'`, bind form submit and click handlers for `.plan-toggle`, `.plan-edit`, `.plan-delete`.

- [ ] **Step 5: Verify plans manually**

Add a task, mark it complete, edit its title, delete it. Expected: each change persists after page refresh; delete prompts for confirmation.

---

### Task 5: Implement mistakes CRUD and subject filtering

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add mistakes render function**

Implement `renderMistakes()` with:

- Filter select `#mistakeFilter`
- Form fields: hidden `#mistakeId`, `#mistakeSubject`, `#mistakeSummary`, `#mistakeReason`, `#mistakeAnswer`, `#mistakeTags`, `#mistakeDate`
- Mistake cards/list showing subject color, date, tags, summary, reason, answer, edit/delete buttons.

- [ ] **Step 2: Add mistake mutation functions**

Implement:

```js
function saveMistakeFromForm(event) {
  event.preventDefault();
  const id = document.getElementById('mistakeId').value;
  const summary = document.getElementById('mistakeSummary').value.trim();
  if (!summary) { alert('请输入题目摘要'); return; }
  const payload = {
    subjectId: document.getElementById('mistakeSubject').value,
    summary,
    reason: document.getElementById('mistakeReason').value.trim(),
    answer: document.getElementById('mistakeAnswer').value.trim(),
    tags: document.getElementById('mistakeTags').value.trim(),
    date: document.getElementById('mistakeDate').value || todayISO()
  };
  if (id) Object.assign(state.mistakes.find(item => item.id === id), payload);
  else state.mistakes.push({ id: uid('mistake'), ...payload });
  saveState();
  renderApp();
}

function deleteMistake(id) {
  if (!confirm('确定删除这条错题记录吗？')) return;
  state.mistakes = state.mistakes.filter(item => item.id !== id);
  saveState();
  renderApp();
}
```

- [ ] **Step 3: Add edit and filter behavior**

Implement `editMistake(id)` to fill the form. Store current filter in `currentMistakeFilter` variable. On `#mistakeFilter` change, update the variable and re-render.

- [ ] **Step 4: Bind mistake events**

In `bindViewEvents()`, when `currentView === 'mistakes'`, bind form submit, filter change, `.mistake-edit`, `.mistake-delete`.

- [ ] **Step 5: Verify mistakes manually**

Create one mistake under a subject, filter to that subject, edit tags, delete it. Expected: chart data updates after returning to dashboard; delete prompts for confirmation.

---

### Task 6: Implement retrospectives and AI review card

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add review render function**

Implement `renderReviews()` with:

- Date input `#reviewDate`
- Period select `#reviewPeriod` daily/weekly
- Summary card showing selected date total study time and mistake count
- Textareas `#reviewCompleted`, `#reviewProblems`, `#reviewImprovements`, `#reviewTomorrow`
- Save button `#reviewForm`
- AI button `#aiReviewBtn`
- AI output card `#aiReviewResult`
- Historical list of saved reviews sorted by date descending.

- [ ] **Step 2: Add review helpers**

Implement:

```js
function reviewFor(date, period) {
  return state.reviews.find(review => review.date === date && review.period === period);
}

function saveReviewFromForm(event) {
  event.preventDefault();
  const date = document.getElementById('reviewDate').value || todayISO();
  const period = document.getElementById('reviewPeriod').value;
  const payload = {
    date,
    period,
    completed: document.getElementById('reviewCompleted').value.trim(),
    problems: document.getElementById('reviewProblems').value.trim(),
    improvements: document.getElementById('reviewImprovements').value.trim(),
    tomorrow: document.getElementById('reviewTomorrow').value.trim()
  };
  const existing = reviewFor(date, period);
  if (existing) Object.assign(existing, payload);
  else state.reviews.push({ id: uid('review'), ...payload, aiAdvice: '' });
  saveState();
  renderApp();
}
```

- [ ] **Step 3: Add review load behavior**

When date or period changes, re-render the review form with existing saved content if present. Keep the default date as today.

- [ ] **Step 4: Verify reviews manually**

Save a daily review, switch date away and back, confirm fields load. Expected: study-time and mistake-count summary changes with selected date.

---

### Task 7: Implement subject management and settings

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add settings render function**

Implement `renderSettings()` with two cards:

1. Subject management form:
   - hidden `#subjectId`
   - `#subjectName`
   - `#subjectColor`
   - list of all subjects with edit/delete buttons
2. AI and timer settings form:
   - `#settingFocusMinutes`
   - `#settingBreakMinutes`
   - `#apiKey`
   - `#baseUrl`
   - `#model`

- [ ] **Step 2: Add subject mutation functions**

Implement:

```js
function saveSubjectFromForm(event) {
  event.preventDefault();
  const id = document.getElementById('subjectId').value;
  const name = document.getElementById('subjectName').value.trim();
  const color = document.getElementById('subjectColor').value || '#5B9BD5';
  if (!name) { alert('请输入科目名称'); return; }
  if (id) Object.assign(state.subjects.find(subject => subject.id === id), { name, color });
  else state.subjects.push({ id: uid('subject'), name, color });
  saveState();
  renderApp();
}

function deleteSubject(id) {
  const used = state.plans.some(plan => plan.subjectId === id) || state.mistakes.some(mistake => mistake.subjectId === id) || state.sessions.some(session => session.subjectId === id);
  const message = used ? '该科目已有学习/计划/错题记录，删除后相关记录会显示为未分类。确定删除吗？' : '确定删除这个科目吗？';
  if (!confirm(message)) return;
  state.subjects = state.subjects.filter(subject => subject.id !== id);
  saveState();
  renderApp();
}
```

- [ ] **Step 3: Add settings save function**

Implement:

```js
function saveSettingsFromForm(event) {
  event.preventDefault();
  state.settings.focusMinutes = Number(document.getElementById('settingFocusMinutes').value || 25);
  state.settings.breakMinutes = Number(document.getElementById('settingBreakMinutes').value || 5);
  state.settings.apiKey = document.getElementById('apiKey').value.trim();
  state.settings.baseUrl = document.getElementById('baseUrl').value.trim().replace(/\/$/, '');
  state.settings.model = document.getElementById('model').value.trim();
  saveState();
  alert('设置已保存');
  renderApp();
}
```

- [ ] **Step 4: Bind settings events**

In `bindViewEvents()`, when `currentView === 'settings'`, bind subject form, settings form, `.subject-edit`, `.subject-delete`.

- [ ] **Step 5: Verify settings manually**

Add a subject, confirm it appears in Pomodoro/Plan/Mistake dropdowns. Edit its color, confirm chart color changes. Save API settings and refresh; expected settings persist.

---

### Task 8: Implement AI reminder and AI review calls

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add OpenAI-compatible API helper**

Implement:

```js
async function callAI(prompt) {
  const { apiKey, baseUrl, model } = state.settings;
  if (!apiKey || !baseUrl || !model) throw new Error('请先在设置中填写 API Key、Base URL 和模型名称。');
  const response = await fetch(`${baseUrl.replace(/\/$/, '')}/chat/completions`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${apiKey}` },
    body: JSON.stringify({ model, messages: [{ role: 'user', content: prompt }], temperature: 0.7 })
  });
  if (!response.ok) throw new Error(`API 请求失败：${response.status} ${response.statusText}`);
  const data = await response.json();
  const text = data?.choices?.[0]?.message?.content;
  if (!text) throw new Error('API 返回内容为空或格式不兼容。');
  return text;
}
```

- [ ] **Step 2: Add today data prompt builder**

Implement:

```js
function buildTodayStudySummary(date = todayISO()) {
  const sessions = state.sessions.filter(session => sameDate(session.endedAt, date));
  const mistakes = mistakesForDate(date);
  const plans = state.plans.filter(plan => plan.dueDate === date);
  const bySubject = state.subjects.map(subject => {
    const minutes = sessions.filter(session => session.subjectId === subject.id).reduce((sum, session) => sum + Number(session.minutes || 0), 0);
    return `${subject.name}: ${minutes}分钟`;
  }).join('\n');
  return { sessions, mistakes, plans, bySubject, totalMinutes: totalMinutesForDate(date) };
}
```

- [ ] **Step 3: Add AI reminder function**

Implement `generateAIReminder()` that builds a short prompt from today's plan completion rate, total minutes, and unfinished plan titles. It should set a loading message in `#aiReminder`, call `callAI`, then render the returned text or an error.

- [ ] **Step 4: Add AI review function**

Implement `generateAIReview()` that collects selected review form fields plus `buildTodayStudySummary(date)`, calls `callAI`, writes result into `#aiReviewResult`, and saves it into the matching review object's `aiAdvice` field if a review exists.

- [ ] **Step 5: Bind AI buttons**

Bind `#aiReminderBtn` on dashboard and `#aiReviewBtn` on reviews.

- [ ] **Step 6: Verify AI behavior manually**

Without API settings, click AI buttons. Expected: readable error instructs user to configure API settings. With valid OpenAI-compatible settings, expected: loading state appears and response text renders in the card.

---

### Task 9: Final polish, persistence verification, and acceptance pass

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add final safety details**

Ensure all delete functions use `confirm()`. Ensure reset timer confirms only when currently running. Ensure subject deletion warns when referenced by existing records.

- [ ] **Step 2: Add empty states**

For each list, render a restrained empty message:

```html
<p class="empty">暂无记录，先添加一条吧。</p>
```

Use empty states for plans, mistakes, reviews, and chart datasets with no values.

- [ ] **Step 3: Add responsive and accessibility checks**

Ensure form labels are visible, buttons have readable text, focus outlines are not removed, and mobile width below 900px keeps all controls usable.

- [ ] **Step 4: Run manual acceptance checklist**

Open `index.html` and verify:

1. Sidebar switches all six pages.
2. Reload keeps localStorage data.
3. Pomodoro requires subject before start.
4. One completed focus session appears in dashboard totals.
5. Plan add/edit/delete/toggle works with confirmation on delete.
6. Mistake add/filter/edit/delete works with confirmation on delete.
7. Review save/load by date works.
8. Subject add/edit/delete updates dropdowns.
9. Chart range switch 7/30 days works.
10. AI buttons show configuration error when settings are missing.

Expected result: all ten checks pass.

- [ ] **Step 5: Report completion**

Since this directory is not a git repository, do not commit. Report that `index.html` is ready to run directly in a browser and summarize any skipped checks or limitations.

---

## Self-Review

- Spec coverage: The plan covers single-file implementation, localStorage persistence, preset subjects, Pomodoro, plans, mistakes, charts, reviews, settings, AI calls, confirmations, and cold minimalist UI.
- Placeholder scan: No TBD/TODO placeholders remain. Each task has concrete functions, IDs, commands or manual checks.
- Type consistency: Data fields are consistent across tasks: `subjects`, `plans`, `mistakes`, `sessions`, `reviews`, and `settings`; IDs and helper names are stable across tasks.
