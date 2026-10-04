---
name: operit-plugin-dev-skill
description: 在 Windows、macOS 或 Linux 桌面端开发、续写、迁移、编译、校验、打包和真机调试 Operit Sandbox Package 与 ToolPkg。使用本地 Operit 官方源码、类型和示例作为权威依据，以 shell、Node.js、TypeScript 与可选 ADB 替代 operit_editor、use_package、grep_code、read_file_part 等 Operit 安卓端专属工具。用户提到 Operit 插件、Sandbox Package、ToolPkg、Compose DSL、插件侧边栏、宿主 hook、桌面小组件或 .toolpkg 时使用。
---

# Operit 插件桌面开发

## 目标与边界

在 PC 上完成与官方 `SandboxPackage_DEV` 相同的开发工作：查官方资料、写普通 Sandbox Package 或 ToolPkg、用官方类型编译、校验并打包。不要调用 Operit 内的 `operit_editor`、`use_package`、`debug_run_sandbox_script`、`grep_code`、`read_file_part` 或 `download_file`。

桌面静态测试不能模拟 Android QuickJS、Java/Kotlin bridge、`Tools.*` 或 Compose DSL 宿主行为。需要验证这些行为时，使用官方 ADB 工具在装有 Operit 的设备上运行；没有设备时明确标记为“仅通过桌面静态验证”。

## 开始前

1. 明确需求、插件类型、已有包位置和成功标准。续写已有包时保留原 `packageId`、插件名和文件结构。
2. 定位 Operit 官方仓库。优先使用用户给出的路径；未给出时只检查当前工作区和常见相邻目录。仍找不到再询问或 clone `https://github.com/AAswordman/Operit.git`。
3. 运行本技能的同步脚本：

   ```powershell
   node <skill目录>\scripts\sync-official.mjs --operit-root D:\Operit
   ```

   macOS/Linux 使用相同 Node 命令和对应路径。脚本把当前官方 guides、`examples/types` 和示例索引同步到本技能，不依赖安卓工具。检查其汇总输出全部成功后再开发。
4. 按需检索，不整读大型文档：

   ```powershell
   rg -n "关键词" <skill目录>\references <skill目录>\types D:\Operit\examples
   ```

5. 需要完整桌面命令时读取 `references/DESKTOP_WORKFLOW.md`；涉及 UI 时读取 `references/COMPOSE_DSL_RULES.md`；出现故障时读取 `references/DEBUG_PLAYBOOK.md`。只有需要历史实验依据时才读取 `references/CASE_STUDIES_CMS_CME.md`。

权威顺序：当前本地 Operit 源码与 `examples/types` > 同步后的官方 guides/types > 本技能经验文档。发生冲突时按更高层修正实现，不凭记忆补接口。

## 规则分级

结论的证据效力不按正式版或 Beta 发布通道区分。记录复现时的 Operit versionName/versionCode、已核实的构建来源或通道、设备权限、插件与调试工具版本、调用路径和复现方法；未知字段明确标记。单一环境实测属于版本相关证据，不能自动提升为平台不变量；环境变化后重新验证相关结论。

平台不变量以当前接口契约为依据；返回形态、并发关联、缓存刷新和终端生命周期需当前环境验证；worker、自愈与缓存参数按项目需求采用。经验文档中的日期和版本限定应保留到实际使用结论的位置。

## 选择插件格式

- 默认使用普通 `.ts`/`.js` Sandbox Package：新增工具、参数、返回值、环境变量或普通资源访问。
- 仅在需要配置/工具箱 UI、导航入口、桌面小组件、资源目录、子包或 lifecycle/message/prompt 等宿主 hook 时使用 ToolPkg。
- 少量配置优先用 `env` 或工具参数，不为未来扩展提前升级格式。
- 若官方类型没有所需能力，停止实现该部分并说明缺口；不要虚构 API。

## UI架构与终端调用铁律
> 这两部分是核心规范，不是可选建议：复杂 UI 架构决策与一切终端调用都必须遵守。详细证据与分级见 `references/COMPLEX_UI_ARCHITECTURE.md` 与 `references/TERMINAL_CALL_RULES.md`。

