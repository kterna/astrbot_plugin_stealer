# WebUI 优化与收藏功能开发总结

**日期**: 2026-06-07（收藏功能上线与第二轮优化）
**分支**: master（本地已完成，待推送）
**状态**: 功能完成，测试通过（126/126）

---

## 零、收藏功能上线收尾（性能 + 接口验证 + 冗余清理）

### 1. 上线前修复

| 问题 | 原因 | 修复 |
|------|------|------|
| 侧边栏 hover 在 1024px 断点崩溃 | `--sidebar-expanded` 变量未定义 | 在 `:root` 中添加 `--sidebar-expanded: 220px` |
| `.stats-bar` 丢失圆角和居中 | 重复定义覆盖了第一个 | 删除第二个重复定义 |
| 批量选择 UI 不更新 | `Set` 在 `ref()` 中 `.add()`/`.delete()` 不触发 Vue 响应式 | 全部改为 `new Set()` 重新赋值 |
| 不可达 catch 块 | `fetchImages` 中两个连续 catch | 删除第二个 |
| `batchUploading` 按钮不禁用 | 未暴露到模板 return 块 | 添加到 return |
| `theme`/`isDarkTheme` 双状态不同步 | 两个 ref 手动同步 | `theme` 改为 `computed` |
| 收藏分类计数一直为 0 | `/images` 响应未返回 `favorite_count` | 作为收藏功能上线保障，后端补充收藏总数，前端读取真实计数 |
| DB 分页列表丢失收藏状态 | `get_emojis_paginated` SELECT 未包含 `is_favorite` | 作为收藏功能上线保障，查询补充 `is_favorite`/`last_used_at` 字段 |
| 分类/图片删除方法不一致 | 前端用 `DELETE`，后端只注册 `POST` | 前端统一改为 `POST` |
| 页码越界重试被请求锁拦截 | `fetchImages` 递归跳转最后一页时 `isFetching` 未释放 | 递归请求前释放请求锁 |

### 2. 性能优化

| 优化项 | 原状态 | 新状态 |
|--------|--------|--------|
| `.item-info` backdrop-filter | 每张卡片创建 compositing layer | 移除，改用纯半透明背景 |
| `.stat-value` 无限动画 | 所有 stat-value 永久播放 pulse | 移除（收益低） |
| `fadeIn` 动画 | 每次翻页重播 | 添加 `animation-fill-mode: both` |
| theme-toggle 三重无限动画 | 3个动画同时运行 | 仅 hover 时播放 |
| blob URL 内存泄漏 | `createObjectURL` 无 `revokeObjectURL` | 清除/覆盖前 revoke |
| Observer/事件监听器泄漏 | 无 `onUnmounted` 清理 | 添加完整清理逻辑 |
| `imageDataUrls` 无界增长 | 翻页永不释放 | 翻页时清理非当前页条目 |
| `originalDataUrls` 泄漏 | 关闭预览不清理 | `closePreview` 时清理 |

### 3. 冗余清理

| 类别 | 删除行数 | 详情 |
|------|----------|------|
| Login CSS 死代码 | ~490行 | 模板中未使用任何 login 类 |
| `.theme-flash` 死代码 | ~18行 | 模板中未使用 |
| `.toast` 死代码 | ~18行 | 只用了 `toast-notification` |
| 工具类死代码 | ~5行 | `.text-gold`/`.text-bright`/`.text-muted`/`.font-cinzel` |
| 被覆盖的 scrollbar 规则 | ~18行 | 第一组被第二组完全覆盖 |
| 重复 `@keyframes shimmer` | ~3行 | 与已有定义完全相同 |
| 过期 vendor prefix | ~5行 | `-webkit-user-select`/`-webkit-appearance`/`-webkit-overflow-scrolling` |
| 确认未使用的 CSS | 少量 | `.spinner`、旧 `.toast-container`、未引用 `pulse`、泛用 `.skeleton` |
| **CSS 总计** | **~621行** | **2901→2280 行（-21%）** |

### 4. JS 代码质量

| 改进项 | 详情 |
|--------|------|
| 移除冗余 `Content-Type` 头 | 10处，`apiFetch` 已自动处理 |
| 提取 `refreshView()` | 替代 4处重复的 `fetchImages+fetchStats` |
| 合并 `prevImage`/`nextImage` | 提取为 `navigateImage(direction)` |
| `searchTimeout` 改为普通变量 | 无需响应式的定时器句柄 |
| 移除死代码 `lastAnalysisResult` | 赋值后从未读取 |
| `normalizeCategories()` | 兼容 API 返回数组或对象格式，避免 mock/旧接口导致分类渲染异常 |

