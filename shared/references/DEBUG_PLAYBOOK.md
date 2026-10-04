# Operit 插件排障手册

> 本文件是 PC 与 Android Pro 两个 Skill 的共同维护源。仓库内只编辑 `shared/references/DEBUG_PLAYBOOK.md`，再运行 `node scripts/sync-shared-references.mjs` 生成两个 Skill 的分发副本。

## 通用顺序

1. 精确定义现象、复现步骤、期望结果和实际结果。
2. 记录 Operit versionName/versionCode、构建来源或已核实的发布通道、插件与调试工具版本、packageId、开发环境、设备和权限模式；未核实的字段明确标记未知。
3. 从当前 types、官方示例和必要的 Operit 源码确认机制。
4. 建立最小复现；一轮只改变一个变量。
5. 在源代码出口、bridge 前后、UI 入口和最终展示点放置带时间与调用 ID 的探针。
6. 修复原因后删除或关闭会影响时序的探针，再完整复测。

## UI 卡死或重复挂载

- 检查 render/UI 构造过程中是否 setState、写文件、写 env 或调用工具。
- 检查 onLoad 是否因状态变化反复生成不同树。
- 统计单位时间 render/onLoad 次数，不凭体感判断。
- 把数据获取与 UI 生成分开，先验证最小静态页面。

## 异步完成但 UI 不刷新

- 证明后台操作确实完成，不把“结果已落盘”和“UI 已收到结果”混为一谈。
- 比较同步事件回调、onLoad 内 await、脱离事件后的 Promise 三种路径。
- 检查完成信号是否写在已结束的 callRuntime 上。
- 根据当前版本探针决定刷新机制，不直接套用旧版 onLoad 长窗口方案。

## 工具无响应或返回错位

- 给每次调用使用不同的可识别参数和结果标记。
- 先串行建立正确基线，再逐步增加并发。
- 同时记录调用开始、结束、参数摘要、结果摘要和异常。
- 若并发触发错配，保留最小复现和版本信息；业务代码对相关调用串行化。

## 安装后仍是旧代码

- 确认编辑的是实际开发目录，而非技能示例或解压临时目录。
- 检查 manifest version、main 路径和目标代码标记。
- 检查外部 packages 目录中对应 `.toolpkg` 的存在与时间。
- UI、注册和 hook 变化后重启 Operit，再复测真实入口。
- 有权限时才进一步检查应用私有缓存；不要把无法读取缓存误报为缓存不存在。

PC 端优先使用官方 `tools/toolpkg/debug_toolpkg.*` 的日志和外部安装包；Android 端优先使用 `operit_editor:debug_install_toolpkg` 的结构化结果。两条路径最后都必须回到真实 UI 或真实工具调用验证。

## worker 启动失败或拖慢 UI

- 区分可执行文件不存在、环境探测超时、进程启动失败和 readiness 未完成。
- readiness 使用有上限的轮询，并记录每次失败原因；不要只固定 sleep 一个经验值。
- UI 路径不应无限同步等待 worker。明确加载、取消和重试状态。
- 宿主退出可能终止其子进程，但具体行为需在当前设备、权限与启动方式验证。
- 简单插件优先移除 worker，而不是为其增加复杂保活机制。

## `debug_run_sandbox_script` 与真实环境不一致

该工具适合验证纯 JS、参数解析和部分 API 探针，不代表已安装包、真实工具调用或 Compose DSL 生命周期。出现差异时，以真实包入口的日志和结果为准。

PC 端同样不要用 Node mock 宣称已验证 Android QuickJS、`Tools.*`、Java/Kotlin bridge 或 Compose DSL。桌面 mock 只验证纯业务逻辑和适配层输入输出。

## 首次进入偶发报错（切走/重进即恢复）

这类问题多为平台首次渲染或 bundle 竞态，先做代码层四查，不要急着改插件逻辑：

1. **本地分支复现**：用 UI Proxy + mock ctx 覆盖关键状态分支（空数据 / 未识别 / 加载态 / 异常入参），确认插件代码静态无缺陷。
2. **安装包完整性**：核对打包目录与实际安装产物，确认不是旧代码或混入开发文件。
3. **组件集合一致性**：对比双端/相关页面引用的组件与注册 id 是否冲突（如 `SelectionContainer` 是否被某侧误引入）。
4. **启用状态确认**：报错时目标包是否确实启用（避免排查到未启用/停用的包）。
5. **关键定性信息**：记录「首次进入才出现、切换 tab 恢复、多次不复现」这类用户反馈——它决定问题归因（平台竞态而非稳定缺陷）。
6. **插件侧防御兜底**：对可疑调用点做类型防御（如主题模块异常时降级内置值），根治仍需平台日志定位；保留现场信息与复现条件。

## ToolPkg 烧录失败 / failed to parse toolpkg（2026-08-13 实锤）

