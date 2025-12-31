# Project Context

## Purpose
本仓库是 The Powder Toy（物理沙盒游戏）的 C++ 代码库与构建/打包脚本集合。

核心目标：
- 提供跨平台的桌面/移动端/（可选）Web 版本的可执行程序（默认目标名为 `powder`）。
- 维护一个实时 2D 物理/化学/电子交互模拟内核，并提供图形渲染与 GUI 交互。
- 支持 Lua 控制台/脚本 API（默认自动选择 LuaJIT；某些平台回退到 Lua 5.2/5.1 或禁用）。

## Tech Stack
- 语言：C++20（Meson 设置 `cpp_std=c++20`），辅以少量 Python 脚本用于资源生成。
- 构建系统：Meson + Ninja
	- 推荐流程（见 SETUP）：`meson setup build` → `meson compile -C build` → `./build/powder`
- 运行时/平台：Linux（主要）、Windows、macOS、Android、Emscripten（WASM）
- 图形与窗口：SDL2
- 依赖库（根据平台/选项可能不同，常见包括）：FFTW3f、LuaJIT/Lua、libcurl（HTTP/在线功能）、libpng、bzip2、JsonCpp、TLS/加密相关库（取决于平台/打包方式）
- 依赖获取策略：
	- 可使用系统库（`-Dstatic=system`）
	- 或使用官方预编译的 tpt-libs 子项目（`-Dstatic=prebuilt`，通过 `subprojects/*.wrap`）

## Project Conventions

### Code Style
- C++：
	- 缩进：以制表符为主（仓库中大量文件使用 tab 缩进；提交时避免把 tab/空格混用造成大范围 diff）
	- 大括号：K&R 风格常见（函数/控制块 `{` 换行；小块 `if` 可能省略大括号，保持与周边一致）
	- 命名：类型/类通常使用 `PascalCase`，函数/方法多用 `PascalCase` 或项目既有风格（以同目录相邻代码为准）；局部变量偏 `camelCase`
	- RTTI：构建默认禁用（Meson `cpp_rtti=false`），新增代码不要依赖 `dynamic_cast`/`typeid`
	- 异常：项目中使用异常处理（例如顶层崩溃处理/错误提示），新增代码遵循既有错误处理路径（UI 错误对话框/日志）
- Python：
	- 主要用于构建/资源生成脚本（例如 resources/gencredits.py）
	- 类型标注：可使用 Python 3 风格类型标注（脚本中已存在 `url : str` 等）

### Architecture Patterns
- 高层分层（以目录为主，不强行引入新架构）：
	- `src/simulation/`：模拟核心、元素/交互、保存渲染等
	- `src/graphics/`：图形渲染与绘制基础设施
	- `src/gui/`：UI、视图/控制器、对话框等
	- `src/client/`：与线上服务交互、保存文件/网络请求（`client/http/...`）
	- `src/prefs/`：偏好设置/持久化
	- `src/common/`：跨平台与通用工具（平台抽象、字符串/随机等）
- 配置/编译期常量：
	- 通过 Meson `configure_file()` 从模板生成头文件（例如 `src/Config.template.h` → `Config.h`、`VcsTag.template.h` → `VcsTag.h`）。
	- 新增编译期开关优先走 Meson option + `configure_file()`，避免在代码里硬编码平台/URL。

### Testing Strategy
- TODO：本仓库当前未发现统一的测试框架/测试目录约定。
- 建议（若引入测试）：优先为纯逻辑模块（如 `common/`、`simulation/` 的无 UI 部分）添加可执行的单元测试目标，并集成到 Meson 里（`meson test`）。

### Git Workflow
- 默认分支：`master`。
- TODO：仓库未提供 CONTRIBUTING/提交规范文件。
- 建议协作方式：
	- 用功能分支 + PR 合并到 `master`
	- 避免把格式化/机械重排与功能修改混在同一提交，以减小 review 成本

## Domain Context
- “元素/材料/粒子”是模拟的基本单位；保存文件（saves）与截图（png）是用户分享与回放的主要载体。
- 在线功能：保存/加载线上存档、获取资源等，受 `-Dhttp` 与服务器配置项影响。
- Lua：提供脚本化/插件化能力（可用于自动化与扩展玩法）。

## Important Constraints
- 跨平台约束很强：新增代码要考虑 Windows/macOS/Linux/Android/Emscripten 的兼容性。
- Emscripten/Android 有特定限制：
	- Emscripten 下部分行为会被禁用或替换（例如退出、数据目录等；见 `src/meson.build` 中的条件配置）。
- Release 构建安全约束：非 Debug 且未启用 HTTPS 强制时会拒绝构建（见 `-Denforce_https` 相关检查）。
- 构建选项很多（见 `meson_options.txt`），修改行为前先确认对应 option 是否已存在，避免引入重复开关。

## External Dependencies
- 在线服务：默认指向 `powdertoy.co.uk`（`server`/`static_server` 等 Meson 选项可配置）。
- GitHub API：资源脚本 `resources/gencredits.py` 会请求 `https://api.github.com/repos/The-Powder-Toy/The-Powder-Toy/contributors` 生成 `resources/credits.json`。
- 系统依赖（Linux 开发常用，见 SETUP）：SDL2、FFTW3、LuaJIT、libcurl、OpenSSL 开发包、JsonCpp、bzip2、libpng 等。