### 5. 本轮验证

| 项目 | 结果 |
|------|------|
| 后端/核心测试 | `pytest tests -q`：126 passed，3 个 Pydantic v2 既有弃用警告 |
| 语法检查 | `python -m py_compile plugin_api.py core/db/database_service.py` 通过；`node --check pages/表情管理/app.js` 通过 |
| 桌面 WebUI | 1280×720 mock bridge 预览：列表、统计、健康状态、搜索、详情弹窗、收藏切换正常，控制台无 error |
| 移动 WebUI | 390×844：侧边栏隐藏、移动分类下拉显示、无横向溢出，控制台无 error |

---

## 一、已完成工作

### 1. 后端：收藏系统

| 文件 | 变更 |
|------|------|
| `core/db/database_service.py` | SCHEMA_VERSION 1→2，新增 `is_favorite` 字段（默认 0），支持 `favorite_only` 筛选查询 |
| `plugin_api.py` | 新增 `POST /images/batch-favorite` 接口；`_build_image_item` 暴露 `is_favorite`/`use_count`/`last_used_at`；`handle_update_image` 支持更新收藏状态；`handle_list_images` 支持 `favorite_only` 参数 |
| `core/search/emoji_selector.py` | 随机模式：收藏项权重 ×3（加权随机选择） |
| `core/search/emoji_smart_select_service.py` | 智能模式：收藏项 bonus +0.3 |
| `core/events/event_handler.py` | 容量清理淘汰时过滤 `is_favorite=1`，收藏表情包永不自动清理 |

### 2. 前端：视觉现代化

| 改造项 | 原状态 | 新状态 |
|--------|--------|--------|
| 整体风格 | 中世纪典籍（厚重边框+金色角标） | 现代玻璃拟态（backdrop-filter + 金色主题保留） |
| 头部 | 渐变+装饰角标，高度不定 | 固定 56px，`backdrop-filter: blur(16px)` |
| 侧边栏 | 220px 实色+厚边框 | 180px 固定，分类标题金色加粗，项间渐变分隔线 |
| 卡片 | 直角+厚边框+强阴影 | 圆角 12px，微妙边框，hover 金色 glow + 上浮 |
| 加载状态 | spinner | 骨架屏（shimmer 动画） |
| 主题切换 | 复杂旋转按钮 | 简洁 toggle，无全屏 flash |

### 3. 前端：性能优化

- **动态 pageSize**：根据屏幕宽高计算每页数量（`每行数量 × 可见行数`）
- **LRU 缓存**：50 张缩略图内存缓存，避免重复 base64 请求
- **resize 防抖**：300ms 内只触发一次重载
- **请求锁**：`isFetching` 标志防止重复请求
- **`content-visibility: auto`**：只渲染可见区域卡片
- **Hash 色块占位**：图片加载前显示从 hash 生成的稳定 HSL 颜色

### 4. 功能覆盖补全

| 功能 | 实现 |
|------|------|
| 健康状态指示器 | 头部圆点（绿/黄/红）+ 文字状态 |
| 使用统计 | 详情面板显示 `use_count` 和 `last_used_at` |
| 收藏分类筛选 | 侧边栏"⭐ 收藏"虚拟分类，点击筛选 |
| 移动端适配 | <768px 侧边栏隐藏，工具栏出现分类下拉选择器 |

### 5. 收藏功能完整前端实现

- 卡片右上角星标按钮（hover 显示 / 已收藏常显）
- 详情面板收藏 toggle（⭐ 已收藏 / ☆ 未收藏）
- 批量操作栏"收藏/取消收藏"按钮
- 收藏切换弹跳动画（`@keyframes starPop`）

---

## 二、关键修复记录

| 问题 | 原因 | 修复 |
|------|------|------|
| 图片不显示 | `v-if/v-else` 导致 `<img>` 不在 DOM 中，`IntersectionObserver` 无法触发 | 改观察 `.item-image` 容器而非 `<img>` |
| 图片不显示（第二次） | `v-show` 让 `<img>` 初始 `display:none`，observer 对不可见元素不触发 | 改回 `v-if/v-else`，将 `data-hash` 放容器上 |
| item-info 不显示 | `.item-slot` 未设置 `display:flex`，`flex:1` 不生效，图片撑出容器 | 添加 `display:flex;flex-direction:column` |
| 侧边栏太窄 | 64px 收缩设计，文字被截断 | 改为 180px 固定宽度 |
| 统计不居中 | 头部元素无 flex 分配 | 添加 `.header-right` 包裹右侧元素 |
| 白天模式刺眼 | 纯白卡片 + 冷灰背景 | 改为暖米色主题（`#e8e4dc` 背景，`#faf8f3` 卡片） |
| 白天模式 scope-pill 看不清 | 浅色文字在白色背景上 | 深色文字（`#8b6914`/`#dc2626`/`#16a34a`） |

