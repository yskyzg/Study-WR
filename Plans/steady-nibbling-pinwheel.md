# Plan: Add Question Type Management and Enhanced Trends

## Context

The app currently supports subjects, mistake records, localStorage persistence, Chart.js dashboard charts, and second-level study durations. The requested change adds a new subject-scoped concept: question types. These types must be managed inside each subject, dynamically used when recording mistakes, and used to filter mistake lists and charts. The dashboard also needs stronger trend analysis: a new mistake-added trend chart and a subject-filtered study-duration chart with session detail tooltips.

The implementation should remain a single-file, directly runnable app in `index.html`, using the existing localStorage state key and current glassmorphism UI style.

## Critical File

- `index.html`

## Recommended Implementation

### 1. Extend the data model and migration

Update subject data to include a `types` array:

```js
{ id, name, color, types: [] }
```

Changes:
- Update `defaultSubjects()` so preset subjects include practical default type lists where appropriate.
- Add a normalization helper, for example `normalizeSubject(subject)`, used by `loadState()` to ensure every subject has `types: []` even for old localStorage data.
- Keep existing mistakes compatible: old mistakes without `questionType` should display as no type / 未标注.
- New mistake records should save:

```js
questionType: '逻辑填空'
```

Keep it as the type name string, scoped by `subjectId`, matching the user requirement that deleted types should not disappear from existing mistake records.

### 2. Add subject-embedded question type management in Settings

Modify `renderSubjectList()` so each subject card includes:
- subject pill and color
- type tags/list under the subject
- `管理题型` button
- existing edit/delete subject buttons

Add a compact modal or inline panel for managing one subject’s types:
- list existing types
- add input
- edit button per type
- delete button per type

Add helpers:
- `getSubjectTypes(subjectId)`
- `typeExists(subject, typeName, exceptOldName = '')`
- `addQuestionType(subjectId, typeName)`
- `renameQuestionType(subjectId, oldName, newName)`
- `deleteQuestionType(subjectId, typeName)`

Rules:
- Trim input names.
- Empty type names are rejected.
- Duplicate type names within the same subject are rejected.
- Deleting a type only removes it from `subject.types`; existing mistakes keep their `questionType` string.
- Save to localStorage and re-render relevant UI after changes.

### 3. Enhance mistake recording with dynamic type selection

Modify the mistake section filter HTML:
- Keep subject filter.
- Add a type filter next to it.
- Type filter options depend on selected subject.

Modify `openMistakeModal()`:
- Add a `questionType` select after subject selection.
- When subject changes, repopulate type options from that subject.
- Type select options:
  - empty option: `请选择题型`
  - each current type in the subject
  - final option: `+ 添加新题型`
- When `+ 添加新题型` is selected, prompt for a type name using a simple `prompt()` or a compact existing modal flow. Prefer `prompt()` for minimal code and fast interaction.
- New type is saved into the selected subject’s `types`, then immediately selected in the mistake modal.
- On save, include `questionType` in the mistake payload.

Modify `renderMistakes()`:
- Apply both filters:
  - subject `all` or concrete `subjectId`
  - type `all` or concrete question type
- Show type tag in each mistake item when `m.questionType` exists.
- If a stored mistake has a deleted type, still display the tag, but it will not be available in filter dropdowns unless it exists in current subject types.

### 4. Add new mistake trend line chart

Modify dashboard visualization area to add a new card:
- title: `错题新增趋势`
- top filters:
  - `mistakeTrendSubjectFilter`: all subjects / concrete subject
  - `mistakeTrendTypeFilter`: all types / current subject’s types
- chart canvas: `mistakeTrendChart`
- optional empty-state text element shown when no data exists.

Add state fields if needed:

```js
app.mistakeTrendSubject = 'all'
app.mistakeTrendType = 'all'
```

Or read directly from DOM filters during render.

Add helpers:
- `populateQuestionTypeSelect(select, subjectId, options)`
- `getMistakeTrendSeries(days)`

Chart behavior:
- X-axis: dates in current range (`app.chartRange`, currently 7/30 days).
- Y-axis: count of mistakes on that date after subject/type filters.
- Cold color line matching existing UI, e.g. cyan-blue.
- Empty state: if all values are 0, show `暂无符合条件的错题数据` while still rendering a flat/empty chart or hide canvas.

### 5. Enhance study duration trend chart

Modify existing `学习趋势` card:
- Add subject filter select, default `全部科目`.
- Keep existing 7/30 range chips.

Update `getDailySeries(days)` to accept subject filter:

```js
getDailySeries(days, subjectId = 'all')
```

It should sum `durationSeconds` for matching sessions and convert to minutes for chart values.

Session detail tooltip:
- Use Chart.js tooltip callbacks for the daily line chart.
- For the hovered date, list matching sessions from that date and subject filter.
- Format each line using `createdAt` time, subject name, and `formatDuration(sessionDurationSeconds(session))`.
- If filter is `all`, include subject name in each detail line.
- If no matching sessions, show `暂无计时明细`.

Because Chart.js native tooltips are callback-based and easy to keep single-file, prefer using the built-in tooltip label callbacks rather than building a custom DOM floating card unless native rendering proves insufficient.

### 6. Keep dashboard and AI summaries consistent

Update `buildDataSummary(date)` so mistakes include `questionType`.

Review/dashboard summaries do not need major structural changes, but chart calculations should use `durationSeconds` through the existing `sessionDurationSeconds()` helper.

### 7. Event binding

Update `bindEvents()`:
- Subject filter for study trend: re-render charts on change.
- Mistake list subject filter: repopulate mistake type filter then render mistakes.
- Mistake list type filter: render mistakes.
- Mistake trend subject filter: repopulate trend type filter then render charts.
- Mistake trend type filter: render charts.
- `管理题型`, type edit, type delete actions through existing document-level event delegation.

### 8. Styling

Add small CSS utilities consistent with existing style:
- type tag rows inside subject cards
- compact management rows for question types
- chart filter rows
- empty chart state text

Do not change the overall theme or restructure unrelated modules.

## Verification

After implementation:

1. Run inline JavaScript syntax verification with Node `vm.Script`.
2. Browser manual checks:
   - Existing localStorage data loads without errors.
   - All old subjects gain `types: []` automatically.
   - Subject settings can add/edit/delete question types.
   - Duplicate type names in one subject are rejected.
   - Mistake modal loads type options after subject selection.
   - `+ 添加新题型` adds a type and selects it immediately.
   - Mistake save persists `questionType`.
   - Mistake list displays type tags.
   - Mistake list filters by subject and type.
   - Deleting a type removes it from dropdowns but existing mistake cards still show the stored type tag.
   - Mistake trend chart updates when subject/type filters change.
   - Study duration trend chart updates when subject filter changes.
   - Study chart tooltip shows per-session details with time, subject, and duration.
3. Confirm no unrelated behavior regressed:
   - Pomodoro countdown/count-up still records sessions.
   - Dashboard cards still update.
   - Plans/reviews/settings still save.
