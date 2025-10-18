# NCE-Flow 项目分析

> 更新时间：2024-10-18

本分析文档聚焦对 NCE-Flow 静态站点项目的整体理解和拆解，供后续维护、功能扩展或架构迁移时参考。

---

## 1. 项目整体架构与目录结构

项目为纯静态站点，所有资源均位于仓库根目录下，可直接通过任意静态文件服务器部署。主要目录及文件说明如下：

```
NCE-Flow/
├── assets/              # 样式与脚本（styles.css, app.js, lesson.js）
├── static/
│   └── data.json        # 四册课程元数据（标题与文件名映射）
├── NCE1~NCE4/           # 各册音频（.mp3）与字幕（.lrc/.lrc.bak）资源
├── images/              # 书籍封面等静态图片
├── index.html           # 首页/课程导航页
├── lesson.html          # 课文点读页
├── book.html            # 兼容旧链接的跳转页
├── CNAME                # GitHub Pages 自定义域名配置
└── README.md            # 项目说明
```

特征总结：
- **无构建产物、无打包流程**，源代码即部署文件。
- 通过 `location.hash` 实现册别与课文的锚点路由导航。
- 所有业务状态通过浏览器本地存储（`localStorage` / `sessionStorage`）管理，支持离线使用。
- 音频与字幕资源按书本分目录存放，文件命名与 `static/data.json` 保持一致，便于静态查找。

## 2. 使用的编程语言、框架与主要技术栈

| 层次         | 选型                              | 说明                                                         |
|--------------|-----------------------------------|--------------------------------------------------------------|
| 标记语言     | HTML5                             | 两个主要页面（`index.html`、`lesson.html`） + 跳转页 `book.html`|
| 样式         | 原生 CSS                          | `assets/styles.css` 使用 CSS 变量、响应式、渐变等效果           |
| 脚本         | 原生 JavaScript（ES6+）           | 无框架、无模块化打包；核心逻辑封装在立即执行函数 (IIFE) 中       |
| 音频字幕数据 | LRC + MP3                         | 课文字幕使用 LRC 标准扩展格式，支持中英双语对齐                 |
| 数据存储     | `localStorage` / `sessionStorage` | 记忆设置、收藏、最近播放、断点进度、自动续播等                 |
| 构建/部署    | 无需构建                          | 直接托管到 GitHub Pages 或任意静态服务器                      |

**未使用任何第三方框架或依赖包**（无 npm、无 CDN），最大化减少部署复杂度并保持离线可用性。

## 3. 核心功能模块与组件

### 3.1 首页（`index.html` + 内联脚本 + `assets/app.js`）

- **数据加载**：通过 `fetch('static/data.json')` 获取课程列表，根据 `location.hash` 决定当前册别。
- **收藏管理**：使用 `localStorage` 的 `nce_favs`（JSON array）存储收藏课程 ID（`NCE#/filename`）。
- **最近学习**：读取 `nce_recents`，展示当前册别下最近一次学习的课程与断点。
- **锚点导航**：顶部 segmented tab 通过 hash 切换册别，页面监听 `hashchange` 重新渲染。
- **语言模式控制**：`assets/app.js` 负责语言 segmented 控件，切换时调整 `<body>` 类名（`lang-en / lang-bi / lang-cn`），首页与课文页共用。

### 3.2 课文页（`lesson.html` + `assets/lesson.js`）

课文页是项目的核心复杂模块，承担音频播放调度、字幕同步与交互。

**主要职责与亮点：**
- **资源解析**：根据锚点 `lesson.html#NCE#/filename` 拼接 MP3/LRC 路径并加载。
- **LRC 解析**：`loadLrc()` 支持多时间标签、元数据 (`[al]`, `[ar]`, `[ti]`, `[by]`)，并能识别中英文行（使用 `|` 分隔或相邻重复时间标签）。
- **精准播放调度**：
  - 连读模式下使用“静音 + 快速 seek + 双层 `requestAnimationFrame`”实现无缝切句。
  - 点读模式确保严格停止在当前句尾，避免漏出下一句音频。
  - 结合 `setTimeout` 与 `requestAnimationFrame` 构建多级调度，适配不同倍速与 iOS 设备。
- **iOS 优化**：首次点击自动执行静音播放以解锁音频上下文 (`unlockAudioSync`)，规避 iOS 自动播放限制。
- **播放设置**（均持久化到 `localStorage`）：
  - 阅读模式：连读 (`continuous`) / 点读 (`single`)。
  - 自动跟随：是否自动滚动定位 (`autoFollow`)。
  - 自动续播下一课 (`autoContinue`)。
  - 播放倍速（0.75x ~ 2.5x）。
