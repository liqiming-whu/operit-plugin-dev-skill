# 手机端开发工作流

## 1. 更新官方技能

每个正式开发任务开始前，按官方 `SandboxPackage_DEV/SKILL.md` 重新下载并运行 `install_or_update.js`。更新完成后确认至少存在：

```text
/sdcard/Download/Operit/skills/SandboxPackage_DEV/
├── SKILL.md
├── examples/packages/
├── references/
└── types/
```

## 2. 准备工作区

通过 `operit_editor:debug_run_sandbox_script` 运行：

```json
{
  "source_path": "/sdcard/Download/Operit/skills/operit-plugin-dev-pro/scripts/prepare_dev_workspace.js",
  "params_json": "{\"package_id\":\"com.example.plugin\"}"
}
```

结果应为：

```text
/sdcard/Download/Operit/dev_package/
├── types/
└── com.example.plugin/
```

脚本只覆盖官方共享类型文件，不删除项目文件。基于已有插件继续开发时，先把当前包内容放入项目目录。

## 3. 编写与编译

- 普通包优先写 `.ts`，从官方示例选择相近结构并编译为 `.js`。
- ToolPkg 的 `typeRoots` 与 `include` 指向 `../types`。
- 模块使用 `import`/`export`，不要以 `/// <reference>` 或 `require()` 组织 TypeScript 项目。
- 接口、返回值和 Compose DSL 属性必须从刚同步的 types 核对。

## 4. 静态检查

运行 `inspect_package.js`：

```json
{
  "source_path": "/sdcard/Download/Operit/skills/operit-plugin-dev-pro/scripts/inspect_package.js",
  "params_json": "{\"target_path\":\"/sdcard/Download/Operit/dev_package/com.example.plugin\"}"
}
```

普通 `.js` 文件检查 METADATA 与导出；ToolPkg 目录检查 manifest、main、subpackage、resource 和 wasm 引用。HJSON manifest 只做存在性提示，复杂字段仍需按官方指南人工核对。

## 5. 安装和真实验证

- 普通包使用 `operit_editor` 当前提供的 JS 包安装/调试工具。
- ToolPkg 使用 `debug_install_toolpkg`，`source_path` 可指向目录、manifest 或 `.toolpkg`。
- 安装成功不等于功能成功。必须调用实际工具或打开实际 UI，并检查返回值、界面和日志。
- `debug_run_sandbox_script` 的运行环境与真实包执行路径不同，尤其不能用它证明 `Tools.Files`、bridge 或 Compose DSL 正常。

## 6. 部署核验

运行 `verify_deployment.js` 检查开发源与外部安装目录：

```json
{
  "source_path": "/sdcard/Download/Operit/skills/operit-plugin-dev-pro/scripts/verify_deployment.js",
  "params_json": "{\"package_id\":\"com.example.plugin\",\"source_path\":\"/sdcard/Download/Operit/dev_package/com.example.plugin\"}"
}
```

若修改了 UI、main 注册或 manifest：

1. 核对开发目录内的新版本和目标代码标记。
2. 核对 `/sdcard/Android/data/com.ai.assistance.operit/files/packages/` 中安装产物存在。
3. 重启 Operit，避免仍在运行的实例继续使用内存旧代码。
4. 再打开真实 UI 或调用工具。

应用私有 `toolpkg_cache` 受权限和实现版本影响，不把它作为通用脚本的强制检查项；需要深入排障时结合当前源码、root/Shizuku 权限和日志检查。


## 7. 当前版本约束：安装结果与环境证据

以下观察来自 2026-09-14，Operit `1.12.1+6`（versionCode 49，Beta 更新计划开启）；设备权限和调试工具版本未随摘要完整记录，目标环境需复验。

- **第 5 步**：使用独立暂存包作为 `debug_install_toolpkg.source_path`，并在替换前备份现有安装包。现场替换失败不等于相同路径必失败，完整路径与安装器版本的排查见 `DEBUG_PLAYBOOK.md`。
- **第 6 步部署核验**：检查完整安装结果的 `data.related_load_errors`，该字段应是非 null、非数组的对象映射，且 `Object.keys(errors).length === 0`。字段缺失、类型错误或未取得刷新结果时标记未验证，不能视为空；此字段由安装工具返回，`verify_deployment.js` 仅做文件存在性核验并列出人工检查项。
- 空错误映射只是必要检查，还需核对实际加载的目标包、manifest 版本与代码标记，并完成第 5、6 节的真实 UI / 工具验证。隔离开发探针中的免重启缓存观察不替代正式部署的重启核验。
- **平台版本与构建记录**：
  - 用 `PackageManager.getPackageInfo(pkg, 0)` 记录 versionName、versionCode；现场分别为 `1.12.1+6`、49。
  - `user_preferences.preferences_pb` 的 `beta_plan_enabled=true` 记录 Beta 更新计划开启；它是可修改的更新偏好，不能单独确定已安装产物的发布通道。构建来源或发布通道需另行核实，无法核实时标记未知。
  - `ApplicationInfo.flags & FLAG_DEBUGGABLE` 只反映可调试标志；现场为 false，不能据此判断发布通道或签名身份。
- **`api_version` 门禁**：声明必须被目标应用支持；现场日志列出 `1.0.0`、`1.0.1`。支持集合及失败原因的核对方式见 `DEBUG_PLAYBOOK.md`。
- **`ctx.callTool` 返回形态**：现场所测工具返回 JSON 文本；目标工具需探针确认，并兼容契约允许的对象与文本结果，见 `COMPOSE_DSL_RULES.md`。