- **现象**：导入/烧录时报 `failed to parse toolpkg` 或 `Failed to parse main registration from 'dist/main.js'`。
- **第一依据是日志，不猜**：`/sdcard/Download/Operit/packageLogs/` 最新日志搜 `parse failed` / `loadToolPkg` / `registerToolPkg` / `JS ERROR`——日志给出精确报错 + 堆栈 + 出错代码上下文（含行号）。
- **修复闭环**：改源码 → `node --check` → 重新 `zip` → 重新 `debug_install_toolpkg` → 重启 Operit，缺一不可。
- **打包前校验（防低级错误带进烧录）**：`node --check dist/main.js` + `node -e "JSON.parse(...)"` 校验 manifest；`.toolpkg` 本质是 zip，`dist/main.js` 与 `manifest.json` 必须位于包根（打包后 `unzip -l` 复核）。

### 常见报错对照表（源码级确认）

| 报错 | 根因 | 修复 |
|---|---|---|
| `registerAiProvider requires an object field: listModels` | handler 字段（listModels/sendMessage/testConnection/calculateInputTokens）必须是对象 `{ function: xxx }`，传了裸函数 | 改为 `listModels: { function: listModels }` 形式 |
| `registerAiProvider.listModels function must be exported from a toolpkg module` | handler 必须是**模块导出**函数（带 `__operit_toolpkg_module_path` 标记），模块内部普通函数不被识别 | 文件末尾加 `exports.listModels = listModels;` 等导出 |
| `requires an object field: xxx`（其他注册 API） | 注册参数格式不符，字段应为对象而非裸值 | 对照 `types/toolpkg.d.ts` 的 Registration 接口定义检查 |
| `Tool not found`（工具调用报错） | **新增工具漏加 METADATA `tools` 数组声明**（subpackage 解析器按 METADATA 注册，非扫描 exports） | 在 METADATA tools 补声明（含参数 schema）；验证 toolCount |
| manifest 报错 / 解析失败 | manifest JSON 非法（逗号/引号/编码） | `node -e "JSON.parse(...)"` 校验；中文内容别损坏编码 |

> 烧录成功但 UI 没刷新 / 行为还是旧的：见上文「安装后仍是旧代码」章节（UI 实例缓存，重启 Operit；区分"没部署"与"部署了但缓存旧"）。烧录成功但包没出现在列表：重启 → `list_sandbox_packages` → 检查 `enabled=false` 需启用。

### registerAiProvider 机制要点（防止复发）

- 注册走 `normalizeNestedFunctionField`（JsToolPkgRegistration.kt）：每个 handler 字段必须是**对象** `{ function: <fn> }`（非对象抛 `requires an object field`）；`function` 必须**模块导出**，经 `resolveDurableFunctionRef` 提取函数名+源码（否则抛 `must be exported from a toolpkg module`）。
- handler **返回值结构**必须对齐 `types/toolpkg.d.ts` 的 `AiProviderRegistration`：

| Handler | 必须返回 |
|---|---|
| `listModels` | `{ models: [{id, name}] }` |
| `sendMessage` | `{ text: string, usage?: {input?, cachedInput?, output?} }` |
| `testConnection` | `{ success: boolean, message?, error? }` |
| `calculateInputTokens` | `{ tokens: number }` |

- 返回结构不对**不会**导致 parse 失败，但上层解析不到数据（token 计 0、连接测试永远失败），排查时先对照 types。


## 当前版本约束：工具名解析、安装刷新与运行时环境

以下现场观察来自 2026-09-14 的 Operit `1.12.1+6`（versionCode 49，Beta 更新计划开启）。摘要未完整记录设备权限、调试工具版本和 Linux 挂载配置；它们是目标环境复验的起点，不是跨版本保证。源码对照固定在官方 `0d2bbdd`（构建声明也是 `1.12.1+6`），不据此认定现场安装器与该快照完全相同。

### 工具无响应：工具名解析三类坑

1. **名称不完整**：现场 `ctx.callTool("share_file", ...)` 未命中预期工具，改用 `extended_file_tools:share_file` 后可调用。包工具使用当前注册的 `包:工具` 名称；不要将这一观察推广到所有内置工具。
2. **前缀用错**：现场工具注册在子包下，实际名称是 `via_bridge:via_status`。从当前注册信息或 manifest 子包声明确认运行时包名，区分容器 id、子包 id 与实际注册名。
3. **解析结果未经核对**：先检查当前 types 和 `resolveToolName` 返回值。包工具结果应含有效的包名与工具名，含 `:` 仅是格式检查；解析能力缺失时使用已确认的注册名。候选验证使用只读探针，仅在明确的“工具不存在”错误后尝试下一候选；业务失败、超时或结果不明确时停止，避免重复副作用。

> 实测探针输出：`resolved=via_bridge:via_status`（全限定、子包 id 正确）。

### 「安装后仍是旧代码」的三种具体形态

