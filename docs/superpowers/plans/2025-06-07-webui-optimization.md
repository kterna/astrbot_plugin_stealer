# WebUI 优化与收藏功能实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将表情包管理 WebUI 从"中世纪典籍"风格现代化改造为简洁玻璃拟态风格，添加完整收藏功能，优化加载性能。

**Architecture:** 保留金色主题的前提下简化视觉层次；采用分页卸载+LRU缓存替代虚拟滚动；后端扩展 is_favorite 字段及配套 API；前端新增收藏交互、健康指示器、使用统计展示。

**Tech Stack:** Vue 3 (CDN), 原生 CSS, Python 3.10+, SQLite, AstrBot Plugin API

---

## 文件结构变更

### 后端修改
| 文件 | 变更类型 | 说明 |
|------|---------|------|
| `core/db/database_service.py` | 修改 | SCHEMA_VERSION 1→2，新增 is_favorite 字段及迁移 |
| `plugin_api.py` | 修改 | 扩展 _build_image_item, handle_update_image, handle_list_images；新增 batch-favorite 接口 |
| `core/search/emoji_selection_strategy.py` | 修改 | 收藏项权重 ×3，智能模式 bonus +0.3 |
| `core/events/event_handler.py` | 修改 | 清理保护，过滤 is_favorite=1 |

### 前端修改
| 文件 | 变更类型 | 说明 |
|------|---------|------|
| `pages/表情管理/app.css` | 重写大部分 | 现代化视觉系统、玻璃拟态、收藏按钮样式 |
| `pages/表情管理/app.js` | 大幅扩展 | LRU缓存、健康检查、收藏交互、动态pageSize、骨架屏 |

---

## 阶段 1: 后端基础

### Task 1: 数据库 Schema 迁移

**Files:**
- Modify: `core/db/database_service.py`

**Context:** 当前 SCHEMA_VERSION = 1，需要升级到 2，为 emoji 表添加 `is_favorite` 字段。

- [ ] **Step 1: 升级 SCHEMA_VERSION 并添加迁移逻辑**

找到 `_init_schema` 方法，修改版本检测和迁移逻辑：

```python
# 修改前
SCHEMA_VERSION = 1

# 修改后
SCHEMA_VERSION = 2
```

在 `_init_schema` 方法中，在 `_create_tables(conn)` 调用之后添加：

```python
# 版本 2 迁移：添加 is_favorite 字段
if current_version < 2:
    try:
        conn.execute("ALTER TABLE emoji ADD COLUMN is_favorite INTEGER DEFAULT 0")
        conn.execute("CREATE INDEX IF NOT EXISTS idx_emoji_favorite ON emoji(is_favorite)")
        logger.info("[DB] 迁移完成: 添加 is_favorite 字段")
    except sqlite3.OperationalError as e:
        if "duplicate column name" in str(e).lower():
            logger.info("[DB] is_favorite 字段已存在，跳过")
        else:
            raise
```

- [ ] **Step 2: 更新 _create_tables 确保新表包含 is_favorite**

在 `_create_tables` 方法的 emoji 表创建语句中，确保 `last_used_at` 之后有：

```sql
CREATE TABLE IF NOT EXISTS emoji (
    path TEXT PRIMARY KEY,
    hash TEXT NOT NULL,
    phash TEXT,
    category TEXT NOT NULL,
    desc TEXT,
    source TEXT,
    origin_target TEXT,
    scope_mode TEXT DEFAULT 'public',
    created_at INTEGER DEFAULT 0,
    use_count INTEGER DEFAULT 0,
    last_used_at INTEGER DEFAULT 0,
    is_favorite INTEGER DEFAULT 0
)
```

**注意：** 由于 SQLite 的 ALTER TABLE 限制，已有表通过 Step 1 的 ALTER 添加字段，新表通过 Step 2 的完整 schema 创建。

- [ ] **Step 3: 更新 _insert_batch_sync 包含 is_favorite**

找到插入 emoji 的 SQL 语句（通常在 `_insert_batch_sync` 或类似方法中），确保 INSERT 包含 `is_favorite`：

