# CODEBUDDY.md This file provides guidance to CodeBuddy when working with code in this repository.

## 1) 仓库已验证事实（仅基于当前代码）

- 这是一个 **Tauri 2 + React + TypeScript** 的桌面应用仓库。
- 前端入口在 `src/`，后端（Tauri/Rust）入口在 `src-tauri/`。
- `src-tauri/tauri.conf.json` 已配置：
  - `beforeDevCommand`: `npm run dev`
  - `beforeBuildCommand`: `npm run build`
- `README.md`（开发与构建章节）要求：
  - Node.js v18+
  - Rust（Tauri 运行时）
  - 开发运行：`npm run tauri dev`
  - 构建：`npm run tauri build`
- `package.json` 的已存在脚本：`dev`、`typecheck`、`build`、`preview`、`tauri`、`release:preflight`、`sync-version`。
- **未发现** `lint`、`test`、`unit test` 脚本；不要在实现中假设它们存在。

## 2) 常用开发命令（每条说明尽量简短）

> 所有命令在仓库根目录执行；`tauri` 命令会通过 npm 脚本调用并先做必要预处理。

### 安装依赖
- `npm install`
- 用途：安装前端依赖与 Tauri CLI 相关 npm 依赖，首次拉取仓库后先执行。

### 前端开发服务（Vite）
- `npm run dev`
- 用途：仅启动前端开发服务器（通常由 Tauri dev 的 beforeDevCommand 自动触发）。

### TypeScript 类型检查
- `npm run typecheck`
- 用途：运行 `tsc --noEmit`，只做类型检查，不产出构建文件。

### 前端构建
- `npm run build`
- 用途：先同步版本再执行 TypeScript 编译与 Vite 打包；通常由 Tauri build 前自动触发。

### Tauri 开发模式
- `npm run tauri dev`
- 用途：启动 Tauri 桌面开发运行，联动前端 dev server 与 Rust 侧应用壳，适合日常联调。

### Tauri 生产构建
- `npm run tauri build`
- 用途：生成桌面发行构建（平台相关产物），发布前使用。

### 版本同步脚本
- `npm run sync-version`
- 用途：执行 `scripts/sync-version.js`，用于在构建链前同步版本信息。

### 发布前检查
- `npm run release:preflight`
- 用途：运行 `scripts/release/preflight.cjs` 进行发布前校验。

### 预览构建结果
- `npm run preview`
- 用途：本地预览 Vite 构建产物（前端层）。

### 关于 lint / test
- 当前仓库 `package.json` **没有** `npm run lint`、`npm test` 或单测脚本。
- 若后续新增，请先在 `package.json` 注册脚本，再更新本文件命令节。

## 3) 高层架构（面向改动定位）

### 3.1 双端结构

- **前端（`src/`）**：React 页面、状态管理、服务调用与 UI 交互。
- **后端（`src-tauri/`）**：Rust 命令层、业务模块层、数据模型层、系统能力（窗口、托盘、更新、OAuth、WebSocket 等）。
- 两端通过 Tauri `invoke` 命令桥接：前端 service 调用命令名，后端 `invoke_handler` 统一注册。

### 3.2 前端组织方式

- `src/App.tsx` 是应用壳：
  - 懒加载多个平台页面与通用页面。
  - 处理全局事件、更新通知、窗口交互等。
- `src/pages/`：按平台或功能拆分页面（如账号页、实例页、设置页等）。
- `src/services/`：封装命令调用，负责把前端动作映射到后端命令。
- `src/stores/`：基于 Zustand 的状态管理。
  - `createProviderAccountStore.ts`：账号类平台的通用 store 工厂。
  - `createInstanceStore.ts`：实例生命周期（list/create/update/start/stop）通用 store 工厂。

### 3.3 后端分层方式

- `src-tauri/src/lib.rs`：Tauri 主入口，负责：
  - 插件初始化（dialog/fs/opener/notification/single-instance/updater/process）
  - 启动后台服务（websocket、web_report）
  - 窗口关闭行为、托盘初始化、启动恢复逻辑（OAuth pending listener 等）
  - 注册全部 `invoke` 命令
- `src-tauri/src/commands/`：命令暴露层（按平台/领域拆分）。
- `src-tauri/src/modules/`：业务实现层（账号、实例、OAuth、配额、调度、系统服务等）。
- `src-tauri/src/models/`：数据结构层（Account、Instance、Quota 等）。

### 3.4 多平台扩展模式

- 代码按“平台 + 通用工厂”结合：
  - 平台专属命令模块：如 `workbuddy`、`codebuddy_cn` 等。
  - 前端通过命令前缀拼接实现复用（例如 `createCodebuddySuiteService.ts` 通过 `codebuddy_cn` / `workbuddy` 前缀生成调用）。
  - 状态层通过工厂函数复用通用行为，再注入平台 service/mapper。
- 结论：新增或改造平台时，优先复用工厂与既有命令命名约定，减少重复实现。

## 4) 变更执行建议（与当前仓库结构对齐）

### 4.1 改前端页面时
- 优先确认：页面在 `src/pages/`，依赖哪些 `stores` 与 `services`。
- 需要跨平台一致行为时，优先在工厂层（`createProviderAccountStore` / `createInstanceStore` / 平台 service 工厂）统一修改。

### 4.2 改后端命令时
- 命令入口在 `commands/*`，核心逻辑通常在 `modules/*`。
- 新命令必须加入 `src-tauri/src/lib.rs` 的 `invoke_handler`，否则前端无法调用。

### 4.3 改命令名或参数时
- 同步更新前端 `services` 与相关类型定义（`src/types/`）。
- 检查对应页面/store 的调用链，避免前后端签名不一致。

## 5) README 与规则文件提炼结果

### 5.1 README 关键点（已提炼）
- 项目定位：多平台 AI IDE 账号/实例管理工具。
- 开发条件：Node.js v18+、Rust。
- 开发与构建主命令：`npm run tauri dev`、`npm run tauri build`。

### 5.2 规则文件检查结果（已按仓库检索）
- 未发现 `AGENTS.md`。
- 未发现已有 `CODEBUDDY.md`（本文件为新建）。
- 未发现 `CLAUDE.md` / `copilot-instructions.md`。

## 6) 文档维护约定

- 本文档只记录**可在仓库中验证**的信息。
- 当脚本、目录结构、命令命名或开发流程变化时，应同步更新本文件。
- 对于 lint/test 等当前不存在的能力，先落地脚本再更新说明，避免文档先于代码“超前”。
