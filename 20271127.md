### 一、项目概况与文件结构

* **架构目标**：面向 **Windows 10 1909+ x64** 的单文件绿色便携版代码打包与还原工具，零外部运行库依赖、纯绿色无残留、启动即用。
* **极简根目录布局**（无业务子目录，代码文件共 3 个）：
  * `Cargo.toml`：声明 `rfd`、`zip`、`windows-sys`、`winresource` 依赖及极致体积优化（`opt-level = "z"`、`lto = true`、`codegen-units = 1`、`panic = "abort"`、`strip = true`）。
  * `build.rs`：编译期纯代码生成 16×16 / 32×32 / 48×48 多尺寸 `.ico` 图标并嵌入 EXE，同时注入 Windows Manifest（Common-Controls v6 现代样式与 `dpiAware`）及软件版本描述。
  * `main.rs`：整合原生 Win32 界面、正向流式打包引擎、反向 ZIP 还原引擎、可控日志与异常兜底机制。
  * `.github/workflows/build.yml`：GitHub Actions 云端自动编译，启用 `RUSTFLAGS="-C target-feature=+crt-static"` 静态链接 C 运行库，输出单文件 `repomix-portable.exe`。

---

### 二、核心功能总结

1. **双向工作流（正向打包 + 反向还原 ZIP）**
   * **正向打包（Pack）**：支持 **Git 链接**、**本地文件夹**、**ZIP 压缩包** 三种输入源，可导出 **Markdown**、**XML**、**Plain** 三种 AI 友好格式文档，支持独立勾选目录树与统计摘要。
   * **反向还原（文档还原转 ZIP）**：支持读取本工具生成的 `.md` / `.xml` / `.txt` 代码说明文件，自动嗅探文档格式并解析全部代码块与相对路径，还原输出标准单层 Deflate 压缩包（`.zip`），对路径穿越（`..`、盘符等）做安全清洗，并在未解析到有效文件时拦截报错。

2. **智能识别与原生拖放**
   * 支持窗口全局拖放（`WM_DROPFILES`）：自动识别拖入对象是文件夹、`.zip` 压缩包还是 `.md` / `.xml` / `.txt` 说明文档，自动切换至对应工作模式并刷新界面状态。
   * 输入框监听 `EN_CHANGE`：粘贴或输入 `https://`、`http://`、`git@`、`ssh://` 链接时自动切到 Git 模式。
   * 选中“文档还原 (转ZIP)”时，自动禁用正向输出格式与内容选项，执行按钮切换为“还原为 ZIP”。

3. **Git 分支解析与高效精准过滤**
   * 自动解析 GitHub `/tree/<branch>` 与 GitLab `/-/tree/<branch>` 网页分支链接，转换为 `git clone --depth 1 --single-branch [--branch <branch>]` 静默执行（`CREATE_NO_WINDOW`），完成后自动清理临时目录。
   * 目录遍历阶段直接跳过忽略目录子树（`.git`、`node_modules`、`build`、`dist`、`target`、`.dart_tool`、`__pycache__`、`.venv`、`venv`、`.idea`、`.vscode`、`.vs`、`.next`、`.nuxt`、`coverage`、`out`、`.gradle`），过滤二进制扩展名、系统杂项文件与 `> 2MB` 超大单文件，并结合前 8KB `\0` 空字节检测与 UTF-8 校验精准剔除无后缀二进制文件。

4. **输出格式规范与汇总反馈**
   * **目录树（Directory Structure）**：采用缩进 ASCII 树形结构（`├──`、`└──`、`│   `），层级清晰且节省上下文空间。
   * **统计摘要（File Summary）**：展示输出格式、已打包文件数、各类跳过统计、总行数、总字符数与估算 Token 数（约字符数 / 4）。
   * **格式安全**：Markdown 动态计算最长反引号围栏长度；XML 对 `path` 属性做实体转义，正文使用 `<![CDATA[...]]>` 包裹并拆分内部 `]]>` 为 `]]]]><![CDATA[>`，彻底避免源码破坏 XML 结构。
   * **自动命名与状态反馈**：正向输出默认为 `<来源名>_YYYY-MM-DD.<ext>`，反向输出默认为 `<文档名>_restored.zip`；完成后状态栏直接展示文件数、Token 估算与产物大小。

5. **默认关闭的纯绿色运行日志**
   * 日志功能**默认关闭**，关闭状态下不产生任何磁盘文件；用户可通过界面复选框随时启停，或通过命令行参数 `--log` 启动。
   * 开启后在程序同级目录生成 `repomix-portable.log`（超过 1MB 自动清空轮转），记录程序版本、DPI、路径操作、Git 命令参数及 stderr、正/反向打包耗时及异常明细。

---

### 三、关键技术实现亮点

1. **低内存流式读写架构**
   * 正向打包先将格式化正文流式写入临时缓存文件并累计精确统计，随后与树形目录、统计摘要流式合并写入目标文件，避免大仓库内存峰值翻倍。
   * 反向还原基于 `BufReader::read_line` 逐行状态机解析说明文档，并直接流式写入 `ZipWriter`，兼顾大文档解析效率与内存占用。

2. **Win32 原生体验与防重入状态管理**
   * 延续并强化 `Cell<bool>` 状态管理，消息处理全程仅使用 `RefCell::borrow()` 只读借用，从根源上避免 Win32 控件启用/赋值同步触发 `WM_CTLCOLOR*` 或 `EN_CHANGE` 导致 `RefCell` 重入崩溃。
   * 消息循环接入 `IsDialogMessageW`，完整支持 `Tab` 切换焦点、方向键切换单选框与 `Enter` 触发执行；使用 `AdjustWindowRectEx` 按客户区尺寸精确反算窗口外框，防止不同主题或高 DPI 下控件裁切；全局复用单一 `SHARED_BG` 画刷句柄减少 GDI 资源开销。

3. **前置校验与异常兜底**
   * 弹出保存路径对话框前优先完成输入校验（本地路径存在性、文档非空、URL 合法性及 `git --version` 可用性检查），不通过立即提示。
   * 后台线程执行期间统一锁定全部输入与选项控件；任务出错或触发 `panic_hook` 时均通过 `MessageBoxW` 明确提示错误详情（开启日志时同步落盘）。

4. **双保险现代样式与零外部依赖**
   * 通过 `build.rs` 编译期嵌入高清图标与 Manifest，同时保留运行期 `SetProcessDPIAware` 与 `CreateActCtxW`（加载 `shell32.dll` 资源 124）兜底，配合静态 CRT 编译，确保 Win10 1909 及以上系统开箱即用。
