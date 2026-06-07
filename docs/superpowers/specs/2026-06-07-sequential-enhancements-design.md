# Design: Sequential Enhancements for Plans, Reviews, and Accuracy Analytics

## Scope

This design extends the existing single-file civil service study app in three sequential modules:

1. Plan module: question type selection and monthly calendar statistics.
2. Review module: subject/type review blocks and export.
3. Mistake module: accuracy fields, statistics, and visualization.

The app remains a directly runnable single-file application in `index.html`, using native HTML/CSS/JavaScript, Chart.js from CDN, and browser `localStorage` under the existing key `civilServiceExamReviewPomodoroStateV1`.

## Non-goals

- No backend, database, login, cloud sync, or build tooling.
- No new framework or package dependency.
- No change to the existing storage key.
- No removal of existing dashboard, timer, plan, mistake, review, subject, or AI settings features.

## Existing Data Principles

Question types continue to live under subjects:

```js
subjects: [
 { id, name, color, types: ['逻辑填空', '片段阅读'] }
]
```

Records that store a question type keep the type name string. If a subject type is later deleted, existing plan, mistake, or review records keep the old string, but deleted types no longer appear in current type dropdowns.

## Module1: Plan Enhancements

### Data changes

Each plan gains an optional `questionType` field:

```js
{
 id,
 title,
 subjectId,
 questionType: '',
 dueDate,
 completed,
 createdAt
}
```

For old plans, missing `questionType` is treated as an empty string and displayed as `无`.

### Plan form behavior

The plan create/edit UI adds a question type select after the subject select.

- The type select is populated from the selected subject's `types` array.
- The first option is `无` with empty value.
- When the subject changes, the type select refreshes and resets to `无` unless editing an existing plan with a matching stored type.
- If a stored type no longer exists, the edit form may still show it as a preserved option so the user can save without silently losing data.

### Plan list behavior

Each plan card/list row displays a compact type tag:

- If `questionType` exists, show the type name.
- Otherwise show `题型：无` or a muted `无` tag, matching the surrounding style.

### Monthly calendar view

The plan page adds a monthly calendar card.

Calendar state is derived from plans only and is not stored separately.

The calendar includes:

- Current month label.
- Previous month button.
- Next month button.
- Weekday header.
- Date grid.
- Status dot in each date cell.

Status rules:

| Date state | Indicator |
| --- | --- |
| No plans | Blank |
| All plans completed | Green dot |
| Some completed and some unfinished | Yellow dot |
| At least one plan exists and none are completed, or unfinished work remains | Red dot |

Clicking a date selects it and renders a detail list below the calendar.

The selected-date detail list shows:

- Plan title.
- Subject name.
- Question type or `无`.
- Completion state.
- Quick toggle to mark complete / cancel complete.

Toggling a plan updates `localStorage`, re-renders the calendar, selected-date detail, plan list, and dashboard summaries that depend on plans.

## Module2: Review Enhancements

### Data changes

Reviews become normalized objects keyed by date:

```js
reviews[date] = {
 content: '整体每日复盘内容',
 subjectReviews: [
 {
 id,
 subjectId,
 questionType: '',
 content: '专项复盘内容'
 }
 ]
}
```

Backward compatibility:

- If an old review value is a string, migrate it to `{ content: oldValue, subjectReviews: [] }`.
- If an old review object lacks `subjectReviews`, treat it as an empty array.

### Review editor behavior

The review editor keeps the existing overall daily review textarea and template behavior.

Below the overall review, add a subject/type review section:

- `添加科目复盘` button.
- Each block contains subject select, type select, textarea, and delete button.
- Type select depends on the selected subject.
- Type is optional; empty value displays as `无`.
- One day can contain multiple subject/type review blocks.

Saving a review stores both the overall content and all subject review blocks.

### Review history behavior

Review history entries can expand to show:

- Overall review content.
- Subject/type review blocks.
- Export actions.

Each subject review block displays subject name, type tag, and content. Missing or deleted subjects/types are displayed safely using stored names when possible, or a fallback label.