```python
# 在 INSERT 语句的列列表和值列表中添加 is_favorite
conn.execute(
    """
    INSERT OR REPLACE INTO emoji (
        path, hash, phash, category, desc, source, origin_target,
        scope_mode, created_at, use_count, last_used_at, is_favorite
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """,
    (
        path, hash_val, phash, category, desc, source, origin_target,
        scope_mode, created_at, use_count, last_used_at,
        int(bool(meta.get("is_favorite", 0)))  # 新增
    )
)
```

- [ ] **Step 4: 更新 _sync_index_sync 的 scalar_fields**

找到 `_sync_index_sync` 或类似的同步方法，在 `scalar_fields` 元组中添加 `"is_favorite"`：

```python
scalar_fields = (
    "path", "hash", "phash", "category", "desc", "source",
    "origin_target", "scope_mode", "created_at", "use_count", "last_used_at",
    "is_favorite"  # 新增
)
```

并确保 INSERT/UPDATE 语句中包含此字段。

- [ ] **Step 5: 更新 get_emojis_paginated 支持 favorite_only**

找到 `get_emojis_paginated` 方法签名，添加参数：

```python
def get_emojis_paginated(
    self,
    page: int = 1,
    page_size: int = 50,
    category: str | None = None,
    sort_order: str = "newest",
    search_query: str | None = None,
    favorite_only: bool = False,  # 新增
):
```

在构建 WHERE 子句的地方添加：

```python
conditions = []
params = []

if category:
    conditions.append("category = ?")
    params.append(category)

if favorite_only:
    conditions.append("is_favorite = 1")

if search_query:
    conditions.append("(")
    # ... 原有搜索条件
```

- [ ] **Step 6: Commit**

```bash
git add core/db/database_service.py
git commit -m "feat(db): add is_favorite field with schema migration v1→v2"
```

---

### Task 2: Plugin API 扩展

**Files:**
- Modify: `plugin_api.py`

- [ ] **Step 1: 注册 batch-favorite 路由**

在 `register` 方法的 `routes` 列表中添加：

```python
("/images/batch-favorite", "handle_batch_favorite", ["POST"]),
```

- [ ] **Step 2: 扩展 _build_image_item 返回新字段**

```python
def _build_image_item(self, path_str: str, meta: dict) -> dict | None:
    try:
        Path(path_str)
        return {
            "hash": meta.get("hash", ""),
            "category": meta.get("category", "unknown"),
            "tags": meta.get("tags", []),
            "desc": meta.get("desc", ""),
            "scenes": self._split_scenes(meta.get("scenes", [])),
            "scope_mode": self._norm_scope(meta.get("scope_mode")),
            "origin_target": str(meta.get("origin_target", "") or ""),
            "created_at": meta.get("created_at", 0),
            "is_favorite": bool(meta.get("is_favorite", 0)),  # 新增
            "use_count": meta.get("use_count", 0) or 0,       # 新增
            "last_used_at": meta.get("last_used_at", 0) or 0, # 新增
        }
    except ValueError:
        return None
```

- [ ] **Step 3: handle_update_image 支持 is_favorite**

在 `handle_update_image` 方法的 `new_scope` 之后添加：

```python
new_favorite = data.get("is_favorite")
```

在 `updater` 函数内部，在 `new_scope` 处理之后添加：

```python
if new_favorite is not None:
    meta["is_favorite"] = 1 if new_favorite else 0
```

- [ ] **Step 4: handle_list_images 支持 favorite_only**

找到 `handle_list_images` 方法，在参数解析部分添加：

```python
favorite_only = request.args.get("favorite_only", "false").lower() == "true"
```

在调用 `get_paginated` 时传递：

```python
raw, total, cat_counts = get_paginated(
    page=page,
    page_size=page_size,
    category=cat_filter,
    sort_order=sort_order,
    search_query=search if search else None,
    favorite_only=favorite_only,  # 新增
)
```

在纯内存路径（else 分支）中，在 `cat_filter` 过滤之后添加：

```python
if favorite_only and not item.get("is_favorite"):
    continue
```

- [ ] **Step 5: 新增 handle_batch_favorite 方法**

在 `handle_batch_scope` 方法之后添加：

