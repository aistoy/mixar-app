# Mixar vs. Upstream Blender 5.0 修改分析与总结报告

本报告针对 **Mixar 3D** 项目分支（基于原始 **Blender 5.0** 核心主干）的代码修改与架构设计进行了全方位的代码级深度剖析。旨在为后续开发与系统维护团队提供一份详尽、清晰、具备高度实操价值的技术架构文档。

---

## 目录
1. [项目定位与核心修改机制（Overlay Pattern）](#1-项目定位与核心修改机制overlay-pattern)
2. [构建系统与编译配置覆盖分析（`cmake/` & `scripts/`）](#2-构建系统与编译配置覆盖分析cmake--scripts)
3. [C++ 底层修改与原生平台定制（GHOST & SystemPaths & GIL Safety）](#3-c-底层修改与原生平台定制ghost--systempaths--gil-safety)
4. [双阶段引导启动与模块加载系统（Bootstrap & Deferred UI Batching）](#4-双阶段引导启动与模块加载系统bootstrap--deferred-ui-batching)
5. [核心 Python 插件模块代码级深度拆解（`src/scripts/mixar/modules/`）](#5-核心-python-插件模块代码级深度拆解srcscriptsmixarmodules)
6. [关键交互模式与全系统设计模式（Patterns）](#6-关键交互模式与全系统设计模式patterns)
7. [后续开发与架构维护指南](#7-后续开发与架构维护指南)

---

## 1. 项目定位与核心修改机制（Overlay Pattern）

Mixar 是基于 **Blender 5.0** 定制的 AI 辅助 3D 内容创作套件。为了在最大程度上保持与 Blender 上游分支的兼容性、简化版本升级难度，Mixar 采用了创新的 **Overlay（层叠覆盖）机制**。

```
                     [ upstream/ ] (Blender 5.0 原始源码 submodule)
                           │
                           ▼ (rsync 复制)
                     [ source/ ] (生成的临时编译源码树)
                           ▲
                           │ (rsync 覆写层叠)
                       [ src/ ] (Mixar 自定义 overlay 源码)
```

### 1.1 构建流程与目录运作
- `upstream/`：作为只读的 Git 子模块存储多 GB 的 Blender 5.0 完整官方源码。
- `src/`：只存放 Mixar 自身修改和新增的代码文件。它严格遵循与 Blender 相同的目录树结构（例如 `src/intern/ghost/` 映射 `upstream/intern/ghost/`）。
- `source/`：实际执行 CMake 编译的工作目录（被 Git 忽略）。在每次 `make build` 时：
  1. `scripts/unix/overlay.sh` 首先清空 `source/` 并将 `upstream/` 复制进去。
  2. 接着将 `src/` 的内容层叠（Overlay）拷贝至 `source/`，直接覆盖冲突的同名文件。
  3. 执行编译。**绝不允许直接修改 `source/` 目录中的文件**，所有修改必须落地在 `src/`。

---

## 2. 构建系统与编译配置覆盖分析（`cmake/` & `scripts/`）

Mixar 对 Blender 原生的 CMake 依赖和打包配置进行了针对性重写，重点集中在编译速度、CUDA/OptiX 支持和运行环境解耦上。

### 2.1 `cmake/mixar_overrides.cmake`
该文件在 Blender 执行 `project()` 调用前被注入，强制覆盖了多项 CMake 缓存变量：
- **编译缓存优化**：检测系统 `PATH`。若存在 `sccache`，在 macOS/Linux 下强制设置 `CMAKE_C_COMPILER_LAUNCHER` / `CMAKE_CXX_COMPILER_LAUNCHER`；在 Windows 下，强制启用 `WITH_WINDOWS_SCCACHE=ON` 触发 Blender 内置的 sccache 编译器发射机制（处理 `/Z7`、调试符号、PDB 映射）。
- **Cycles 硬件加速**：
  - `WITH_CYCLES_DEVICE_CUDA=ON`、`WITH_CYCLES_CUDA_BINARIES=ON` 及 `WITH_CUDA_DYNLOAD=ON`，支持运行时动态加载 CUDA 驱动及渲染加速核心。
  - `WITH_CYCLES_DEVICE_OPTIX=ON`，支持 NVIDIA OptiX 光线追踪硬件加速。
- **编译器指令配置**：在 Windows/MSVC 下强制追加 C++ 编译选项 `/DWIN32 /D_WINDOWS /W3 /GR /EHsc`，保障第三方库与底层代码的稳健链接。

### 2.2 `src/CMakeLists.txt`
- 在项目根级的 `CMakeLists.txt` 中修改了主程序名称 `project(Mixar)`，并将输出的可执行文件名和窗口标识符统一注册为 `mixar`。
- 修改 Windows 下的 AppID 宏定义为 `mixar.<major>.<minor>`（派生自 `VERSION` 文件），自定义 Friendly Name 为 `"Mixar 5.0"`，以便操作系统级别的高级通知和应用图标合并。

### 2.3 `scripts/generate_config.py` 与安全防护 (C4 Guard)
- 在构建期间调用，读取本地的 `.env` 或构建系统的环境变量，并于运行时安装包的资源目录中生成 `config/mixar.json`。
- **安全拦截防漏机制**：为了防止在生产环境（Prod）发布包中意外泄露开发旁路凭证（Dev Bypass Credentials），脚本在编译期间内嵌了 “C4 守卫”（C4 Guard）。若 `MIXAR_ENV` 不为 `Dev`（如 `Prod` 或 `UAT`），且检测到设置了任何 `DEV_BYPASS_*` 变量，构建将抛出致命错误并强制中断：
  ```python
  if environment != "Dev" and bypass_any_set:
      sys.stderr.write("ERROR: DEV_BYPASS_* variables are set but MIXAR_ENV != 'Dev'. ...")
      sys.exit(1)
  ```
- 同时在 Python 侧生成静态文件 `source/scripts/mixar/config/_build_env.py`，写死当前编译环境，并在 C++ 侧生成环境头文件 `mixar_env_config.h`。

### 2.4 依赖打包与内置 Python 包管理 (`scripts/python_requirements.txt`)
- 在构建的最后，通过内置或系统的 pip 模块，强制将 `scripts/python_requirements.txt` 中列出的核心 AI 运算、WebSocket 连接、密钥链访问等依赖包安装到 Blender 嵌入式 Python 引擎的 `site-packages` 目录内，保障客户端开箱即用。

---

## 3. C++ 底层修改与原生平台定制（GHOST & SystemPaths & GIL Safety）

Mixar 的窗口管理和多线程模型是重灾区，共包含约 150 个修改的 C++ 文件，其中以 **GHOST（通用便捷操作系统工具包）** 及 **Python 解释器 GIL 安全隔离** 最为底层和关键。

### 3.1 跨平台 GHOST 浮动窗口定制（Win32 subclassing & Cocoa AppKit）
为了在 3D 视图上方实现像 AI Chat Bubble 这样无边框、悬浮、支持拖拽、自带圆角与毛玻璃模糊背景的全新 UI，Mixar 在 GHOST 库（`src/intern/ghost/`）中扩展并导出了一套 `extern "C"` 专有 API，使得 Python Addon 可以通过 `ctypes` 完全控制操作系统级原生窗口：

- **`Mixar_WindowSetChromeless` / `Mixar_WindowSetBorderless`**：
  - **Win32 实现**：通过 `SetWindowLongPtr` 消除 `WS_CAPTION` / `WS_THICKFRAME`，并安装自定义窗口子类 `mixar_min_size_subclass_proc` 接管 `WM_GETMINMAXINFO` 和 `WM_NCHITTEST`，实现 chromeless 窗口仍能在边缘被鼠标调整大小。
  - **Cocoa 实现**：通过设置 `NSWindowStyleMaskFullSizeContentView` 并修改 `titlebarAppearsTransparent = YES`、`titleVisibility = NSWindowTitleHidden`，移除标准 macOS Title Bar，打造全浸式（Full-bleed）界面。
- **`Mixar_WindowSetCornerRadius` / `Mixar_WindowSetBlurBehind`**：
  - **Win32 实现**：利用 DWM（Desktop Window Manager）属性 `DWMWA_WINDOW_CORNER_PREFERENCE` 设为 `DWMWCP_ROUND` 实现 Win11 级窗口圆角，并通过 `DwmEnableBlurBehindWindow` 启用磨砂玻璃效果。
  - **Cocoa 实现**：向 NSWindow 的 contentview 层注入 `NSVisualEffectView`（设置 material 为 `NSVisualEffectMaterialHUDWindow`，state 为 `NSVisualEffectStateActive`），结合 CoreAnimation 层的 `cornerRadius` 裁剪，实现完美的 macOS 原生模糊悬浮毛玻璃。
- **双向联动父子窗口布局跟踪 (Window Parenting & Repositioning)**：
  - 当浮动 Chat Bubble 需要“吸附”在 Blender 主窗口上方并随其拖拽、缩放而实时联动时，传统的 Python Timer 轮询位置极度卡顿。
  - **Win32 实现**：通过 `mixar_install_parent_hook` 强力注入 Win32 Subclass 子类化函数，拦截主窗口的 `WM_MOVE` / `WM_SIZE` 等消息。一旦主窗口移动，立即通过 `mixar_reposition_child` 在同一微秒内计算 Delta 像素偏移量并调用 `SetWindowPos` 重绘子窗口，彻底消除了视觉延迟。
  - **Cocoa 实现**：采用 Objective-C 的 `NSNotificationCenter`，为子窗口绑定监听主窗口的 `NSWindowDidMoveNotification` 和 `NSWindowDidResizeNotification`。在回调中，实时更新子窗口相对于父窗口的边界约束，保持紧密跟随。
- **模态弹窗与隐藏逻辑（`Mixar_FloatingDocksSuppressForModal`）**：
  - 避免在 Blender 打开标准文件选择、确认弹窗等系统级模态（Modal）会话时，由于悬浮窗持续处于 `HWND_TOP` 或 `NSStatusWindowLevel` 导致点击穿透或死锁。
  - 设计了 Suppress/Restore 状态栈：在进入模态状态时，调用 `Mixar_FloatingDocksSuppressForModal` 遍历所有标记为 `mixar_floating_dock` 的窗口并将其安全隐藏（`SW_HIDE` 或 `orderOut:`）；退出模态时，通过 `Mixar_FloatingDocksRestoreAfterModal` 恢复显示。
  - **解决 NVIDIA 驱动黑屏/崩溃问题**：由于在隐藏窗口中继续递交 OpenGL/Vulkan Present 缓冲帧会导致 NVIDIA 显卡驱动在系统休眠唤醒后崩溃于 `DrvPresentBuffers`。C++ 侧通过 `Mixar_WindowIsVisible` 拦截隐藏窗口的全局渲染更新（`wm_draw_update` 中的 `Mixar_WindowIsVisible` 检查），并在重新显示时主动标记全局重绘，规避了 NVIDIA 驱动死锁崩溃问题。

### 3.2 系统存储路径重定位 (`GHOST_SystemPaths`)
修改了 GHOST 内部的文件系统路径获取函数，确保用户配置、首选项和缓存数据独立于原始 Blender 存放：
- **Windows**：`GHOST_SystemPathsWin32.cc` 中的 `GetAppData` 目录从 `\\Blender Foundation\\Blender` 重构为 `\\Mixar\\Mixar\\`。
- **macOS**：`GHOST_SystemPathsCocoa.mm` 定向至 `~/Library/Application Support/Mixar/`。
- **Linux**：`GHOST_SystemPathsUnix.cc` 定向至 `~/.config/mixar/`。

### 3.3 Python GIL 线程安全安全漏洞修正 (`py_capi_utils.cc`)
- 在 Blender 原生架构中，多线程 Python 插件（如多路并行 auth 请求、图片/Mesh 生成的后台下载轮询）极易触发内存破坏（Alloc Corruption）甚至在 `PyUnicode_New` 中段错误崩溃。原因在于原生的 `PyC_IsInterpreterActive()` 使用了 **进程全局的当前线程状态检测**。对于 Python <= 3.11，这会错误地返回“是否有 *任意* 线程持有了 GIL”，从而导致 C++ 原生操作符在释放 GIL 期间（如 `WM_operator_call_py` 前后），后台的 Python 子线程并发调用 Python C-API 抢占导致内存冲突。
- Mixar 对此进行了底层重构，将 `PyC_IsInterpreterActive()` 替换为 **线程局部级 GIL 校验** `PyGILState_Check()`，彻底消除了后台 Daemon 线程并发访问引起的启动段错误和随机崩溃。

---

## 4. 双阶段引导启动与模块加载系统（Bootstrap & Deferred UI Batching）

为了防止包含上千个 Python 文件的庞大 AI 模块群在 Blender 启动时造成阻塞、导致界面假死或加载菊花，Mixar 设计了优雅的 **双阶段异步/延时引导加载方案**。

入口文件位于：`src/scripts/startup/bootstrap/__init__.py`

```
[ 阶段 1：同步引导 (Synchronous Bootstrap) ]
 - 注册顶级 sys.modules 虚拟合成包。
 - 同步加载 mixar/bootstrap/*.py 中的核心服务（网络连接、缓存、更新自检等）。
                  │
                  ▼
[ 阶段 2：延时 UI 模块分批加载 (Deferred UI Batching) ]
 - 注册 bpy.app.timers，每帧执行一次
 - 限制每帧加载时长不超过 4ms (ui_batch_budget_ms)
 - 依序加载：1. Properties (优先级 0) -> 2. Operators/Core (优先级 1) -> 3. Panels (优先级 2)
```

### 4.1 阶段 1：同步引导与合成包封装 (`_setup_mixar_packages`)
- **零包初始化机制**：为了让第三方库和相对引用能在 `sys.modules` 中正常运行，在不污染底层目录的前提下，引导程序自动扫描并为 `mixar`、`mixar.modules`、`mixar.bootstrap`、`mixar.config` 注册合成包。
- **引导模块注册**：依次加载 `mixar/bootstrap/` 下的物理组件（如 `agent_connection`，`sandbox_supervisor` 等），并在执行 `spec.loader.exec_module` 之前立即将模块名注册到 `sys.modules`，防止循环依赖导致产生双重模块实例与全局状态发散。

### 4.2 阶段 2：时间预算化延时 UI 模块加载 (`_load_ui_batch_tick`)
- **4ms 帧预算屏障**：UI 模块（共 415+ 个文件）的完整注册需要消耗 1~2 秒。为避免卡死，Mixar 通过 `bpy.app.timers` 在 Blender 的绘图主循环中注册了计时器微任务 `_load_ui_batch_tick`。
- **时序依赖控制（Sort Priority）**：
  在扫描 UI 目录时，系统会对所有模块文件的注册顺序进行拓扑排序：
  1. **Properties（优先级 0）**：必须最先注册。所有 PropertyGroup 在 Blender 系统中充当数据承载骨架。
  2. **Operators / Core（优先级 1）**：各类操作算子。
  3. **Panels / Menus / Headers（优先级 2）**：界面绘制层。由于面板在渲染时会通过属性反射来绑定 Operator，因此若算子未注册完成即加载面板，会导致 Blender 发生严重的 Optype 悬空引用并崩溃。
- 每帧加载时，累加消耗时长。一旦超过配置文件中的性能时预算上限（默认 `ui_batch_budget_ms = 4`），则主动 `return 0.0` 挂起，把控制权归还给 Blender 的原生事件处理器，维持了 60 帧无卡顿的流畅启动体验。

---

## 5. 核心 Python 插件模块代码级深度拆解（`src/scripts/mixar/modules/`）

这是 Mixar 自定义 Python 业务层的大本营，承载了所有核心特性。

### 5.1 `paint` (核心图层绘画模块，59MB)
- **底层架构**：该模块是体积最大的核心模块，承载了 Photoshop 风格的堆叠图层绘图系统。
- **技术实现**：
  - 基于 Blender 的节点树（ShaderNodeTree）实现。每个图层实际上是一个封装的节点组（NodeGroup），通过动态构建连线控制材质通道、遮罩（Mask）、图层混合模式、非破坏性修改器（Modifiers）等。
  - 支持多象限 UDIM 纹理烘焙、过程化程序材质和 Decal 贴花映射。
  - 深度结合 C++ 重写了其底层像素绘画内核，大幅提高了高分辨率纹理绘制的响应速度。

### 5.2 `space_mixie_chat` (AI 智能体聊天逻辑)
- **长连接中枢**：通过 JSON-RPC 2.0 over WebSocket 架构与 hosted AI 智能体后台建立双向实时通信。
- **消息流处理**：使用 Server-Sent Events (SSE) 协议对生成式的回复进行流式分发、在渲染画布中动态排版、提供支持 Markdown 和 Thinking 思考步骤的可视化渲染。
- **远程脚本沙盒（Sandboxed Script Execution）**：
  AI 智能体的底层指令实质上是生成一小段 Blender Python 脚本传给客户端执行。该模块提供了带有超时监控、错误自愈及主动回收机制的 Headless 运行环境，能够在不干扰用户当前 3D 视口的前提下并行、安全地测试脚本效果。

### 5.3 `agent_viewport_lock` (视口锁定视觉处理)
- 当 AI 智能体在非激活场景或后台沙盒中大量创建、修改 Mesh、排布材质时（TURN 模式为 BUSY/MODIFYING），为防止用户在 3D 视图中进行并发的手动冲突操作，该模块将触发一个 **视口锁定状态**。
- **技术实现**：
  - 使用 `SpaceView3D.draw_handler_add` 挂载 `POST_PIXEL` 绘图处理函数。
  - 在主 3D 视口边缘渲染一层动态呼吸、由浅变深的绿色内发光光晕（Breathing Inner Glow Halo）。
  - 通过注册一个 Modal 拦截器来消耗（Consume）一切属于视口内的鼠标选择、顶点编辑、物体变换等输入事件，但优雅地释放（Pass Through）摄像机平移、旋转及缩放事件，允许用户在 AI 施工期间自由观察。
  - 该锁定强力绑定在 `scene.mixie_chat_active_turn_mode` 上，无法因用户在中途切换下拉菜单而绕过，杜绝了并发读写造成的 blend 树崩溃。

### 5.4 `agent_bubble` (悬浮聊天气泡窗口)
- 属于纯 Python 实现的 persistent modal operator。
- 利用前述 GHOST C++ 特制 API 创建独立窗口，将 Chat 交互、状态药丸、快捷输入框直接“画”在 3D 视口边缘，并与 `space_mixie_chat` 后端共享同一个 ConnectionManager 和 Scene Message Store 数据。
- **生命周期崩溃拦截（Crash-class Fixes）**：
  - 为了防止含有这些自定义 GHOST 窗口指针的数据被 Blender 的标准文件保存流程（交互保存、自动保存、启动项保存）意外序列化到 `.blend` 文件中导致坏档：
    在 C++ 的 `BLO_write_file` 链路中强行剥离其 window 指针（`wm_agent_bubble_windows_unlink_for_write`）。
    在读取 `.blend` 时增加主动剥离拦截（`wm_file_read_strip_agent_bubble_windows`）。
    同时，通过 GHOST 底层的窗口关闭（`wm_window_close`）和内存释放（`wm_window_free`）双重安全钩子（ED_agent_bubble_windows_closed / freed），一并清除全局静态 GHOST 裸指针缓存。

### 5.5 `agent_scene_strip` (C++ 底层多智能体视口监控)
- **底层 C++ 区域扩展**：替代了原先已被废弃的 `SPACE_SCENE_GRID` 空间，在 View3D 区域底部注册了自定义 Region 类型 `RGN_TYPE_EXECUTE`（`view3d_agent_strip.cc`）。
- **多场景离屏实时监视**：
  - Mixar 支持在多场景中并行运行 AI 智能体。为了让用户在当前活动场景中直观监视其他未激活场景中的智能体工作进度，该 Region 会以非阻塞方式对其他所有场景执行离屏渲染（Offscreen Rendering）。
  - 生成 3D 视口缩略图平铺在视口下方，每 0.1 秒通过 `TIMERNOTIFIER` 结合 `DEG_get_update_count`（依赖图变更计数检测）动态刷新缩略图，用户点击相应的缩略图平铺块即可在不丢失任何视口遮罩的前提下，瞬间完成当前活动场景的热切换。

### 5.6 `moodboard` (情感板与多维生成面板)
- 包含无损粘贴板、剪贴板监控服务。
- 整合了 **Image Gen**（图片生成）、**AI Render**（AI 渲染 / 深度图生成）、**Model Gen**（3D 模型快速生成 / 混元模型接入）、**Texture Gen**（纹理贴图合成及材质脚本排队）、**Retopology**（拓扑重构）、**UV Unwrap** 等标签页。
- 全面解耦了历史遗留 subtab 逻辑，统一通过 Generation Catalog 实现数据驱动的 UI 布局（见第 6 节）。

---

## 6. 关键交互模式与全系统设计模式（Patterns）

Mixar 在插件系统设计与 AI 与 DCC 交互上建立了三个标杆式的通用架构模式。

### 6.1 统一任务异步队列调度架构（Unified Job Queue）
为了解决各个 AI 特性独自轮询、线程竞争和数据丢失的问题，Mixar 设计了 **Unified Job Queue (统一任务队列)** 系统：

```
[ AI 算子调用 (如 Model Gen, Retopo) ]
                 │
                 ▼
[ Enqueue Engine (moodboard/core/generation_enqueue.py) ]
                 │ (按 payload 规整化组装)
                 ▼
[ JobQueueService (modules/common/job_queue/) ]
                 │
     ┌───────────┴───────────┐ (按任务类型分流创建)
     ▼                       ▼
 [ AsyncGLBJob ]         [ SyncImageJob ]
  - 轮询任务ID            - 提交立即同步处理
  - 下载二进制 GLB         - 图片多线程落盘
  - 触发导入 & Stamping     - 自动渲染缩略图
```

- **统一契约接口**：通过 `POST /job-queue/jobs` 统一下发，返回任务 ID 后启动 `GET /job-queue/jobs/{id}` 高效轮询。
- **模型驱动参数多路分发 (Fan-out)**：
  以 `Retopology` 为例，同一个 Retopo 队列，底层可以根据服务特征动态路由到 `Hunyuan` 拓扑（参数包括 `polygon_type`, `face_level`）或者 `Tripo` 拓扑（参数包括 `quad`, `face_limit`），客户端通过 `retopology_enqueue.py` 动态构建对应的 json 对象并配置回调，所有的队列状态管理（PENDING → RUNNING_SUBMIT → RUNNING_POLL → RUNNING_DOWNLOAD → SUCCESS/FAILED）完全由 `QueueManager` 自动化管理。

### 6.2 数据库驱动的动态参数表单渲染（Generation Catalog & Dynamic Params）
为了避免每次 AI 服务端后台升级参数、更换模型时，客户端都要重新发版、重写 Python 属性定义，Mixar 开发了 **Schema 驱动的参数系统**：
- **`generation_catalog_cache.py`**：在 Blender 启动时自动通过 `GET /api/v1/generation-catalog` 缓存完整的服务、模型及参数配置 schema，并利用本地 ETag (`If-None-Match`) 实现秒级增量更新和本地持久化。
- **WindowManager 动态参数挂载**：
  - 传统 Blender 插件将参数挂载在 `bpy.types.Scene`，这会导致 Undo/Redo 时由于属性树重置触发异常崩溃。
  - Mixar 将所有根据 schema 生成的属性（包括各种滑动条、开关、字符串下拉等）**动态挂载在 `bpy.types.WindowManager` 指针下**。
  - `draw_service_params()` 会读取 Schema 中的 `visible_if`、`order`、`group` 动态构建表单面板。
  - `collect_params()` 会自动扫描激活的 UI Widget 字段并将其转换为标准 snake_case 的 Python Dict 返回给 Enqueue Engine，实现了**零发版即可同步服务器最新模型参数配置**。

### 6.3 主子进程沙盒生命周期守护（Sandbox Supervision & Parent Liveness Check）
- 为防止沙盒 Headless 后台 Blender 子进程在模型创建完毕或异常出错后成为僵尸进程、常驻后台耗尽用户 GPU 显存，设计了 **多层级的生命周期守护**。
- **C++ + Python 双重检测**：
  - 启动子进程时，父进程通过环境变量向子进程注入自身 PID (`MIXAR_SANDBOX_PARENT_PID`)、生存周期 TTL (`MIXAR_SANDBOX_IDLE_TTL_S`)。
  - **Windows 守护**：子进程在底层使用 Win32 API 轮询或注册父进程句柄通知。一旦父进程句柄无效，即刻引发主动终止。
  - **Linux/macOS 守护**：子进程定期监控主进程 PID 是否由于意外退出被 init(1) 接管。
  - 父进程在 `unregister()` (Blender 退出) 钩子中，通过 `sandbox_supervisor.py` 主动向子进程发出优雅关闭信号（`SIGTERM`），并在 5 秒超时后升级为强制杀死（`SIGKILL`），保证不遗留任何后台资源消耗。

---

## 7. 后育开发与架构维护指南

在后续对 Mixar 的升级迭代、错误排查和功能开发中，请严格遵守以下核心守则：

### 7.1 文件修改物理红线
- **禁止修改编译产物**：切勿直接在 `source/`、`build/` 目录下对代码进行就地编辑。它们均属于编译中间目录。所有修改必须落在宿主目录 `src/` 中，然后通过 `make build` 进行 Overlay 合并与重新生成。
- **代码物理长度限制**：为了保障极致的运行性能和可读性，任何 Python/C++ 源文件的长度均**不得超过 500 行**。复杂业务逻辑应进行垂直拆分、下沉至组件或通用工具包（`modules/common/utils/`）中。

### 7.2 模块注册与引用规则
- **避免硬编码 UI 依赖**：不要在 Panel/Menu 的绘制代码中直接硬编码调用特定 AI 的属性字段。应当使用 `generation_params.draw_service_params` 和 `collect_params` 统一接入 schema 参数系统。
- **遵循 bootstrap 分阶段加载**：
  - 新增的基础通用数据、网络缓存组件，应归类在 `src/scripts/mixar/bootstrap/` 并包含 `register()` / `unregister()`，这些组件在同步初始化阶段运行。
  - 新增的面板、操作符必须放在 `modules/{module_name}/ui/` 下。其生命周期由 `bootstrap` 统一自动发现和延迟分批（4ms/帧）注册，**禁止在子模块的 `__init__.py` 中进行大范围的同步循环加载**，否则会破坏时间预算，引发启动假死。

### 7.3 GIL 安全与多线程开发限制
- **Python-C 接口交互线程锁定**：任何在多线程、Timer 回调、网络长连接事件回调中访问 Blender 数据集（`bpy.data`）或进行 UI 更新的操作，**必须返回到主线程执行**。
- 请善用 `mixar.modules.common.api` 提供的 `start_executor` 线程池机制，绝不能随便使用原生 Python Thread 进行未经 GIL 安全加锁的 `bpy` 核心操作。

---

*本报告是为 Mixar 开发与维护量身打造的高级技术文档。遵循上述设计思想与操作规范，将极大提高开发效能，并确保产品的架构稳定与技术演进优势。*
