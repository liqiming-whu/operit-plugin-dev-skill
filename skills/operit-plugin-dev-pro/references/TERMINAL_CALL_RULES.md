# 终端调用约束与证据等级
> 本文件是 PC 与 Android Pro 两个 Skill 的共同维护源。仓库内只编辑 `shared/references/TERMINAL_CALL_RULES.md`，再运行 `node scripts/sync-shared-references.mjs` 生成两个 Skill 的分发副本。
> 依据：官方 examples 统一使用 `Tools.System.terminal` 会话 API；CME 多轮实机教训（2026-08-08 hiddenExec 坏会话 / 32s 探测阻塞 / proot 整树 kill；2026-08-16 Hermes/Termux 服务孤儿化）。

## 平台不变量：只有两个入口
| API | 用途 | 典型场景 |
|---|---|---|
| `Tools.System.terminal.create(name)` / `exec(sessionId, cmd)` / `input` / `screen` / `close` | 持久终端会话，带状态、可交互 | 启动服务、安装依赖、SSH、tmux 长任务 |
| `Tools.System.shell(cmd)` | 一次性执行，无会话 | 单条短命令（如 `screencap`） |

官方插件（github / linux_ssh / file_converter / sidebar_* / super_admin）全部走这两个入口，无例外。禁止直接 fork/exec 脱离平台管理的进程。

## 平台不变量：运行期零依赖（最重要的一条）
1. **被启动的服务进程必须脱离 terminal 生命周期**：`nohup setsid ... &` 启动、输出重定向到日志文件、PID 写入文件；健康判断一律以 HTTP / 进程探测为准，**不得把 terminal 会话存活当作服务健康前提**。
2. 原因（实锤）：Operit 退出/划掉 → proot 连带被杀 → 挂在 terminal session 下的进程整树死亡；不存在"进程跨 App 重启存活"的状态。
3. terminal 只做两件事：**投递启动命令（毫秒级返回）+ 健康确认（有界轮询）**。重活（装依赖、等健康）放后台或 JS 侧轮询，绝不在终端里同步等待。

## 平台不变量：禁止 hiddenExec（生产代码）
1. `hiddenExec` 已被证实可制造跨重启残留的 proot+bash 坏会话（CME P0-C3 删除生产链，2026-08-10）。
2. 官方 linux_ssh / code_runner 仍在用 hiddenExec，**不因对齐官方而松动**：一律用 visible terminal + exec / execStreaming。
3. 探测类命令不做同步等待：存在性校验下沉到启动脚本（bash `[ -x ]`，毫秒级），不在 JS 侧 8s×N 阻塞（ANR 根因）。

## 当前版本约束：会话生命周期（先验证再固化）
1. **命名复用，不做投递即弃**：控制类命令固定复用同一命名会话（如 `cme_worker_ctrl`），不要每次 create 一次性投递后遗弃——残留会话会累积，且 Operit 重启早期 create 有 executor 竞态风险（CME main.js 实锤：重启数秒内创建可能产生坏会话，后续调用永久卡）。
2. **补 close 语义**：任务完成后 close 会话（或空闲回收），避免 Android 侧会话泄漏；资源生命周期问题要在设计里闭环，不能靠重启 App 自愈。
3. **启动期延迟**：Operit 重启早期（数秒内）不创建/不复用 terminal session、不部署、不 kill——`onAppCreate` 保守延迟（10s）后再触发拉起。

## 当前版本约束：超时与等待
1. **一切等待有界**：health 轮询（如 1.5s × 45s）、命令 exec 超时（如 15s）、日志读取超时（如 4s），全部显式设界；超时返回结构化失败（含 logTail），不无限重试。
2. **安装类命令优先前台 execStreaming**：pip/npm 安装用流式执行等待自然结束，输出直接回流；不要 nohup + 轮询日志尾部（有历史残留误判问题）。
3. **启动类命令轻提交 + 后台执行**：`nohup setsid bash start.sh & echo submitted`（毫秒级返回），配合单飞锁（原子 mkdir / 文件锁）+ 有界 health 轮询。

## 项目策略：并发与竞态防护
1. **single-flight 硬裁决**：并发触发启动时只允许一个真实执行（原子互斥），其余直接退出——启动脚本内做，不依赖调用方自觉。
2. **失败自驱重试**：轮询探测失败后自行排队下一次（有界次数），不依赖 render/调用方重试。
3. **快速失败**：依赖缺失（如 NO_VENV）要能提前识别并立即返回 NEED_INSTALL，不空等超时。

## 项目策略：审计与回滚边界
1. 不回退 hiddenExec、不恢复隐式业务拉起、不用宽泛 kill（按 PID 精确管理，PID 文件为准）。
2. 任何终端调用改动都要进版本库且可回滚；服务进程管理统一收敛到独立模块（如 `worker_runtime`），对外只暴露语义接口（`ensureWorker/installDeps/restartWorker`），内部 transport 用 `Tools.System.terminal`。
3. 与 `COMPLEX_UI_ARCHITECTURE.md` 配合：Web 服务/Worker 的启动投递、健康确认、资源同步均按本文件执行。


## 当前版本约束：`setsid` 与运行时回收

- **版本相关证据**：2026-09-14，Operit `1.12.1+6`（versionCode 49，Beta 更新计划开启）中，用 `nohup setsid ... &` 启动的所测服务 `PID=SID=20005`，调用方 shell 的 `SID=29469`。设备权限与运行时配置未随摘要完整记录，采用前需复验。
- SID 不同证明新会话已建立，不证明所有关闭路径都不发送信号。`setsid` 建立新会话，`nohup` 设置忽略 `SIGHUP`；宿主主动按 PID、进程树或运行时范围清理仍可能终止服务。健康检查还需覆盖实际 terminal close 与运行时重启。[setsid](https://man7.org/linux/man-pages/man2/setsid.2.html)、[nohup](https://man7.org/linux/man-pages/man1/nohup.1.html)。
- 可用 `ps` 的 PID/SID 输出核对会话；读取 `/proc/<pid>/stat` 第 6 字段时需正确解析带括号的 comm，不能直接按空白切分。
- 现场报告运行时重启或回收会终止服务；独立会话不能保证跨 App 或 Linux 运行时重启存活。关闭路径与回收范围以目标环境测试为准。

## 项目策略：显式恢复服务

- 现场项目在服务被终止后，通过带自愈行为的状态工具重新拉起服务。这属于该项目恢复策略，不是只读状态查询的默认契约。
- 状态查询只报告健康状态；需要恢复时调用显式 `ensureWorker` / `restartWorker` 入口。启动由 single-flight 和有界健康轮询管理，遵守上文“不恢复隐式业务拉起”的边界。
- 恢复需等运行时就绪；启动期延迟与重试参数按目标环境验证，不能保证 App 被终止期间仍可恢复。健康以 HTTP / 进程探测为准。