- **进度记录**：定时将 `currentTime` 与句子索引保存至 `nce_lastpos`，离开页面或刷新后可恢复。
- **最近列表增强**：播放或跳转时更新 `nce_recents`，便于首页展示上次学习。
- **邻接课文**：解析 `static/data.json` 获得上一课/下一课链接，支持自动跳转提示。
- **键盘/可访问性**：设置弹窗启用焦点陷阱、支持 `Escape` 关闭等基本可访问性优化。

### 3.3 UI 与交互组件

- **Segmented Controls**：统一使用 `.segmented` 样式实现多种切换控件（语言、阅读模式、自动续播）。
- **收藏按钮**：SVG 星标按钮与课程卡片悬浮联动，状态切换后触发列表重新渲染。
- **通知栏**：`showNotification()` 在自动续播等场景弹出浮层提示。
- **设置面板**：模态对话框包含播放速度、阅读模式、自动续播与跟随开关，支持“恢复默认”。

## 4. 代码组织方式与设计模式

- **立即执行函数 (IIFE)**：`assets/app.js` 与 `assets/lesson.js` 均采用 IIFE 封装作用域，避免全局变量污染，仅通过 `window.NCE_APP` 暴露必要接口。
- **事件驱动**：大量使用 DOM 事件（点击、`hashchange`、`timeupdate`、`play/pause`）驱动 UI 状态与业务逻辑。
- **状态集中管理**：
  - 将播放状态、模式、最近记录等统一保存到 `localStorage`/`sessionStorage`，页面之间通过约定的 key 协作。
  - 以 `idx`、`segmentEnd` 等变量复合维护播放进度。
- **播放调度模式**：结合 `setTimeout` 与 `requestAnimationFrame` 构建“远端定时 + 近端精确”的双层调度，确保倍速与 iOS 兼容性。
- **渐进增强**：针对不支持 `color-mix` 等 CSS 功能的浏览器提供 `@supports not` 回退；对 iOS 进行特定处理，保证体验一致。

## 5. 依赖关系与外部库

- **零第三方依赖**：无 npm、无 CDN、无外链脚本。
- **纯标准 API**：所有功能依托浏览器内建能力（Fetch、Audio、Storage、History/Location 等）。
- **外部资源**：仅引用仓库内静态资源（MP3/LRC/图片）。CNAME 对应的是 GitHub Pages 自定义域名 `nce.luzhenhua.cn`。

## 6. 配置文件与环境设置

- `CNAME`：GitHub Pages 自定义域名绑定。
- `README.md`：提供本地启动方式（Python `http.server`、`npx serve` 或 VSCode Live Server），提醒需 HTTP 服务以绕过浏览器文件协议对 `fetch` 的限制。
- 无 `.env`、`.gitignore`、`package.json` 或其他环境配置文件，部署环境依赖极低。

## 7. 构建与部署方式

- **构建**：无构建流程；代码即静态资源，可直接使用。
- **部署**：
  - 官方示例部署在 GitHub Pages，结合 `CNAME` 绑定独立域名。
  - Releases 提供整包下载，解压后通过任意静态服务器（或离线环境）即可访问。
  - 如需本地调试，执行 `python -m http.server 8000`（或其他静态服务）访问 `http://localhost:8000`。
- **离线支持**：由于所有音频与字幕资源内置，部署后无需外网请求即可使用（首次加载需注意缓存体积）。

---

## 8. 补充观察与建议

1. **数据扩展**：新增课程时需同时更新 `static/data.json` 与相应的 MP3/LRC 文件，并确保命名一致。
2. **资源体积**：`NCE1~NCE4` 音频体积较大，打包分发时需考虑带宽；若未来计划 Web 打包，可引入按需加载或按册拆分下载。
3. **可维护性**：`lesson.js` 逻辑较长（700+ 行），如需大规模迭代，可考虑拆分模块或引入构建工具进行代码分层。
4. **跨端支持**：已针对 iOS 做专门优化，Android/桌面浏览器使用标准 API，整体兼容性良好。
5. **测试策略**：当前无自动化测试；如后续需要回归保障，可考虑引入 Playwright/Puppeteer 做端到端测试。不过这会引入新的依赖与构建流程，需要在“零依赖”目标与自动化之间权衡。

---

以上分析覆盖了项目结构、技术栈、核心功能、代码组织、依赖状况、配置及部署方式，可作为新成员上手或规划改造时的速读材料。若需更细致的业务流程或交互说明，可在此基础上扩展用例级文档。
