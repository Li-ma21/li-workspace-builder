---
name: "li-workspace-builder"
description: "Builds a single-file personal workspace with Morandi-color UI, project hierarchy, calendar kanban, Supabase cloud sync, and PWA. Invoke when user wants a personal productivity workspace / kanban / calendar app / 个人工作台 / 看板 / 日历 / 项目管理 / 备忘录 / 个人效率工具 / Todo app."
---

# Personal Workspace Builder (Li 工作台)

构建一个**单文件 HTML + Supabase 云端同步 + PWA 离线可用**的个人工作台，莫兰迪色系，移动端响应式，手机可添加到主屏幕当 App 用。

## 最终交付物

| 文件 | 作用 |
|---|---|
| `index.html` | 全部 UI + 逻辑（内联 CSS + JS） |
| `sw.js` | Service Worker（离线缓存） |

数据存储：Supabase 云端（主）+ localStorage（离线回退）

## 设计规范

### 莫兰迪色系（CSS 变量命名）

```css
--bg-page:        #E8E2D6;   /* 页面背景：米灰 */
--bg-sidebar:      #DCD3C5;   /* 侧边栏：暖灰 */
--bg-card:        #F2EDE3;   /* 卡片：奶白 */
--bg-card-2:      #EAE2D3;   /* 卡片次：浅米 */
--bg-input:       #FBF8F1;   /* 输入框：象牙 */
--accent:         #9C8E7E;   /* 主强调：灰咖 */
--accent-soft:    #C9BBA8;   /* 强调浅：驼灰 */
--accent-green:   #8FA089;   /* 灰绿 */
--accent-blue:    #9AA5B0;   /* 灰蓝 */
--accent-rose:    #C2A095;   /* 灰粉 */
--accent-amber:   #C2A878;   /* 灰琥珀 */
--text-main:      #4A443B;   /* 主文字：深褐灰 */
--text-soft:      #7A7163;   /* 次文字：暖灰 */
--text-mute:      #A99E8E;   /* 弱文字：浅灰 */
--border:         #C8BDA9;   /* 边框：浅米咖 */
--border-soft:    #D9CFBC;   /* 软边框 */
```

### 布局结构

```
┌─────────────────────────────────────────────┐
│  侧边栏（固定）  │      主区域（动态切换）      │
│  ┌─────────┐   │  ┌───────────────────────┐ │
│  │ ▦ 首页总览│   │  │ 首页 / 项目页         │ │
│  ├─────────┤   │  │                       │ │
│  │ 项目1    │   │  │ 首页：                │ │
│  │ 项目2    │   │  │   统计卡 + 日历 +     │ │
│  │ ...      │   │  │   当日详情 + 项目卡片   │ │
│  ├─────────┤   │  │                       │ │
│  │ ☁ 同步态 │   │  │ 项目页：              │ │
│  │ ⤓⤒备份   │   │  │   描述 / 计划 /       │ │
│  │ + 新增项目│   │  │   TODO / 备忘        │ │
│  └─────────┘   │  └───────────────────────┘ │
└─────────────────────────────────────────────┘
```

## 数据模型

```js
state = {
  projects: [{
    id, name, desc,
    plans: [{ id, module, planStart, planEnd, content }],
    todos: [{ id, module, planStart, planEnd, status, text, remark }],
    memos: [{ id, text, invalid }]
  }],
  dailyTasks: [{ id, module, text, active }],   // 全局每日任务
  checkins: { "YYYY-MM-DD": [dailyTaskId,...] },  // 打卡记录
  activeId: null
}
nextId: number
```

**时间字段命名规则**：用 `planStart` / `planEnd`，不用 `start` / `end` / `date`。旧数据迁移时自动：
- `planStart = pl.start || getToday()`，删 `start`
- `planEnd = t.date || ''`，删 `date`

## Supabase 配置

### 数据库建表 SQL

```sql
create table workspace (
  id bigint primary key,
  data jsonb not null,
  updated_at timestamptz default now()
);
alter table workspace disable row level security;
```

### 前端常量

```js
const SUPABASE_URL = 'https://xxx.supabase.co';     // 从 Data API 页面复制
const SUPABASE_KEY = 'eyJ...anon public key...';     // Legacy anon key
const SYNC_ID = 1;                                    // 单用户固定行 id
```

### 云端同步策略

- **启动流程**：先用 localStorage 秒开 → 后台拉云端 → 以云端为准（空云端则推本地）
- **写入策略**：本地立即保存 + 800ms 防抖推云端
- **离线回退**：断网时降级本地模式，联网后自动补推
- **同步状态**：侧边栏底部显示 loading/synced/syncing/offline，点击手动拉云端
- **upsert**：用 `Prefer: resolution=merge-duplicates` 头实现 upsert

```js
// 云端写入（upsert）
async function cloudPush(){
  const body = { id: SYNC_ID, data: { state, nextId }, updated_at: new Date().toISOString() };
  await fetch(SUPABASE_URL + '/rest/v1/workspace', {
    method: 'POST',
    headers: { 'apikey': SUPABASE_KEY, 'Authorization': 'Bearer ' + SUPABASE_KEY,
               'Content-Type': 'application/json', 'Prefer': 'resolution=merge-duplicates' },
    body: JSON.stringify(body)
  });
}
```

