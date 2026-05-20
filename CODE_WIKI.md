# 雨课堂助手 (Yuketang Helper) - 代码知识库

> **项目版本**: 1.21.9
> **最后更新**: 2026-05-20
> **项目类型**: 浏览器用户脚本 (Tampermonkey Userscript)
> **适用平台**: Chrome / Edge
> **许可证**: MIT

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [目录结构](#3-目录结构)
4. [核心模块详解](#4-核心模块详解)
   - 4.1 [入口与启动 (index.js)](#41-入口与启动-indexjs)
   - 4.2 [核心工具 (core/)](#42-核心工具-core)
   - 4.3 [网络拦截 (net/)](#43-网络拦截-net)
   - 4.4 [状态管理 (state/)](#44-状态管理-state)
   - 4.5 [AI 服务 (ai/)](#45-ai-服务-ai)
   - 4.6 [截图服务 (capture/)](#46-截图服务-capture)
   - 4.7 [业务逻辑 (tsm/)](#47-业务逻辑-tsm)
   - 4.8 [用户界面 (ui/)](#48-用户界面-ui)
5. [关键数据流](#5-关键数据流)
6. [依赖关系](#6-依赖关系)
7. [构建与运行](#7-构建与运行)
8. [配置说明](#8-配置说明)
9. [版本历史](#9-版本历史)

---

## 1. 项目概述

### 1.1 项目定位

雨课堂助手是一个基于 Tampermonkey 的浏览器用户脚本，为[雨课堂 (yuketang.cn)](https://www.yuketang.cn/)、[荷塘雨课堂 (pro.yuketang.cn)](https://pro.yuketang.cn/) 和[长江雨课堂 (changjiang.yuketang.cn)](https://changjiang.yuketang.cn/) 提供智能化课堂辅助功能。项目最初灵感来源于 [yuketang-helper](https://github.com/hotwords123/yuketang-helper.git)，现已发展为具有 AI 视觉分析能力的全功能助手。

### 1.2 核心功能

| 功能 | 描述 |
|------|------|
| **习题提醒** | 新习题发布时通过浏览器通知、自定义悬浮弹窗、提示音提醒用户 |
| **AI 智能解答** | 支持多种 VLM (Vision Language Model) API，结合幻灯片截图进行题目分析 |
| **自动作答** | 自动检测并作答新发布的习题，支持随机延时避免行为模式检测 |
| **课件浏览** | 查看、浏览课件幻灯片，支持全部/习题模式切换 |
| **PDF 导出** | 将课件幻灯片导出为高清 PDF |
| **OCR 识别** | 对幻灯片中的文字进行 OCR 识别，提取可复制文本 |
| **课件翻译** | 将 OCR 识别的文字翻译为指定语言 |
| **自动进入课堂** | 检测正在上课的课程并通过 API 自动进入 |
| **补交习题** | 支持在下课前通过 /retry 接口补交未作答或需修改的习题 |

### 1.3 支持的题目类型

| 类型码 | 题型 | internalType (Step1) | 作答格式 |
|--------|------|---------------------|----------|
| 1 | 单选题 | `single_choice` | 选择单个选项字母，如 `['A']` |
| 2 | 多选题 | `multiple_choice` | 选择多个选项字母，如 `['A','B','C']` |
| 3 | 投票题 | `single_choice` | 选择单个选项字母 |
| 4 | 填空题 | `fill_in` | 填写文本数组，如 `['内容1','内容2']` |
| 5 | 主观题 | `subjective` | 返回 `{ content: '文本', pics: [] }` 对象 |

### 1.4 支持的 AI 提供商

项目使用统一 OpenAI 兼容协议封装，支持多 "档案 (Profile)" 管理，允许用户配置多个 API 提供商并动态切换：

| 提供商 | 典型 baseUrl | 示例模型 |
|--------|-------------|----------|
| MoonShot (Kimi) | `https://api.moonshot.cn/v1/chat/completions` | `moonshot-v1-8k-vision-preview` |
| 硅基流动 | `https://api.siliconflow.cn/v1/chat/completions` | `Qwen/Qwen3-VL-32B-Instruct` |
| 阿里云百炼 | `https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions` | `qwen3-vl-plus` |
| 并行科技 | `https://ai.paratera.com/v1/chat/completions` | `GLM-4V-Plus` |

---

## 2. 整体架构

### 2.1 分层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                   雨课堂网页 (yuketang.cn / pro / changjiang)     │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  WebSocket   │  │  XHR / Fetch │  │  Vue.js App (Vuex)     │ │
│  │  (实时消息)   │  │  (API 请求)  │  │  (#app.__vue__)       │ │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬────────────┘ │
│         │                 │                       │              │
│  ╔══════╧═════════════════╧═══════════════════════╧══════════════╗
│  ║              网络拦截层 (net/)                               ║
│  ║  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────┐ ║
│  ║  │ ws-interceptor.js│ │xhr-interceptor.js│ │fetch-interceptor│║
│  ║  │ 自定义WebSocket类 │ │ 自定义XHR类       │ │ 包装原生fetch  │║
│  ║  │ 消息路由分发      │ │ API拦截+结果提取  │ │ 补充slide数据  │║
│  ║  └──────────────────┘ └──────────────────┘ └──────────────┘ ║
│  ╚══════════╤═══════════════════════════════════════════════════╝
│             │                                                    │
│  ╔══════════╧═══════════════════════════════════════════════════╗
│  ║              状态管理层 (state/)                             ║
│  ║  ┌────────────────────────┐  ┌────────────────────────────┐ ║
│  ║  │        repo.js         │  │        actions.js          │ ║
│  ║  │  • presentations: Map  │  │  • onFetchTimeline()       │ ║
│  ║  │  • slides: Map         │◄─│  • onUnlockProblem()       │ ║
│  ║  │  • problems: Map       │  │  • handleAutoAnswer()      │ ║
│  ║  │  • problemStatus: Map  │  │  • launchLessonHelper()    │ ║
│  ║  │  • 自动进入课堂元数据   │  │  • startAutoJoinLoop()     │ ║
│  ║  └────────────────────────┘  └────────────────────────────┘ ║
│  ╚══════╤═══════════════════════════════════════════════════════╝
│         │                                                        │
│  ╔══════╧═══════════════════════════════════════════════════════╗
│  ║              服务层                                          ║
│  ║  ┌───────────────┐  ┌───────────────┐  ┌──────────────────┐ ║
│  ║  │    ai/        │  │   capture/    │  │     tsm/         │ ║
│  ║  │  openai.js    │  │ screenshoot.js│  │  ai-format.js   │ ║
│  ║  │  (通用AI封装)  │  │ (截图+下载)   │  │  answer.js      │ ║
│  ║  └───────────────┘  └───────────────┘  └──────────────────┘ ║
│  ╚══════╤═══════════════════════════════════════════════════════╝
│         │                                                        │
│  ╔══════╧═══════════════════════════════════════════════════════╗
│  ║              UI 层 (ui/)                                     ║
│  ║  ┌─────────────────────────────────────────────────────────┐ ║
│  ║  │                  ui-api.js (统一接口门面)                 │ ║
│  ║  └─────────────────────────────────────────────────────────┘ ║
│  ║  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐ ║
│  ║  │ toolbar  │ │ panels/  │ │  toast   │ │   styles       │ ║
│  ║  │ (工具栏)  │ │settings  │ │ (提示)   │ │  (CSS注入)     │ ║
│  ║  │          │ │ai        │ │          │ │                │ ║
│  ║  │          │ │presentation││         │ │                │ ║
│  ║  │          │ │problem-list││         │ │                │ ║
│  ║  │          │ │active-prob││         │ │                │ ║
│  ║  │          │ │tutorial   │ │          │ │                │ ║
│  ║  │          │ │auto-answer│ │          │ │                │ ║
│  ║  └──────────┘ └──────────┘ └──────────┘ └────────────────┘ ║
│  ╚══════════════════════════════════════════════════════════════╝
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 架构特点

1. **拦截器模式 (Interceptor Pattern)**: 通过重写 `window.WebSocket`、`window.XMLHttpRequest` 和 `window.fetch` 来透明拦截雨课堂的通信数据，雨课堂自身代码无感知
2. **集中状态管理**: 使用全局 `repo` 对象（纯对象 + Map 结构）集中管理所有运行时数据，避免状态分散
3. **ES6 模块化**: 按功能域拆分为独立模块，通过 Rollup 打包为单文件 IIFE 格式输出
4. **OpenAI 协议抽象**: 统一的聊天补全接口封装，支持多 AI 提供商的无缝切换和两步 Agent 工作流
5. **Vision 融合分析**: 结合幻灯片高分辨率图片和文本结构化信息，通过两步（VLM 结构化提取 + LLM 推理作答）提升答题准确率

---

## 3. 目录结构

```
ykt-helper/                          # 主项目目录（当前开发版本）
├── src/                             # 源代码
│   ├── index.js                     # 主入口 — 按顺序启动所有子系统
│   ├── core/                        # 核心工具层
│   │   ├── env.js                  # 环境适配：封装 GM API、动态脚本加载
│   │   ├── storage.js              # localStorage 封装：StorageManager 类
│   │   ├── types.js                # 常量定义：题目类型、默认配置
│   │   └── vuex-helper.js          # Vue/Vuex 辅助：获取雨课堂应用状态
│   ├── net/                         # 网络拦截层
│   │   ├── ws-interceptor.js       # WebSocket 拦截 + 主动建链
│   │   ├── xhr-interceptor.js      # XHR 拦截 + 课堂签到/状态 API
│   │   └── fetch-interceptor.js    # Fetch 拦截（补充幻灯片数据）
│   ├── state/                       # 状态管理层
│   │   ├── repo.js                 # 数据仓库：Map 结构存储所有运行时数据
│   │   └── actions.js              # 业务动作：自动作答、自动进入课堂、路由监听
│   ├── ai/                          # AI 服务层
│   │   ├── openai.js               # 🌟 统一 OpenAI 协议封装（当前主入口）
│   │   ├── kimi.js                 # Kimi API 专用实现（历史遗留，含 prompt 模板）
│   │   ├── deepseek.js             # DeepSeek API（备用，基本未维护）
│   │   ├── gemini.js               # Gemini API 预留接口
│   │   └── openrouter.js           # OpenRouter API 预留接口
│   ├── capture/                     # 截图服务
│   │   └── screenshoot.js          # DOM截图、幻灯片图片下载、图片压缩
│   ├── tsm/                         # 雨课堂业务逻辑层
│   │   ├── ai-format.js            # Prompt 生成 + AI 答案解析
│   │   └── answer.js               # 答题提交（answer/retry/submitAnswer）
│   └── ui/                          # 用户界面层
│       ├── styles.css              # 全局样式定义
│       ├── styles.js               # 样式注入模块（import .css 为字符串注入）
│       ├── toast.js                # 轻量 Toast 提示组件
│       ├── toolbar.js              # 左下角浮动工具栏
│       ├── ui-api.js               # 🌟 UI 统一接口门面
│       └── panels/                 # 面板组件（HTML模板 + JS逻辑）
│           ├── settings.html       # 设置面板模板
│           ├── settings.js         # 设置面板：AI配置、档案管理、音效
│           ├── ai.html             # AI 对话面板模板
│           ├── ai.js               # AI 面板：提问交互、Markdown渲染
│           ├── presentation.html   # 课件面板模板
│           ├── presentation.js     # 课件面板：浏览、下载、OCR、翻译
│           ├── problem-list.html   # 题目列表模板
│           ├── problem-list.js     # 题目列表：历史题目、补交
│           ├── active-problems.html# 活跃题目模板
│           ├── active-problems.js  # 活跃题目：当前可答题列表
│           ├── tutorial.html       # 教程面板模板
│           ├── tutorial.js         # 教程面板：使用说明
│           └── auto-answer-popup.js# 自动作答结果弹窗
├── flat/                            # 扁平化旧版源码（兼容参考）
├── dist/                            # 构建产物
│   └── ykt-helper-1211.user.js     # Rollup 打包后的最终脚本
├── package.json                     # npm 配置
├── package-lock.json
├── rollup.config.mjs               # Rollup 构建配置
├── userscript.meta.js              # Tampermonkey 元数据模板
├── .npmrc
└── readme.md                        # 开发者文档

release/                             # 发布版本归档
├── ykt-helper-1180.user.js
├── ...
└── ykt-helper-1211.user.js

ykt-helper-allinone/                 # 旧版 Vue 实现（已完全归档）
├── old/
│   ├── src/
│   │   ├── components/
│   │   ├── App.vue
│   │   └── main.js
│   └── yuketang-helper-old.js

static/                              # 项目图片资源
├── 1.png ~ 5.png, 7.png            # 功能截图
├── ai.png, auto.png, agent.png     # 功能图示
├── ocr.png, tr.png, ppt.png        # 功能图示
```

---

## 4. 核心模块详解

### 4.1 入口与启动 (index.js)

**文件**: [src/index.js](file:///workspace/ykt-helper/src/index.js)

**职责**: 用户脚本的唯一入口，负责按严格顺序初始化所有子系统。

**完整启动流程**:

```javascript
(function main() {
  // 1. 延迟挂载保护：如果脚本在 DOM 就绪后才挂载，自动刷新一次
  if (maybeAutoReloadOnMount()) return;

  // 2. 防僵尸会话：每 1 分钟自动刷新页面（课堂页内跳过）
  startPeriodicReload({
    intervalMs: 1 * 60 * 1000,
    onlyWhenHidden: false,
    skipLessonPages: true
  });

  // 3. 注入全局 CSS + 加载 FontAwesome 图标库
  injectStyles();
  loadFA();  // IIFE 动态加载 FontAwesome CDN

  // 4. 挂载所有 UI 面板到 DOM（隐藏状态）
  ui._mountAll();

  // 5. 安装网络拦截器（重写原生 WebSocket / XHR）
  installWSInterceptor();
  installXHRInterceptor();
  // fetch-interceptor 在导入时自动执行

  // 6. 渲染工具栏
  installToolbar();

  // 7. 启动自动作答轮询（500ms 间隔）
  actions.startAutoAnswerLoop();

  // 8. 启动课堂助手（检测 lessonId、加载持久化数据、初始化路由监听）
  actions.launchLessonHelper();
})();
```

**关键函数**:

| 函数 | 行号 | 说明 |
|------|------|------|
| `maybeAutoReloadOnMount()` | L17-L33 | 通过 `sessionStorage` 防重复，仅在脚本晚于 DOM 挂载时执行一次刷新，确保拦截器尽早生效 |
| `startPeriodicReload(opts)` | L34-L65 | 周期性刷新页面：`intervalMs` 控制间隔，`onlyWhenHidden` 限制仅后台刷新，`skipLessonPages` 在课堂页内跳过 |

---

### 4.2 核心工具 (core/)

#### 4.2.1 env.js — 环境适配器

**文件**: [src/core/env.js](file:///workspace/ykt-helper/src/core/env.js)

**职责**: 封装 Tampermonkey 专有 API，提供运行环境无关的工具函数。

```javascript
export const gm = {
  notify(opt)       // 包装 GM_notification，若不可用则静默失败
  addStyle(css)     // 优先 GM_addStyle，降级为 <style> 标签注入
  xhr(opt)          // 包装 GM_xmlhttpRequest（绕过 CORS），未注入时抛错
  uw                // unsafeWindow（沙箱穿透）或回退到 window
};

export function loadScriptOnce(src)        // 动态加载外部 JS，自动去重
export async function ensureHtml2Canvas()  // 按需加载 html2canvas（CDN），返回函数引用
export async function ensureJsPDF()        // 按需加载 jsPDF（CDN），返回 window.jspdf
export function randInt(l, r)              // [l, r] 闭区间随机整数
export async function ensureFontAwesome()  // 按需加载 FontAwesome CSS
```

#### 4.2.2 storage.js — 存储管理器

**文件**: [src/core/storage.js](file:///workspace/ykt-helper/src/core/storage.js)

**职责**: 基于 `localStorage` 的键值存储封装，支持 JSON 序列化和 Map 结构持久化。

```javascript
export class StorageManager {
  constructor(prefix)           // 'ykt-helper:'
  get(key, defaultValue)        // JSON.parse(localStorage.getItem(prefix+key)) || defaultValue
  set(key, value)               // localStorage.setItem(prefix+key, JSON.stringify(value))
  remove(key)                   // localStorage.removeItem(prefix+key)
  getMap(key)                   // 读取为 Map 对象（经由数组反序列化）
  setMap(key, map)              // 将 Map 序列化为 [[k,v],...] 数组存储
  alterMap(key, fn)             // 原子操作：读取 → 回调修改 → 写回 Map
}

export const storage = new StorageManager('ykt-helper:');
```

**关键设计**: `alterMap` 提供读取-修改-写入的原子操作模式，用于课件容量裁剪（`repo.setPresentation` 中限制 `maxPresentations` 上限）。

#### 4.2.3 types.js — 类型与常量

**文件**: [src/core/types.js](file:///workspace/ykt-helper/src/core/types.js)

**职责**: 定义题目类型映射表和默认配置模板。

```javascript
export const PROBLEM_TYPE_MAP = {
  1: '单选题',
  2: '多选题',
  3: '投票题',
  4: '填空题',
  5: '主观题',
};

export const DEFAULT_CONFIG = {
  notifyProblems: true,
  autoAnswer: false,
  autoAnswerDelay: 3000,
  autoAnswerRandomDelay: 2000,
  iftex: true,
  ai: {
    provider: 'kimi',
    kimiApiKey: '',
    apiKey: '',
    endpoint: 'https://api.moonshot.cn/v1/chat/completions',
    model: 'moonshot-v1-8k',
    visionModel: 'moonshot-v1-8k-vision-preview',
    ocrApi: '',                 // OCR 专用 API 端点（可选）
    ocrApiKey: '',              // OCR 专用 API Key
    translateApi: '',           // 翻译专用 API 端点（可选）
    translateApiKey: '',        // 翻译专用 API Key
    translateModel: '',         // 翻译专用模型
    temperature: 0.3,
    maxTokens: 1000,
  },
  profiles: [{
    id: 'default',
    name: 'Kimi',
    baseUrl: 'https://api.moonshot.cn/v1/chat/completions',
    apiKey: '',
    model: 'moonshot-v1-8k',
    visionModel: 'moonshot-v1-8k-vision-preview',
  }],
  activeProfileId: 'default',
  showAllSlides: false,
  maxPresentations: 5,
};
```

#### 4.2.4 vuex-helper.js — Vue 状态访问

**文件**: [src/core/vuex-helper.js](file:///workspace/ykt-helper/src/core/vuex-helper.js)

**职责**: 通过 `unsafeWindow` 访问雨课堂 Vue 应用的内部 Vuex Store 状态。

```javascript
export function getVueApp()                    // 获取 document.querySelector('#app').__vue__
export function getCurrentMainPageSlideId()    // 从 store.state.currSlide 获取当前 slideId
export function watchMainPageChange(callback)  // 使用 store.watch 监听 currSlide 变化
export function waitForVueReady()              // 轮询等待 Vue 应用初始化完成
```

---

### 4.3 网络拦截 (net/)

#### 4.3.1 ws-interceptor.js — WebSocket 拦截与连接管理

**文件**: [src/net/ws-interceptor.js](file:///workspace/ykt-helper/src/net/ws-interceptor.js)

**职责**: 
- 重写 `window.WebSocket`，拦截所有 WebSocket 通信
- 解析雨课堂服务器推送的消息并分发给 `actions` 处理
- 支持为自动进入课堂主动建立 WebSocket 连接

**核心类**:

```javascript
class MyWebSocket extends WebSocket {
  static handlers = [];        // 全局处理器注册表
  static addHandler(h);        // 添加连接处理器
  intercept(cb)                // 拦截发送侧（注入消息监听）
  listen(cb)                   // 拦截接收侧（注入 message 事件监听）
}
```

**消息路由表** (ws.listen 中的 switch-case):

| WebSocket 操作码 | 分发目标 | 功能 |
|------------------|----------|------|
| `fetchtimeline` | `actions.onFetchTimeline(message.timeline)` | 批量处理时间线中已发布题目 |
| `unlockproblem` | `actions.onUnlockProblem(message.problem)` | 单题解锁：触发提醒 + 排程自动作答 |
| `lessonfinished` | `actions.onLessonFinished()` | 课程结束通知 |

**额外功能**: 递归搜索每个消息中的 `url` 字段，通过 `window.dispatchEvent(new CustomEvent('ykt:url-change', ...))` 广播，并更新 `repo.currentSelectedUrl`。

**主动建链函数**:

```javascript
export function connectOrAttachLessonWS({ lessonId, auth })
// 1. 判断当前域名选择 ws 端点（pro 用 wss://pro.yuketang.cn/wsapp/，否则用 www）
// 2. 创建 WebSocket，open 时发送 hello 握手：{ op:'hello', role:'student', auth, lessonid }
// 3. 记录到 repo.lessonSockets 和 repo.listeningLessons
```

#### 4.3.2 xhr-interceptor.js — XHR 拦截与 API 封装

**文件**: [src/net/xhr-interceptor.js](file:///workspace/ykt-helper/src/net/xhr-interceptor.js)

**职责**: 重写 `XMLHttpRequest` 拦截雨课堂 API 请求/响应，并提供自动进入课堂所需的 API 函数。

**核心类**:

```javascript
class MyXHR extends XMLHttpRequest {
  static handlers = [];
  static addHandler(h);
  intercept(cb)   // 在 load 事件中调用 cb(JSON.parse(responseText), sendPayload)
}
```

**拦截的 API 端点**:

| API 路径 | 触发动作 |
|----------|----------|
| `/api/v3/lesson/presentation/fetch` | `actions.onPresentationLoaded(id, resp.data)` |
| `/api/v3/lesson/problem/answer` | `actions.onAnswerProblem(problemId, result)` |
| `/api/v3/lesson/problem/retry` | `actions.onAnswerProblem(first.problemId, first.result)` |

**自动进入课堂 API 函数**:

```javascript
export async function getOnLesson()
// 多 URL 自动探测，获取正在上课的课堂列表
// 候选 URL: /api/v3/classroom/on-lesson, /mooc-api/v1/lms/classroom/on-lesson, /apiv3/classroom/on-lesson
// 返回: [{ lessonId, status, ... }]

export async function checkinClass(lessonId, opts)
// 多 URL 自动探测，课堂签到获取 lessonToken
// 候选：同域 v3、pro 域名 v3、www 域名 v3、mooc 旧网关、apiv3
// 返回: { token, setAuth, raw }

export async function getActivePresentationId(lessonId)
// 多 URL 自动探测，获取当前活跃课件的 presentationId
```

#### 4.3.3 fetch-interceptor.js — Fetch 补充拦截

**文件**: [src/net/fetch-interceptor.js](file:///workspace/ykt-helper/src/net/fetch-interceptor.js)

**职责**: 拦截原生 `fetch` 请求，补充提取响应中的 `slides` 数组并灌入 `repo.slides`。以 IIFE 形式自动执行，通过 `window.__YKT_FETCH_PATCHED__` 防重复注入。

---

### 4.4 状态管理 (state/)

#### 4.4.1 repo.js — 数据仓库

**文件**: [src/state/repo.js](file:///workspace/ykt-helper/src/state/repo.js)

**职责**: 集中存储所有运行时数据，是各模块间数据传递的唯一汇聚点。

**完整数据结构**:

```javascript
export const repo = {
  // ─── 课件与题目数据 ───
  presentations: new Map(),      // presId -> { id, slides[], ... }
  slides: new Map(),             // slideId -> { id, cover, coverAlt, problem, thumbnail, ... }
  problems: new Map(),           // problemId -> { problemId, problemType, body, options[], blanks[], answers[] }
  problemStatus: new Map(),      // problemId -> { presentationId, slideId, startTime, endTime, done, autoAnswerTime, answering }
  encounteredProblems: [],       // [{ problemId, problemType, body, options, slide, presentationId, result }]

  // ─── 当前会话状态 ───
  currentPresentationId: null,
  currentSlideId: null,
  currentLessonId: null,         // 从 URL /lesson/fullscreen/v3/{lessonId} 提取
  currentSelectedUrl: null,      // 从 WebSocket 消息中提取（教师端推送的 URL）

  // ─── 自动进入课堂多课堂状态 ───
  listeningLessons: new Set(),   // 已建立 WS 监听的 lessonId 集合
  lessonTokens: new Map(),       // lessonId -> token
  lessonSockets: new Map(),      // lessonId -> WebSocket 实例
  autoJoinRunning: false,        // 自动加入轮询开关
  autoJoinedLessons: new Set(),  // 标记为自动进入连接的课堂
  forceAutoAnswerLessons: new Set(),

  // ─── 方法 ───
  setPresentation(id, data),               // 设置课件 + localStorage 持久化 + 容量裁剪
  upsertSlide(slide),                      // 更新/插入单张幻灯片
  upsertProblem(prob),                     // 更新/插入单个题目
  pushEncounteredProblem(prob, slide, pid),// 追加遭遇题目记录
  loadStoredPresentations(),               // 从 localStorage 按 currentLessonId 分组加载
  markLessonConnected(lessonId, ws, token),
  isLessonConnected(lessonId),
  markLessonAutoJoined(lessonId, enabled),
};
```

#### 4.4.2 actions.js — 业务动作处理器

**文件**: [src/state/actions.js](file:///workspace/ykt-helper/src/state/actions.js)

**职责**: 处理所有核心业务逻辑，是连接网络拦截、AI 服务和 UI 的核心调度器。

**导出函数**:

```javascript
export const actions = {
  // ─── 网络事件响应 ───
  onFetchTimeline(timeline),               // 遍历 timeline，对 type==='problem' 的项调用 onUnlockProblem
  onPresentationLoaded(id, data),          // 课件数据写入 repo → 提取内部 slides/problems → 更新 UI
  onUnlockProblem(data),                   // 🌟 核心：习题解锁 → 计算时间窗口 → 提醒 → 排程自动作答
  onLessonFinished(),                      // 浏览器原生通知
  onAnswerProblem(problemId, result),      // 更新 problem.result 和 encounteredProblems 中的 result

  // ─── 自动作答 ───
  handleAutoAnswer(problem),               // 公共入口，内部调用 handleAutoAnswerInternal
  startAutoAnswerLoop(),                   // 500ms 间隔轮询到期 autoAnswerTime 并触发作答

  // ─── 手动操作 ───
  submit(problem, content),                // 解析手动输入 → submitAnswer
  parseManual(problemType, content),       // 按题型解析手动输入的答案文本
  navigateTo(presId, slideId),             // 更新 currentPresentationId/currentSlideId → 刷新 UI

  // ─── 课堂助手 ───
  launchLessonHelper(),                    // 提取 lessonId、打标签、加载持久化课件、启动自动加入

  // ─── 自动进入课堂 ───
  startAutoJoinLoop(),                     // 5s 间隔轮询 getOnLesson → checkinClass → connectOrAttachLessonWS
  stopAutoJoinLoop(),
  maybeStartAutoJoin(),                    // 条件启动（需 ui.config.autoJoinEnabled 为 true）
  installRouterRearm(),                    // 监听 history.pushState/replaceState + popstate + visibilitychange
  startAutoClickOnOnLessonBar(),           // 在主页自动点击"正在上课"条 → 路由跳转至课堂
};
```

**自动作答内部流程** (`handleAutoAnswerInternal`):

```
输入：problem 对象
│
├─ 1. 门控检查：status 是否存在 / 是否正在作答 / 是否已有结果 / 是否超时
│
├─ 2. 无 AI API Key 分支
│      └─ makeDefaultAnswer(problem)
│         ├─ 单选/多选/投票 → ['A']
│         ├─ 填空 → [' 1']
│         └─ 主观题 → { content: '略', pics: [] }
│      └─ submitAnswer(problem, parsed)
│
└─ 3. 有 AI API Key 分支（融合模式）
       ├─ captureSlideImage(slideId)             ← 优先使用幻灯片高清图片
       │  └─ 失败时回退：captureProblemForVision()  ← DOM截图
       ├─ formatProblemForVision(problem, TYPE_MAP, hasTextInfo)
       ├─ queryAIVision(imageBase64, textPrompt, aiCfg, { problemType })
       ├─ parseAIAnswer(problem, aiAnswer)
       └─ submitAnswer(problem, parsed, { startTime, endTime, ... })
```

**无 AI 默认答案生成** (`makeDefaultAnswer`):

适用于未配置 API Key 的场景，生成占位答案确保流程不中断。

---

### 4.5 AI 服务 (ai/)

#### 4.5.1 openai.js — 统一 AI 接口（当前主模块）

**文件**: [src/ai/openai.js](file:///workspace/ykt-helper/src/ai/openai.js)

**职责**: 提供统一的 OpenAI 兼容协议封装，支持文本模式、Vision 模式（单步/两步）、OCR 和翻译功能。仅此模块被 `actions.js` 实际引用，其他 AI 模块为历史遗留。

**核心导出**:

```javascript
// 文本模式（单步 Vision/LLM）
export async function queryAI(question, aiCfg)
// 直接使用文本模型，不做图像分析

// Vision 模式（支持单步/两步自动切换）
export async function queryAIVision(imageBase64, textPrompt, aiCfg, options = {})
// options: { disableTwoStep, twoStepDebug, timeout, problemType }
// 自动判断：若 profile 设置了独立 textModel ≠ visionModel，则启用两步工作流

// OCR 文字识别
export async function queryOCRVision(imageBase64, aiCfg)
// 使用专用 OCR API（ocrApi/ocrApiKey）或回退到默认 VLM

// 课件翻译
export async function queryTranslationText(text, targetLanguage, aiCfg)
// 使用专用翻译 API（translateApi/translateApiKey/translateModel）或回退到默认 LLM
```

**内部关键函数**:

```javascript
function getActiveProfile(aiCfg)
// 从 profiles 数组中按 activeProfileId 查找，回退到旧版 kimiApiKey 兼容模式
// 若 profiles 为空但存在旧版 kimiApiKey，自动构造 legacy profile

function makeChatUrl(profile)
// 智能拼接 /chat/completions 后缀，兼容多种 API 格式

function chatCompletion(profile, payload, debugLabel, timeoutMs)
// 底层 OpenAI 协议请求封装，使用 GM_xmlhttpRequest 发送 POST

async function singleStepVisionCall(profile, cleanBase64List, textPrompt, options)
// 一步 Vision：直接发送图片+文本到 VLM，获取答案
```

**两步 Vision 工作流** (`queryAIVision` 的核心算法):

```
前置条件：profile.visionModel ≠ profile.model（即拥有独立 VLM 和 LLM）
若 disableTwoStep 或模型相同 → 回退到 singleStepVisionCall

Step 1 — VLM 结构化提取:
  输入: 幻灯片截图 + 可选题目标注文本
  系统提示: 详细的题型识别规则（含选项字母规则）
  输出: JSON { question_type, stem, options, image_facts, requires_image_for_solution }
  失败处理: 回退到 singleStepVisionCall

题型合并逻辑:
  1. 优先使用传入的 problemType（后端可靠数据）
  2. 其次使用 VLM 推断的 question_type
  3. 忽略 'unknown' / 'visual_only' 类型
  4. 全部缺失时回退为 'subjective'

requires_image_for_solution === true → 回退到 singleStepVisionCall（图片依赖题）

Step 2 — LLM 纯文本推理:
  输入: 结构化题目（stem + options + image_facts）
  输出: 格式化的答案（含 答案: + 解释:）
  失败处理: 回退到 singleStepVisionCall
```

#### 4.5.2 kimi.js — Kimi API 专用实现

**文件**: [src/ai/kimi.js](file:///workspace/ykt-helper/src/ai/kimi.js)

**状态**: 历史遗留（未被 actions 引用），包含与 openai.js 类似的 prompt 模板和 Kimi Vision 调用逻辑。

```javascript
export async function queryKimi(question, aiCfg)        // 文本模式
export async function queryKimiVision(imageBase64, textPrompt, aiCfg)  // Vision 单步模式
```

#### 4.5.3 其他 AI 模块

| 文件 | 状态 | 说明 |
|------|------|------|
| `deepseek.js` | 备用 | DeepSeek API 调用实现 |
| `gemini.js` | 预留 | 仅占位，无实现代码 |
| `openrouter.js` | 预留 | 仅占位，无实现代码 |

---

### 4.6 截图服务 (capture/)

#### 4.6.1 screenshoot.js

**文件**: [src/capture/screenshoot.js](file:///workspace/ykt-helper/src/capture/screenshoot.js)

**职责**: 为 Vision AI 提供题目图像数据。

```javascript
// DOM 截图（使用 html2canvas）
export async function captureProblemScreenshot()
// 选择器优先级：.ques-title → .problem-body → .ppt-inner → .ppt-courseware-inner → body

// 从 repo 获取幻灯片高清图片并下载为 base64
export async function captureSlideImage(slideId)
// 图片 URL 优先级：slide.coverAlt → slide.cover → slide.image → slide.thumbnail
// 下载后自动压缩：>1MB 则降至 0.5 质量

// Vision API 用的 base64（DOM 截图 → canvas.toDataURL）
export async function captureProblemForVision()

// 内部工具：下载图片 → Canvas → base64（自动处理 CORS）
async function downloadImageAsBase64(url)
```

---

### 4.7 业务逻辑 (tsm/)

#### 4.7.1 ai-format.js — Prompt 生成与答案分析

**文件**: [src/tsm/ai-format.js](file:///workspace/ykt-helper/src/tsm/ai-format.js)

**职责**: 为 AI 生成精确的 Prompt 并解析 AI 返回的答案。

```javascript
// 生成纯文本方式 Prompt
export function formatProblemForAI(problem, TYPE_MAP)

// 生成 Vision 融合模式 Prompt
export function formatProblemForVision(problem, TYPE_MAP, hasTextInfo)
// hasTextInfo 为 true 时，附加要求 "若图片内容与文本冲突，以图片为准"

// 格式化题目内容用于 UI 展示
export function formatProblemForDisplay(problem, TYPE_MAP)

// 核心：解析 AI 回答为结构化答案
export function parseAIAnswer(problem, aiAnswer)
```

**`parseAIAnswer` 解析规则**:

| 题型 | 解析策略 | 正则/算法 |
|------|----------|-----------|
| 单选题 (type=1) / 投票题 (type=3) | 匹配第一个大写字母 | `/答案[:：]\s*([A-Z])/` → 回退 `answerLine.match(/[A-Z]/)` |
| 多选题 (type=2) | 顿号/逗号/连续字母分隔 | 顿号 → 逗号 → 连续字母正则 → 去重排序 |
| 填空题 (type=4) | 按分隔符拆分 + 格式清理 | 清理类型标识 → 按 `,，;；\s+` 拆分 → 短文本(≤50字)返回数组，否则返回 `{content}` 对象 |
| 主观题 (type=5) | 保留完整文本 | 清理"主观题/论述题"前缀 → 返回 `{ content, pics: [] }` |

**填空/主观题多行答案支持**: 从"答案:"行开始，逐行收集直到遇到"解释:"或文本结束

#### 4.7.2 answer.js — 答题提交

**文件**: [src/tsm/answer.js](file:///workspace/ykt-helper/src/tsm/answer.js)

**职责**: 封装雨课堂答题提交的全部逻辑。

```javascript
// 正常答题
export async function answerProblem(problem, result, options)
// POST /api/v3/lesson/problem/answer  { problemId, problemType, dt, result }

// 补交答案（超时后）
export async function retryAnswer(problem, result, dt, options)
// POST /api/v3/lesson/problem/retry  { problems: [{ problemId, problemType, dt, result }] }

// 🌟 智能提交入口（自动判断 answer / retry）
export async function submitAnswer(problem, result, submitOptions)
```

**`submitAnswer` 决策逻辑**:

```
1. autoGate 判定（针对自动进入课堂场景）:
   ├─ ui.config.autoAnswer 为 true 或 课堂在 forceAutoAnswerLessons 中
   └─ → 随机等待 (autoAnswerDelay ~ autoAnswerDelay + autoAnswerRandomDelay)

2. 路由决策:
   ├─ pastDeadline (now >= endTime) 或 forceRetry → retryAnswer
   │  └─ dt 计算: 优先 startTime + offset，其次 endTime - offset，最后 now - offset
   └─ 否则 → answerProblem
```

**请求头常量**:

```javascript
const DEFAULT_HEADERS = () => ({
  'Content-Type': 'application/json',
  'xtbz': 'ykt',
  'X-Client': 'h5',
  'Authorization': 'Bearer ' + localStorage.getItem('Authorization'),
});
```

---

### 4.8 用户界面 (ui/)

#### 4.8.1 toolbar.js — 浮动工具栏

**文件**: [src/ui/toolbar.js](file:///workspace/ykt-helper/src/ui/toolbar.js)

**职责**: 在页面左下角渲染一个半透明浮动工具栏，提供所有功能的快捷入口。

**工具栏按钮**:

| DOM ID | 图标 | 功能 | 交互 |
|--------|------|------|------|
| `#ykt-btn-bell` | 🔔 | 习题提醒切换 | click 切换 `ui.config.notifyProblems` |
| `#ykt-btn-pres` | 📄 | 课件浏览面板 | click 切换 `ui.showPresentationPanel()` |
| `#ykt-btn-ai` | 🤖 | AI 解答面板 | click 切换 `ui.showAIPanel()` |
| `#ykt-btn-auto-answer` | ✨ | 自动作答开关 | click 切换 `ui.config.autoAnswer` |
| `#ykt-btn-settings` | ⚙️ | 设置面板 | click 切换 `ui.toggleSettingsPanel()` |
| `#ykt-btn-help` | ❓ | 使用教程 | click 切换 `ui.toggleTutorialPanel()` |

#### 4.8.2 ui-api.js — UI 统一接口门面

**文件**: [src/ui/ui-api.js](file:///workspace/ykt-helper/src/ui/ui-api.js)

**职责**: 整合所有 UI 模块，提供统一的接口给业务层。同时负责配置的初始化、持久化和面板层级管理。

**配置初始化**: 在模块顶层执行 `Object.assign({}, DEFAULT_CONFIG, storage.get('config', {}))`，并对所有可能缺失的深层字段做防御性默认值填充。

**导出对象** `ui`:

```javascript
export const ui = {
  config,                    // 配置对象（响应式引用 _config）
  saveConfig(),              // 持久化到 localStorage('ykt-helper:config')

  // ─── 面板控制（自动管理 z-index）───
  showPresentationPanel(visible),
  showProblemListPanel(visible),
  showAIPanel(visible),
  toggleSettingsPanel(),
  toggleTutorialPanel(),
  _mountAll(),               // 初始化时一次性挂载全部面板到 DOM

  // ─── 数据更新代理 ───
  updatePresentationList: PresPanel.updatePresentationList,
  updateSlideView: PresPanel.updateSlideView,
  askAIForCurrent: AIPanel.askAIForCurrent,
  updateProblemList: ProbListPanel.updateProblemList,
  updateActiveProblems: ActivePanel.updateActiveProblems,

  // ─── 通知系统 ───
  notifyProblem(problem, slide),    // 自定义悬浮弹窗（右下角，可拖拽）
  toast,                            // 轻量 Toast 组件
  nativeNotify: gm.notify,          // 浏览器原生通知

  // ─── 工具 ───
  getProblemDetail(problem),        // 格式化题目为可读文本
  setCustomNotifyAudio({ src, name }),
  updateAutoAnswerBtn(),
  _bringToFront(panelElement),      // z-index 递增提升面板
  _playNotifySound(volume),         // 自定义音频 → Web Audio API 合成音回退
  _playNotifyTone(volume),          // Web Audio API 生成"叮-咚"提示音
};
```

**面板层级管理**: 使用 `currentZIndex` 全局计数器，每次面板置顶时自增，避免面板相互遮挡。

**通知弹窗 (notifyProblem)**: 创建可拖拽的 `div` 面板，包含缩略图、题目标题、题目详情、关闭按钮。支持自定义提示音（URL 音频 → Web Audio API 合成 "叮-咚" 双音色回退）。

#### 4.8.3 toast.js — 轻量提示

**文件**: [src/ui/toast.js](file:///workspace/ykt-helper/src/ui/toast.js)

纯函数组件，创建居中顶部浮现的文字提示，默认 2 秒后淡出消失。

#### 4.8.4 面板组件 (panels/)

| 面板 | 主要 JS 文件 | 核心功能 |
|------|-------------|----------|
| **设置面板** | [settings.js](file:///workspace/ykt-helper/src/ui/panels/settings.js) | API Key 管理、AI 档案 (Profile) 增删改、自动作答参数、音效设置、OCR/翻译专用 API 配置、自动进入课堂选项、配置重置 |
| **AI 面板** | [ai.js](file:///workspace/ykt-helper/src/ui/panels/ai.js) | 提问交互、Markdown 渲染、支持选中多张 PPT 组合提问、OCR 按钮集成 |
| **课件面板** | [presentation.js](file:///workspace/ykt-helper/src/ui/panels/presentation.js) | 幻灯片浏览（全部/问题模式切换）、单页/多页选中提问、PDF 导出、OCR 文字识别、翻译功能、课件列表管理 |
| **题目列表** | [problem-list.js](file:///workspace/ykt-helper/src/ui/panels/problem-list.js) | 历史题目展示、手动答题区域、强制补交（提供 JSON 模板） |
| **活跃题目** | [active-problems.js](file:///workspace/ykt-helper/src/ui/panels/active-problems.js) | 显示当前可作答的活跃题目列表 |
| **教程面板** | [tutorial.js](file:///workspace/ykt-helper/src/ui/panels/tutorial.js) | 使用说明和版本信息 |
| **自动作答弹窗** | [auto-answer-popup.js](file:///workspace/ykt-helper/src/ui/panels/auto-answer-popup.js) | 显示 AI 自动作答结果，支持手动修改后提交 |

**面板架构模式**: 每对面板采用 HTML 模板 + JS 逻辑分离架构，HTML 文件通过 `@bkuri/rollup-plugin-string` 在构建时作为字符串内联进 JS 模块，JS 文件暴露 `mount*Panel`、`show*Panel`、`toggle*Panel`、`update*` 等标准接口。

---

## 5. 关键数据流

### 5.1 新习题发布 → 自动作答 完整链路

```
雨课堂服务器 WebSocket push
    │
    ▼
ws-interceptor.js: ws.listen()
    │ 解析 message.op === 'unlockproblem'
    │ 提取 { prob, sid, pres, dt, limit }
    ▼
actions.onUnlockProblem(data)
    │
    ├─ 从 repo.problems.get(prob) 获取题目详情
    │ 从 repo.slides.get(sid) 获取幻灯片
    │
    ├─ 构建 problemStatus 记录:
    │    { startTime: dt, endTime: dt + 1000*limit, done: false, answering: false }
    │
    ├─ 门控检查: 已超时? 已有结果? → 跳过
    │
    ├─ notifyProblems 开关 → ui.notifyProblem(problem, slide)
    │    └─ 创建悬浮弹窗 + 播放提示音
    │
    └─ autoAnswer 开关 → 设置 status.autoAnswerTime = now + delay + random
         │
         ▼
startAutoAnswerLoop() — 500ms 轮询
    │ 检测到 now >= status.autoAnswerTime
    ▼
handleAutoAnswerInternal(problem)
    │
    ├─ 无 API Key 分支:
    │    makeDefaultAnswer() → submitAnswer() → 完成
    │
    └─ 有 API Key 分支:
         │
         ├─ captureSlideImage(slideId)
         │    └─ coverAlt → cover → image → thumbnail → DOM截图(回退)
         │
         ├─ formatProblemForVision(problem, TYPE_MAP, hasTextInfo)
         │
         ├─ queryAIVision(imageBase64, textPrompt, aiCfg, { problemType })
         │    ├─ Step1: VLM 结构化提取 JSON
         │    │    └─ 失败 → 单步回退
         │    ├─ 题型合并: problemType(后端) > question_type(VLM) > 'subjective'
         │    ├─ requires_image_for_solution? → 单步回退
         │    └─ Step2: LLM 纯文本推理作答
         │         └─ 失败 → 单步回退
         │
         ├─ parseAIAnswer(problem, aiAnswer)
         │    └─ 按题型正则解析 → 结构化答案
         │
         ├─ submitAnswer(problem, parsed, { startTime, endTime, lessonId })
         │    ├─ 超时? → retryAnswer
         │    └─ 未超时 → answerProblem
         │
         └─ updateStatus + 显示弹窗
```

### 5.2 课件数据加载流

```
雨课堂 XHR/Fetch 请求
    │
    ├─ xhr-interceptor.js ──► /api/v3/lesson/presentation/fetch
    │   └─ intercept → actions.onPresentationLoaded(id, resp.data)
    │
    └─ fetch-interceptor.js ──► 任意包含 slides 的响应
        └─ 提取 json.data.slides → repo.slides.set(sid, slide)

actions.onPresentationLoaded(id, data)
    │
    ├─ repo.setPresentation(id, data)
    │    └─ localStorage 持久化 (按 courseId 分组 + 容量裁剪)
    │
    ├─ 遍历 pres.slides:
    │    ├─ repo.upsertSlide(slide)
    │    ├─ 若有 slide.problem:
    │    │    ├─ repo.upsertProblem(slide.problem)
    │    │    └─ repo.pushEncounteredProblem(slide.problem, slide, id)
    │
    └─ ui.updatePresentationList()
         └─ PresPanel 重新渲染课件列表
```

### 5.3 自动进入课堂流

```
launchLessonHelper()
    │
    ├─ 提取 lessonId (URL regex)
    ├─ repo.loadStoredPresentations()
    └─ maybeStartAutoJoin()
         │
         └─ startAutoJoinLoop() — 5s 轮询
              │
              ├─ getOnLesson() — 多 URL 自动适配
              │    └─ 返回 status===1 的课堂列表
              │
              ├─ 遍历: 跳过已连接的课堂
              │
              ├─ checkinClass(lessonId) — 多 URL 自动适配
              │    └─ 返回 { token, setAuth }
              │
              ├─ connectOrAttachLessonWS({ lessonId, auth: token })
              │    └─ 创建 WebSocket → 发送 hello 握手
              │
              └─ repo.markLessonAutoJoined(lessonId, true)

同时: startAutoClickOnOnLessonBar() (主页)
    ├─ tryApiJumpFirst() — 优先 API 跳转
    │    └─ getOnLesson → location.assign()
    └─ attachGuardAndTrigger() — DOM 接管 onlesson 条点击
         └─ MutationObserver 监听 .onlesson .jump_lesson__bar
```

---

## 6. 依赖关系

### 6.1 模块依赖图

```
index.js (入口)
│
├── core/env.js ◄────────── 被几乎所有模块依赖（GM API封装 + 工具函数）
├── core/storage.js ◄──────  repo.js, ui-api.js
├── core/types.js ◄────────  ui-api.js, actions.js, ai-format.js
├── core/vuex-helper.js ◄── ai.js (AI面板)
│
├── net/ws-interceptor.js ──► state/actions.js, state/repo.js
├── net/xhr-interceptor.js ──► state/actions.js
├── net/fetch-interceptor.js ──► state/repo.js, state/actions.js
│
├── state/repo.js ◄────────  被 net/, state/, ui/ 多个模块依赖
├── state/actions.js ──►     state/repo.js, ui/ui-api.js, tsm/answer.js,
│                            ai/openai.js, capture/screenshoot.js,
│                            tsm/ai-format.js, ui/panels/auto-answer-popup.js,
│                            net/ws-interceptor.js, net/xhr-interceptor.js
│
├── ai/openai.js ──►         core/env.js
├── ai/kimi.js ──►           core/env.js (历史遗留，未被 actions 引用)
│
├── capture/screenshoot.js ──► core/env.js, state/repo.js
│
├── tsm/answer.js ──►        ui/ui-api.js, state/repo.js
├── tsm/ai-format.js ──►     (无外部依赖，纯函数工具)
│
└── ui/
    ├── ui/styles.js ──►     core/env.js
    ├── ui/toolbar.js ──►    ui/ui-api.js
    ├── ui/toast.js ──►      (独立无依赖)
    ├── ui/ui-api.js ──►     core/env.js, core/storage.js, core/types.js,
    │                        state/repo.js, ui/toast.js, 各 panels
    └── ui/panels/*.js ──►   ui/ui-api.js, state/repo.js, state/actions.js,
                             ai/openai.js, tsm/ai-format.js, capture/screenshoot.js
```

### 6.2 外部运行时依赖

| 依赖 | 加载方式 | 用途 | 状态 |
|------|----------|------|------|
| Tampermonkey GM API | 浏览器扩展内置 | 跨域请求、样式注入、系统通知、标签管理、沙箱穿透 | 必须 |
| FontAwesome 6.4.0 | CDN 动态 `<link>` (index.js) | 工具栏图标渲染 | 必须 |
| html2canvas | CDN 按需加载 (env.js) | DOM 截图（Vision 回退方案） | 可选 |
| jsPDF 2.5.1 | `@require` CDN (userscript.meta.js) | 课件导出 PDF | 必须 |
| MathJax 3 | `@require` CDN (userscript.meta.js) | LaTeX 公式渲染 | 必须 |

### 6.3 构建时依赖

| 包 | 版本 | 用途 |
|----|------|------|
| rollup | ^4.21.2 | 模块打包器 |
| @rollup/plugin-node-resolve | ^15.2.3 | 解析 node_modules 依赖 |
| @rollup/plugin-commonjs | ^25.0.7 | CommonJS → ESM 转换 |
| @rollup/plugin-replace | ^5.0.7 | 编译期 `__BUILD_TIME__` / `__BUILD_VERSION__` 替换 |
| @rollup/plugin-terser | ^0.4.4 | 轻量压缩（不混淆，保留注释） |
| @rollup/plugin-typescript | ^11.1.6 | TypeScript 支持（.ts 文件） |
| @bkuri/rollup-plugin-string | ^1.0.0 | 将 .html/.css 文件作为字符串导入 |
| typescript | ^5.6.3 | 类型检查（编译时） |

---

## 7. 构建与运行

### 7.1 环境要求

- Node.js ≥ 18
- npm

### 7.2 构建步骤

```bash
cd ykt-helper

npm install         # 安装构建依赖
npm run build       # 生产构建 → dist/ykt-helper-1211.user.js
npm run dev         # 开发模式（文件监听自动重建）
```

### 7.3 Rollup 构建配置详解

**文件**: [rollup.config.mjs](file:///workspace/ykt-helper/rollup.config.mjs)

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `input` | `src/index.js` | ES6 模块入口 |
| `output.format` | `iife` | 立即执行函数，适配 Userscript 运行环境 |
| `output.banner` | `meta` 字符串 | 将 `userscript.meta.js` 中的 Tampermonkey 元数据头部注入产物顶部 |
| `output.inlineDynamicImports` | `true` | 禁用代码分割，所有代码打包为单文件 |
| `plugins[0]` | `string()` | 允许 `import html from './x.html'` 等字符串导入 |
| `plugins[1]` | `resolve({ browser: true })` | 浏览器端模块解析 |
| `plugins[2]` | `commonjs()` | CJS 兼容 |
| `plugins[3]` | `replace()` | 注入 `__BUILD_TIME__` 和 `__BUILD_VERSION__` |
| `plugins[4]` | `terser({ mangle: false, beautify: true })` | 美化输出，保留所有注释，不混淆变量名 |
| `treeshake.moduleSideEffects` | `true` | 保留模块副作用，避免过度摇树 |

### 7.4 用户脚本元数据

**文件**: [userscript.meta.js](file:///workspace/ykt-helper/userscript.meta.js)

关键元数据：

```
@name         AI雨课堂助手（JS版）
@version      1.21.9
@match        https://pro.yuketang.cn/web/*
@match        https://changjiang.yuketang.cn/web/*
@match        https://*.yuketang.cn/lesson/fullscreen/v3/*
@match        https://*.yuketang.cn/v2/web/*
@grant        GM_addStyle, GM_notification, GM_xmlhttpRequest,
              GM_openInTab, GM_getTab, GM_getTabs, GM_saveTab, unsafeWindow
@connect      *（运行时可访问所有域名）
@run-at       document-start（尽早注入，确保拦截器先于雨课堂代码生效）
@require      jspdf, mathjax（预加载 CDN 依赖）
```

### 7.5 部署方式

1. 在 Chrome/Edge 浏览器安装 [Tampermonkey](https://www.tampermonkey.net/) 扩展
2. 将 `dist/ykt-helper-1211.user.js` 的内容导入 Tampermonkey（新建脚本粘贴 / 直接安装）
3. 访问雨课堂页面，左下角出现工具栏图标即安装成功

---

## 8. 配置说明

### 8.1 运行时持久化配置

配置存储在 `localStorage`，键为 `ykt-helper:config`，JSON 格式。初始化时与 `DEFAULT_CONFIG` 合并。

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `notifyProblems` | boolean | true | 新习题提醒开关 |
| `autoAnswer` | boolean | false | 自动作答开关 |
| `autoAnswerDelay` | number | 3000 | 自动作答基础等待时间(ms) |
| `autoAnswerRandomDelay` | number | 2000 | 随机延迟增量范围(ms) |
| `autoJoinEnabled` | boolean | false | 自动进入课堂开关 |
| `autoAnswerOnAutoJoin` | boolean | true | 自动进入后是否立即开启自动作答 |
| `notifyPopupDuration` | number | 5000 | 习题通知弹窗显示时长(ms) |
| `notifyVolume` | number | 0.6 | 提示音音量(0-1) |
| `customNotifyAudioSrc` | string | '' | 自定义提示音 URL |
| `customNotifyAudioName` | string | '' | 自定义提示音名称 |
| `iftex` | boolean | true | 启用 MathJax/LaTeX 渲染 |
| `showAllSlides` | boolean | false | 课件面板是否显示全部页面（否则仅习题） |
| `maxPresentations` | number | 5 | 本地缓存课件数量上限 |
| `ai.profiles` | array | `[{...}]` | AI 配置档案列表 |
| `ai.activeProfileId` | string | 'default' | 当前激活档案 ID |
| `ai.ocrApi` | string | '' | OCR 专用 API 地址 |
| `ai.ocrApiKey` | string | '' | OCR 专用 API Key |
| `ai.translateApi` | string | '' | 翻译专用 API 地址 |
| `ai.translateApiKey` | string | '' | 翻译专用 API Key |
| `ai.translateModel` | string | '' | 翻译专用模型名称 |

### 8.2 AI 档案 (Profile) 结构

每个 profile 对象包含以下字段：

```javascript
{
  id: '唯一标识符',
  name: '显示名称（如 Kimi、硅基流动）',
  baseUrl: 'https://api.xxx.com/v1/chat/completions',  // OpenAI 兼容端点
  apiKey: 'sk-...',
  model: '文本模型（如 moonshot-v1-8k）',
  visionModel: '视觉模型（如 moonshot-v1-8k-vision-preview）',
}
```

---

## 9. 版本历史

| 版本 | 主要特性 |
|------|----------|
| 1.21.1 | 课件翻译功能 |
| 1.21.0 | OCR 系统：独立 OCR API 配置、课件文字识别与复制 |
| 1.20.3 | 高清图片下载（PDF 导出升级） |
| 1.20.2 | 多张图片输入、自动刷新优化、题型识别修复 |
| 1.20.1 | Agent 模式：两步 AI 工作流（VLM 结构化 + LLM 推理） |
| 1.20.0 | 多 AI 提供商支持：OpenAI 统一协议封装、档案管理 |
| 1.19.2 | 子 frame PPT 获取、补交逻辑回归 |
| 1.19.1 | Markdown 渲染器、Edge 兼容性维护 |
| 1.19.0 | Fetch 拦截器、自定义习题提示音、代码重构 |
| 1.18.7 | 自动进入课堂功能、设置持久化 |
| 1.18.5 | 无 AI 环境随机答题模式、修改 AI 答案 |
| 1.18.0 | 全面转向 Vision 模式（弃用纯文本模式） |
| 1.17.0 | Kimi Vision 模型支持 |
| 1.16.0 | 大模型答案弹窗 |
| 1.15.0 | 答案模糊匹配 |
| 1.14.0 | 自动刷新页面 |
| 1.13.0 | 自动作答功能 |
| 1.12.0 | AI 作答功能首发 |

---

## 附录

### A. 环境适配表

脚本自动检测运行环境（基于 `location.hostname`）：

| 域名 | 环境类型 | WebSocket 端点 | 说明 |
|------|----------|---------------|------|
| `www.yuketang.cn` | 标准雨课堂 | `wss://www.yuketang.cn/wsapp/` | 清华校内可用 |
| `pro.yuketang.cn` | 荷塘雨课堂 | `wss://pro.yuketang.cn/wsapp/` | 清华专属平台 |
| `changjiang.yuketang.cn` | 长江雨课堂 | — | 专属平台 API 兼容 |

### B. 日志规范

所有模块使用统一的日志格式：

```
[雨课堂助手][级别][模块名] 消息内容
```

级别标识：
- `DBG` — 调试信息（用于开发诊断）
- `INFO` — 正常运行时信息
- `WARN` — 非致命警告
- `ERR` — 错误信息

### C. 安全与隐私

1. API Key 仅存储在浏览器 `localStorage` 中，不上传至任何第三方服务器
2. 题目数据仅在用户主动触发或开启自动作答时发送给配置的 AI API 端点
3. 脚本不收集、不上传用户的任何个人信息
4. 自动作答使用随机延迟避免机械式行为模式
5. 答案正确性不保证，用户应自行判断

### D. 后续开发方向

根据 `todolist.md`，项目规划的 2.0 大版本开发方向包括：
- 多 Agent 图像识别方案（当前为单 Agent + 两步工作流）
- API Quota 耗尽时自动切换备用 API
- 多轮对话支持（含错误回答的 ReAct/ReGeneration 反馈机制）
- 学习助手功能（OCR 获取全部页信息 + 用户修改问题）
- 更好的多模型切换（LLM 与 VLM 区分）