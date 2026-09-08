# Li 工作台 (li-workspace-builder)

一个 **Trae Skill**，用于构建单文件个人工作台应用。

## 功能特性

- 🎨 **莫兰迪色系 UI** — 统一配色方案，温和舒适
- 📁 **项目层级管理** — 多项目独立管理计划、TODO、备忘
- 📅 **全局任务日历** — 首页日历看板 + 当日详情
- ☁️ **Supabase 云端同步** — 主存储，本地 localStorage 离线回退
- 📱 **PWA 离线可用** — Service Worker 缓存，可添加到手机主屏幕
- 💾 **备份/恢复** — JSON 导出导入，数据安全可控

## 使用方法

在 Trae 中加载此 skill 后执行即可生成完整的工作台应用。

## 仓库结构

```
├── SKILL.md              # Skill 定义文件
└── examples/
    ├── index.html         # 运行示例（单文件应用）
    └── sw.js              # Service Worker
```

## 部署建议

1. 将 `index.html` 和 `sw.js` 上传到 Netlify Drop
2. 手机浏览器打开 → 添加到主屏幕
3. 桌面出现莫兰迪色 LI 图标，全屏无地址栏启动