### 复杂 UI：先决策，再动手
- 多 Tab / 长列表 / 密集交互 / 表单 / 需要 Web 生态的界面，**必须先读 `references/COMPLEX_UI_ARCHITECTURE.md` 再设计**；简单界面（1-2 屏、列表 <100 条）保持纯 Compose（遵守 `COMPOSE_DSL_RULES.md`），不要默认上复杂架构。
- 官方范本：薄 Compose 壳 + UI.WebView + web_runtime + 本地 Web 服务 + 前端。壳内只做注册入口、启动状态机、WebView 渲染；业务交互在浏览器。
- 硬边界（违反即返工）：部署/恢复入口必须留在 Compose 壳（防自举死锁）；Web 服务运行期必须脱离 terminal 生命周期（`nohup setsid`，健康以 HTTP 为准）；不回退 hiddenExec；Web 与工具面共享同一数据权威。
- 该架构能避免：120s action 窗口卡死、render 期 setState 无限重建、bridge 并发响应错配、异步更新 UI 不刷新、首帧无持久状态、复杂列表性能上限、自举死锁。

### 终端调用：只走两个入口
- 一切执行命令 / 启动进程 / 探测服务只走 `Tools.System.terminal`（会话：create/exec/input/screen/close）与 `Tools.System.shell`（一次性），遵守 `references/TERMINAL_CALL_RULES.md`。
- 会话命名复用并补 close（不投递即弃）；被启动的服务进程运行期必须脱离 terminal 生命周期；生产代码禁用 hiddenExec；一切等待有界；启动期延迟；single-flight 防并发。
- 违反任一条都会在实机上以 ANR / 坏会话 / 进程被杀的形式暴露（CME 2026-08-08 / 2026-08-10 实锤）。
## 普通 Sandbox Package

1. 从 `references/SCRIPT_DEV_GUIDE.md` 检索 `METADATA`、`main`、`exports`、目标 `Tools` API，并在 `examples/index.json` 中找相近官方示例。
2. 用 `types/index.d.ts`、对应能力类型和 `types/results.d.ts` 约束代码。所有 `Tools.*` 异步调用都要 `await`。
3. 优先编写 TypeScript，再编译为最终 JavaScript。模块按官方示例使用 `import`/`export`；不要用 `/// <reference>` 或 CommonJS `require()` 组织 TS 源码。
4. 保持 METADATA 中工具名、参数和导出函数一致。若需求要求入口函数，提供并测试 `main`。
5. 运行静态校验：

   ```powershell
   node <skill目录>\scripts\validate-package.mjs <package.js>
   ```

6. 涉及宿主返回值、Android 文件路径或 Java bridge 时，再执行“真机验证”。

## ToolPkg

### 项目布局

让共享类型成为各插件项目的兄弟目录，不复制进包：

```text
workspace/
├── types/                 # 从本技能 types/ 同步
└── com.example.plugin/
    ├── manifest.json
    ├── src/main.ts
    ├── src/ui/<id>/index.ui.ts
    ├── dist/
    ├── packages/          # 可选
    ├── resources/         # 可选
    ├── i18n/              # 可选
    └── tsconfig.json
```

从 `<skill目录>/types/` 整体复制 `types`，不要零散修补。`tsconfig.json` 以 `D:\Operit\examples` 中同型官方项目为模板，通常使用 `target: es2020`、`module: commonjs`、`moduleResolution: node`、`outDir: ./dist`，并让 `typeRoots`/`include` 指向 `../types`。

### 实现规则

1. 先在 `types/toolpkg.d.ts` 核对注册函数和字段，再写 `registerToolPkg()`。不要从旧文档猜名字。
2. UI 使用默认导出的 `Screen(ctx)` 与 `runtime: "compose_dsl"`。组件与属性以 `types/compose-dsl*.d.ts` 为准。
3. 颜色使用 `ctx.MaterialTheme.colorScheme`；不要硬编码主题色。图标以 `types/material-icons.d.ts` 为准。
4. 导航功能按当前官方示例核对 `registerToolboxUiModule`、`registerUiRoute` 和 `registerNavigationEntry`；桌面小组件参考 `desktop_widget_demo`；消息/Prompt hook 参考 `message_insert`。
5. render 保持无副作用；异步工作放事件处理器或生命周期入口。依赖顺序、共享状态或尚未验证响应关联时串行 bridge 调用；独立请求需要并发时，先在当前 Operit 版本用可识别结果建立最小探针。
6. 编译并校验：

   ```powershell
   npx tsc -p <插件目录>\tsconfig.json
   node <skill目录>\scripts\validate-package.mjs <插件目录>
   ```

