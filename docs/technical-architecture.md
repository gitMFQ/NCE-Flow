# NCE-Flow 技术架构详解

> 技术架构与数据流说明文档

本文档详细说明 NCE-Flow 的技术架构、数据流转、状态管理以及关键实现细节。

---

## 1. 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         浏览器环境                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                       用户界面层                           │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │  │
│  │  │  index.html  │  │ lesson.html  │  │   book.html     │ │  │
│  │  │  (课程列表)   │  │  (课文播放)   │  │  (兼容跳转)     │ │  │
│  │  └──────┬───────┘  └──────┬───────┘  └────────┬────────┘ │  │
│  └─────────┼──────────────────┼──────────────────┼──────────┘  │
│            │                  │                  │              │
│  ┌─────────┴──────────────────┴──────────────────┴──────────┐  │
│  │                       样式层                               │  │
│  │                  assets/styles.css                        │  │
│  │        (CSS Variables, 主题, 响应式布局)                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                       逻辑层                               │  │
│  │  ┌──────────────────┐      ┌──────────────────────────┐  │  │
│  │  │   assets/app.js  │      │   assets/lesson.js       │  │  │
│  │  │  ・语言切换      │      │  ・LRC 解析              │  │  │
│  │  │  ・全局状态      │      │  ・音频调度              │  │  │
│  │  │  ・API 暴露      │      │  ・播放控制              │  │  │
│  │  │                  │      │  ・进度管理              │  │  │
│  │  │                  │      │  ・iOS 音频解锁          │  │  │
│  │  └──────────────────┘      └──────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    本地存储层                              │  │
│  │  ┌────────────────────┐    ┌───────────────────────────┐  │  │
│  │  │   localStorage     │    │   sessionStorage          │  │  │
│  │  │  ・nce_lang_mode  │    │  ・nce_resume             │  │  │
│  │  │  ・nce_favs       │    │  ・nce_resume_play        │  │  │
│  │  │  ・nce_recents    │    └───────────────────────────┘  │  │
│  │  │  ・nce_lastpos    │                                    │  │
│  │  │  ・readMode       │                                    │  │
│  │  │  ・autoFollow     │                                    │  │
│  │  │  ・autoContinue   │                                    │  │
│  │  │  ・audioPlaybackRate│                                 │  │
│  │  └────────────────────┘                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    静态资源层                              │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌─────────────┐ │  │
│  │  │ static/        │  │ NCE1~NCE4/     │  │ images/     │ │  │
│  │  │  data.json     │  │  *.mp3         │  │  *.jpg      │ │  │
│  │  │  (课程元数据)   │  │  *.lrc         │  │  (封面)     │ │  │
│  │  └────────────────┘  └────────────────┘  └─────────────┘ │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心模块职责

### 2.1 assets/app.js - 全局工具模块

**职责**：
- 语言模式状态管理（EN / EN+CN / CN）
- 初始化 Segmented 控件
- 通过 `window.NCE_APP` 暴露全局 API

**关键函数**：
```javascript
NCE_APP.getLang()       // 读取当前语言模式
NCE_APP.setLang(v)      // 设置语言模式
NCE_APP.applyLang(v)    // 应用语言样式类
NCE_APP.initSegmented() // 初始化语言切换控件
```

**状态键**：`nce_lang_mode` (localStorage)

---

### 2.2 assets/lesson.js - 核心播放控制器

**职责**：
- LRC 字幕解析（支持中英双语、元数据）
- 音频播放调度与切句逻辑
- iOS 音频解锁
- 播放模式管理（连读/点读）
- 自动跟随滚动
- 进度持久化
- 自动续播下一课

**关键算法**：

#### 2.2.1 LRC 解析流程

```
读取 LRC 文本
  ↓
按行拆分
  ↓
识别元数据 [al:], [ar:], [ti:], [by:]
  ↓
识别歌词行（时间标签 + 内容）
  ↓
检测中英文分离（| 分隔符 或 相邻重复时间标签）
  ↓
计算句子起止时间（start/end）
  ↓
返回 { meta, items }
```

