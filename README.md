开发者指南（Developer Guide）
本文面向希望理解、运行、调试、贡献「子沐智能中转路由」的开发者。 面向终端用户的快速入门见 README.md。

1. 技术栈
层级	技术	版本/说明
桌面壳	Electron	^33.0.0（主进程 + 渲染进程 + preload 隔离）
后端代理	Node.js + Express	^4.21.0，OpenAI 兼容端点
前端	原生 JS（DOM + CSS）	无框架，直接操作 DOM
本地推理	llama.cpp server	应用内下载，Qwen2.5-7B-Instruct（Q4_K_M）
测试	Node.js 原生	16 套 test-*.js，覆盖单测/集成/UI/加固/长会话
打包	electron-builder	^25.0.0，输出 portable + nsis
运行环境：Node.js >= 18（推荐 20/22），Windows 10+ / macOS / Linux。

2. 目录结构
llm-router-app/
├── main.js                 # Electron 主进程入口（IPC 注册、托盘、窗口管理）
├── preload.js              # 预加载脚本（渲染进程 → 主进程的桥梁）
├── package.json            # 项目元数据、脚本、electron-builder 配置
├── package-lock.json       # 依赖锁版本
├── README.md               # 面向用户的快速入门 + 功能说明
├── LICENSE                 # MIT License
├── .npmrc                  # 国内镜像（npmmirror）
│
├── src/                    # 核心后端逻辑（Node.js，约 8000 行）
│   ├── config.js           # 配置中心：18 预设渠道 / 价格表 / 路由 / 记忆 / 远程
│   ├── router.js           # 路由引擎（~3000 行）：策略 / 回退 / 熔断 / 缓存 / 压缩 / 意图 / 影子
│   ├── proxy.js            # OpenAI 兼容代理层（流式 / 认证 / 重放）
│   ├── local-engine.js     # 本地模型引擎（llama.cpp 子进程管理）
│   ├── brain.js            # 路由大脑（决策 / 经验 / 自我进化）
│   ├── tools.js            # 大脑工具集（12 个内置工具 + 沙箱）
│   ├── intent.js           # 请求级意图/复杂度识别
│   ├── prose-compress.js   # 规则 / 抽取 / 结构化压缩引擎（25 条规则）
│   ├── cache.js            # 1GB LRU 精确缓存（含流式重放）
│   ├── semantic-cache.js   # 语义缓存（嵌入相似度）
│   ├── healthcheck.js      # 健康检测 + Key 自动恢复（带乐观锁 CAS）
│   ├── model-memory.js     # 内置记忆系统（画像 / 画像 / 功能分析）
│   ├── trace.js            # 请求 Trace 日志（JSONL 持久化 + 导出）
│   ├── slo.js              # SLO 监控（可用性 / 延迟 / 熔断）
│   ├── guard.js            # 输入敏感词护栏
│   ├── mandatory-guard.js  # 不可关闭的强制内容拦截
│   ├── remote.js           # 远程访问（账号密码 / Key / Nginx / 一键迁移）
│   ├── downloader.js       # 模型文件下载器
│   └── ...
│
├── renderer/               # 渲染进程前端
│   ├── app.js              # 主应用 UI（渠道管理 / 路由 / 设置 / 大脑）
│   └── assets/             # 静态资源（图标 / 背景）
│
├── docs/                   # 技术文档（本文件即其中之一）
│   ├── ADR-001-local-engine.md
│   ├── INTERACTION-AUDIT.md
│   ├── SECURITY-AUDIT.md
│   ├── OMNIROUTE-BENCHMARK.md
│   └── OMNIROUTE-DEEP-DIVE.md
│
├── test-*.js               # 16 套测试（详见第 5 节）
├── build/                  # 构建资源（图标等）
└── dist/                   # electron-builder 输出（gitignore，不纳入）
3. 架构总览
┌─────────────────────────────────────────────────────────────┐
│  客户端（任意 OpenAI 兼容客户端：LobsterAI / OpenWebUI / curl） │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  代理层（src/proxy.js · Express）                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  认证 / 请求限流 / 流式 SSE / 头信息透传 / 缓存命中重放 │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────────┘
                           │ IPC / require