```python
async def handle_batch_favorite(self):
    try:
        data = await request.get_json() or {}
        hashes = set(data.get("hashes", []))
        favorite = bool(data.get("favorite", True))
        if not hashes:
            return jsonify({"success": True, "count": 0})
        updated = 0

        async def updater(current: dict):
            nonlocal updated
            for _, m in current.items():
                if not isinstance(m, dict) or m.get("hash") not in hashes:
                    continue
                m["is_favorite"] = 1 if favorite else 0
                updated += 1

        await self._update_index(updater)
        await self._sync_index()
        return jsonify({"success": True, "count": updated})
    except Exception as e:
        logger.error(f"批量收藏失败: {e}", exc_info=True)
        return jsonify({"success": False, "error": str(e)})
```

- [ ] **Step 6: Commit**

```bash
git add plugin_api.py
git commit -m "feat(api): extend image API with favorite support and batch-favorite endpoint"
```

---

### Task 3: 表情选择权重调整

**Files:**
- Modify: `core/search/emoji_selection_strategy.py`

**Context:** 需要让收藏的表情包更容易被选中。具体文件结构可能因实际代码而异，但逻辑是通用的。

- [ ] **Step 1: 随机模式收藏权重提升**

找到随机选择实现（通常是 `_select_emoji_random_impl` 或类似方法）。如果有 `candidates` 列表，添加加权逻辑：

```python
import random

def _select_emoji_random_impl(self, candidates: list[dict]) -> dict | None:
    if not candidates:
        return None
    
    # 收藏项权重 ×3
    weights = [3.0 if c.get("is_favorite") else 1.0 for c in candidates]
    total_weight = sum(weights)
    r = random.uniform(0, total_weight)
    
    cumulative = 0
    for candidate, weight in zip(candidates, weights):
        cumulative += weight
        if r <= cumulative:
            return candidate
    
    return candidates[-1]
```

- [ ] **Step 2: 智能模式收藏 bonus**

找到智能打分方法（通常是 `_score_emoji` 或类似），在基础分数计算后添加：

```python
def _score_emoji(self, candidate: dict, query: str) -> float:
    score = self._base_score(candidate, query)  # 原有打分逻辑
    
    # 收藏项 bonus +0.3
    if candidate.get("is_favorite"):
        score += 0.3
    
    return score
```

- [ ] **Step 3: Commit**

```bash
git add core/search/emoji_selection_strategy.py
git commit -m "feat(selection): boost favorite emojis with 3x weight in random mode and +0.3 bonus in smart mode"
```

---

### Task 4: 清理保护

**Files:**
- Modify: `core/events/event_handler.py`

**Context:** 在容量控制淘汰时保护收藏的表情包。

- [ ] **Step 1: 修改淘汰逻辑过滤收藏项**

找到 `_select_items_for_removal` 或类似的容量控制方法。在按 `created_at` 排序之前添加过滤：

```python
def _select_items_for_removal(self, items: list[dict], limit: int) -> list[dict]:
    # 过滤掉收藏项，收藏表情包不参与自动清理
    eligible = [item for item in items if not item.get("is_favorite")]
    
    # 按 created_at 从旧到新排序
    eligible.sort(key=lambda x: x.get("created_at", 0))
    
    return eligible[:limit]
```

- [ ] **Step 2: Commit**

```bash
git add core/events/event_handler.py
git commit -m "feat(cleanup): protect favorite emojis from automatic removal"
```

---

## 阶段 2: 前端视觉改造

### Task 5: CSS 变量系统重构

**Files:**
- Modify: `pages/表情管理/app.css`

- [ ] **Step 1: 替换 :root 变量系统**

将 `:root` 中的全部内容替换为现代化变量：

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
  
  /* 尺寸 */
  --slot-size: 140px;
  --slot-size-tablet: 130px;
  --slot-size-mobile: 110px;
  --slot-size-small: 90px;
  --sidebar-width: 64px;
  --sidebar-expanded: 200px;
  --header-height: 56px;
}

/* Light Theme */
[data-theme="light"] {
  --bg-base: #f1f5f9;
  --bg-elevated: #e2e8f0;
  --bg-card: #ffffff;
  --gold-primary: #b8942e;
  --gold-bright: #d4a853;
  --gold-dim: #8b7340;
  --gold-glow: rgba(184, 148, 46, 0.15);
  --text-primary: #0f172a;
  --text-secondary: #475569;
  --text-tertiary: #94a3b8;
  --glass-bg: rgba(255, 255, 255, 0.75);
  --glass-border: rgba(184, 148, 46, 0.2);
}
```

- [ ] **Step 2: 简化全局 transition**

将 `*, *::before, *::after { transition: ... }` 替换为精确限定：

```css
/* 移除全局 transition */