---

## 三、Git 提交记录

```
361b07c perf(frontend): debounce resize, add fetch lock...
a3b0242 fix(css): flex layout for item-slot, soften light theme...
64d07ba fix(frontend): observe item-image container...
b4ce257 feat(frontend): add mobile category selector...
77f6180 feat(frontend): dynamic pageSize based on screen size...
20ff0a4 fix(frontend): fix image loading deadlock...
f8c2869 fix(css): add missing --bg-card-hover...
0d8d6ec feat(frontend): add LRU cache, health indicator...
766a340 fix(css): add missing --bg-card-hover...
ede65f8 style(css): modernize visual system...
--- (第二轮优化) ---
fix(js): Set reactivity bugs, remove dead code, fix memory leaks
perf(css): remove item-info backdrop-filter, optimize animations
style(css): remove ~620 lines dead code, clean vendor prefixes
refactor(js): extract refreshView, merge navigateImage, remove redundant headers
```

---

## 四、已知问题 / 待细化方向

| 问题 | 优先级 | 建议 |
|------|--------|------|
| 移动端侧边栏完全隐藏 | 中 | 当前仅通过工具栏下拉选择分类，可考虑汉堡菜单 |
| 分类数量多时侧边栏滚动 | 低 | 已支持滚动，但滚动条样式可美化 |
| Light theme 持续调优 | 中 | 当前为暖米色，用户反馈"有些刺眼"，可继续微调饱和度和对比度 |
| ~~批量选择 UI 不更新~~ | ~~高~~ | ✅ 已修复：Set 响应式改为 new Set() 重新赋值 |
| ~~内存泄漏（blob URL / observer）~~ | ~~中~~ | ✅ 已修复：添加 revoke + onUnmounted 清理 |
| ~~侧边栏 hover 崩溃~~ | ~~高~~ | ✅ 已修复：定义 --sidebar-expanded 变量 |

---

## 五、架构决策备忘

1. **虚拟滚动方案**：采用"分页卸载 + LRU 缓存"而非真虚拟滚动（复杂度低，兼容性好）
2. **图片加载策略**：`IntersectionObserver` 观察 `.item-image` 容器（非 `<img>`），解决 `display:none` 死循环
3. **响应式策略**：侧边栏 180px 固定（桌面），768px 以下隐藏 + 工具栏下拉
4. **主题方向**：保留金色主题，现代化改造（非换色）
9. **接口兼容策略**：前端删除类操作与后端注册方法保持 `POST` 一致，减少 bridge 映射差异带来的风险
6. **内存管理**：翻页时清理非当前页 `imageDataUrls`，关闭预览时清理 `originalDataUrls`，blob URL 用前 revoke
7. **Vue 响应式**：`Set` 在 `ref()` 中必须整体替换才能触发更新，禁止 `.add()`/`.delete()`/`.clear()` 原地修改
8. **动画策略**：无限动画仅用于 hover/交互状态，静态元素用 `transition` 替代，减少 GPU 持续消耗
5. **性能策略**：`content-visibility: auto` + `contain: layout style paint` + 防抖 + 请求锁

---

## 六、API 变更

### 新增端点
```
POST /images/batch-favorite
Body: { hashes: string[], favorite: boolean }
Response: { success: true, count: number }
```

### 扩展端点
```
GET /api/images?page=1&size=24&favorite_only=true
POST /api/images/update
Body: { hash: string, is_favorite: boolean }
POST /api/images/delete
POST /api/categories/delete
```

### 响应字段扩展
```json
{
  "hash": "...",
  "is_favorite": true,
  "use_count": 12,
  "last_used_at": 1705312800,
  "favorite_count": 8
}
```

---

## 七、移交说明

**当前分支状态**：`master`，本地已完成，可直接 `git push`

**核心文件**：
- `pages/表情管理/app.css` — 完整视觉系统（约 2280 行，已清理 620 行死代码）
- `pages/表情管理/app.js` — 前端逻辑（约 2090 行）
- `plugin_api.py` — API 层
- `core/db/database_service.py` — 数据层

**测试命令**：`pytest tests -q`（126 passed）