## PWA 配置

### index.html head 内联

```html
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="default" />
<meta name="apple-mobile-web-app-title" content="工作台" />
<meta name="mobile-web-app-capable" content="yes" />
<meta name="theme-color" content="#E8E2D6" />

<!-- apple-touch-icon（SVG base64 内联，莫兰迪色圆形 + LI 字样） -->
<link rel="apple-touch-icon" href="data:image/svg+xml;base64,..." />

<!-- manifest（data URL 内联，display=standalone） -->
<link rel="manifest" href='data:application/manifest+json,{"name":"Li 的工作台","short_name":"工作台","display":"standalone","background_color":"#E8E2D6","theme_color":"#E8E2D6","icons":[...]}' />
```

### sw.js（网络优先 + 缓存回退）

```js
const CACHE = 'li-workspace-v1';
self.addEventListener('fetch', e => {
  if(e.request.method !== 'GET') return;
  e.respondWith(
    fetch(e.request).then(res => {
      const copy = res.clone();
      caches.open(CACHE).then(c => c.put(e.request, copy)).catch(()=>{});
      return res;
    }).catch(() => caches.match(e.request).then(r => r || caches.match('./')))
  );
});
```

### 注册 SW

```js
if('serviceWorker' in navigator && location.protocol.startsWith('http')){
  navigator.serviceWorker.register('./sw.js').catch(()=>{});
}
```

注：`file://` 下无法注册 SW，但 PWA 配置保留，部署到 https 后自动生效。

## 功能清单与交互约定

| 模块 | 交互 | 数据持久化 |
|---|---|---|
| **项目创建** | prompt 输入名称 | save() |
| **项目重命名** | 双击侧边栏项目名 或 项目页标题旁 ✎ | save() |
| **项目删除** | 项目页"删除项目"按钮（带确认） | save() |
| **计划新增** | + 新增，默认 planStart=planEnd=今天，自动聚焦模块名 | save() |
| **计划编辑** | 直接在输入框改（模块名 / 开始 / 完成 / 内容） | input 事件触发 save() |
| **TODO 新增** | 底部表单：模块 / 开始 / 完成 / 状态 / 内容，默认开始=完成=今天 | save() |
| **TODO 状态切换** | 点状态徽章循环：未开始→进行中→已完成→取消 | save() + renderTodos() |
| **TODO 编辑** | 双击任务行 | save() |
| **备忘勾选** | 行首圆点点击 = 标记失效（空心 + 删除线） | save() |
| **备忘编辑** | 双击文本 | save() |
| **每日任务** | 全局，不属于项目，日历渲染到哪天就显示哪天 | save() |
| **每日打卡** | 勾选 → 写入 checkins[日期] | save() + renderDayDetail() |
| **日历圆点** | 🔵有 planEnd TODO / 🟡有每日任务 / 🟢当日全部完成 | renderCalendar() |
| **导出备份** | 侧边栏 ⤓ 按钮，下载 JSON | 仅本地下载 |
| **导入恢复** | 侧边栏 ⤒ 按钮，覆盖本地并推云端 | save() |

## 部署到手机 App 形态

1. Netlify Drop（https://app.netlify.com/drop）拖入 `index.html` + `sw.js`
2. 拿到 netlify.app 网址
3. 手机 Safari/Chrome 打开 → 添加到主屏幕
4. 桌面出现莫兰迪色 LI 图标，全屏无地址栏启动

## 关键陷阱与修复

1. **TODO 日历匹配字段**：日历蓝点/当日截止判断必须用 `t.planEnd`，不能用 `t.date`（迁移后 date 已删）
2. **日历圆点 allDone 判断**：TODO 已完成/取消 或 每日任务全部打卡才算 green
3. **Service Worker 不能用 Blob URL 注册**：scope 无法覆盖页面，必须用独立 sw.js 文件
4. **manifest 用 data URL**：避免额外文件；`application/manifest+json` content-type 正确
5. **迁移代码只跑一次**：用 `state.dailyMigratedV1` / `normalizeState()` 幂等保护
6. **侧边栏 active 状态**：activeId=null 代表首页总览，所有项目 activeId 匹配才高亮
7. **renderHome 依赖 DOM**：stats / calendar / projCards 都有独立 render 函数，首页/项目页切换时分别渲染

## 浏览器验证清单

部署后在 `http://localhost:8789/` 起静态服务验证：

- [ ] 首页加载：工作台总览 + 4 统计卡 + 日历 + 当日详情 + 项目卡片
- [ ] 项目页：4 模块全宽，无日历侧栏
- [ ] 计划行：4 输入框（模块名 / 开始 / 完成 / 内容）+ 左侧彩色条
- [ ] TODO 行：模块标签 + 文本 + 日期（开始 → 完成）+ 状态徽章 + 删除
- [ ] 新增计划/TODO：默认日期=今天，自动聚焦
- [ ] 状态循环切换：未开始→进行中→已完成→取消
- [ ] 日历蓝点：有 planEnd 那天显示
- [ ] 项目重命名：双击侧边栏名 / 点 ✎ → 名称同步更新
- [ ] PWA manifest：DevTools Application → Manifest 可见
- [ ] SW 注册：DevTools Application → Service Workers 显示 scope
- [ ] 无红色 JS 报错