### 打包

使用随技能提供的跨平台打包器，避免 PowerShell 对非 `.zip` 扩展名的差异：

```powershell
node <skill目录>\scripts\pack-toolpkg.mjs <插件目录> --output <输出文件>.toolpkg
node <skill目录>\scripts\validate-package.mjs <输出文件>.toolpkg
```

打包器保留插件目录内容，但排除 `.git`、`node_modules`、TypeScript 源码、源码映射、日志和常见临时/编辑器文件。打包前确保运行所需资源都位于插件目录，且 manifest 中的 `main`、subpackage、resource 和 wasm 路径存在。

## 真机验证

PC 上不使用 `operit_editor`。设备已通过 ADB 授权时，使用 Operit 仓库自带工具：

```powershell
# 普通脚本：按目标入口选择 execute_js / execute_js_dir / run_sandbox_script
D:\Operit\tools\adb\execute_js.bat <script.js> <exportName> <args-json>

# ToolPkg：打包、推送、广播安装并查看相关 logcat
D:\Operit\tools\toolpkg\debug_toolpkg.bat <插件目录或.toolpkg>
```

macOS/Linux 使用对应 `.sh`；也可直接运行 `tools/toolpkg/debug_toolpkg.py`，但遵守当前项目的 Python 环境规范。记录设备、Operit 版本、调用参数和日志。不得把 `debug_run_sandbox_script` 的结果当成真实工具环境的替代证据。

## 交付检查

- 已说明选择普通包或 ToolPkg 的理由。
- 已从当前官方源码同步并检索接口，未虚构 API。
- TypeScript（如有）编译通过；校验脚本通过。
- `.toolpkg` 根目录直接包含 manifest，manifest 引用的文件均存在。
- 未把 `node_modules`、`.git`、日志、临时文件或无意保留的 TS 源码打入包。
- 涉及宿主行为时已真机验证；否则明确剩余风险和建议的 ADB 命令。
- 发布前核对 manifest 版本与产物版本。若使用 GitHub Release，先确认公开范围与许可证，不擅自公开代码。

## 资源导航

- `references/PC_TOOL_EQUIVALENTS.md`：安卓专属工具到桌面命令的映射与能力边界。
- `references/DESKTOP_WORKFLOW.md`：从官方资料同步到 ADB 真机验证的完整桌面流程。
- `references/SCRIPT_DEV_GUIDE.md`：同步自官方仓库的普通包指南。
- `references/TOOLPKG_FORMAT_GUIDE.md`：同步自官方仓库的 ToolPkg 指南。
- `references/COMPOSE_DSL_RULES.md`：按证据等级整理的公共 UI/异步规则。
- `references/COMPLEX_UI_ARCHITECTURE.md`：复杂 UI 架构设计规范（决策表、官方薄壳+WebView 分层、硬边界）。核心规范，涉及复杂 UI 必读。
- `references/TERMINAL_CALL_RULES.md`：终端调用约束（两个入口、运行期零依赖、禁 hiddenExec、有界等待）。核心规范，一切终端调用必读。
- `references/DEBUG_PLAYBOOK.md`：PC 与 Android 共用的按症状排障流程。
- `references/CASE_STUDIES_CMS_CME.md`：有版本范围的 CMS/CME 历史案例。
- `types/*.d.ts`：同步自 `Operit/examples/types` 的完整类型快照。
- `examples/index.json`：由同步脚本生成的官方普通脚本与 ToolPkg 示例索引；源码仍从 Operit 仓库读取。
- `scripts/sync-official.mjs`：同步官方资料。
- `scripts/validate-package.mjs`：校验普通包、ToolPkg 目录或 `.toolpkg`。
- `scripts/pack-toolpkg.mjs`：跨平台打包 ToolPkg。
