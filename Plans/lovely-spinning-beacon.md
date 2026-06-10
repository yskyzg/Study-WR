# 计划：AI 聊天多会话 + 持久化

## 背景

当前问题：
1. `aiChatHistory` 全量发送，无上限保护，长对话会触发 API context length 错误
2. 刷新页面后聊天记录全部丢失（从未持久化）
3. 只有一个对话，无法区分不同话题

目标：
- 支持多个独立对话会话（新建、切换、删除）
- 每次发消息后自动保存到 localStorage
- 刷新后恢复上次的对话状态

---

## 数据结构

新增 localStorage key：`aiChatSessions`

```js
// 存储格式
[
  {
    id: "sess_1234567890",
    title: "新对话",          // 自动取第一条用户消息前 15 字
    createdAt: 1234567890,   // Date.now()
    history: [...],          // 等价于原 aiChatHistory
    summary: ""              // 等价于原 aiChatSummary
  },
  ...
]
```

图片（base64）不持久化，保存时从 history 中剥离 `image_url` 部分，避免 localStorage 超额。

---

## 变量改动

原变量保留（作为"当前会话的工作副本"）：
- `aiChatHistory` — 仍代表当前会话的消息列表
- `aiChatSummary` — 仍代表当前会话的摘要

新增：
```js
const AI_CHAT_SESSIONS_KEY = "aiChatSessions";
let aiSessions = [];
let aiCurrentSessionId = null;
```

---

## 新增函数

```
loadSessions()        → 从 localStorage 读取会话列表
saveSessions()        → 序列化整个 aiSessions 写入 localStorage
saveCurrentSession()  → 把 aiChatHistory/aiChatSummary 写回当前会话再 saveSessions
createNewSession()    → 新建会话，压入 aiSessions 头部，切换到它，清空聊天框
switchSession(id)     → saveCurrentSession → 加载目标会话 → 渲染聊天框 → 更新列表
deleteSession(id)     → confirm → 从数组删除 → 若删当前则切到第一个（或新建）
updateSessionTitle(text) → 第一次发消息时把 "新对话" 改为 text.slice(0,15)
renderSessionList()   → 清空面板中的 .ai-session-item → 重新渲染
renderChatBox(sess)   → 清空 #aiChatBox → 逐条调用 appendChatBubble 重建气泡
```

修改现有函数：
- `sendAiChat` → 发消息后调用 `updateSessionTitle` + `saveCurrentSession` + `renderSessionList`
- `compressAiChat` → 压缩后调用 `saveCurrentSession`
- 清空按钮事件 → 清空 history/summary → `saveCurrentSession` → 重置 title → `renderSessionList`
- `initAiChatSection` → 加载会话 → 若无则 `createNewSession` → 否则 `switchSession(aiSessions[0].id)` → 绑定面板事件

上限：超过 50 个会话时，`createNewSession` 自动删掉最老的一条。

---

## HTML 改动（仅内置 AI 解答卡片内部）

在现有内容外层加一个 flex 布局容器：

```
.card
  header（标题 + 清空当前对话按钮）       ← 不变
  .ai-chat-layout                         ← 新增 flex 行
    .ai-sessions-panel#aiSessionsPanel    ← 新增左侧列表
      button#aiNewSessionBtn "+ 新对话"
      .ai-session-item（×N，JS 渲染）
    div.ai-chat-main（flex:1）             ← 原有内容包裹
      .ai-ctx-bar
      提示词选择行
      #aiChatBox
      .ai-chat-input-row
```

---

## CSS 新增样式

在现有 `.ai-ctx-summary-badge` 后追加：

```css
.ai-chat-layout { display:flex; gap:0; }
.ai-sessions-panel { width:190px; flex-shrink:0; display:flex; flex-direction:column;
  gap:4px; border-right:1px solid rgba(148,163,184,.2); padding-right:10px; margin-right:12px;
  max-height:calc(65vh + 60px); overflow-y:auto; }
.ai-sessions-new-btn { ... }   /* 新建按钮 */
.ai-session-item { cursor:pointer; padding:7px 8px; border-radius:10px; display:flex;
  align-items:flex-start; gap:4px; border:1px solid transparent; }
.ai-session-item:hover { background:rgba(99,153,218,.1); }
.ai-session-item.active { background:rgba(99,153,218,.15); border-color:rgba(99,153,218,.3); }
.ai-session-title { font-size:13px; ... 单行截断 }
.ai-session-date { font-size:11px; color:var(--muted); }
.ai-session-del { ... 小×按钮 }
/* 小屏响应式 */
@media(max-width:640px) { .ai-chat-layout { flex-direction:column; }
  .ai-sessions-panel { width:100%; max-height:120px; border-right:none;
    border-bottom:1px solid rgba(148,163,184,.2); padding:0 0 8px; margin:0 0 10px; } }
```

---

## 修改文件

只修改 `e:/code/Review/index.html`，涉及：
1. CSS 块（约 13480 行后）— 追加新样式
2. HTML 块（约 14285–14331 行）— 重构卡片内部布局
3. JS 块（约 17412–17734 行）— 变量声明 + 新增函数 + 修改现有函数 + initAiChatSection

---

## 验证

1. `node -e "new vm.Script(...)"` 语法检查
2. 浏览器打开，控制台无报错
3. 发一条消息 → 刷新页面 → 消息仍在
4. 新建对话 → 切换回旧对话 → 内容正确
5. 删除对话 → 自动切换到其他对话
6. 超过 50 个会话时最老的被删除