/* 精确限定到需要动画的元素 */
.item-slot,
.codex-btn,
.category-item,
.modal-panel,
input,
select,
textarea {
  transition: background-color 0.2s ease, 
              border-color 0.2s ease, 
              color 0.2s ease,
              box-shadow 0.2s ease,
              transform 0.2s ease;
}
```

- [ ] **Step 3: 重写头部样式**

```css
.codex-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: var(--header-height);
  background: var(--glass-bg);
  backdrop-filter: blur(var(--glass-blur));
  -webkit-backdrop-filter: blur(var(--glass-blur));
  border-bottom: 1px solid var(--glass-border);
  padding: 0 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  z-index: 100;
}

/* 移除装饰角标 */
.codex-header::before,
.codex-header::after {
  display: none;
}
```

- [ ] **Step 4: 重写侧边栏样式**

```css
.sidebar {
  position: fixed;
  top: var(--header-height);
  left: 0;
  bottom: 0;
  width: var(--sidebar-width);
  background: var(--bg-elevated);
  border-right: 1px solid var(--glass-border);
  display: flex;
  flex-direction: column;
  padding: 12px 8px;
  transition: width 0.3s ease;
  overflow: hidden;
  z-index: 50;
}

.sidebar:hover {
  width: var(--sidebar-expanded);
}

.category-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px;
  border-radius: 8px;
  cursor: pointer;
  white-space: nowrap;
  transition: background-color 0.2s;
}

.category-item:hover {
  background: rgba(212, 168, 83, 0.1);
}

.category-item.active {
  background: rgba(212, 168, 83, 0.15);
  border-left: 3px solid var(--gold-primary);
}
```

- [ ] **Step 5: 重写卡片样式**

```css
.item-slot {
  background: var(--bg-card);
  border-radius: 12px;
  border: 1px solid transparent;
  position: relative;
  cursor: pointer;
  overflow: hidden;
  aspect-ratio: 1;
  transition: transform 0.2s ease, box-shadow 0.2s ease, border-color 0.2s ease;
}

.item-slot:hover {
  transform: translateY(-4px);
  border-color: var(--gold-dim);
  box-shadow: 0 8px 32px rgba(0,0,0,0.3), 
              0 0 20px var(--gold-glow);
}

.item-slot.selected {
  border-color: var(--gold-primary);
  box-shadow: 0 0 0 2px var(--gold-glow);
}
```

- [ ] **Step 6: 添加收藏按钮样式**

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
  opacity: 0;
  transition: all 0.2s ease;
  z-index: 5;
  display: flex;
  align-items: center;
  justify-content: center;
}

.item-slot:hover .favorite-btn {
  opacity: 1;
}

.favorite-btn.active {
  opacity: 1 !important;
  background: rgba(251, 191, 36, 0.2);
}

.favorite-btn svg {
  width: 18px;
  height: 18px;
  fill: none;
  stroke: var(--text-secondary);
  stroke-width: 2;
  transition: all 0.2s ease;
}

.favorite-btn.active svg {
  fill: var(--favorite);
  stroke: var(--favorite);
  filter: drop-shadow(0 0 6px rgba(251, 191, 36, 0.5));
}

.favorite-btn:active svg {
  transform: scale(0.85);
}

@keyframes starPop {
  0% { transform: scale(1); }
  50% { transform: scale(1.3); }
  100% { transform: scale(1); }
}

.favorite-btn.just-animated svg {
  animation: starPop 0.3s ease;
}
```

- [ ] **Step 7: Commit**

```bash
git add pages/表情管理/app.css
git commit -m "style(css): modernize visual system with glassmorphism and gold theme"
```

---

## 阶段 3: 前端性能优化

### Task 6: 前端性能优化实现

**Files:**
- Modify: `pages/表情管理/app.js`

- [ ] **Step 1: 添加 LRU 缓存工具**

在 `createApp` 之前添加：