┌──────────────────────────▼──────────────────────────────────┐
│  路由引擎（src/router.js）                                   │
│  ┌────────────┬────────────┬────────────┬──────────────┐   │
│  │ 意图识别   │ 策略选择   │ 韧性体系   │ 省钱优化      │   │
│  │ intent.js  │ 10 种策略  │ 熔断/退避  │ 缓存/压缩/亲和│   │
│  │ auto/intent│ priority/  │ 429/熔断   │ 1GB LRU+语义 │   │
│  │            │ round-robin│ 跨模型回退 │ 10 压缩引擎   │   │
│  └────────────┴────────────┴────────────┴──────────────┘   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  路由大脑（src/brain.js）· 内置 Qwen2.5-7B 本地推理     │ │
│  │  12 个工具 + 定时巡检 + 经验沉淀 + 自我进化             │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  可观测性：Trace（src/trace.js）+ 记忆（model-memory）  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
进程模型：

进程	职责
主进程（main.js）	IPC 路由、窗口管理、托盘、远程服务、本地引擎生命周期
预加载（preload.js）	渲染进程 API 白名单，暴露 window.api
渲染进程（renderer/app.js）	UI 交互，通过 window.api 调用后端
数据隔离：渲染进程不能直接访问 Node.js API；所有后端访问必须通过 preload 暴露的 IPC 通道。

4. 核心模块职责
4.1 代理层（src/proxy.js）
Express 服务器，默认 localhost:3456
认证：Bearer 渠道 Key → X-Provider-Key 头透传
流式：SSE 分块转发 + 消费统计
缓存命中时 SSE 重放（reconstructFromChatSse）
4.2 路由引擎（src/router.js）
入口：routeModel(model, opts) → routeWithFallback(model, opts)

数据流：

接收模型名 + 消息 + 参数
computeAffinityKey：取 system 前缀做缓存亲和
selectProviders：按策略/驻留/SLO/熔断/配额选出可用渠道
tryRouteModel：遍历候选渠道 → selectKey → callProvider
成功 → recordSuccess + 配额记账 + 缓存写入
失败 → 错误分类 → 同渠道换 Key / 切渠道 / 跨模型回退
finishSuccessTrace：Trace 收尾 + 影子双跑 + 意图成本归因
10 种路由策略：auto / priority / weighted / round-robin / random / strict-random / p2c / least-used / cost-optimized / quota-aware

4.3 本地引擎（src/local-engine.js）
子进程管理：下载 llama-server.exe + GGUF 模型
启动：spawn(llama-server, ...)，HTTP 本地 127.0.0.1:随机端口
工具循环：chatWithTools 支持 function calling 多轮
状态：running 布尔值；空闲超时自动休眠（可被 autoWakeHook 唤醒）
4.4 路由大脑（src/brain.js）
内置 Qwen2.5-7B 通过本地引擎推理
工具沙箱：TOOL_CATEGORY（只读 / 写 / 控制）+ 单次决策写操作上限
触发：定时巡检 + 事件唤醒 + 手动
经验：每次决策记录基线 → 下次评估成效 → 好经验沉淀
4.5 配置中心（src/config.js）
18 个预设渠道 + 用户自定义
价格表：内置 50+ 模型价格，支持自定义
持久化：userData/config.json（Electron app.getPath('userData')）
高频写：persistDebounced()（30s 合并落盘，quota/stats 用）
5. 运行与构建
5.1 环境准备
# 克隆
git clone https://github.com/你的用户名/llm-smart-router.git
cd llm-smart-router

# 安装依赖
npm install

# 开发模式运行（Electron 窗口 + 后端代理）
npm start
5.2 运行测试
# 全部测试
for f in test-*.js; do node "$f"; done

# 单个测试
node test-deep.js          # 深测（路由 / 配额 / 熔断 / 压缩 / 缓存）
node test-hardening.js     # 加固测试（安全 / 边界 / 异常输入）
node test-long-session.js  # 长会话（多轮对话 / 上下文增长）
node test-omniroute.js     # OmniRoute 对标回归
5.3 构建打包
# Windows 便携版 + 安装版（electron-builder）
npm run build

# 仅便携版
npm run build-portable

# 仅安装版
npm run build-nsis
产物输出到 dist/（gitignore，不纳入源码包）。