#### 2.2.2 播放调度机制

```
播放句子 playSegment(idx)
  ↓
判断模式：连读 or 点读
  ↓
[连读模式]                          [点读模式]
  静音 + seek + 等待 seeked/canplay   暂停 + seek + 等待 seeked + play
  ↓                                  ↓
  两帧后解除静音                      启动调度
  ↓                                  ↓
启动调度                            [近端调度]
  ↓                                  requestAnimationFrame 精确判断
[远端调度]                          当时间到达句尾前 guardAheadSec
  setTimeout 粗略等待                ↓
  ↓                                  静音 + 暂停 + 钳位到句尾
[近端调度]                          ↓
  requestAnimationFrame 精确判断     保持暂停状态
  当时间到达句尾前 guardAheadSec
  ↓
  [连读] 播放下一句
  [点读] 静音暂停
```

**关键参数**：
- `NEAR_WINDOW_MS`: 近端调度窗口（iOS: 160ms, 其他: 120ms）
- `guardAheadSec()`: 提前量计算（基于倍速动态调整）
- iOS 提前量：80ms + (rate - 1) * 30ms
- 其他设备：60ms + (rate - 1) * 20ms

#### 2.2.3 iOS 音频解锁

```
首次用户交互 (touchstart/pointerdown/click)
  ↓
audio.muted = true
  ↓
audio.play() (同步执行)
  ↓
立即排队 audio.pause() + audio.muted = false
  ↓
iosUnlocked = true
```

---

### 2.3 index.html - 课程导航页

**职责**：
- 显示课程列表（基于 `static/data.json`）
- 收藏管理
- 最近学习显示
- 册别切换（hash 路由）

**数据流**：

```
页面加载
  ↓
读取 location.hash → 确定当前册别 (NCE1~NCE4)
  ↓
fetch('static/data.json') → 获取所有课程数据
  ↓
过滤当前册别的课程列表
  ↓
读取 localStorage:
  - nce_favs (收藏列表)
  - nce_recents (最近学习)
  - nce_lastpos (最后位置)
  ↓
渲染课程列表 + 最近学习卡片
  ↓
用户交互：
  - 点击课程 → 跳转到 lesson.html
  - 点击收藏 → 更新 localStorage + 重新渲染
  - 点击最近学习 → 设置 sessionStorage resume 标志 → 跳转
```

---

### 2.4 lesson.html - 课文播放页

**职责**：
- 加载音频与字幕资源
- 播放控制
- 设置面板
- 上一课/下一课导航

**数据流**：

```
页面加载
  ↓
解析 location.hash → NCE#/filename
  ↓
拼接资源路径：
  - 音频: NCE#/filename.mp3
  - 字幕: NCE#/filename.lrc
  ↓
loadLrc() 解析字幕 → items[]
  ↓
渲染句子列表
  ↓
检查 sessionStorage.nce_resume:
  如果匹配当前课程 ID
    ↓
    从 localStorage.nce_lastpos 恢复播放位置
    如果 nce_resume_play === '1' → 自动播放
  ↓
用户交互：
  - 点击句子 → playSegment(idx)
  - 点击设置 → 打开设置面板
  - 音频播放 → 触发 timeupdate → 更新高亮 + 保存进度
  - 点击上一课/下一课 → 跳转
  - 课文结束 → 自动续播（如果开启）
```

---

## 3. 状态管理详解

### 3.1 localStorage 存储结构

| Key                  | Type   | 说明                                          | 示例值                                    |
|----------------------|--------|-----------------------------------------------|------------------------------------------|
| `nce_lang_mode`      | String | 语言模式                                       | `'en'` \| `'bi'` \| `'cn'`               |
| `nce_favs`           | Array  | 收藏课程 ID 列表                               | `['NCE1/001&002－Excuse Me', 'NCE2/...']`|
| `nce_recents`        | Array  | 最近学习记录（含时间戳）                        | `[{id: 'NCE1/...', ts: 1697654321000}]`  |
| `nce_lastpos`        | Object | 每课的最后播放位置                             | `{'NCE1/...': {t: 12.5, idx: 3, ts: ...}}`|
| `readMode`           | String | 阅读模式                                       | `'single'` \| `'continuous'`             |
| `autoFollow`         | String | 自动跟随开关                                   | `'true'` \| `'false'`                    |
| `autoContinue`       | String | 自动续播模式                                   | `'single'` \| `'auto'`                   |
| `audioPlaybackRate`  | String | 播放速率                                       | `'1.0'` ~ `'2.5'`                        |