### TXT export

TXT export generates a local download with a clear date title and separators.

Example structure:

```text
2026-06-07复盘
====================

【整体复盘】
...

【分科/题型复盘】
--- 言语理解与表达 /逻辑填空 ---
...
```

The export includes both overall content and subject/type blocks.

### PDF export

PDF export uses browser-native printing:

- Render a clean print-friendly area or reuse an export preview area.
- Apply print-specific CSS to hide app chrome and reduce visual noise.
- Call `window.print()`.
- The user can save as PDF from the browser print dialog.

No third-party PDF library is introduced.

## Module3: Mistake Accuracy Statistics

### Data changes

Each mistake may include:

```js
{
 totalQuestions:0,
 wrongQuestions:0
}
```

Missing or zero values mean the record does not participate in accuracy statistics.

### Mistake form behavior

The mistake modal adds two numeric inputs:

- 本次做题总数
- 本次错题数

Validation:

- Both values are optional as a pair.
- If entered, total must be greater than0.
- Wrong count must be greater than or equal to0.
- Wrong count must be less than or equal to total count.

If validation fails, show a friendly alert and do not save.

### Mistake list behavior

Each mistake card displays accuracy information when available:

```text
正确率：18/20，90%
```

If no valid accuracy fields exist, display a muted fallback such as:

```text
正确率：未记录
```

### Aggregate statistics

Statistics are computed from mistake records with valid `totalQuestions` and `wrongQuestions`.

Supported dimensions:

1. All subjects.
2. Single subject.
3. Specific type under a selected subject.

For each dimension, calculate:

- Cumulative total questions.
- Cumulative wrong questions.
- Cumulative correct questions.
- Overall accuracy percentage.

Formula:

```js
correct = totalQuestions - wrongQuestions
accuracy = correct / totalQuestions
```

### Visualization

Use a bar chart first because it is clearer for comparing weak subjects/types.

The chart supports two granularities:

1. By subject: one bar per subject.
2. By type: after selecting a subject, one bar per current type under that subject, plus optionally an `未标注` bucket for records without type.

Empty state:

- If no valid accuracy data exists, show `暂无正确率数据`.
- Keep the chart area visually consistent with other dashboard/chart cards.

## UI Style

New UI elements should match the current cold glassmorphism style:

- Translucent cards.
- Pale blue / light purple / cyan-blue accents.
- Rounded controls.
- Soft shadows.
- Compact tags and status dots.
- Responsive layout that stacks cleanly on small screens.

## Implementation Order

Implement and verify in this order:

1. Plan type selection and calendar statistics.
2. Review subject/type blocks and export.
3. Mistake accuracy fields, aggregate stats, and chart.

This sequence reduces risk because each module can be validated independently before moving to the next.

## Verification

After editing `index.html`, run JavaScript syntax verification by extracting the inline script and checking it with Node `vm.Script`.

Manual browser checks:

### Plans

- Old plans without `questionType` still render.
- New plan can select subject and optional type.
- Plan list shows type tag or `无`.
- Calendar defaults to current month.
- Previous/next month buttons work.
- Date status dots match plan completion state.
- Clicking a date shows that date's plans.
- Quick complete/cancel updates localStorage and calendar state.

### Reviews

- Old review strings load correctly.
- Overall daily review still saves.
- Subject review blocks can be added, edited, deleted, and saved.
- Type dropdown updates when subject changes.
- History expands and shows subject/type details.
- TXT export downloads readable content.
- PDF export opens browser print flow with clean print style.

### Mistake accuracy

- Mistake modal validates total/wrong counts.
- Wrong count cannot exceed total count.
- Mistake card displays per-record accuracy.
- Aggregate total, wrong, correct, and accuracy values update.
- Chart switches between subject and type granularity.
- Empty state appears when no valid accuracy data exists.

### Regression checks

- Dashboard cards still render.
- Existing charts still update.
- Pomodoro countdown/count-up still records sessions.
- Subject/type management still works.
- Existing mistake type filtering still works.
- AI settings remain local-only and unchanged.