6. 配置结构
{
  routing: {
    strategy: 'priority',          // priority | round-robin | random | weighted | auto | ...
    maxRetries: 3,                 // 最多切几个渠道
    maxKeyRetries: null,           // null=同渠道所有 Key 都试；N=预算模式
    timeout: 60000,                // 单次请求超时（ms）
    failover: true,                // 启用跨渠道回退
    fallback: true,                // 启用跨模型回退
    circuitBreaker: { enabled, threshold, cooldownMs },
    quota: { enabled, period, limits },
    promptCacheAffinity: true,     // 提示缓存亲和
    lkgp: true,                    // 最后成功渠道优先
    // ...更多见 src/config.js 默认值
  },
  providers: [
    { id, name, baseUrl, keys: [], models: [], enabled, region, pathStyle, ... }
  ],
  local: { backend: 'llama', ... },      // 本地引擎配置
  remote: { enabled, port, ... },         // 远程访问配置
  // ... cache / logging / quality / guard / semanticCache / slo / ...
}
7. IPC 通道总览
前缀	数量	说明
config:*	3	配置读写 / 重置
provider:*	3	渠道 CRUD
health:*	3	健康检测
route:* / proxy:*	3	路由测试 / 代理状态
stats:* / cache:*	4	统计 / 缓存
logs:*	4	Trace 日志（含 status 端点）
smart:*	1	质量/护栏/语义缓存/SLO/驻留/影子配置
memory:*	5	记忆系统
local:*	7	本地引擎（下载/启动/停止/工具）
remote:*	7	远程访问 + 迁移（含限流）
brain:*	6	路由大脑（状态/运行/撤销栈）
data:* / price:*	4	用量统计 / 价格管理
app:* / window:* / clipboard:* / bg:*	7	应用级（自启/托盘/窗口/剪贴板/背景）
共 73 个 IPC 通道，详见 main.js 中的 ipcMain.handle 注册点。

8. 测试策略
测试套件	覆盖范围
test-smoke.js	冒烟测试：启动 / 配置 / 路由 / 代理基础流程
test-deep.js	深测：路由策略 / 配额软降级 / 熔断 / 压缩 / 缓存 / 回退链
test-hardening.js	加固：边界输入 / 异常状态 / 竞态 / 安全护栏
test-integration.js	集成：完整请求生命周期（代理 → 路由 → 上游 → 响应）
test-local.js	本地引擎：子进程 / 工具循环 / 下载
test-brain.js	路由大脑：工具沙箱 / 最小变更 / 撤销栈
test-memory.js	记忆系统：画像 / 同步 / 导出
test-remote.js	远程访问：登录 / 迁移 / Nginx 配置
test-intent.js	意图识别：代码/翻译/数学/总结/创意/闲聊分类
test-omniroute.js	OmniRoute 对标回归
test-long-session.js	长会话：多轮对话 / 上下文增长 / token 统计
test-theme.js	UI 主题：亮/暗模式切换
test-ui.js	UI 交互：渠道管理 / 设置面板
test-mandatory.js	强制拦截：不可关闭的内容护栏
test-backend.js	后端基础：模型列表 / 健康检测 / Key 管理
9. 贡献指南
Fork → Clone → npm install
创建功能分支：git checkout -b feat/xxx
运行测试：for f in test-*.js; do node "$f"; done
提交前确保 npm run build 正常（至少 build-portable）
PR 时说明：改动文件 / 动机 / 测试结果
代码风格：

2 空格缩进
变量/函数用 camelCase
类名用 PascalCase
异步用 async/await，避免回调地狱
错误处理：不要 catch 后静默 console.error，要么抛要么记录
10. 已知限制与路线图
项目	状态
流式 token 精确统计	✅ 已修复（C3 + proxy.js 流式用法回填）
多 Key 同渠道重试	✅ 已修复（M6 双层循环）
健康检测乐观锁	✅ 已修复（M1 CAS）
语义缓存 variant 隔离	✅ 已修复（H2）
前端无框架重构	⏳ 待评估（当前原生 JS 可维护，暂不需 Vue/React）
多平台打包	⏳ 待评估（当前仅 Windows，macOS/Linux 需要 CI）
11. 许可证
MIT License — 详见根目录 LICENSE 文件。开发者指南（Developer Guide）
本文面向希望理解、运行、调试、贡献「子沐智能中转路由」的开发者。 面向终端用户的快速入门见 README.md。

