# 项目架构概览（The Powder Toy）

本文面向首次读代码的开发者：解释本仓库的**目录分层**、**构建产物**、**启动/主循环**、以及模拟/渲染/GUI/网络/Lua 的关系。

## 1. 顶层目录

- `src/`：核心 C++ 代码（桌面端/Android/Emscripten 共用大部分逻辑）
- `resources/`：资源模板与生成脚本（图标、credits、desktop 文件等）
- `android/`：Android 打包/构建脚本与模板
- `docs/`：项目文档（从现在开始新增文档统一放这里）
- `subprojects/`：第三方依赖（含预编译 tpt-libs 的 `.wrap`）
- `meson.build` / `meson_options.txt`：Meson 构建入口与可配置选项

## 2. `src/` 内部模块划分

`src/` 以“按职责分目录”的方式组织：

- `src/simulation/`：模拟核心与元素系统
  - 粒子/元素：`Particle.*`、`Element.*`、`elements/`
  - 模拟循环与物理：`Simulation.*`、`Air.*`、`gravity/`
  - 保存/快照：`Snapshot*`、`SaveRenderer.*`
  - 工具系统：`Tool*`、`simtools/`
- `src/gui/`：GUI 视图、对话框与业务控制器
  - 游戏主界面：`gui/game/`（GameModel / GameView / GameController）
  - 选项、搜索、预览、存档、本地浏览器等：`gui/options/`、`gui/search/`、`gui/preview/`、`gui/save/`...
  - UI 引擎接口层：`gui/interface/`（由 `ui::Engine` 驱动）
- `src/graphics/`：渲染与像素缓冲抽象
  - `Graphics.*` / `Renderer.*` / `VideoBuffer.*`
- `src/client/`：在线功能与服务端交互
  - 核心客户端：`Client.*`、`ClientListener.*`
  - 保存/用户：`SaveFile.*`、`SaveInfo.*`、`User.*`
  - HTTP：`client/http/`（请求、TLS、Websocket 等实现）
- `src/lua/`：Lua API 与脚本集成
  - Lua 与引擎/模拟/图形/HTTP 的绑定：`LuaSimulation.*`、`LuaGraphics.*`、`LuaHttp.*`...
- `src/common/`：通用工具与平台抽象
  - 平台层：`common/platform/`（CWD、文件系统、崩溃处理、Console、StackTrace 等）
  - 其他：字符串、随机、向量/几何、延迟执行等
- `src/prefs/`：偏好设置与持久化（例如 `GlobalPrefs`）
- `src/tasks/`：异步任务/可取消任务的基础设施（Task/TaskWindow 等）
- 入口/胶水代码（位于 `src/` 根目录）：
  - 主程序：`PowderToy.cpp`、`PowderToySDL.cpp`、`PowderToySDLCommon.cpp`
  - Emscripten 特化：`PowderToySDLEmscripten.cpp`
  - 渲染器/字体工具：`PowderToyRenderer.cpp`、`PowderToyFontEditor.cpp`

## 3. 构建系统（Meson）与产物

项目使用 Meson + Ninja。

### 3.1 关键构建特性

- C++ 标准：C++20
- RTTI：默认禁用（不要新增依赖 `dynamic_cast/typeid` 的代码路径）
- 配置头生成：通过模板生成编译期常量：
  - `src/Config.template.h` → `Config.h`
  - `src/VcsTag.template.h` → `VcsTag.h`
  - `simulation/ElementNumbers.template.h` → `ElementNumbers.h`
  - `simulation/ToolNumbers.template.h` → `ToolNumbers.h`

### 3.2 主要目标（targets）

Meson 会将核心代码拆成 3 个静态库依赖，然后链接到不同可执行目标：

- `common`（通用/平台/部分图形基础）
- `gui`（UI 系统）
- `simulation`（模拟核心）

在此基础上生成：

- 主程序：`powder`（实际名称由 `-Dapp_exe` 决定）
  - Android 下会生成 `shared_library(app_exe)` 作为 APK 的 native 库
- 工具：`render`（渲染相关工具）
- 工具：`font`（字体编辑器）

依赖获取策略（概念上）：

- 使用系统库（`-Dstatic=system`）
- 或使用预编译的 tpt-libs（`-Dstatic=prebuilt`，对应 `subprojects/tpt-libs-prebuilt-*.wrap`）

