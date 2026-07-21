# Mixar vs. Upstream Blender 5.0 修改分析与总结报告

本报告针对 **Mixar 3D** 项目分支（基于原始 **Blender 5.0** 核心主干）的代码修改与架构设计进行了全方位的代码级深度剖析，并**重点聚焦于 Mixar 的 UI 实现方式**。旨在为后续开发与系统维护团队提供一份详尽、清晰、具备极高实操价值的底层架构文档。

---

## 目录
1. [项目定位与层叠修改机制（Overlay Pattern）](#1-项目定位与层叠修改机制overlay-pattern)
2. [构建系统与编译配置覆盖（`cmake/` & `scripts/`）](#2-构建系统与编译配置覆盖cmake--scripts)
3. [C++ 底层修改与原生平台定制（GHOST & SystemPaths & GIL Safety）](#3-c-底层修改与原生平台定制ghost--systempaths--gil-safety)
4. [双阶段引导启动与模块加载（Bootstrap & Deferred UI Batching）](#4-双阶段引导启动与模块加载bootstrap--deferred-ui-batching)
5. [重点突破：Mixar UI 实现方式深度剖析](#5-重点突破mixar-ui-实现方式深度剖析)
6. [核心 Python 插件模块代码级拆解（`src/scripts/mixar/modules/`）](#6-核心-python-插件模块代码级拆解srcscriptsmixarmodules)
7. [关键交互模式与全系统设计模式（Patterns）](#7-关键交互模式与全系统设计模式patterns)
8. [后续开发与架构维护指南](#8-后续开发与架构维护指南)

---

## 1. 项目定位与层叠修改机制（Overlay Pattern）

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

## 2. 构建系统与编译配置覆盖（`cmake/` & `scripts/`）

Mixar 对 Blender 原生的 CMake 依赖和打包配置进行了针对性重写，重点集中在编译速度、CUDA/OptiX 支持和运行环境解耦上。

### 2.1 `cmake/mixar_overrides.cmake`
该文件在 Blender 执行 `project()` 调用前被注入，强制覆盖了多项 CMake 缓存变量：
- **编译缓存优化**：检测系统 `PATH`。若存在 `sccache`，在 macOS/Linux 下强制设置 `CMAKE_C_COMPILER_LAUNCHER` / `CMAKE_CXX_COMPILER_LAUNCHER`；在 Windows 下，强制启用 `WITH_WINDOWS_SCCACHE=ON` 触发 Blender 内置的 sccache 编译器发射机制。
- **Cycles 硬件加速**：
  - `WITH_CYCLES_DEVICE_CUDA=ON`、`WITH_CYCLES_CUDA_BINARIES=ON` 及 `WITH_CUDA_DYNLOAD=ON`，支持运行时动态加载 CUDA 驱动及渲染加速核心。
  - `WITH_CYCLES_DEVICE_OPTIX=ON`，支持 NVIDIA OptiX 光线追踪硬件加速。
- **编译器指令配置**：在 Windows/MSVC 下强制追加 C++ 编译选项 `/DWIN32 /D_WINDOWS /W3 /GR /EHsc`，保障第三方库与底层代码的稳健链接。

### 2.2 `src/CMakeLists.txt`
- 修改主程序名称 `project(Mixar)`，并将输出的可执行文件名和窗口标识符统一注册为 `mixar`。
- 修改 Windows 下的 AppID 宏定义为 `mixar.<major>.<minor>`（派生自 `VERSION` 文件），自定义 Friendly Name 为 `"Mixar 5.0"`，以便操作系统级别的高级通知和应用图标合并。

### 2.3 `scripts/generate_config.py` 与安全防护 (C4 Guard)
- 在构建期间调用，读取本地的 `.env` 并于运行环境中生成 `config/mixar.json`。
- **安全拦截防漏机制**：为了防止在生产环境（Prod）发布包中意外泄露开发旁路凭证（Dev Bypass Credentials），若 `MIXAR_ENV` 不为 `Dev`（如 `Prod` 或 `UAT`），且检测到设置了任何 `DEV_BYPASS_*` 变量，构建将抛出致命错误并强制中断。
- 同时在 Python 侧生成静态文件 `source/scripts/mixar/config/_build_env.py`，写死当前编译环境，并在 C++ 侧生成环境头文件 `mixar_env_config.h`。

---

## 3. C++ 底层修改与原生平台定制（GHOST & SystemPaths & GIL Safety）

Mixar 的窗口管理和多线程模型是重灾区，共包含约 150 个修改的 C++ 文件，其中以 **GHOST（通用便捷操作系统工具包）** 及 **Python 解释器 GIL 安全隔离** 最为底层和关键。

### 3.1 跨平台 GHOST 浮动窗口定制（Win32 subclassing & Cocoa AppKit）
为了在 3D 视图上方实现像 AI Chat Bubble 这样无边框、悬浮、支持拖拽、自带圆角与毛玻璃模糊背景的全新 UI，Mixar 在 GHOST 库（`src/intern/ghost/`）中扩展并导出了一套 `extern "C"` 专有 API，使得 Python Addon 可以通过 `ctypes` 完全控制操作系统级原生窗口：

- **`Mixar_WindowSetChromeless` / `Mixar_WindowSetBorderless`**：
  - **Win32 实现**：通过 `SetWindowLongPtr` 消除 `WS_CAPTION` / `WS_THICKFRAME`，并安装自定义窗口子类 `mixar_min_size_subclass_proc` 接管 `WM_GETMINMAXINFO` 和 `WM_NCHITTEST`，实现 chromeless 窗口仍能在边缘被鼠标调整大小。
  - **Cocoa 实现**：通过设置 `NSWindowStyleMaskFullSizeContentView` 并修改 `titlebarAppearsTransparent = YES`、`titleVisibility = NSWindowTitleHidden`，移除标准 macOS Title Bar，打造全浸式（Full-bleed）界面。
- **`Mixar_WindowSetCornerRadius` / `Mixar_WindowSetBlurBehind`**：
  - **Win32 实现**：利用 DWM 属性 `DWMWA_WINDOW_CORNER_PREFERENCE` 设为 `DWMWCP_ROUND` 实现 Win11 级窗口圆角，并通过 `DwmEnableBlurBehindWindow` 启用磨砂玻璃效果。
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
- 在 Blender 原生架构中，多线程 Python 插件（如多路并行 auth 请求、图片/Mesh 生成的后台下载轮询）极易触发内存破坏甚至在 `PyUnicode_New` 中段错误崩溃。原因在于原生的 `PyC_IsInterpreterActive()` 使用了 **进程全局的当前线程状态检测**。对于 Python <= 3.11，这会错误地返回“是否有 *任意* 线程持有了 GIL”，从而导致 C++ 原生操作符在释放 GIL 期间（如 `WM_operator_call_py` 前后），后台的 Python 子线程并发调用 Python C-API 抢占导致内存冲突。
- Mixar 对此进行了底层重构，将 `PyC_IsInterpreterActive()` 替换为 **线程局部级 GIL 校验** `PyGILState_Check()`，彻底消除了后台 Daemon 线程并发访问引起的启动段错误和随机崩溃。

---

## 4. 双阶段引导启动与模块加载（Bootstrap & Deferred UI Batching）

为了防止包含上千个 Python 文件的庞大 AI 模块群在 Blender 启动时造成阻塞、导致界面假死或加载菊花，Mixar 设计了优雅的 **双阶段异步/延时引导加载方案**。

入口文件位于：`src/scripts/startup/bootstrap/__init__.py`

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

## 5. 重点突破：Mixar UI 实现方式深度剖析

Mixar 在 3D 内容创作流程（DCC）中不仅保留了 Blender 传统的面板和视口，更融合了大量**原生 C++ 区域扩展、操作系统底层无缝悬浮窗、高性能 GPU 渲染技术以及完全由数据驱动的动态表单绘制**。以下是其 UI 实现的核心物理支柱：

```
                    ┌──────────────────────────────────────────────┐
                    │               Mixar 复合 UI 架构              │
                    └──────────────────────────────────────────────┘
                                           │
         ┌───────────────────┬─────────────┴───────┬──────────────────────┐
         ▼                   ▼                     ▼                      ▼
  【C++ 窗体/编辑器空间】 【OS 原生悬浮窗口】    【高性能 GPU 2D 渲染】   【Schema 数据驱动 UI】
   - space_mixie_chat     - Agent Bubble/Pill    - SpaceView3D 视口渲染  - 动态 WindowManager 挂载
   - 底部 Agent Strip     - ctypes API 穿透      - GPUBatch / 顶点着色   - JSON 参数 Schema 自动解析
   - RGN_TYPE_EXECUTE     - 父子窗口无迟滞跟随    - 呼吸发光 halo, 卡片   - 避免 Undo/Redo 数据坏档
```

### 5.1 C++ 原生编辑器空间与离屏渲染视口监视 (C++ Native Space & Offscreen Render)
除了基于 Python 的插件 UI 外，Mixar 还在底层直接扩展了 Blender 原生 C++ 窗口和区域：
- **原生编辑器空间（Custom Spaces）**：在 C++ 侧注册了 `space_mixie_chat` (AI 智能体对话)、`space_mixar_properties` (属性管理器)、`space_mixar_layers` (图层管理器) 等，深度参与 Blender 的 DNA / RNA 数据流以及 Area/Region 划分。
- **并行智能体 3D 视图底部监视栏（`agent_scene_strip`）**：
  - 原理：在 `editors/space_view3d/view3d_agent_strip.cc` 中向 3D 视口底部注册了一个专用的 `RGN_TYPE_EXECUTE` 原生区域。
  - 功能：当 blend 文件包含多个 Scene 且多个 AI 智能体在非激活场景后台并行工作时，该区域会发起 **Offscreen FBO 离屏渲染**。利用主场景的渲染管线去静默渲染其他后台场景。
  - 实现：利用 `TIMERNOTIFIER` 触发 0.1s 定时轮询，结合 `DEG_get_update_count`（检测非激活场景的依赖图变更），仅在有后台修改时才重新渲染后台场景，并以 OpenGL Texture Tiles（平铺片）的方式实时绘制在当前活动视口底栏，提供无缝的多场景监视，点击即可平滑热切换活动场景。

### 5.2 操作系统原生级悬浮窗口与 ctypes 穿透交互（GHOST Native Floating Windows）
这是 AI 智能体界面（Agent Bubble & Status Pill）最具特色的实现方式。
- **Python-ctypes 跨平台映射调用**：
  Mixar 在 `GHOST_SystemWin32.cc` 和 `GHOST_SystemCocoa.mm` 中通过 `extern "C"` 导出了三十余个 `Mixar_Window*` 底层窗体控制器。Python Addon 通过 `ctypes` 动态加载当前进程，动态获得这些 API 并完全接管 Agent 浮动窗体的外观与交互。
- **无边框（Chromeless）与毛玻璃透射（Blur Behind）**：
  - 在 Windows 上使用 DWM 并添加子类监听，在 macOS 上利用 AppKit 的 `NSVisualEffectView` HUD 材质和 layer 属性裁剪，直接实现抗锯齿圆角、无标题栏和系统原生磨砂玻璃。
- **极其平滑的父子窗体跟随机制（Parent-Child Tracking）**：
  - 如果使用 Python Timer 循环检测 Blender 位置并更新 Bubble，会出现严重的拖动延迟、卡顿和撕裂。
  - Win32 下在父窗口（Blender）中注入 subclass 拦截 `WM_MOVE` / `WM_SIZE`，在同一消息帧中通过 `PostMessage(MIXAR_WM_PARENT_TRACK_REPOSITION)` 触发子窗口即时重绘。
  - Cocoa 下通过 `NSNotificationCenter` 注册 AppKit 本地观察者（观察主 NSWindow 移动和缩放事件），保证悬浮 Bubble 移动时永远与 Blender 主窗口严格像素级同步对齐。

### 5.3 视口级高性能 GPU 顶点/着色器绘图（GPU Viewport Draw Handlers）
Mixar 在 3D 视口内绘制的大量高阶动效、遮罩和 Onboarding 指导步骤，并不是利用常规 UI 面板绘制的，而是直接通过 `SpaceView3D.draw_handler_add` 注入到 **`WINDOW` 区域的 `POST_PIXEL` 阶段**。
- **工作状态呼吸发光光晕（`agent_viewport_lock`）**：
  - 目的：在后台智能体工作期间，在 3D 视口边缘渲染软绿色的内发光呼吸光晕。
  - 实现：
    ```python
    # 1. 动态计算呼吸 Alpha (利用 time.monotonic 搭配 sine 曲线平滑过渡)
    phase = (time.monotonic() % HALO_PERIOD_S) / HALO_PERIOD_S
    s = 0.5 - 0.5 * math.cos(2.0 * math.pi * phase)
    alpha = HALO_ALPHA_MIN + (HALO_ALPHA_MAX - HALO_ALPHA_MIN) * s

    # 2. 从内置 Shader 获取顶点平滑着色器
    shader = gpu.shader.from_builtin('SMOOTH_COLOR')

    # 3. 构造 4 条边缘的渐变矩形顶点与颜色数据 (边缘不透明度为 alpha，向视口中心指数级递减)
    # 利用 batch_for_shader 离线编译缓冲，并在 POST_PIXEL 中直接递交 GPU 绘制
    batch = batch_for_shader(shader, 'TRIS', {"pos": verts, "color": colors}, indices=indices)
    gpu.state.blend_set('ALPHA')
    shader.bind()
    batch.draw(shader)
    ```
- **悬浮新手引导步骤卡片（`onboarding` card renderer）**：
  - 核心功能：高自由度的 Freeform 卡片排版，包括图标槽、标题阴影、垂直渐变背景按钮、步骤指示小圆点。
  - **垂直渐变圆角矩形（`_draw_rounded_rect_gradient`）**：
    为了给按钮等元素增加细腻的 Apple 风格渐变质感，重构了圆角矩形顶点扇（Triangle Fan）生成器，将顶点分成上半部分和下半部分，并根据相对于卡片边界的高度比（$t = \frac{v_y - y}{h}$）线性混合 `bottom_color` 和 `top_color`。利用内置的 `SMOOTH_COLOR` 渐变着色器交由 GPU 在光栅化阶段插值生成丝滑渐变。
  - **Faux-Bold 仿粗体抗锯齿文本绘制（`_draw_text_double`）**：
    由于内置字体的 BLF 库不支持通过修改加粗属性来改变粗细。为了在不增加字体多文件大小的前提下，实现卡片标题和按钮文字“加粗”以凸显层级，Mixar 在绘制关键标签时，使用 1 像素的横向偏移在同一帧绘制同一个 Blf 字符串两次：
    ```python
    blf.position(_FONT_ID, label_x, label_y, 0)
    blf.draw(_FONT_ID, label)
    blf.position(_FONT_ID, label_x + 1, label_y, 0)
    blf.draw(_FONT_ID, label)  # 巧妙 thicken 字形笔画
    ```
  - **多视口裁剪坐标重定位（`x_offset` / `y_offset`）**：
    当 Onboarding 卡片需要在窗口中心展示时，Blender 的多视口分割结构会导致绘图回调在每个 Area 单独被裁剪，出现卡片被分割线切断的 BUG。
    Mixar 在 `draw_card` 中引入了全局窗口坐标偏移，绘制回调将全局 Window 坐标投影在各 Area 坐标系中并进行 `x_offset` / `y_offset` 负向补偿，使得同一个绘制命令可以无障碍地跨视口边界渲染。

### 5.4 动态数据库 Schema 驱动参数表单（Dynamic Form Engine）
为了避免每次后台增加 AI 模型或调整算法滑块时，前端都必须重新排版、发版的低效循环，Mixar 开发了完全由参数 schema 驱动的 UI 生成引擎（位于 `modules/common/generation_params/`）：
- **全动态属性挂载**：
  在 Blender 中，如果直接修改 `bpy.types.Scene` 属性组，Undo/Redo 的历史回滚很容易让这些内存结构崩溃、损坏 `.blend` 文件。Mixar 的 UI 引擎将所有 schema 生成的参数动态定义在 **`bpy.types.WindowManager`** 下（通过运行时 main 线程的 Timer 定时调度重组）。WindowManager 上的临时属性不保存在 `.blend` 中，不进入 Undo 历史，保证了数据纯净与运行时稳定性。
- **Schema 自动构建解析与 Widget 渲染（`core/engine.py`）**：
  参数 Schema 支持定义类型（int, float, enum, bool, text）、默认值、滑块范围，以及高级排版约束如 `order`（布局排序）、`group`（折叠组）、`visible_if`（条件显示联动）。
  在 UI 绘制时，通过 `draw_service_params()` 面板绘制器，根据接收到的 catalog 参数 schema，逐个反射读取对应的 Widget 类型并渲染出 Blender 交互滑块或下拉列表，用户修改后通过 `collect_params()` 自动化过滤隐藏项并输出规范的 snake_case Payload。

---

## 6. 核心 Python 插件模块代码级拆解（`src/scripts/mixar/modules/`）

这是 Mixar 自定义 Python 业务层的大本营，承载了所有核心特性。

- **`paint` (核心图层绘画模块)**：基于 Blender 节点树 ShaderNodeTree 实现，每个图层为一封装的节点组。深度结合 C++ 重写其底层像素绘画内核，保障高清晰度纹理渲染时的极致响应。
- **`space_mixie_chat` (AI 智能体对话)**：封装了流式生成回复、Markdown 与思考步骤排版组件，并在后台提供了超时监控、异常恢复、对主窗口无干扰的沙盒 Headless 沙盒执行管理器。
- **`moodboard` (情感板与多维生成面板)**：包含剪贴板监控服务。整合了多项 AI 生成面板，目前已完全重构为从统一的 `Generation Catalog` 中读取模型列表和参数表单。
- **`onboarding` (功能引导 tour)**：管理 GPU 绘制的圆角卡片流，通过获取 `region.view2d.tot_rect` 动态捕捉并聚焦特定工具栏卡片位置。
- **`operation_history` (操作日志与动作搜集)**：使用 Timer 监视系统依赖图和 userop，实时记录手动操作和 agent 执行的 Python 沙盒脚本，提供非对称的操作自愈对齐。
- **`uv_editor` (高级双空间 UV 编辑器)**：为 UV 空间定制了 mutually exclusive tool panels，支持 toolbar 在窄屏自适应展开。
- **`common` (通用通信、自更新与任务调度中枢)**：封装了 HTTP 客户端、自更新强制推送Toast，以及核心的 `job_queue` 体系。
- **`auth` (认证鉴权)**：支持 OAuth PKCE 双向密钥链存储（Win Credential Manager / MacOS Keychain）。

---

## 7. 关键交互模式与全系统设计模式（Patterns）

### 7.1 统一任务异步队列调度架构（Unified Job Queue）
为了解决各个 AI 特性独自轮询、线程竞争和数据丢失的问题，Mixar 设计了 **Unified Job Queue (统一任务队列)** 系统：
- **统一契约接口**：通过 `POST /job-queue/jobs` 统一下发，返回任务 ID 后启动 `GET /job-queue/jobs/{id}` 高效轮询。
- **Generic Jobs 整合**：
  - `AsyncGLBJob`：自动请求、多线程下载 GLB，并在导入成功后触发 staging 重命名与 unwrap。
  - `SyncImageJob`：处理快速图像生成，支持图像异步落盘和 Moodboard 画布自动加载。

### 7.2 主子进程沙盒生命周期守护（Sandbox Supervision & Parent Liveness Check）
- 沙盒子进程在 Blender 原生命令行参数上追加了 `--background -noaudio --python headless_main.py`。
- **生命周期断链守护**：
  沙盒启动后，子进程通过 `MIXAR_SANDBOX_PARENT_PID` 捕捉父进程状态。在 Windows 环境下，通过 Win32 API 检索进程树，一旦父进程被强制关闭或退出，子进程会立刻自毁，确保不遗留任何僵尸 Blender 实例耗尽用户 GPU。

---

## 8. 后续开发与架构维护指南

在后续对 Mixar 的升级迭代、错误排查和功能开发中，请严格遵守以下核心守则：

### 8.1 UI 开发与文件物理红线
- **禁止修改编译产物**：切勿直接在 `source/`、`build/` 目录下对代码进行就地编辑。它们均属于编译中间目录。所有修改必须落在宿主目录 `src/` 中，然后通过 `make build` 进行 Overlay 合并与重新生成。
- **代码物理长度限制**：为了保障极致的运行性能和可读性，任何 Python/C++ 源文件的长度均**不得超过 500 行**。复杂业务逻辑应进行垂直拆分、下沉至组件或通用工具包（`modules/common/utils/`）中。

### 8.2 UI 开发加载守则
- **优先使用 WindowManager 进行临时属性挂载**：
  编写新的 UI 功能组件时，如非必要，禁止在 `bpy.types.Scene` 挂载临时、交互性质的状态变量，应当使用 `bpy.types.WindowManager` 属性组，避免坏档或 undo 引起崩溃。
- **遵守时序加载规则**：
  新增的 Panel、Menu 等绘制类必须放在子模块的 `ui/` 目录下，交由 deferred UI bootstrap 系统延时分批注册。严禁在同步启动阶段（`bootstrap/` 下的脚本或 `__init__.py` 的顶层）直接载入含有 UI 绘制的代码，避免破坏 4ms 帧预算屏障导致出现界面启动菊花或 Optype 空指针崩溃。

### 8.3 视口渲染安全性限制
- 编写任何 `POST_PIXEL` 绘图回调函数时，必须使用完整的 `try-except Exception` 结构包裹所有 GPU 指令和 BLF 命令。由于视口绘制回调属于每一帧重复执行的高频逻辑，任何未捕获的 Python 异常都会直接导致 Blender 强行断开并解绑 draw handler，严重时还会直接引发 OpenGL 上下文死锁崩溃。

---

*本报告是为 Mixar 开发与维护量身打造的高级技术文档。遵循上述设计思想与操作规范，将极大提高开发效能，并确保产品的架构稳定与技术演进优势。*