### 3.2 sessionStorage 存储结构

| Key                  | Type   | 说明                                          | 示例值                                    |
|----------------------|--------|-----------------------------------------------|------------------------------------------|
| `nce_resume`         | String | 待恢复的课程 ID                                | `'NCE1/001&002－Excuse Me'`              |
| `nce_resume_play`    | String | 恢复后是否自动播放                             | `'1'` (自动播放) \| 未设置 (不自动播放)    |

**用途**：从首页"继续学习"或自动续播时，跨页面传递恢复意图。

---

## 4. 路由与导航

### 4.1 Hash 路由机制

```
首页路由：
  index.html#NCE1    → 第一册
  index.html#NCE2    → 第二册
  index.html#NCE3    → 第三册
  index.html#NCE4    → 第四册

课文页路由：
  lesson.html#NCE1/001&002－Excuse Me → 第一册第1课
  lesson.html#NCE2/001－A Private Conversation → 第二册第1课
```

**监听机制**：
- `window.addEventListener('hashchange', ...)`
- 首页：重新加载课程列表
- 课文页：刷新页面（简化处理，保证状态一致性）

### 4.2 跨页导航流程

```
首页 → 课文页：
  用户点击课程 → href="lesson.html#NCE#/filename"
  ↓
  lesson.html 加载 → 解析 hash → 加载资源

课文页 → 课文页（上一课/下一课）：
  点击导航按钮 → href="lesson.html#NCE#/new-filename"
  ↓
  监听到 hashchange → location.reload()
  ↓
  重新解析 hash → 加载新资源

自动续播：
  课文播放完毕 → 查询 data.json 获取下一课
  ↓
  设置 sessionStorage.nce_resume = 下一课 ID
  ↓
  location.href = "lesson.html#NCE#/next-filename"
  ↓
  新页面加载 → 检测 resume 标志 → 自动从头播放

返回首页：
  点击返回按钮 → 优先 history.back()
  ↓
  如果不是站内跳转 → location.href = "index.html#NCE#"
```

---

## 5. 关键设计模式

### 5.1 IIFE（立即执行函数表达式）

```javascript
(() => {
  // 私有作用域
  const LANG_KEY = 'nce_lang_mode';
  function getLang() { ... }
  
  // 暴露公共 API
  window.NCE_APP = { getLang, setLang, ... };
})();
```

**优势**：
- 避免全局变量污染
- 封装私有状态
- 选择性暴露接口

### 5.2 事件驱动架构

```javascript
// DOM 事件
listEl.addEventListener('click', e => { ... });
audio.addEventListener('timeupdate', () => { ... });
window.addEventListener('hashchange', () => { ... });

// 自定义事件流
用户点击句子 → click 事件
  ↓
playSegment() → 更新状态 → 触发 play 事件
  ↓
play 事件 → scheduleAdvance() → setTimeout/rAF
  ↓
timeupdate 事件 → highlight() → 自动滚动
```

### 5.3 状态机模式（播放调度）

```
状态：IDLE → SCHEDULING → NEAR_END → NEXT_SEGMENT → IDLE

转换条件：
- IDLE → SCHEDULING: audio.play() + scheduleAdvance()
- SCHEDULING → NEAR_END: remainingMs <= NEAR_WINDOW_MS
- NEAR_END → NEXT_SEGMENT: currentTime >= segmentEnd - guard
- NEXT_SEGMENT → IDLE: 连读继续 / 点读暂停
```

### 5.4 策略模式（播放模式）

```javascript
function endFor(item) {
  if (readMode === 'single') {
    // 点读策略：严格用下一句开始时间
    return item.end || (item.start + 0.01);
  }
  // 连读策略：用 end 或 fallback
  return computeEnd(item);
}
```