1. 技术栈
层级	技术	版本/说明
桌面壳	Electron	^33.0.0（主进程 + 渲染进程 + preload 隔离）
后端代理	Node.js + Express	^4.21.0，OpenAI 兼容端点
前端	原生 JS（DOM + CSS）	无框架，直接操作 DOM
本地推理	llama.cpp server	应用内下载，Qwen2.5-7B-Instruct（Q4_K_M）
测试	Node.js 原生	16 套 test-*.js，覆盖单测/集成/UI/加固/长会话
打包	electron-builder	^25.0.0，输出 portable + nsis
运行环境：Node.js >= 18（推荐 20/22），Windows 10+ / macOS / Linux。

2. 目录结构
llm-router-app/
├── main.js                 # Electron 主进程入口（IPC 注册、托盘、窗口管理）
├── preload.js              # 预加载脚本（渲染进程 → 主进程的桥梁）
├── package.json            # 项目元数据、脚本、electron-builder 配置
├── package-lock.json       # 依赖锁版本
├── README.md               # 面向用户的快速入门 + 功能说明
├── LICENSE                 # MIT License
├── .npmrc                  # 国内镜像（npmmirror）
│
├── src/                    # 核心后端逻辑（Node.js，约 8000 行）
│   ├── config.js           # 配置中心：18 预设渠道 / 价格表 / 路由 / 记忆 / 远程
│   ├── router.js           # 路由引擎（~3000 行）：策略 / 回退 / 熔断 / 缓存 / 压缩 / 意图 / 影子
│   ├── proxy.js            # OpenAI 兼容代理层（流式 / 认证 / 重放）
│   ├── local-engine.js     # 本地模型引擎（llama.cpp 子进程管理）
│   ├── brain.js            # 路由大脑（决策 / 经验 / 自我进化）
│   ├── tools.js            # 大脑工具集（12 个内置工具 + 沙箱）
│   ├── intent.js           # 请求级意图/复杂度识别
│   ├── prose-compress.js   # 规则 / 抽取 / 结构化压缩引擎（25 条规则）
│   ├── cache.js            # 1GB LRU 精确缓存（含流式重放）
│   ├── semantic-cache.js   # 语义缓存（嵌入相似度）
│   ├── healthcheck.js      # 健康检测 + Key 自动恢复（带乐观锁 CAS）
│   ├── model-memory.js     # 内置记忆系统（画像 / 画像 / 功能分析）
│   ├── trace.js            # 请求 Trace 日志（JSONL 持久化 + 导出）
│   ├── slo.js              # SLO 监控（可用性 / 延迟 / 熔断）
│   ├── guard.js            # 输入敏感词护栏
│   ├── mandatory-guard.js  # 不可关闭的强制内容拦截
│   ├── remote.js           # 远程访问（账号密码 / Key / Nginx / 一键迁移）
│   ├── downloader.js       # 模型文件下载器
│   └── ...
│
├── renderer/               # 渲染进程前端
│   ├── app.js              # 主应用 UI（渠道管理 / 路由 / 设置 / 大脑）
│   └── assets/             # 静态资源（图标 / 背景）
│
├── docs/                   # 技术文档（本文件即其中之一）
│   ├── ADR-001-local-engine.md
│   ├── INTERACTION-AUDIT.md
│   ├── SECURITY-AUDIT.md
│   ├── OMNIROUTE-BENCHMARK.md
│   └── OMNIROUTE-DEEP-DIVE.md
│
├── test-*.js               # 16 套测试（详见第 5 节）
├── build/                  # 构建资源（图标等）
└── dist/                   # electron-builder 输出（gitignore，不纳入）
3. 架构总览
┌─────────────────────────────────────────────────────────────┐
│  客户端（任意 OpenAI 兼容客户端：LobsterAI / OpenWebUI / curl） │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  代理层（src/proxy.js · Express）                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  认证 / 请求限流 / 流式 SSE / 头信息透传 / 缓存命中重放 │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────────────────┘
                           │ IPC / require