## 4. 启动流程（桌面端）

桌面端主入口在 `src/PowderToy.cpp`：

1. `main()`
   - 调用 `Platform::SetupCrt()`
   - 通过 `Platform::InvokeMain(argc, argv)` 转入 `Main()`（便于跨平台封装/异常处理）
2. `Main()` 初始化顺序（简化）：
   - `SDL_Init(0)`（先初始化 SDL 基础设施，视频子系统稍后在 `SDLOpen()` 初始化）
   - 解析命令行参数（例如 `open`、`ptsave:`、`scale:`、`proxy:`、`disable-network` 等）
   - 处理数据目录（`ddir` / 默认目录），再加载偏好：`GlobalPrefs`
   - 初始化网络与客户端：
     - `http::RequestManager::Create(...)`
     - `Client`（在线功能的主对象）
   - 初始化引擎与 UI：
     - `ui::Engine` + `Graphics`
     - `SDLOpen()` 创建窗口/renderer/texture，并设置逻辑分辨率（`WINDOWW/WINDOWH`）
   - 初始化模拟数据与游戏控制器：
     - `SimulationData`
     - `GameController`
     - `engine.ShowWindow(gameController->GetView())`
   - 可选：处理 `open`/`ptsave` 自动打开存档
   - 进入主循环：`MainLoop()`

> 这里使用了一个“显式单例”容器（`ExplicitSingletons`）来控制全局对象的创建/销毁顺序，避免静态初始化顺序问题。

## 5. 主循环与帧调度（SDL）

主循环位于 `src/PowderToySDLCommon.cpp`：

- `MainLoop()`：只要 `ui::Engine::Ref().Running()` 为真，就不断调用 `EngineProcess()`

`EngineProcess()` 位于 `src/PowderToySDL.cpp`，做三件事：

1. 事件处理（Input）
   - `SDL_PollEvent()` → `EventProcess()`
   - 把键鼠/文本输入/拖拽文件等事件分发给 `ui::Engine`（`onKeyPress`、`onMouseMove`...）
2. 逻辑更新（Tick/SimTick）
   - 通过 `FrameSchedule` 对“模拟 tick”和“绘制 draw”做节流
   - 典型顺序：
     - `engine.Tick()`：UI/逻辑 tick（常用于状态推进）
     - `engine.SimTick()`：模拟推进（粒子/空气/元素交互等）
3. 绘制（Draw）
   - `engine.Draw()` 将当前帧渲染到 `Graphics`/像素缓冲
   - `SDLSetScreen()` 处理窗口模式变化
   - `blit(engine.g->Data())`：将像素缓冲提交到 SDL texture 并 `Present`

简化的数据流：

```
SDL events
   ↓
ui::Engine (input dispatch)
   ↓
GameController / GameModel
   ↓
Simulation (tick)
   ↓
Renderer/Graphics (draw)
   ↓
SDL texture → window
```

## 5.1 Tick / SimTick / Draw 关键调用链（按文件引用）

下面按“从外到内”的顺序列出关键类/方法与调用链，方便快速定位代码。

### A) 主循环驱动（SDL → Engine）

- `src/PowderToySDLCommon.cpp`
  - `MainLoop()`：while 循环，持续调用 `EngineProcess()`
- `src/PowderToySDL.cpp`
  - `EngineProcess()`：
    - 处理事件：`SDL_PollEvent()` → `EventProcess()` → `ui::Engine` 的 `onKeyPress/onMouseMove/...`
    - 逻辑 tick：满足节流条件时调用 `ui::Engine::Tick()`
    - 模拟 tick：满足节流条件时调用 `ui::Engine::SimTick()`
    - 绘制：满足节流条件时调用 `ui::Engine::Draw()`，并 `blit(engine.g->Data())`

### B) UI 引擎（Engine → Window → Component）

- `src/gui/interface/Engine.h`
  - 关键类：`ui::Engine`
  - 关键方法：`Tick()` / `SimTick()` / `Draw()`
- `src/gui/interface/Engine.cpp`
  - `Engine::Tick()`
    - `state_->DoTick()`（`state_` 是当前激活的 `ui::Window`）
  - `Engine::SimTick()`
    - `state_->DoSimTick()`
  - `Engine::Draw()`
    - 处理“窗口切换时的冻结背景 + 淡入”
    - `state_->DoDraw()`
    - `g->Finalise()`（提交/收尾渲染）