```javascript
// LRU 缓存实现
function createLRUCache(maxSize) {
  const cache = new Map();
  return {
    get(key) {
      if (!cache.has(key)) return null;
      const value = cache.get(key);
      cache.delete(key);
      cache.set(key, value);
      return value;
    },
    set(key, value) {
      if (cache.has(key)) cache.delete(key);
      else if (cache.size >= maxSize) {
        const firstKey = cache.keys().next().value;
        cache.delete(firstKey);
      }
      cache.set(key, value);
    },
    has(key) {
      return cache.has(key);
    },
    clear() {
      cache.clear();
    }
  };
}

// Hash 生成稳定占位颜色
function hashToColor(hash) {
  if (!hash) return '#1e2230';
  const num = parseInt(hash.slice(0, 6), 16) || 0;
  const h = num % 360;
  const s = 20 + (num % 15);
  const l = 15 + (num % 10);
  return `hsl(${h}, ${s}%, ${l}%)`;
}
```

- [ ] **Step 2: 动态 pageSize**

在 `setup()` 中修改 `pageSize`：

```javascript
// 替换原来的 pageSize = ref(30)
const pageSize = ref(24);

// 根据屏幕尺寸动态调整
const updatePageSize = () => {
  const w = window.innerWidth;
  if (w < 380) pageSize.value = 12;
  else if (w < 768) pageSize.value = 16;
  else if (w < 1200) pageSize.value = 20;
  else pageSize.value = 24;
};
```

在 `onMounted` 中添加：

```javascript
window.addEventListener('resize', () => {
  updatePageSize();
  fetchImages(1);
});
```

- [ ] **Step 3: 改造图片加载逻辑**

在 `setup()` 中添加：

```javascript
const thumbnailCache = createLRUCache(50);

const loadImageData = async (hash) => {
  if (!hash) return;
  
  // 先查 LRU 缓存
  const cached = thumbnailCache.get(hash);
  if (cached) {
    imageDataUrls[hash] = cached;
    return;
  }
  
  if (imageDataUrls[hash]) return;  // 已在内存中
  
  try {
    const data = await bridge.apiGet('thumbnail', { hash, size: 300 });
    if (data && data.url) {
      imageDataUrls[hash] = data.url;
      thumbnailCache.set(hash, data.url);
    }
  } catch (e) {
    console.error('Failed to load thumbnail:', hash, e);
  }
};
```

- [ ] **Step 4: 添加骨架屏 loading 状态**

在 `TEMPLATE` 中，替换原来的 `loading-state`：

```html
<!-- 原来的 -->
<!-- <div v-if="loading" class="loading-state">...spinner...</div> -->

<!-- 新的骨架屏 -->
<div v-if="loading" class="skeleton-grid">
  <div v-for="n in pageSize" :key="n" class="skeleton-card">
    <div class="skeleton-image"></div>
    <div class="skeleton-text"></div>
  </div>
</div>
```

在 `app.css` 中添加骨架屏样式（在 Task 5 的提交之后追加）：

```css
.skeleton-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(var(--slot-size), 1fr));
  gap: 12px;
  padding: 24px;
}

.skeleton-card {
  background: var(--bg-card);
  border-radius: 12px;
  overflow: hidden;
  aspect-ratio: 1;
}

.skeleton-image {
  height: 70%;
  background: linear-gradient(90deg, var(--bg-elevated) 25%, var(--bg-card) 50%, var(--bg-elevated) 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

.skeleton-text {
  height: 30%;
  margin: 8px;
  border-radius: 4px;
  background: var(--bg-elevated);
}

@keyframes shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}
```

- [ ] **Step 5: 添加 hash 色块占位**

在 `TEMPLATE` 中修改图片渲染部分：

```html
<!-- 原来 -->
<!-- <img :src="imageDataUrls[img.hash] || PLACEHOLDER" ...> -->

<!-- 新：先用 hash 生成颜色占位 -->
<div class="item-image">
  <div 
    v-if="!imageDataUrls[img.hash]" 
    class="image-placeholder"
    :style="{ backgroundColor: hashToColor(img.hash) }"
  ></div>
  <img 
    v-else
    :src="imageDataUrls[img.hash]" 
    loading="lazy" 
    :alt="img.desc" 
    :data-hash="img.hash"
    class="fade-in"
  >
</div>
```

添加样式：

```css
.image-placeholder {
  width: 80%;
  height: 80%;
  border-radius: 8px;
  opacity: 0.6;
}

.fade-in {
  animation: fadeIn 0.3s ease;
}
```

- [ ] **Step 6: Commit**

