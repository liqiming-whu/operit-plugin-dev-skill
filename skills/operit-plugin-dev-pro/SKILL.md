---
name: operit-plugin-dev-pro
description: 在 Operit Android App 内开发、续写、调试、安装和排查 Sandbox Package 或 ToolPkg 的手机端增强工作流。与官方 SandboxPackage_DEV 配合使用，专门处理 /sdcard 开发目录、types 同步、operit_editor 调试安装、Compose DSL 生命周期、真机探针、部署未生效和 proot/worker 问题。用户在手机端开发 Operit 插件、使用 operit_editor、debug_run_sandbox_script、debug_install_toolpkg，或遇到 UI 不刷新、工具无响应、安装后仍是旧代码时使用。
---

# Operit 插件手机端开发增强

## 定位

与官方 `SandboxPackage_DEV` 同时使用：

- 官方技能负责最新格式、types、内置示例、API 和发布流程。
- 本技能负责 Android 端工作区准备、调试安装、真机验证和复杂故障定位。

不要复制官方技能已经维护的类型和格式指南。不得用本技能的历史案例覆盖当前官方 types 或源码。

## 开始前

1. 明确目标、插件格式、已有包位置和成功标准。续写已有包时保留原 packageId、名称和目录结构。
2. 按官方 `SandboxPackage_DEV` 第一部分重新运行安装/更新脚本。
3. 从官方技能检索当前 types、guide 和相近示例；不要凭旧记忆写 API。
4. 选择格式：普通工具优先使用 Sandbox Package；只有 UI、资源、子包、导航、widget 或宿主 hook 才使用 ToolPkg。
5. 使用 `/sdcard/Download/Operit/dev_package/{packageId}/` 作为项目目录；共享类型位于兄弟目录 `/sdcard/Download/Operit/dev_package/types/`。

## 准备工作区

调用 `operit_editor:debug_run_sandbox_script` 执行 `scripts/prepare_dev_workspace.js`：

```json
{
  "source_path": "/sdcard/Download/Operit/skills/operit-plugin-dev-pro/scripts/prepare_dev_workspace.js",
  "params_json": "{\"package_id\":\"com.example.plugin\"}"
}
```

脚本创建项目目录，并把官方技能的完整 `types/` 同步到共享类型目录。它不会清空或重建已有项目。

## 开发流程

1. 普通包：从官方 `SCRIPT_DEV_GUIDE.md` 检索 METADATA、目标工具和返回类型，优先写 TS，再编译为 JS。
2. ToolPkg：从官方 `TOOLPKG_FORMAT_GUIDE.md` 和 `types/toolpkg.d.ts` 核对 manifest、注册函数与 hook；从同型官方示例复制 tsconfig 结构。
3. Compose DSL：仅在涉及 UI 时读取 `references/COMPOSE_DSL_RULES.md`。把规则按“平台不变量、当前版本约束、项目策略”区分，不把案例参数当默认值。
4. 编译后执行 `scripts/inspect_package.js` 检查项目结构和 manifest 引用。
5. 普通脚本使用 `operit_editor` 的相应 JS 包调试入口；ToolPkg 使用 `debug_install_toolpkg`。安装后调用真实工具或打开真实 UI，不以 `debug_run_sandbox_script` 结果替代宿主验证。
6. 执行 `scripts/verify_deployment.js` 核对开发源和外部安装包。涉及 UI/注册缓存时重启 Operit，再验证实际界面和日志。

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
## 调试决策

出现问题时先读取 `references/DEBUG_PLAYBOOK.md`，按症状定位：

- UI 卡死或反复挂载：先查 render 副作用与状态写入。
- 异步完成但 UI 不刷新：先查 action/onLoad 生命周期和当前源码行为。
- 工具无响应或结果错位：先建立最小并发探针；在当前版本未证明并发安全前串行化相关调用。
- 安装成功但代码未变化：检查源文件、安装包、manifest 版本，再重启 Operit 排除内存旧实例。
- worker 启动失败：分开检查环境不存在、启动超时、readiness 未完成和进程已被宿主终止。

只有需要历史证据或复现 CMS/CME 架构时才读取 `references/CASE_STUDIES_CMS_CME.md`。其中的版本号、设备行为、TTL、路径和 worker 结论都属于案例范围，采用前必须在当前项目验证。

## 规则分级

结论的证据效力不按正式版或 Beta 发布通道区分。记录复现时的 Operit versionName/versionCode、已核实的构建来源或通道、设备权限、插件与调试工具版本、调用路径和复现方法；未知字段明确标记。单一环境实测属于版本相关证据，不能自动提升为平台不变量；环境变化后重新验证相关结论。

始终遵守：

- render/UI 树构造保持无副作用。
- API 与字段以刚更新的官方 types 和示例为准。
- 调试探针不改变业务控制流，并记录真实参数、结果和时间。
- 运行时问题必须在真实 Operit 工具/UI 路径验证。

需要当前版本验证：

- `ctx.callTool` 并发响应关联；未验证时默认串行相关调用，但不要永久禁止所有并发。
- 异步 `setState`、onLoad/action 窗口和重绘触发方式。
- 模块实例生命周期、env 持久化和 ToolPkg 缓存刷新。
- `ctx.callTool` 返回形态、安装结果字段、终端关闭与运行时回收的边界。

仅在需求成立时采用：

- debounce、TTL、乐观更新和分页的具体数值。
- Python/proot worker、常驻服务与文件完成信号。
- CMS/CME 的固定路径、模型启动时间和恢复策略。

## 验证与交付

- 官方技能已更新，项目使用当前完整 types。
- 已说明选择普通包或 ToolPkg 的原因。
- TS 编译通过，检查脚本未发现缺失引用。
- 已通过真实工具调用或真实 UI 验证宿主行为。
- ToolPkg 安装后核对源包、外部安装包、manifest 版本；UI/注册变化后重启 Operit。
- 未能验证的 QuickJS、bridge、权限、生命周期或设备行为必须单独列为风险。
- 发布继续遵循官方 `SandboxPackage_DEV` 的市场发布流程。

## 资源导航

- `references/MOBILE_WORKFLOW.md`：完整手机端命令与 `operit_editor` 调用顺序。
- `references/COMPOSE_DSL_RULES.md`：按证据等级整理的 UI/异步规则。
- `references/COMPLEX_UI_ARCHITECTURE.md`：复杂 UI 架构设计规范（决策表、官方薄壳+WebView 分层、硬边界）。核心规范，涉及复杂 UI 必读。
- `references/TERMINAL_CALL_RULES.md`：终端调用约束（两个入口、运行期零依赖、禁 hiddenExec、有界等待）。核心规范，一切终端调用必读。
- `references/DEBUG_PLAYBOOK.md`：按症状组织的排障流程。
- `references/CASE_STUDIES_CMS_CME.md`：CMS/CME 历史实验和版本战役，仅作案例证据。
- `scripts/prepare_dev_workspace.js`：创建工作区并同步官方 types。
- `scripts/inspect_package.js`：静态检查普通包或 ToolPkg 目录。
- `scripts/verify_deployment.js`：检查开发源与外部安装产物。