- `src/gui/interface/Window.h`
  - 关键类：`ui::Window`
  - 关键“模板方法”（可被派生类重写的 `On*`）：
    - `OnTick()` / `OnSimTick()` / `OnDraw()`
  - 关键“封装调用”（由 Engine 调用，内部再调用 `On*` 与组件）：
    - `DoTick()` / `DoSimTick()` / `DoDraw()`
- `src/gui/interface/Window.cpp`
  - `Window::DoTick()`
    - 处理文本输入启停（`ui::Engine::StartTextInput/StopTextInput`）
    - 遍历组件：`Component::Tick()`
    - 调用 `OnTick()`
  - `Window::DoSimTick()`
    - 调用 `OnSimTick()`（默认很薄，通常由具体 Window 实现“每帧模拟侧更新”）
  - `Window::DoDraw()`
    - 调用 `OnDraw()`
    - 依次 `Component::Draw(...)`（并在 debugMode 下画调试覆盖）

### C) 游戏主窗口（GameView → GameController → GameModel → Simulation）

- `src/gui/game/GameView.h`
  - `class GameView : public ui::Window`
  - 覆盖：`void OnSimTick() override;`
- `src/gui/game/GameView.cpp`
  - `GameView::OnSimTick()`
    - `c->Update();`（`c` 是 `GameController*`）
    - `wantFrame = true;`
- `src/gui/game/GameController.h`
  - `class GameController : public ClientListener, public ExplicitSingleton<GameController>`
  - 关键方法：`Update()`（驱动游戏态更新与模拟推进）、`BeforeSimDraw()/AfterSimDraw()`（Lua/命令接口的渲染钩子）
- `src/gui/game/GameController.cpp`
  - `GameController::Update()`（节选逻辑）：
    - 读取鼠标位置、采样：`Simulation::GetSample(...)`
    - 若模拟在跑：`gameModel->UpdateUpTo(NPART);`（推进模拟）
    - 否则：`gameModel->BeforeSim();`（即使暂停也做一些准备/事件）
- `src/gui/game/GameModel.h`
  - 关键成员：`Simulation *sim; Renderer *ren;`
- `src/gui/game/GameModel.cpp`
  - `GameModel::UpdateUpTo(int upTo)`
    - 第一次进入一帧更新时：`BeforeSim()`
    - 逐段更新粒子：`sim->UpdateParticles(sim->debug_nextToUpdate, upTo)`
    - 更新到末尾：`AfterSim()`，并重置 `debug_nextToUpdate`
  - `GameModel::BeforeSim()`
    - 若模拟在跑：`CommandInterface::Ref().HandleEvent(BeforeSimEvent{})`
    - `sim->BeforeSim(willUpdate)`
  - `GameModel::AfterSim()`
    - `sim->AfterSim()`
    - `CommandInterface::Ref().HandleEvent(AfterSimEvent{})`

### D) 模拟核心（Simulation）

- `src/simulation/Simulation.h`
  - 关键类：`class Simulation : public RenderableSimulation`
    - `RenderableSimulation` 持有“渲染需要的状态”（如 gravity 输入/输出、signs、各种场图、粒子数组等）
  - 关键方法（与每帧推进相关）：
    - `void BeforeSim(bool willUpdate);`
    - `void UpdateParticles(int start, int end);`（更新 `[start, end)` 的粒子范围）
    - `void AfterSim();`
    - （辅助）`void SimulateGoL();`、`void RecalcFreeParticles(...)`、`void CheckStacking()` 等
- `src/simulation/Simulation.cpp`
  - `Simulation::UpdateParticles(...)`：粒子更新的核心入口（通常由 GameModel 分段调用以支持调试/分帧）
  - `Simulation::BeforeSim(...)` / `Simulation::AfterSim()`：每帧更新前后钩子（用于准备/收尾、计数、触发某些批处理逻辑等）

> 结论：项目的“模拟推进”并不直接发生在 `ui::Engine::SimTick()` 里，而是由当前活跃窗口（通常是 `GameView`）在其 `OnSimTick()` 中调用 `GameController::Update()` 间接驱动。