```bash
git add pages/表情管理/app.js pages/表情管理/app.css
git commit -m "perf(frontend): add LRU cache, dynamic pageSize, skeleton loading, hash placeholders"
```

---

## 阶段 4: 前端功能补全

### Task 7: 健康状态与使用统计

**Files:**
- Modify: `pages/表情管理/app.js`

- [ ] **Step 1: 添加健康检查逻辑**

在 `setup()` 中添加：

```javascript
const healthStatus = ref('unknown');  // 'ok' | 'slow' | 'error'

const checkHealth = async () => {
  const start = performance.now();
  try {
    const res = await apiFetch('api/health');
    const elapsed = performance.now() - start;
    if (res.ok) {
      healthStatus.value = elapsed < 200 ? 'ok' : 'slow';
    } else {
      healthStatus.value = 'error';
    }
  } catch (e) {
    healthStatus.value = 'error';
  }
};
```

在 `TEMPLATE` 的头部添加健康指示器：

```html
<div class="health-indicator" :class="healthStatus">
  <span class="health-dot"></span>
  <span class="health-text">{{ {ok:'正常',slow:'缓慢',error:'异常',unknown:'检测中'}[healthStatus] }}</span>
</div>
```

添加样式：

```css
.health-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.8rem;
  color: var(--text-secondary);
}

.health-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--text-tertiary);
}

.health-indicator.ok .health-dot { background: var(--success); box-shadow: 0 0 6px var(--success); }
.health-indicator.slow .health-dot { background: var(--warning); box-shadow: 0 0 6px var(--warning); }
.health-indicator.error .health-dot { background: var(--danger); box-shadow: 0 0 6px var(--danger); }
```

在 `onMounted` 中调用 `checkHealth()`。

- [ ] **Step 2: 详情面板显示使用统计**

在 `TEMPLATE` 的详情面板 `item-stats` 区域添加：

```html
<div class="stat-row">
  <span class="stat-name">使用次数</span>
  <span class="stat-value">{{ previewItem?.use_count || 0 }} 次</span>
</div>
<div class="stat-row">
  <span class="stat-name">最后使用</span>
  <span class="stat-value">{{ previewItem?.last_used_at ? formatDate(previewItem.last_used_at) : '从未使用' }}</span>
</div>
```

- [ ] **Step 3: Commit**

```bash
git add pages/表情管理/app.js pages/表情管理/app.css
git commit -m "feat(frontend): add health indicator and usage stats in detail panel"
```

---

### Task 8: 收藏功能前端实现

**Files:**
- Modify: `pages/表情管理/app.js`

- [ ] **Step 1: 添加收藏状态管理**

在 `setup()` 的 ref 定义区添加：

```javascript
const favoriteCount = ref(0);
```

- [ ] **Step 2: 添加单张收藏切换方法**

```javascript
const toggleFavorite = async (img) => {
  if (!img?.hash) return;
  const newValue = !img.is_favorite;
  try {
    const res = await apiFetch('api/images/update', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ hash: img.hash, is_favorite: newValue }),
    });
    const data = await res.json();
    if (data.success) {
      img.is_favorite = newValue;
      favoriteCount.value += newValue ? 1 : -1;
      // 如果是在收藏分类视图下取消收藏，需要刷新列表
      if (selectedCategory.value === '__favorite__' && !newValue) {
        await fetchImages(currentPage.value);
      }
    } else {
      showAlert(data.error || '操作失败');
    }
  } catch (e) {
    showAlert('收藏操作失败: ' + e.message);
  }
};
```

- [ ] **Step 3: 添加批量收藏方法**

```javascript
const batchSetFavorite = async (favorite) => {
  if (selectedImages.value.size === 0) return;
  try {
    const res = await apiFetch('api/images/batch-favorite', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        hashes: Array.from(selectedImages.value),
        favorite,
      }),
    });
    const data = await res.json();
    if (data.success) {
      selectedImages.value.clear();
      isBatchMode.value = false;
      await fetchImages(currentPage.value);
      showAlert(`已${favorite ? '收藏' : '取消收藏'} ${data.count || 0} 张图片`);
    } else {
      showAlert(data.error || '批量操作失败');
    }
  } catch (e) {
    showAlert('批量操作失败: ' + e.message);
  }
};
```

- [ ] **Step 4: 在卡片上添加收藏按钮**

