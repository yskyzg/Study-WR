# Weekly AI Review Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a daily/weekly AI review mode so the review page can generate either a single-day recap or a rolling 7-day recap from the same button.

**Architecture:** Keep the change localized to `index.html`. Add a small review-mode switch in the review section, extend the data-summary helper so it can aggregate either one day or a closed date range, then branch the AI prompt based on mode. Reuse the existing plain-text formatting and copy/clear controls so the output pipeline stays unchanged.

**Tech Stack:** Vanilla HTML, CSS, JavaScript, browser `localStorage`, existing OpenAI-compatible chat/completions flow.

---

### Task 1: Extend the review section UI

**Files:**
- Modify: `index.html:13703-13767`

- [ ] **Step 1: Add the review mode control and weekly hint text**

Insert a compact mode selector next to the existing review date field and keep the current AI buttons unchanged.

```html
<div class="form-row">
  <div class="field">
    <label for="reviewDate">复盘日期</label>
    <input type="date" id="reviewDate" />
  </div>
  <div class="field">
    <label for="reviewMode">复盘模式</label>
    <select id="reviewMode">
      <option value="daily">每日</option>
      <option value="weekly">每周</option>
    </select>
  </div>
</div>
<div class="small muted" id="reviewModeHint" style="margin-top: 6px"></div>
```

Use the existing field styling already present in the section; do not add new layout primitives unless the current spacing breaks.

- [ ] **Step 2: Verify the markup fits the current card layout**

Open the page and confirm the review card still reads cleanly on desktop and narrow widths, with the new mode selector aligned to the existing date input.

- [ ] **Step 3: Commit the UI-only change**

```bash
git add index.html
git commit -m "feat: add review mode selector UI"
```

---

### Task 2: Add date-range summary helpers

**Files:**
- Modify: `index.html:16307-16334`

- [ ] **Step 1: Write the failing helper shape in place**

Add a new function that accepts a start date and end date and returns the same summary shape as the current single-day helper, but aggregated across a closed interval.

```js
function buildDataSummaryForRange(startDate, endDate) {
  const plans = app.state.plans
    .filter((p) => p.dueDate >= startDate && p.dueDate <= endDate)
    .map((p) => ({
      title: p.title,
      subject: subjectName(p.subjectId),
      estimatedPomodoros: p.estimatedPomodoros,
      completed: p.completed,
      dueDate: p.dueDate,
    }));
  const sessions = app.state.sessions
    .filter((s) => s.date >= startDate && s.date <= endDate)
    .map((s) => ({
      subject: subjectName(s.subjectId),
      duration: formatDuration(sessionDurationSeconds(s)),
      date: s.date,
    }));
  const mistakes = app.state.mistakes
    .filter((m) => m.date >= startDate && m.date <= endDate)
    .map((m) => ({
      subject: subjectName(m.subjectId),
      questionType: m.questionType || "",
      summary: m.summary,
      reason: m.reason,
      tags: m.tags,
      date: m.date,
    }));
  return {
    dateRange: { startDate, endDate },
    totalStudyMinutes: totalMinutesForRange(startDate, endDate),
    plans,
    sessions,
    mistakes,
  };
}
```

- [ ] **Step 2: Add the supporting total calculation for the range**

Implement a closed-interval minutes aggregator that mirrors the existing single-day calculation and sums session durations across all dates in the interval.

```js
function totalMinutesForRange(startDate, endDate) {
  return app.state.sessions
    .filter((s) => s.date >= startDate && s.date <= endDate)
    .reduce((sum, s) => sum + sessionDurationSeconds(s), 0) / 60;
}
```

If the project already has a session/date helper used by `sessionsOn(date)` or `mistakesOn(date)`, reuse that pattern rather than introducing a second way to resolve dates.

- [ ] **Step 3: Verify the new helper returns the same object shape plus range metadata**

Call the helper from the console with a known range and confirm it returns `plans`, `sessions`, `mistakes`, and a usable date-range field for prompt construction.

- [ ] **Step 4: Commit the data helper change**

```bash
git add index.html
git commit -m "feat: add ranged review summary helper"
```

---

### Task 3: Branch AI review generation by mode

**Files:**
- Modify: `index.html:16404-16437`
- Modify: `index.html:16512-16588`

- [ ] **Step 1: Write the mode-aware generation logic**

Replace the single-date path in `generateAiReview()` with a small branch that reads `reviewMode` and either keeps the current single-day summary or converts the chosen date into a rolling 7-day window.