## 6. GUI / Controller / Simulation 的关系

从头文件依赖和初始化顺序来看，核心“游戏态”大致是：

- `ui::Engine`：全局 UI 引擎，负责窗口栈、输入分发、tick/simTick/draw 调度
- `GameController`：游戏主控制器（同时实现 `ClientListener`，能接收在线事件）
  - 管理多个子控制器/窗口（登录、搜索、预览、选项、本地浏览等）
  - 处理键鼠操作（画笔、选择元素、复制粘贴、缩放、撤销/重做等）
- `SimulationData`：偏向“静态/共享”的模拟元数据（元素表、工具表、文本资源等）
- `Simulation`：运行时模拟状态（粒子场、空气、重力、热量、电流等）

### 6.1 对象依赖方向图（更细）

下面的图强调“谁调用谁 / 谁依赖谁”（非所有权图；箭头表示主要依赖/调用方向）：

```mermaid
flowchart TD
  SDL[SDL event loop\nEngineProcess()] --> Engine[ui::Engine\nTick/SimTick/Draw]

  Engine --> Window[ui::Window state_\nDoTick/DoSimTick/DoDraw]
  Window --> Components[ui::Component\nTick/Draw]

  Window --> GameView[GameView\nOnSimTick/OnDraw]
  GameView --> GameController[GameController\nUpdate()]

  GameController --> GameModel[GameModel\nUpdateUpTo/BeforeSim/AfterSim]
  GameModel --> Simulation[Simulation\nBeforeSim/UpdateParticles/AfterSim]

  GameModel --> Renderer[Renderer\n(sim render path)]
  Renderer --> RenderState[RenderableSimulation\n(pmap/parts/air/grav...)]
  Simulation --> RenderState

  GameController --> Client[Client\n(online features)]
  Client --> Http[client/http\nrequests]

  GameModel --> CommandInterface[CommandInterface\n(BeforeSim/AfterSim events)]
  GameController --> CommandInterface
  CommandInterface --> Lua[LuaScriptInterface\nLua bindings]
  Lua --> Simulation
  Lua --> Renderer
  Lua --> Client
```

## 7. 在线功能（Client/HTTP）

- `src/client/Client.*`：在线功能入口（登录、加载/保存线上存档、通知等）
- `src/client/http/`：HTTP 请求与底层连接实现（平台差异会在这里体现）
- `ClientListener`：UI/控制器通过监听器模式接收客户端事件（例如新通知、登录状态变化）

## 8. Lua 扩展

Lua 层主要位于 `src/lua/`，提供对以下能力的脚本化访问（按文件名可直观判断）：

- 模拟：`LuaSimulation.*` / 元素：`LuaElements.*`
- 图形与渲染：`LuaGraphics.*` / `LuaRenderer.*`
- 网络：`LuaHttp.*` / socket：`LuaSocket*`
- 平台：`LuaPlatform.*` / 文件系统：`LuaFileSystem.*`

Lua 入口通常通过 `CommandInterface` / `LuaScriptInterface` 与 `GameController`、`GameModel` 绑定。

## 9. 平台差异（Android / Emscripten）

构建层面：

- Android：主目标会生成 `shared_library(app_exe)` 并进入 `android/` 子目录继续打包。
- Emscripten：
  - 禁用部分能力（例如 `ALLOW_QUIT=false`、数据目录能力受限）
  - 输出会产生 `.js + .wasm`，并提供 `serve-locally` 运行目标。

运行时层面：

- `Config.h` 中会注入平台相关的编译期常量（例如 `DEFAULT_TOUCH_UI`、`FORCE_WINDOW_FRAME_OPS` 等）。

## 10. 读代码的推荐路径

1. 启动与主循环：`src/PowderToy.cpp` → `src/PowderToySDLCommon.cpp` → `src/PowderToySDL.cpp`
2. UI 主控制器：`src/gui/game/GameController.*`（再顺藤摸瓜到 GameModel/GameView）
3. 模拟核心：`src/simulation/Simulation.*` 与 `src/simulation/elements/`
4. 渲染与像素通道：`src/graphics/Graphics.*`、`src/graphics/Renderer.*`
5. 在线与任务：`src/client/Client.*`、`src/tasks/*`
6. Lua：`src/lua/*`