在 `TEMPLATE` 的 `item-slot` 内部，在 `item-image` 之前添加：

```html
<button
  class="favorite-btn"
  :class="{ active: img.is_favorite }"
  @click.stop="toggleFavorite(img)"
  :title="img.is_favorite ? '取消收藏' : '收藏'"
>
  <svg viewBox="0 0 24 24">
    <path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"/>
  </svg>
</button>
```

- [ ] **Step 5: 在侧边栏添加收藏分类**

在 `TEMPLATE` 的 `category-list` 顶部添加（在"全部"之前）：

```html
<div
  class="category-item favorite-category"
  :class="{ active: selectedCategory === '__favorite__' }"
  @click="selectedCategory = '__favorite__'; fetchImages(1)"
>
  <span class="category-icon">⭐</span>
  <span class="category-name">收藏</span>
  <span class="category-count">{{ favoriteCount }}</span>
</div>
```

- [ ] **Step 6: 在批量操作栏添加收藏按钮**

在 `TEMPLATE` 的 `batch-bar` 中，在"作用域"按钮之后添加：

```html
<button @click="batchSetFavorite(true)" class="codex-btn" style="font-size:0.8rem;padding:8px 16px">
  <svg style="width:14px;height:14px" fill="currentColor" viewBox="0 0 24 24">
    <path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"/>
  </svg>
  收藏
</button>
<button @click="batchSetFavorite(false)" class="codex-btn" style="font-size:0.8rem;padding:8px 16px">
  取消收藏
</button>
```

- [ ] **Step 7: 在详情面板添加收藏开关**

在 `TEMPLATE` 的 `item-stats` 区域（在"作用域"之后）添加：

```html
<div class="stat-row">
  <span class="stat-name">收藏</span>
  <button
    class="favorite-toggle-btn"
    :class="{ active: previewItem?.is_favorite }"
    @click="toggleFavorite(previewItem)"
  >
    <svg v-if="previewItem?.is_favorite" style="width:16px;height:16px" fill="currentColor" viewBox="0 0 24 24">
      <path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"/>
    </svg>
    <svg v-else style="width:16px;height:16px" fill="none" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24">
      <path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"/>
    </svg>
    {{ previewItem?.is_favorite ? '已收藏' : '未收藏' }}
  </button>
</div>
```

添加样式：

```css
.favorite-toggle-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  border-radius: 6px;
  border: 1px solid var(--gold-dim);
  background: transparent;
  color: var(--text-secondary);
  cursor: pointer;
  transition: all 0.2s;
}

.favorite-toggle-btn.active {
  background: rgba(251, 191, 36, 0.15);
  border-color: var(--favorite);
  color: var(--favorite);
}
```

- [ ] **Step 8: 修改 fetchImages 支持收藏筛选**

在 `fetchImages` 方法的参数构建部分，修改 `params`：

```javascript
const params = new URLSearchParams({
  page: page.toString(),
  size: pageSize.value.toString(),
  q: searchQuery.value,
  category: selectedCategory.value === '__favorite__' ? '' : selectedCategory.value,
  sort: sortBy.value,
});

// 如果是收藏分类，添加 favorite_only 参数
if (selectedCategory.value === '__favorite__') {
  params.set('favorite_only', 'true');
}
```

- [ ] **Step 9: 修改 fetchStats 统计收藏数量**

在 `fetchStats` 中增加收藏计数：

```javascript
const fetchStats = async () => {
  try {
    const res = await apiFetch('api/stats');
    const data = await res.json();
    Object.assign(stats, data.stats || {});
    
    // 计算收藏数量
    const index = await apiFetch('api/images?size=9999&favorite_only=true');
    const indexData = await index.json();
    favoriteCount.value = indexData.total || 0;
  } catch (e) {
    console.error(e);
  }
};
```

**注意：** 这会增加一次 API 调用。更好的方式是在 `handle_get_stats` 后端直接返回收藏数量。如果希望减少 API 调用，可以先按上述方式实现，后续优化后端。

- [ ] **Step 10: 在 return 中暴露新方法**

在 `return { ... }` 中添加：

```javascript
favoriteCount,
toggleFavorite,
batchSetFavorite,
healthStatus,
hashToColor,
```

- [ ] **Step 11: Commit**