```js
async function generateAiReview() {
  const target = $("aiReviewResult");
  const plainTextTarget = $("aiReviewPlainText");
  const copyBtn = $("copyAiReviewBtn");
  target.textContent = "";
  plainTextTarget.value = "AI 复盘生成中...";
  copyBtn.disabled = true;
  try {
    const date = $("reviewDate").value || todayISO();
    const mode = $("reviewMode").value || "daily";
    const summary =
      mode === "weekly"
        ? buildDataSummaryForRange(addDaysISO(date, -6), date)
        : buildDataSummary(date);
    const content = await callAi([
      {
        role: "system",
        content:
          mode === "weekly"
            ? "你是公务员考试备考教练。请基于最近 7 天数据输出纯中文纯文本周复盘建议，使用分点结构，不要使用 Markdown、不要使用标题符号 #、不要使用星号、不要使用粗体。层级请按‘一、二、三……’、‘（一）（二）（三）……’、‘1、2、3……’、‘(1)(2)(3)……’ 这几种形式之一组织。"
            : "你是公务员考试备考教练。请基于数据输出纯中文纯文本复盘建议，使用分点结构，不要使用 Markdown、不要使用标题符号 #、不要使用星号、不要使用粗体。层级请按‘一、二、三……’、‘（一）（二）（三）……’、‘1、2、3……’、‘(1)(2)(3)……’ 这几种形式之一组织。",
      },
      {
        role: "user",
        content:
          mode === "weekly"
            ? `请根据以下最近 7 天备考数据生成周复盘，结构包括：本周完成情况、本周主要问题、下周改进重点、需要关注的科目 / 错题。数据：${JSON.stringify(summary, null, 2)}`
            : `请根据以下备考数据生成复盘，结构包括：完成情况、问题诊断、改进措施、明日重点。数据：${JSON.stringify(summary, null, 2)}`,
      },
    ]);
    const plainText = formatReviewPlainText(content);
    target.textContent = "";
    plainTextTarget.value = plainText;
    copyBtn.disabled = !plainText.trim();
    $("clearAiReviewBtn").disabled = !plainText.trim();
  } catch (err) {
    const message = err.message || String(err);
    target.textContent = "";
    plainTextTarget.value = message;
    copyBtn.disabled = !message.trim();
    $("clearAiReviewBtn").disabled = !message.trim();
  }
}
```

Keep the existing plain-text formatter and copy/clear buttons exactly as they are.

- [ ] **Step 2: Add the helper for date subtraction**

If no reusable date helper already exists for subtracting days from an ISO string, add one small helper near the other date utilities.

```js
function addDaysISO(isoDate, deltaDays) {
  const date = new Date(`${isoDate}T00:00:00`);
  date.setDate(date.getDate() + deltaDays);
  return toISODate(date);
}
```

- [ ] **Step 3: Update the mode hint in the UI bind/render path**

Set the hint text based on the selected mode so the user can see that daily mode is single-day and weekly mode is a rolling 7-day window.

```js
function renderReviewModeHint() {
  const mode = $("reviewMode").value || "daily";
  $("reviewModeHint").textContent =
    mode === "weekly"
      ? "每周模式会按所选日期往前回溯 7 天。"
      : "每日模式只复盘所选日期当天的数据。";
}
```

Wire this helper into the existing render flow and change events for `reviewMode` and `reviewDate`.

- [ ] **Step 4: Verify both modes hit the right summary path**

Test the button twice:
- daily mode should keep the current single-day output and existing plain-text behavior
- weekly mode should include data from the previous 6 days plus the selected date

- [ ] **Step 5: Commit the generation change**

```bash
git add index.html
git commit -m "feat: support weekly AI review generation"
```

---

### Task 4: Verify the review flow end to end

**Files:**
- Modify: none
- Test: browser/manual verification against `index.html`

- [ ] **Step 1: Open the app and verify the daily path still works**

Confirm the review card loads, the daily mode is selected by default, and generating AI review still populates the plain-text box and copy button.

- [ ] **Step 2: Switch to weekly mode and verify the rolling range**

Pick a date with known sessions, generate a weekly review, and confirm the prompt summary includes sessions and mistakes from the prior 6 days plus the selected date.

- [ ] **Step 3: Verify the clear/copy controls still work**

Copy the generated text, clear it, and confirm the buttons and textarea state reset correctly.

- [ ] **Step 4: Run the lightweight syntax check if available**

Extract the inline script and validate it with Node or the repo's existing approach for checking `index.html` script syntax.

- [ ] **Step 5: Report the changed files and verification results**

Summarize the final behavior and mention any manual browser checks that were performed.

---

### Spec coverage check

- Daily mode unchanged: Task 3 and Task 4
- Weekly mode with 7-day rolling window: Task 2 and Task 3
- Same AI button and plain-text pipeline: Task 1 and Task 3
- No new storage or backend changes: all tasks stay in `index.html`
- Verification of behavior: Task 4
