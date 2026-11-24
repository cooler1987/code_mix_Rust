以下是该项目的**功能总结**与**技术实现总结**：

---

### 一、项目概况与文件结构

- **架构目标**：将原本基于 Flutter（体积大、多 DLL）的代码打包工具重构为 **Windows x64 单文件绿色便携版 EXE**，要求零外部运行库依赖（兼容 Win10 1909+）、极小体积、启动即用。
- **极简文件布局**：
  - `Cargo.toml`：依赖声明与极致体积压缩编译配置。
  - `main.rs`：唯一的全部主体源码文件（约 750 行），整合界面、逻辑、系统上下文配置于一体，无需 `src/` 目录及 `build.rs`、`app.manifest` 等辅助文件。
  - `.github/workflows/build.yml`：GitHub Actions 云端自动构建脚本。

---

### 二、核心功能总结

1. **三类代码源输入与原生拖放**
   - 支持 **本地文件夹**、**`.zip` 压缩包**、**Git 仓库链接** 三种输入方式。
   - 支持窗口全局拖放（`WM_DROPFILES`），自动识别拖入对象是文件夹还是 `.zip` 文件并切换对应输入模式。
   - Git 链接采用 `git clone --depth 1` 浅克隆并静默执行，ZIP 与 Git 临时目录在任务完成后自动清理。

2. **精准过滤与完整保留工作流配置**
   - 目录过滤基于路径组件精确匹配（`Path::components()`），仅忽略 `.git`、`node_modules`、`build`、`dist`、`target`、`.dart_tool` 等独立目录段，**彻底解决子串匹配误过滤 `.github/workflows` 的问题**。
   - 自动过滤图片、压缩包、可执行程序等二进制扩展名及 `.ds_store`、`thumbs.db` 等系统杂项文件，读取阶段自动跳过非 UTF-8 文件。

3. **多格式输出与内容控制**
   - 支持 **Markdown**、**XML**、**Plain** 三种 AI 友好输出格式；其中 Markdown 会根据文件内容动态计算最长连续反引号长度生成代码块围栏，防止内含 Markdown 文件破坏外层格式。
   - 支持独立勾选是否生成**文件摘要（File Summary）**和**目录结构（Directory Structure）**。

4. **自动命名规则**
   - 根据拖入的文件夹名、ZIP 文件名（去扩展名）或 Git 仓库名，过滤 Windows 非法字符后自动追加当天本地日期（`YYYY-MM-DD`）与对应格式后缀，例如 `项目名_2026-11-24.md`。

---

### 三、关键技术实现亮点

1. **纯 Win32 原生界面 + 单文件激活现代样式**
   - 基于 `windows-sys` 直接调用 Win32 API 构建界面，搭配 `rfd` 调起系统原生文件/文件夹选择弹窗。
   - 在 `main.rs` 入口直接调用 `SetProcessDPIAware()` 实现高分屏缩放，并通过 `CreateActCtxW` 加载系统 `shell32.dll` 内置清单资源（ID `124`），在**不附加外部 `.manifest` 文件**的前提下激活 Windows Common Controls v6 现代控件样式。

2. **多线程异步打包与消息通信**
   - 耗时打包任务放在 `std::thread::spawn` 后台线程执行，保持窗口响应不卡顿。
   - 线程结束后通过 `OnceLock<Mutex<...>>` 传递状态，并使用 `PostMessageW` 发送自定义窗口消息 `WM_APP_DONE` 通知 UI 线程恢复按钮并刷新状态栏。

3. **防重入状态管理与异常兜底**
   - 针对 Win32 控件禁用/文本更新会**同步触发** `WM_CTLCOLORSTATIC` / `WM_CTLCOLORBTN` 重绘回调的特性，使用 `Cell<bool>` 管理处理状态、全程仅使用不可变借用 `borrow()`，避免 `RefCell` 重入崩溃。
   - 启动时注册 `std::panic::set_hook`，异常发生时通过 `MessageBoxW` 弹出明确错误信息，杜绝静默闪退。

4. **体积优化与 Win10 1909 零依赖构建**
   - `zip` 仅启用纯 Rust 实现的 `deflate-flate2-zlib-rs` 特性，避免引入额外 C 工具链与冗余算法。
   - Release 编译配置启用 `opt-level = "z"`、`lto = true`、`codegen-units = 1`、`panic = "abort"` 与 `strip = true`。
   - 云端构建通过 `RUSTFLAGS="-C target-feature=+crt-static"` 静态链接 C 运行库，确保成品单文件在未安装 VC++ Redistributable 的 Windows 10 1909 系统上可直接双击运行。

---

### 四、后续可选演进方向（供参考）

- **完全摆脱本地 Git 依赖**：针对 GitHub 链接模式，未来可直接通过轻量 HTTP 请求下载仓库源码 ZIP 并交由内置 `zip` 模块解压，实现即便是未安装 Git 的电脑也能直接打包远程仓库。
- **进一步极限压缩体积**：若希望继续缩减几百 KB 体积，可将 `rfd` 替换为直接调用 Win32 `GetSaveFileNameW` / `GetOpenFileNameW` 等系统接口，进一步减少传递依赖。