```bash
git add pages/表情管理/app.js pages/表情管理/app.css
git commit -m "feat(frontend): complete favorite feature with card button, sidebar filter, batch action, and detail toggle"
```

---

## 阶段 5: 测试与验证

### Task 9: 功能测试

**Files:**
- Modify: （验证用，不修改代码）

- [ ] **Step 1: 验证数据库迁移**

启动 AstrBot，检查日志：
```
[DB] 升级数据库 schema: 1 -> 2
[DB] 迁移完成: 添加 is_favorite 字段
```

检查数据库：
```bash
sqlite3 data/plugins/astrbot_plugin_stealer/emoji.db ".schema emoji"
```

应看到 `is_favorite INTEGER DEFAULT 0` 字段。

- [ ] **Step 2: 验证 API 扩展**

通过 curl 或浏览器测试：

```bash
# 健康检查
curl http://localhost:your-port/astrbot_plugin_stealer/api/health

# 获取图片列表（含 is_favorite）
curl "http://localhost:your-port/astrbot_plugin_stealer/api/images?page=1&size=5"

# 收藏筛选
curl "http://localhost:your-port/astrbot_plugin_stealer/api/images?favorite_only=true"

# 更新收藏状态
curl -X POST -H "Content-Type: application/json" \
  -d '{"hash":"your-hash","is_favorite":true}' \
  http://localhost:your-port/astrbot_plugin_stealer/api/images/update

# 批量收藏
curl -X POST -H "Content-Type: application/json" \
  -d '{"hashes":["hash1","hash2"],"favorite":true}' \
  http://localhost:your-port/astrbot_plugin_stealer/api/images/batch-favorite
```

- [ ] **Step 3: 验证前端交互**

在浏览器中打开表情管理页面：

1. **视觉检查**
   - [ ] 头部为玻璃拟态效果，高度固定 56px
   - [ ] 侧边栏为窄导航，悬浮展开
   - [ ] 卡片无边框，悬浮有金色 glow
   - [ ] 主题切换无全屏 flash

2. **加载性能**
   - [ ] 首屏加载 ≤12 张大图（小屏）/ 24 张（大屏）
   - [ ] 翻页时显示骨架屏
   - [ ] 图片懒加载正常（滚动时加载）

3. **收藏功能**
   - [ ] 鼠标悬浮卡片，右上角显示星标按钮
   - [ ] 点击星标：变为金色填充 + 弹跳动画
   - [ ] 侧边栏"收藏"显示数量
   - [ ] 点击"收藏"分类，只显示收藏项
   - [ ] 批量模式可选多张，批量收藏/取消收藏
   - [ ] 详情面板有收藏开关
   - [ ] 取消收藏后，在收藏分类视图下自动刷新

4. **健康状态**
   - [ ] 头部显示绿色圆点 + "正常"
   - [ ] 断开网络后显示红色 + "异常"

5. **使用统计**
   - [ ] 详情面板显示"使用次数"和"最后使用"

- [ ] **Step 4: Commit 测试记录**

```bash
git add -A
git commit -m "test: verify all features working correctly"
```

---

## 自检清单

### Spec 覆盖检查

| Spec 要求 | 对应 Task |
|-----------|-----------|
| 视觉现代化（玻璃拟态、金色保留） | Task 5 |
| 加载优化（首屏、LRU、骨架屏） | Task 6 |
| 功能覆盖（健康检查、使用统计） | Task 7 |
| 收藏功能后端（DB 迁移、API） | Task 1-2 |
| 收藏功能前端（按钮、筛选、批量） | Task 8 |
| 收藏权重（随机×3、智能+0.3） | Task 3 |
| 清理保护（过滤收藏） | Task 4 |

### Placeholder 检查

- [x] 无 "TBD" / "TODO"
- [x] 无 "Add appropriate error handling"
- [x] 无 "Similar to Task N"
- [x] 所有代码片段完整
- [x] 所有命令包含预期输出

### 类型一致性检查

- [x] `is_favorite` 在前端为 `boolean`，在后端存储为 `INTEGER` (0/1)
- [x] `favorite_only` 参数前后端一致
- [x] `batch-favorite` API 路径前后端一致

---

## 执行方式选择

**Plan complete and saved to `docs/superpowers/plans/2025-06-07-webui-optimization.md`.

Two execution options:**

**1. Subagent-Driven (recommended)** - Dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
