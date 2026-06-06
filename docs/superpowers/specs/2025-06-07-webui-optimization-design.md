# WebUI 优化与收藏功能设计文档

**日期**: 2025-06-07
**项目**: astrbot_plugin_stealer — 表情包管理 WebUI
**主题**: 视觉现代化、加载速度优化、功能覆盖补全、收藏功能实现

---

## 目录

1. [设计目标](#1-设计目标)
2. [视觉设计系统](#2-视觉设计系统)
3. [加载速度优化](#3-加载速度优化)
4. [功能覆盖补全](#4-功能覆盖补全)
5. [收藏功能详细设计](#5-收藏功能详细设计)
6. [实施计划概要](#6-实施计划概要)

---

## 1. 设计目标

### 1.1 问题诊断

当前 WebUI 采用"中世纪典籍"风格（Codex Theme），视觉元素过于厚重：
- 金色装饰边框、装饰角标、多重阴影叠加，导致信息密度低
- 所有元素 CSS transition 全局应用，低端设备掉帧
- 图片加载策略原始，大量 base64 同时渲染造成内存压力
- 前端未覆盖后端全部 API 能力（如健康检查、使用统计）
- 缺少收藏功能（用户核心需求）

### 1.2 目标定义

| 维度 | 当前状态 | 目标状态 |
|------|---------|---------|
| 视觉风格 | 中世纪厚重 | 现代简洁 + 金色主题保留 |
| 首屏加载 | 全量加载 30 张 | 首屏 12~24 张 + 智能预加载 |
| 内存占用 | 随翻页线性增长 | LRU 缓存上限 50 张 |
| 功能覆盖 | ~85% | 100% |
| 收藏功能 | 无 | 完整前后端实现 |

---

## 2. 视觉设计系统

### 2.1 色彩体系

保留天国拯救金色主题，但简化为三层结构：

```css
:root {
  /* 背景层 */
  --bg-base: #0a0d14;
  --bg-elevated: #111520;
  --bg-card: #161b2a;
  
  /* 金色强调 */
  --gold-primary: #d4a853;
  --gold-bright: #f0d78c;
  --gold-dim: #8b7340;
  --gold-glow: rgba(212, 168, 83, 0.15);
  
  /* 功能色 */
  --success: #4ade80;
  --warning: #fbbf24;
  --danger: #f87171;
  --favorite: #fbbf24;
  
  /* 文字层次 */
  --text-primary: #f8fafc;
  --text-secondary: #a1a7b6;
  --text-tertiary: #5e6577;
  
  /* 玻璃效果 */
  --glass-bg: rgba(17, 21, 32, 0.75);
  --glass-border: rgba(212, 168, 83, 0.12);
  --glass-blur: 16px;
}
```

### 2.2 布局改造

| 元素 | 改造前 | 改造后 |
|------|--------|--------|
| 侧边栏 | 220px 宽，实色+厚边框 | 64px 图标导航，悬浮展开 200px 玻璃面板 |
| 头部 | 渐变+装饰角标，高度不定 | 玻璃拟态，固定 56px，统计信息嵌入右侧 |
| 统计栏 | 独立厚重方块 | 小型胶囊标签，整合入头部 |
| 网格 | 固定 160px，gap 16px | 响应式 auto-fill，min 140px，gap 12px |
| 卡片 | 厚边框+强阴影，直角 | 无边框，圆角 12px，悬浮金色 glow |
| 主题切换 | 复杂旋转太阳/月亮按钮 | 简洁 toggle，保留动画但减少体积 |

### 2.3 动效规范

- **页面加载**: 卡片 staggered fade-in（每项延迟 30ms）
- **悬浮**: translateY(-4px) + 金色 glow，150ms ease-out
- **主题切换**: 无全屏 flash，CSS 变量过渡 200ms
- **收藏切换**: 星标缩放弹跳 300ms（50% 时 scale 1.3）
- **全局**: 移除 `* { transition }`，精确限定到 `.item-slot`, `.codex-btn`, `.modal-panel`

### 2.4 卡片收藏按钮设计

```css
.favorite-btn {
  position: absolute;
  top: 8px;
  right: 8px;
  width: 32px;
  height: 32px;
  background: rgba(0,0,0,0.4);
  border: none;
  border-radius: 8px;
  cursor: pointer;
  opacity: 0;              /* 默认隐藏 */
  transition: all 0.2s;
}

.item-slot:hover .favorite-btn {
  opacity: 1;              /* 悬浮显示 */
}

.favorite-btn.active {
  opacity: 1 !important;   /* 已收藏始终显示 */
  background: rgba(251, 191, 36, 0.2);
}

.favorite-btn.active svg {
  fill: var(--favorite);
  filter: drop-shadow(0 0 6px rgba(251, 191, 36, 0.5));
}
```

---

## 3. 加载速度优化

### 3.1 策略选择

采用**分页卸载 + LRU 缓存**方案（而非真虚拟滚动）：

- 复杂度低，改动范围小
- 对当前分页模型兼容
- 性能提升明显（DOM 数量恒定在单页大小）

### 3.2 图片加载管线

```
用户滚动/翻页 → 卡片进入视口 →
  1. 立即显示 hash 色块占位（从 hash 前 6 位生成稳定 HSL 颜色）
  2. 并行请求缩略图（bridge.apiGet thumbnail）
  3. 缩略图到达 → opacity 0→1 淡入（200ms）
  4. 点击/预览 → 预加载原图（originalDataUrls）
```

### 3.3 LRU 内存缓存

```javascript
const imageCache = new Map();
const MAX_CACHE = 50;

function getCachedImage(hash) {
  if (imageCache.has(hash)) {
    const url = imageCache.get(hash);
    imageCache.delete(hash);  // 移到最新
    imageCache.set(hash, url);
    return url;
  }
  return null;
}

function setCachedImage(hash, url) {
  if (imageCache.size >= MAX_CACHE) {
    const firstKey = imageCache.keys().next().value;
    imageCache.delete(firstKey);  // 淘汰最旧
  }
  imageCache.set(hash, url);
}
```

### 3.4 首屏加载优化

- `pageSize` 动态：大屏 24 / 中屏 16 / 小屏 12
- 统计信息 API 优先于图片列表
- 骨架屏替代 spinner（`shimmer` 动画）

### 3.5 CSS 性能

- 移除 `* { transition: background-color 0.3s, ... }`
- 改为 `.item-slot, .codex-btn, input, select { transition: ... }`
- 动画仅使用 `transform` 和 `opacity`
- 卡片尝试 `content-visibility: auto`（渐进增强）

---

## 4. 功能覆盖补全

### 4.1 前端缺失功能清单

| 功能 | 后端支持 | 前端状态 | 实现方案 |
|------|---------|---------|---------|
| `/health` 健康检查 | ✅ | ❌ | 页面加载时调用，头部显示状态圆点（绿/黄/红） |
| `use_count` 使用统计 | ✅ (DB 字段) | ❌ | 详情面板新增"使用次数"字段 |
| `last_used_at` | ✅ (DB 字段) | ❌ | 详情面板新增"最后使用"字段 |
| 排序方式"最多使用" | ❌ | ❌ | 后端新增 sort=most_used，前端加选项 |
| 收藏筛选 | 需新增 | ❌ | 侧边栏"⭐ 收藏"虚拟分类 |

### 4.2 健康状态指示器

```html
<!-- 头部右侧 -->
<div class="health-indicator" :class="healthStatus">
  <span class="health-dot"></span>
  <span class="health-text">{{ healthStatusText }}</span>
</div>
```

状态映射：
- `green`: API 响应 < 200ms → "正常"
- `yellow`: API 响应 200~500ms → "缓慢"
- `red`: API 无响应或报错 → "异常"

---

## 5. 收藏功能详细设计

### 5.1 数据模型

`emoji` 表新增字段（SCHEMA_VERSION 1 → 2）：

```sql
ALTER TABLE emoji ADD COLUMN is_favorite INTEGER DEFAULT 0;
CREATE INDEX idx_emoji_favorite ON emoji(is_favorite);
```

### 5.2 后端改动

#### database_service.py

- `_init_schema`: 版本检测 + ALTER TABLE 迁移
- `_insert_batch_sync`: INSERT 包含 `is_favorite`
- `_sync_index_sync`: INSERT/UPDATE 包含 `is_favorite`
- `get_emojis_paginated`: 新增 `favorite_only: bool = False` 参数

#### plugin_api.py

1. **`_build_image_item`**: 返回值增加 `is_favorite`, `use_count`, `last_used_at`
2. **`handle_update_image`**: 支持 `is_favorite` 字段更新
3. **`handle_list_images`**: 支持 `favorite_only` 查询参数
4. **新增 `POST /images/batch-favorite`**:
   ```python
   async def handle_batch_favorite(self):
       data = await request.get_json() or {}
       hashes = set(data.get("hashes", []))
       favorite = bool(data.get("favorite", True))
       # 批量更新 is_favorite
   ```

#### emoji_selection_strategy.py

- 随机模式：收藏项权重 ×3
- 智能模式：收藏项 bonus +0.3

#### event_handler.py

- `_select_items_for_removal`: 过滤 `is_favorite=1`，收藏项永不清理

### 5.3 前端改动

#### 数据结构

```javascript
// images 数组元素扩展
{
  hash: "...",
  category: "happy",
  is_favorite: false,        // 新增
  use_count: 12,             // 新增
  last_used_at: 1705312800,  // 新增
  // ... 原有字段
}
```

#### 交互设计

1. **卡片收藏按钮**: 右上角星标，悬浮显示/已收藏常显，点击切换 + 弹跳动画
2. **侧边栏收藏分类**: "⭐ 收藏"虚拟分类，点击筛选 `favorite_only=true`
3. **批量操作栏**: 新增"收藏"/"取消收藏"按钮
4. **详情面板**: 收藏 toggle 开关

#### 前端 API 调用

```javascript
// 单张切换收藏
const toggleFavorite = async (img) => {
  const newValue = !img.is_favorite;
  const res = await apiFetch('api/images/update', {
    method: 'POST',
    body: JSON.stringify({ hash: img.hash, is_favorite: newValue }),
  });
  if (res.ok) img.is_favorite = newValue;
};

// 批量收藏
const batchSetFavorite = async (favorite) => {
  await apiFetch('api/images/batch-favorite', {
    method: 'POST',
    body: JSON.stringify({ hashes: [...selectedImages], favorite }),
  });
};
```

### 5.4 收藏权重策略

**随机模式**:
1. 候选列表中收藏项权重 ×3
2. 加权随机选择

**智能模式**:
1. BM25 + 文本相似度基础打分
2. 收藏项额外 +0.3 bonus
3. 分数相近时收藏项优先

### 5.5 清理保护

容量控制淘汰逻辑：
- 按 `created_at` 从旧到新排序
- 过滤掉 `is_favorite=1`
- 收藏表情包永远不会被自动清理
- 用户手动删除不受限制

---

## 6. 实施计划概要

### 阶段 1: 后端基础
1. 数据库 schema 迁移（SCHEMA_VERSION 1→2，新增 `is_favorite`）
2. `plugin_api.py` 接口扩展（`_build_image_item`, `handle_update_image`, `handle_list_images`, 新增 `batch-favorite`）
3. 表情选择权重调整（`emoji_selection_strategy.py`）
4. 清理保护（`event_handler.py`）

### 阶段 2: 前端视觉改造
1. CSS 变量系统重构
2. 布局改造（侧边栏、头部、网格）
3. 卡片样式简化 + 收藏按钮
4. 动效优化（移除全局 transition，精确限定）

### 阶段 3: 前端性能优化
1. 首屏加载优化（动态 pageSize）
2. 骨架屏组件
3. LRU 缓存实现
4. hash 色块占位

### 阶段 4: 前端功能补全
1. 健康状态指示器
2. 使用统计展示（详情面板）
3. 收藏分类筛选（侧边栏）
4. 批量收藏操作

### 阶段 5: 测试与验证
1. 功能测试（收藏/取消/批量/筛选）
2. 性能测试（大量图片加载流畅度）
3. 主题切换测试
4. 移动端响应式测试

---

## 附录: 变更文件清单

### 后端
- `core/db/database_service.py`
- `plugin_api.py`
- `core/search/emoji_selection_strategy.py`
- `core/events/event_handler.py`
- `core/processing/emoji_smart_select_service.py`（如有）

### 前端
- `pages/表情管理/app.css`
- `pages/表情管理/app.js`
- `pages/表情管理/index.html`（可能不需要改动）

### 文档
- `docs/feature-favorite.md`（已有，参考用）
- `docs/superpowers/specs/2025-06-07-webui-optimization-design.md`（本文档）
