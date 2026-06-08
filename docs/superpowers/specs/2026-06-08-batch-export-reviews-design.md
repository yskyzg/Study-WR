# 设计文档：历史复盘批量导出

**日期：** 2026-06-08  
**范围：** index.html 单文件，仅修改历史复盘区域

---

## 功能目标

在"历史复盘"区域新增批量选择与导出功能，用户可筛选、勾选多条复盘记录，选择导出格式后打包为 ZIP 下载。

## UI 布局

"历史复盘"卡片从上到下：

1. **筛选 tab**（不变）：全部 / 每日复盘 / 每周复盘
2. **批量操作栏**（新增，始终可见）：
   - 左侧：`☐ 全选` checkbox + `已选 N 条` 文字
   - 右侧：格式 radio（TXT / HTML / 打印为PDF）+ `导出 ZIP` 按钮（无选中时 disabled）
   - PDF 说明小字（位于 radio 下方）：
     > 选择「打印为PDF」时，将导出带打印样式的 HTML 文件，用浏览器打开后按 Ctrl+P（Mac: ⌘P）另存为 PDF。
3. **历史记录列表**：每条记录左侧加复选框；查看 / 展开 / 删除按钮不变

## 文件命名规则

| 格式 | 路径示例 |
|------|---------|
| TXT | `复盘_2026-06-01_daily.txt` |
| HTML | `复盘_2026-06-01_daily.html` |
| 打印为PDF | `pdf/复盘_2026-06-01_daily.html` |

ZIP 包名：`复盘导出_N条_YYYYMMDD.zip`

## 技术方案

- **ZIP 库**：引入 JSZip（CDN，与现有 Chart.js CDN 一致）
- **状态**：`app._selectedReviews`（`Set<string>`，存 reviewKey）；切换筛选 tab 时清空
- **选中逻辑**：复选框 `change` 事件更新 Set，重新渲染批量操作栏状态（已选数、按钮 disabled）
- **全选**：全选 checkbox 根据当前过滤后列表与 Set 的交集判断 indeterminate / checked 状态
- **内容生成**：
  - TXT：复用 `exportReviewTxt` 的文本生成逻辑（抽取为 `buildReviewTxt(review)`）
  - HTML：复用 `exportReviewPdf` 的 HTML 生成逻辑（抽取为 `buildReviewHtml(review)`）
  - 打印为PDF：与 HTML 相同内容，放入 `pdf/` 子目录
- **导出入口**：新增 `batchExportZip()` 函数

## 边界条件

- 切换筛选 tab → 清空已选 Set，重新渲染
- 导出 0 条时按钮 disabled，不可触发
- JSZip 加载失败时 alert 提示

## 不改动的内容

- 现有单条 TXT / PDF 导出按钮
- 番茄钟、仪表盘、错题、计划、设置等其他模块
- localStorage 结构
- 整体视觉风格