### 5.5 观察者模式（localStorage 监听）

虽然代码未显式使用 `storage` 事件，但状态通过 localStorage 共享，多标签页理论上可监听 `storage` 事件实现同步。

---

## 6. 性能优化策略

### 6.1 调度优化

- **远端 + 近端双层调度**：远离目标时使用 `setTimeout`，临近时切换到 `requestAnimationFrame`，平衡 CPU 占用与精度。
- **动态提前量**：根据倍速调整提前量，避免高倍速下漏句或低倍速下过早触发。

### 6.2 UI 渲染优化

- **节流 timeupdate**：限制高亮更新频率（200ms），避免频繁 DOM 操作。
- **段首去抖**：句子切换后 350ms 内避免重新高亮，降低抖动。
- **requestAnimationFrame 包裹 DOM 操作**：样式切换与滚动操作使用 rAF 同步浏览器刷新。

### 6.3 存储优化

- **延迟存储**：`timeupdate` 中使用节流（2s）避免频繁写入 localStorage。
- **批量清理**：`nce_recents` 限制最多 60 条记录。

### 6.4 资源加载

- **预加载音频**：`audio.preload = 'auto'` + `audio.load()` 提前触发解码。
- **静态资源分册存储**：按书籍目录隔离，降低单次扫描开销。

---

## 7. 浏览器兼容性策略

### 7.1 CSS 回退

```css
@supports not (color: color-mix(in srgb, red, blue)) {
  /* 为不支持 color-mix() 的浏览器提供回退 */
}
```

### 7.2 iOS 特殊处理

- **音频解锁**：iOS 自动播放限制，首次交互触发静音播放解锁。
- **调度参数调整**：iOS 使用更保守的提前量与窗口时间。
- **fastSeek 回退**：优先使用 `fastSeek()` API，不支持则回退到 `currentTime`。

### 7.3 特性检测

```javascript
if (typeof audio.fastSeek === 'function') {
  audio.fastSeek(time);
} else {
  audio.currentTime = time;
}
```

---

## 8. 数据格式规范

### 8.1 static/data.json

```json
{
  "1": [
    { "title": "Excuse Me", "filename": "001&002－Excuse Me" },
    ...
  ],
  "2": [...],
  "3": [...],
  "4": [...]
}
```

### 8.2 LRC 字幕格式

```
[al:新概念英语（一）]
[ar:MP3 同步字幕版（美音）]
[ti:Excuse Me!]
[by:...]
[00:15.11]Excuse me!|打扰一下！
[00:16.66]Yes?|是的？
```

**扩展支持**：
- 多时间标签：`[00:15.11][00:15.11]` 表示重复行（中英文分离）
- 管道分隔：`English text|中文翻译`

---

## 9. 安全性考虑

- **无用户输入存储**：所有 localStorage 数据由应用自身生成，无 XSS 风险。
- **静态资源**：无服务端动态内容，无 SQL 注入等后端风险。
- **同源策略**：Fetch 请求限于同源资源，无跨域问题。

---

## 10. 未来扩展建议

1. **模块化重构**：
   - 引入 ES Modules (`type="module"`)
   - 拆分 `lesson.js` 为多个职责单一的模块

2. **构建工具**：
   - 考虑引入 Vite/Rollup 进行打包优化
   - 代码压缩、Tree Shaking

3. **测试覆盖**：
   - 单元测试：LRC 解析、调度算法
   - E2E 测试：Playwright/Cypress 验证播放流程

4. **渐进式 Web 应用（PWA）**：
   - 添加 Service Worker 实现完全离线
   - manifest.json 支持安装到主屏幕

5. **数据库替代 localStorage**：
   - IndexedDB 支持更大存储容量
   - 结构化查询

6. **多语言国际化**：
   - 提取 UI 文本为配置
   - 支持界面多语言切换

---

以上架构文档描述了 NCE-Flow 的核心技术实现与设计思想，可作为深入理解代码逻辑或规划重构时的参考。