┌──────────────────────────▼──────────────────────────────────┐
│  路由引擎（src/router.js）                                   │
│  ┌────────────┬────────────┬────────────┬──────────────┐   │
│  │ 意图识别   │ 策略选择   │ 韧性体系   │ 省钱优化      │   │
│  │ intent.js  │ 10 种策略  │ 熔断/退避  │ 缓存/压缩/亲和│   │
│  │ auto/intent│ priority/  │ 429/熔断   │ 1GB LRU+语义 │   │
│  │            │ round-robin│ 跨模型回退 │ 10 压缩引擎   │   │
│  └────────────┴────────────┴────────────┴──────────────┘   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  路由大脑（src/brain.js）· 内置 Qwen2.5-7B 本地推理     │ │
│  │  12 个工具 + 定时巡检 + 经验沉淀 + 自我进化             │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  可观测性：Trace（src/trace.js）+ 记忆（model-memory）  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
进程模型：

进程	职责
主进程（main.js）	IPC 路由、窗口管理、托盘、远程服务、本地引擎生命周期
预加载（preload.js）	渲染进程 API 白名单，暴露 window.api
渲染进程（renderer/app.js）	UI 交互，通过 window.api 调用后端
数据隔离：渲染进程不能直接访问 Node.js API；所有后端访问必须通过 preload 暴露的 IPC 通道。

4. 核心模块职责
4.1 代理层（src/proxy.js）
Express 服务器，默认 localhost:3456
认证：Bearer 渠道 Key → X-Provider-Key 头透传
流式：SSE 分块转发 + 消费统计
缓存命中时 SSE 重放（reconstructFromChatSse）
4.2 路由引擎（src/router.js）
入口：routeModel(model, opts) → routeWithFallback(model, opts)

数据流：

接收模型名 + 消息 + 参数
computeAffinityKey：取 system 前缀做缓存亲和
selectProviders：按策略/驻留/SLO/熔断/配额选出可用渠道
tryRouteModel：遍历候选渠道 → selectKey → callProvider
成功 → recordSuccess + 配额记账 + 缓存写入
失败 → 错误分类 → 同渠道换 Key / 切渠道 / 跨模型回退
finishSuccessTrace：Trace 收尾 + 影子双跑 + 意图成本归因
10 种路由策略：auto / priority / weighted / round-robin / random / strict-random / p2c / least-used / cost-optimized / quota-aware

4.3 本地引擎（src/local-engine.js）
子进程管理：下载 llama-server.exe + GGUF 模型
启动：spawn(llama-server, ...)，HTTP 本地 127.0.0.1:随机端口
工具循环：chatWithTools 支持 function calling 多轮
状态：running 布尔值；空闲超时自动休眠（可被 autoWakeHook 唤醒）
4.4 路由大脑（src/brain.js）
内置 Qwen2.5-7B 通过本地引擎推理
工具沙箱：TOOL_CATEGORY（只读 / 写 / 控制）+ 单次决策写操作上限
触发：定时巡检 + 事件唤醒 + 手动
经验：每次决策记录基线 → 下次评估成效 → 好经验沉淀
4.5 配置中心（src/config.js）
18 个预设渠道 + 用户自定义
价格表：内置 50+ 模型价格，支持自定义
持久化：userData/config.json（Electron app.getPath('userData')）
高频写：persistDebounced()（30s 合并落盘，quota/stats 用）
5. 运行与构建
5.1 环境准备
# 克隆
git clone https://github.com/你的用户名/llm-smart-router.git
cd llm-smart-router

# 安装依赖
npm install

# 开发模式运行（Electron 窗口 + 后端代理）
npm start
5.2 运行测试
# 全部测试
for f in test-*.js; do node "$f"; done

# 单个测试
node test-deep.js          # 深测（路由 / 配额 / 熔断 / 压缩 / 缓存）
node test-hardening.js     # 加固测试（安全 / 边界 / 异常输入）
node test-long-session.js  # 长会话（多轮对话 / 上下文增长）
node test-omniroute.js     # OmniRoute 对标回归
5.3 构建打包
# Windows 便携版 + 安装版（electron-builder）
npm run build

# 仅便携版
npm run build-portable

# 仅安装版
npm run build-nsis
产物输出到 dist/（gitignore，不纳入源码包）。