| 形态 | 现场线索 | 处理 |
|---|---|---|
| **包文件丢失** | 移走 `.toolpkg` 后，读资源工具报空内容 `Step error:`，不读资源的工具正常 | 检查外部安装包、资源声明与加载日志；该错误文本不能单独确定根因 |
| **UI module 缓存** | 磁盘 UI 与预期摘要一致，页面行为仍旧；现场调整 module id 与 `params.rev` 后观察到免重启刷新 | 可在隔离开发探针中同步调整 id 及所有引用，验证缓存行为；正式部署仍按重启流程核验，不把递增 id 作为发版规则 |
| **工具名解析错误** | 界面按钮全失败、直调工具正常 | 见上 |

### `debug_install_toolpkg` 的安装线索

- **替换安装包时失败**：现场返回 `Unknown error`，日志出现 `Archive path differs from target; replacing target archive before copy.` 后失败，并观察到包文件丢失。官方快照对相同规范化路径会跳过复制；摘要不足以证明“相同路径必失败”。复现时记录源、目标完整路径、路径别名和安装器版本。使用独立暂存目录（如 `/sdcard/Download/Operit/tmp_xxx.toolpkg`），操作前备份现有包。[复制分支](https://github.com/AAswordman/Operit/blob/0d2bbdd3072cf304159ff34910f7aec4ec63d74e/examples/operit_editor.ts#L3158-L3164)。
- **`Duplicate package name`**：表示注册名称冲突。结合 `data.related_load_errors` 的源路径，检查普通包、ToolPkg 容器和子包的实际注册名，确认冲突双方后处理。不要仅凭错误文本删除文件；手工安装的可读性需按当前设备权限、目录属性与访问环境验证，不固定 UID/GID。[重复名称检查](https://github.com/AAswordman/Operit/blob/0d2bbdd3072cf304159ff34910f7aec4ec63d74e/app/src/main/java/com/ai/assistance/operit/core/tools/packTool/PackageManager.kt#L1184-L1197)。

### `api_version` 门禁与日志定位

- manifest 的 `api_version` 必须被目标 Operit 构建支持。所测环境的错误日志列出 `1.0.0`、`1.0.1`，其中 `1.0.1` 要求 Operit `1.12.1+4` 起；支持集合以目标构建源码和加载结果为准。[兼容规则](https://github.com/AAswordman/Operit/blob/0d2bbdd3072cf304159ff34910f7aec4ec63d74e/app/src/main/java/com/ai/assistance/operit/core/tools/packTool/ToolPkgApiVersion.kt#L74-L112)。
- 遇到 `ToolPkg container did not appear after debug install: <id>`，先检查完整安装结果的 `data.related_load_errors` 与 `refresh_result`；工具可携带结构化原因，再结合本次安装的应用日志定位。
- `packageLogs/` 同时包含应用错误、终端命令与对话文本。先选本次安装时间段的日志，再检索错误标记，并结合目标包 id、源路径和上下文核对来源；关键词匹配本身不能排除命令或对话文本：

  ```bash
  grep -rnE 'E/ToolPkg|loadToolPkg|IllegalArgumentException|parse failed' /sdcard/Download/Operit/packageLogs/
  ```

- 实测示例（manifest 写 `api_version: "9.9.9"`）：

  ```
  E/ToolPkg PKG: loadToolPkgFromExternalFile failed, source=.../com.test.apiversion.toolpkg
  java.lang.IllegalArgumentException: ToolPkg API version '9.9.9' is not supported by Operit 1.12.1+6.
    Supported ToolPkg API versions: 1.0.0, 1.0.1. ToolPkg API 1.0.1 requires Operit 1.12.1+4 or newer.
    at com.ai.assistance.operit.core.tools.packTool.ToolPkgApiCompatibility.requireSupported(ToolPkgApiVersion.kt:108)
    at com.ai.assistance.operit.core.tools.packTool.ToolPkgLoader.loadToolPkgFromExternalFile(ToolPkgLoader.kt:29)
  ```

  现场对照：保持包结构相同，仅将 `api_version` 改为 `1.0.0` 后安装成功，`related_load_errors` 为空。该对照支持本次版本拒绝的归因，不证明包功能或其他构建的兼容性。

### Android 与 Linux 文件访问的现场对照

下表记录所测文件的结果。Linux 挂载、Android 权限模式与具体工具会改变可见范围，不能推广为整个目录的访问保证。

| 路径 | `environment:"android"` | `environment:"linux"` |
|---|---|---|
| App 内部 `/data/user/0/<pkg>/files/...` | ✅ | ✅ |
| `/sdcard/...`（**未**授予「所有文件访问权限」时） | ❌ | ✅ |
| `/sdcard/...`（已授予「所有文件访问权限」后） | ✅ | ✅ |

- 现场 `Tools.Files.share` 在 linux 环境返回 `File sharing is not supported in Linux environment`；目标环境按当前接口和实际调用验证。
- Android 应用侧文件可优先放到 `Java.getApplicationContext().getFilesDir()` 返回的目录，并通过实际消费文件的工具验证可读性；Linux 侧需另行确认挂载与权限。
- 诊断时同时记录路径、调用工具、执行环境、权限与挂载信息，用同一只读操作做对照。