6. 配置结构
{
  routing: {
    strategy: 'priority',          // priority | round-robin | random | weighted | auto | ...
    maxRetries: 3,                 // 最多切几个渠道
    maxKeyRetries: null,           // null=同渠道所有 Key 都试；N=预算模式
    timeout: 60000,                // 单次请求超时（ms）
    failover: true,                // 启用跨渠道回退
    fallback: true,                // 启用跨模型回退
    circuitBreaker: { enabled, threshold, cooldownMs },
    quota: { enabled, period, limits },
    promptCacheAffinity: true,     // 提示缓存亲和
    lkgp: true,                    // 最后成功渠道优先
    // ...更多见 src/config.js 默认值
  },
  providers: [
    { id, name, baseUrl, keys: [], models: [], enabled, region, pathStyle, ... }
  ],
  local: { backend: 'llama', ... },      // 本地引擎配置
  remote: { enabled, port, ... },         // 远程访问配置
  // ... cache / logging / quality / guard / semanticCache / slo / ...
}
7. IPC 通道总览
前缀	数量	说明
config:*	3	配置读写 / 重置
provider:*	3	渠道 CRUD
health:*	3	健康检测
route:* / proxy:*	3	路由测试 / 代理状态
stats:* / cache:*	4	统计 / 缓存
logs:*	4	Trace 日志（含 status 端点）
smart:*	1	质量/护栏/语义缓存/SLO/驻留/影子配置
memory:*	5	记忆系统
local:*	7	本地引擎（下载/启动/停止/工具）
remote:*	7	远程访问 + 迁移（含限流）
brain:*	6	路由大脑（状态/运行/撤销栈）
data:* / price:*	4	用量统计 / 价格管理
app:* / window:* / clipboard:* / bg:*	7	应用级（自启/托盘/窗口/剪贴板/背景）
共 73 个 IPC 通道，详见 main.js 中的 ipcMain.handle 注册点。

8. 测试策略
测试套件	覆盖范围
test-smoke.js	冒烟测试：启动 / 配置 / 路由 / 代理基础流程
test-deep.js	深测：路由策略 / 配额软降级 / 熔断 / 压缩 / 缓存 / 回退链
test-hardening.js	加固：边界输入 / 异常状态 / 竞态 / 安全护栏
test-integration.js	集成：完整请求生命周期（代理 → 路由 → 上游 → 响应）
test-local.js	本地引擎：子进程 / 工具循环 / 下载
test-brain.js	路由大脑：工具沙箱 / 最小变更 / 撤销栈
test-memory.js	记忆系统：画像 / 同步 / 导出
test-remote.js	远程访问：登录 / 迁移 / Nginx 配置
test-intent.js	意图识别：代码/翻译/数学/总结/创意/闲聊分类
test-omniroute.js	OmniRoute 对标回归
test-long-session.js	长会话：多轮对话 / 上下文增长 / token 统计
test-theme.js	UI 主题：亮/暗模式切换
test-ui.js	UI 交互：渠道管理 / 设置面板
test-mandatory.js	强制拦截：不可关闭的内容护栏
test-backend.js	后端基础：模型列表 / 健康检测 / Key 管理
9. 贡献指南
Fork → Clone → npm install
创建功能分支：git checkout -b feat/xxx
运行测试：for f in test-*.js; do node "$f"; done
提交前确保 npm run build 正常（至少 build-portable）
PR 时说明：改动文件 / 动机 / 测试结果
代码风格：

2 空格缩进
变量/函数用 camelCase
类名用 PascalCase
异步用 async/await，避免回调地狱
错误处理：不要 catch 后静默 console.error，要么抛要么记录
10. 已知限制与路线图
项目	状态
流式 token 精确统计	✅ 已修复（C3 + proxy.js 流式用法回填）
多 Key 同渠道重试	✅ 已修复（M6 双层循环）
健康检测乐观锁	✅ 已修复（M1 CAS）
语义缓存 variant 隔离	✅ 已修复（H2）
前端无框架重构	⏳ 待评估（当前原生 JS 可维护，暂不需 Vue/React）
多平台打包	⏳ 待评估（当前仅 Windows，macOS/Linux 需要 CI）
11. 许可证
MIT License — 详见根目录 LICENSE 文件。
