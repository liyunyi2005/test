## 项目概览

**本工程是一个基于 LVGL v8.2 + SDL2 的“手表 UI”PC 模拟器工程**，用于在 PC 上开发和调试手表界面，无需真实硬件即可完成大部分 UI 和交互逻辑的设计与验证。  

工程整体思路来自 `OV-Watch` 项目（见 `user_test/version.h` 中的版本和作者信息），在其基础上做了如下封装和集成：

- **LVGL 图形库 + 驱动层**：完整集成 `lvgl` 源码与 `lv_drivers`，通过 SDL2 在 PC 窗口中模拟 240×280 的手表屏幕。
- **页面管理框架**：在 `user_test/Func` 中实现了 `PageManager`，提供页面栈、前进/后退、返回首页等导航能力。
- **发布订阅消息总线**：`pubsub` 模块实现简单的发布-订阅机制，用于如“键盘事件触发页面返回”等解耦交互。
- **硬件抽象层 (HAL/Middleware)**：`HWDataAccess` 将 RTC、IMU、BLE、电源、背光等硬件访问统一封装，方便在 MCU 与 PC 模拟之间切换。
- **完整多页面 UI**：`user_test/GUI_App/Screens` 下包含首页、菜单、计步、心率、SPO2、环境信息、日历、计时、游戏（2048/记忆翻牌）等诸多页面，UI 由 SquareLine Studio 生成并手工扩展。

在 PC 上运行时，程序入口位于 `main.c`，通过 SDL2 打开一个窗口，初始化 LVGL 后调用 `ui_init()` 构建并运行整套手表 UI。

---

### 运行效果简要说明

程序启动后会：

- 初始化 LVGL (`lv_init`) 与 SDL2 显示和输入驱动 (`sdl_init` 等)，创建 240×280 的模拟显示窗口并设置主题、鼠标光标等（见 `main.c` 中 `hal_init()`）。
- 通过 `Pages_init()` 将首页 (`Page_Home`) 推入页面栈并显示为当前页面。
- UI 采用 **页面栈** 组织结构：  
  - 底部为 `HomePage`（表盘或主界面）。  
  - 通过菜单、按钮等可以跳转到 `MenuPage`、`EnvPage`、`HRPage`、`SPO2Page`、`TimerPage`、`Game2048Page` 等多个功能界面。  
  - `Page_Back()` 实现返回上一级页面，`Page_Back_Bottom()` 返回栈底首页。
- 通过 `SDL_KeyBoard_Publisher` + 订阅回调 `SDL_KeyBoard_Subscriber()` 实现键盘事件触发页面返回（当前示例中直接调用 `Page_Back()`）。
- `main()` 中主循环持续调用 `lv_timer_handler()`，驱动 LVGL 完成刷新与定时任务。

---

### 工程结构说明

- **根目录**
  - `main.c`：程序入口，初始化 LVGL 与 SDL2，调用 `ui_init()`，并在循环中调用 `lv_timer_handler()`。
  - `CMakeLists.txt`：推荐的跨平台构建配置，生成可执行文件 `bin/main`。
  - `Makefile`：旧的基于 Make 的构建脚本（已标注为 **deprecated**，推荐使用 CMake）。
  - `Dockerfile`：基于 Ubuntu 18.04 的 Docker 构建环境说明，可用于在容器中构建和运行模拟器。
  - `lv_conf.h`：LVGL v8.2 的主配置文件，已启用大量组件、字体和 Demo，用于 PC 端开发调试。
  - `lv_drv_conf.h`：LVGL 驱动层配置文件，本工程启用的是 **SDL 驱动**，分辨率为 240×280 (`SDL_HOR_RES` / `SDL_VER_RES`)。
  - `confdef.txt`：LVGL 配置模板，仅用于记录推荐配置（如 `LV_MEM_SIZE`、`LV_COLOR_DEPTH` 等），实际使用以 `lv_conf.h` 为准。
  - `mouse_cursor_icon.c`：自定义鼠标光标图像资源，`hal_init()` 中通过 `LV_IMG_DECLARE(mouse_cursor_icon)` 加载并显示。
  - `SDL2.dll`：Windows 下 SDL2 动态库，构建后会被拷贝到 `bin/` 目录以供运行。

- **`lvgl/`**  
  LVGL 官方源码与示例：
  - `src/`：LVGL 核心、部件、绘制、字体等源码。
  - `examples/`、`demos/`：官方示例和演示程序（工程中可按需调用，如 `lv_demo_widgets()` 等）。
  - `docs/`：LVGL 官方文档源码（Sphinx），与本工程逻辑无直接耦合。

- **`lv_drivers/`**  
  LVGL 官方驱动集合，本工程主要使用：
  - `sdl/`：`sdl.c`、`sdl_gpu.c` 等，提供 SDL 显示与输入驱动（`sdl_init()`、`sdl_display_flush()`、`sdl_mouse_read()` 等接口）。
  - 其余如 `wayland/`、`win32drv/`、`display/` 下的多种控制器驱动当前未启用，面向其他平台可复用。

- **`user_test/`（项目应用层核心）**
  - `version.h`：项目版本与作者信息（如 `VERSION_MAJOR / MINOR / PATCH`，作者英/中文名、GitHub 链接等）。
  - **`GUI_App/`**：UI 工程（多由 SquareLine Studio 生成）
    - `ui.c` / `ui.h`：UI 初始化入口 `ui_init()` 声明与实现，内部逻辑包括：
      - 初始化 `SDL_KeyBoard_Publisher` 并订阅 `SDL_KeyBoard_Subscriber()`。  
      - 设置默认主题 `lv_theme_default_init()`。  
      - 调用 `Pages_init()` 初始化页面栈与首页。  
      - 创建一个 1 秒周期的 `lv_timer`，用于打印页面栈状态等调试信息。
    - `ui_helpers.c` / `ui_helpers.h`：SquareLine 自带 UI 辅助函数，比如：
      - 通用属性设置函数（`_ui_bar_set_property`、`_ui_label_set_property` 等）。
      - 控件数值与文本联动（`_ui_slider_set_text_value` 等）。
      - 屏幕切换与简单动画（`_ui_screen_change`、`_ui_anim_callback_*` 系列）。
    - `Screens/Inc` / `Screens/Src`：每个 UI 界面的头文件与实现：
      - 如 `ui_HomePage.[ch]`、`ui_MenuPage.[ch]`、`ui_EnvPage.[ch]`、`ui_HRPage.[ch]`、`ui_SPO2Page.[ch]`、`ui_Game2048Page.[ch]` 等。
      - 每个页面通常会导出一个 `Page_t` 结构（如 `Page_Home`、`Page_Menu`），并实现对应的 `init()` / `deinit()` 与 `lv_obj_t *` 根对象。
    - `Fonts/`、`IMGs/`：UI 使用的字体文件与图片资源。
  - **`Func/`**：功能与框架逻辑
    - `Inc/PageManager.h` & `Src/PageManager.c`：页面栈与导航管理：
      - `Page_t`：包含 `init`、`deinit` 函数指针和 `lv_obj_t **page_obj`。
      - `PageStack_t`：页面指针数组 + 栈顶索引，栈深由 `MAX_DEPTH` 定义（当前为 6）。
      - 提供接口：
        - `Pages_init()`：初始化栈，推入 `Page_Home` 并加载其屏幕。
        - `Page_Load(Page_t *newPage)`：将新页面压栈并以动画方式切入。
        - `Page_Back()`：弹出当前页面并返回上一页，如栈空则自动推入 `Home + Menu` 并切到 `MenuPage`。
        - `Page_Back_Bottom()`：弹到栈底首页。
        - `Page_Get_NowPage()`：获取当前页面（栈顶）。
    - `Inc/pubsub.h` & `Src/pubsub.c`：简单发布-订阅消息总线：
      - `PubSub_Message_t`：包含 `id` 与 `data[256]` 的消息结构体。
      - `PubSub_Publisher_t` 内含单向链表 `SubscriberNode`，每个节点是一个函数指针 `Subscriber`。
      - 提供接口：
        - `Publisher_init()` / `Publisher_subscribe()` / `Publisher_unsubscribe()` / `Publisher_publish()`。
      - 全局变量 `SDL_KeyBoard_Publisher` 用于处理 SDL 键盘相关的事件分发，`ui.c` 中订阅的 `SDL_KeyBoard_Subscriber()` 在收到消息后调用 `Page_Back()`。
    - `Inc/HWDataAccess.h` & `Src/HWDataAccess.c`：硬件抽象中间层：
      - 用 `HW_USE_RTC`、`HW_USE_IMU`、`HW_USE_BLE`、`HW_USE_BAT`、`HW_USE_LCD` 等宏控制是否直接访问底层 HAL/BSP。
      - 在 **PC 仿真** 场景下，这些宏通常关闭，函数会返回假定值（例如 RTC 时间固定为 2024-06-23 11:59:55），便于在没有硬件的情况下运行 UI。
      - 主要接口包括：`HW_RTC_Get_TimeDate()`、`HW_RTC_Set_Time()`、`HW_RTC_Set_Date()`、`HW_MPU_Wrist_Enable/Disable()`、`HW_BLE_Enable/Disable()`、`HW_Power_Shutdown()`、`HW_LCD_Set_Light()` 等。
    - `Inc/StrCalculate.h` & `Src/StrCalculate.c`：字符串表达式计算模块：
      - 实现一个简单的中缀表达式求值器，支持 `+ - * /` 与小数。
      - 使用 `NumStack` 和 `SymStack` 两个栈来处理数字和符号，`NumSymSeparate()` 进行字符串拆分和优先级处理，`StrCalculate()` 进行最终计算。
      - 该模块可被如“计算器页面”等 UI 所复用。

- **`bin/`、`build/`、`Debug/` 等**
  - `bin/main.exe`：在 Windows 下通过 CMake/Make 构建产物。
  - `SDL2.dll`：运行时依赖的 SDL2 动态库。
  - `build/`：CMake 生成的中间文件（`CMakeCache.txt`、`Makefile`、`*.dir` 等）。
  - `Debug/`：Eclipse/Make 方式构建时的中间产物和配置，不是必须项。

---

### 构建与运行

#### 1. 直接运行已构建好的二进制（Windows 推荐）

在仓库根目录已经有：

- `bin/main.exe`
- `bin/SDL2.dll`（由 CMake `file(COPY SDL2.dll DESTINATION ../bin)` 拷贝而来）

你可以直接：

```bash
cd bin
main.exe
```

如果系统缺少 VC 运行库或 SDL 依赖，请先安装对应的运行环境（如 SDL2 Runtime 或 MSVC 运行库）。

#### 2. 使用 CMake 构建（推荐方式）

工程已提供标准 `CMakeLists.txt`，使用方式如下：

```bash
cd build        # 或自行创建一个空目录
cmake ..        # 生成工程（Windows 下可用 MinGW 或 VS 编译器）
cmake --build . --config Release --parallel
```

构建完成后：

- 可执行文件位于 `bin/main` 或 `bin/main.exe`。
- `SDL2.dll` 会自动被拷贝到 `bin/` 目录。

运行：

```bash
cd ../bin
./main      # Linux/macOS
main.exe    # Windows
```

**注意事项：**

- Windows 下请保证可以找到 SDL2 开发包或使用仓库中提供的 `SDL2.dll`；  
  CMake 中通过 `find_package(SDL2 REQUIRED SDL2)` 查找 SDL2，需要正确配置 `SDL2_DIR` 或将 SDL2 安装到常规搜索路径。
- 编译时已启用 `LV_CONF_INCLUDE_SIMPLE` 宏，确保编译器 include path 中包含 `lv_conf.h` 所在的根目录（`CMakeLists.txt` 中已通过 `INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR})` 处理）。

#### 3. 使用 Makefile 构建（不推荐，仅供参考）

根目录的 `Makefile` 来自早期 LVGL PC 模拟器工程，当前已在文件顶部明确标注：

> Using Make to build this project is deprecated, please switch to CMake

如果你在 Linux 环境，并已安装 `gcc`、`make`、SDL2 开发包与 `pkg-config`，仍可尝试：

```bash
make -j4
./demo
```

但由于工程已明显向 CMake 迁移，后续修改不再保证 Makefile 的完整性，**不建议在新环境上继续使用**。

#### 4. 使用 Docker 运行（Linux/macOS）

`Dockerfile` 提供了一个 Ubuntu 18.04 的示例环境，主要操作步骤：

1. 构建镜像：

   ```bash
   docker build -t lvgl_simulator .
   ```

2. 运行容器（仅命令行，GUI 视宿主系统配置而定）：

   ```bash
   docker run lvgl_simulator
   ```

如需在宿主机上显示 GUI，需要结合宿主系统的图形转发机制：

- **Linux + X11** (示例)：

  ```bash
  xhost +
  docker run -e DISPLAY=$DISPLAY -v /tmp/.X11-unix/:/tmp/.X11-unix:ro -t lvgl_simulator
  ```

- **macOS**：可参考 `Dockerfile` 注释中的链接，通过 XQuartz + 端口转发等方式实现（该方案与本工程逻辑无关，仅为环境配置问题）。

---

### LVGL 配置与驱动说明

- **核心配置文件：`lv_conf.h`**
  - `LV_COLOR_DEPTH` 设置为 16，`LV_COLOR_16_SWAP` 为 0，与 SquareLine Studio 生成工程要求保持一致（`ui.c` 中有编译期检查）。
  - 配置了较大的 `LV_MEM_SIZE`（128 KB）与复杂绘制、渐变、抗锯齿等高级特性，适合 PC 端模拟复杂 UI。
  - 启用了绝大多数常用组件和扩展部件（如 `LV_USE_LABEL`、`LV_USE_BTN`、`LV_USE_IMG`、`LV_USE_LIST`、`LV_USE_TABVIEW`、`LV_USE_CHART` 等）。
  - 启用了 `LV_BUILD_EXAMPLES` 与多种 Demo（`LV_USE_DEMO_WIDGETS`、`LV_USE_DEMO_BENCHMARK`、`LV_USE_DEMO_STRESS`、`LV_USE_DEMO_MUSIC` 等），方便快速验证。
  - 文本编码使用 UTF-8 (`LV_TXT_ENC UTF8`)，便于使用中文。

- **驱动配置文件：`lv_drv_conf.h`**
  - 启用 `USE_SDL`，关闭了 `USE_MONITOR`、`USE_WIN32DRV`、`USE_WAYLAND` 等其他 PC 驱动。
  - 显示分辨率为 `SDL_HOR_RES 240`、`SDL_VER_RES 280`，缩放因子 `SDL_ZOOM 1`（如需在 PC 上放大窗口，可调整 `SDL_ZOOM` 值）。
  - 输入设备方面，本工程主要通过 SDL 驱动中的鼠标、键盘、滚轮接口实现：
    - `LV_INDEV_TYPE_POINTER` + `sdl_mouse_read()`（鼠标）
    - `LV_INDEV_TYPE_KEYPAD` + `sdl_keyboard_read()`（键盘）
    - `LV_INDEV_TYPE_ENCODER` + `sdl_mousewheel_read()`（滚轮）
  - `hal_init()` 中统一完成显示缓冲 (`lv_disp_draw_buf_t`) 与显示驱动 (`lv_disp_drv_t`) 注册，并将鼠标图像对象作为光标绑定到指针输入设备。

---

### 应用层架构与交互流程

- **入口流程**
  1. `main()` 中调用 `lv_init()` 完成 LVGL 核心初始化。
  2. 调用 `hal_init()`：
     - 初始化 SDL 窗口与输入。
     - 创建显示缓冲与注册显示驱动。
     - 设置默认主题（蓝红配色，可在此修改）。
     - 创建输入设备并绑定到 `lv_group`，以支持键盘/编码器导航。
     - 创建自定义鼠标光标。
  3. 调用 `ui_init()`：
     - 初始化键盘事件发布者 `SDL_KeyBoard_Publisher` 并订阅 `SDL_KeyBoard_Subscriber()`。
     - 重新设置主题（可覆盖 `hal_init()` 中主题设置）。
     - 调用 `Pages_init()` 初始化页面栈与首页。
     - 创建 1 秒周期的调试 `lv_timer`。
  4. 进入 while(1) 循环，周期性调用 `lv_timer_handler()` 完成 LVGL 调度。

- **页面与导航**
  - 每个页面都通过 `Page_t` 结构统一抽象，`PageManager` 模块负责栈式管理：
    - `Page_Load()`：压入栈顶并以动画方式切换。
    - `Page_Back()`：弹栈返回上一个页面，若弹空则自动回到 `Home + Menu` 场景。
    - `Page_Back_Bottom()`：一键返回栈底页（默认首页）。
  - 页面本身通常在各自 `ui_*.c` 中实现 UI 创建函数，并在 `Page_t` 中记录 `page_obj` 根对象指针。

- **发布订阅总线**
  - 使用链表维护订阅者列表，Publisher 广播消息时依次调用各个订阅函数。
  - 当前工程中示例用途：
    - `SDL_KeyBoard_Subscriber()`：收到消息后打印日志并调用 `Page_Back()`，用于模拟“按键返回上一页”的行为。
  - 后续可以拓展为：
    - 按消息 `id` 区分业务类型（如按键、通知、同步事件等）。
    - 拆分多个 Publisher（如 BLE 消息、传感器消息等）。

- **硬件抽象**
  - `HWDataAccess` 内所有函数都通过宏控制是否访问真实 HAL/BSP。
  - 在 PC 仿真模式下，可以简单地将这些宏置为 0，通过返回固定/模拟数据支撑 UI 逻辑。
  - 在 MCU 上移植时，只需在 `.h` 中启用对应宏并接入实际 HAL 即可，UI 层与 PageManager 基本无需改动。

---

### 二次开发与扩展指南

- **新增一个页面的大致步骤**
  1. 在 `user_test/GUI_App/Screens/Inc` 与 `Src` 下新增 `ui_MyPage.h` / `ui_MyPage.c`（可用 SquareLine 生成，或参考现有页面手写）。
  2. 在 `ui_MyPage.c` 中：
     - 创建根对象（通常是一个全屏 `lv_obj_t *`），设置布局与样式。
     - 实现 `static void MyPage_init(void)` 和 `static void MyPage_deinit(void)`。
     - 定义 `Page_t Page_MyPage = { MyPage_init, MyPage_deinit, &my_page_root_obj };` 并在头文件中 `extern` 声明。
  3. 在合适的位置通过 `Page_Load(&Page_MyPage);` 进行跳转（如在菜单按钮点击事件中）。

- **接入真实硬件数据**
  - 在 MCU 工程中复用本套 `user_test` 目录时：
    - 仅需：保证 LVGL 配置、字体、分辨率与 PC 端保持一致（或进行适配）。
    - 在 `HWDataAccess` 中：
      - 将 `HW_USE_RTC`、`HW_USE_IMU`、`HW_USE_BLE` 等宏置为 1。
      - 实现对应的 BSP/HAL 函数（如 `RTC_SetTime`、`KT6328_Enable`、`Power_DisEnable` 等）。
  - UI 层调用的始终是 `HW_*` 抽象接口，从而实现“一套 UI，PC + MCU 双端运行”。

- **适配不同分辨率或外观**
  - 调整 `lv_drv_conf.h` 中的 `SDL_HOR_RES` / `SDL_VER_RES`，以及 `lv_conf.h` 中的 `LV_DPI_DEF` 等参数。
  - 修改主题初始化参数（如默认调色板、暗黑模式开关 `LV_THEME_DEFAULT_DARK`）即可快速更换整体观感。

---

### 常见问题（FAQ）

- **Q：Windows 下找不到 SDL2 或链接失败怎么办？**  
  **A**：请确认已安装 SDL2 开发包，并在 CMake 中正确配置 `SDL2_DIR`；或直接使用仓库同级提供的 `SDL2.dll`，并在系统中安装对应的 VC 运行库。

- **Q：分辨率和 UI 在 PC 上太小怎么办？**  
  **A**：在 `lv_drv_conf.h` 中增大 `SDL_ZOOM`（例如改为 2 或 3），或在 SquareLine/LVGL 中整体按比例放大各控件尺寸。

- **Q：如何在 PC 上调试页面栈与导航逻辑？**  
  **A**：`ui.c` 中的 `main_timer()` 每秒会打印当前页面对象指针和 `PageStack.top`，你可以在此添加更详细的调试输出，或直接使用调试器在 `PageManager.c` 的关键函数中下断点。

- **Q：如何只使用 LVGL 官方 Demo 而不用当前表盘 UI？**  
  **A**：在 `main.c` 中将 `ui_init();` 注释掉，改为调用 `lv_demo_widgets();` 或其他 Demo 初始化函数即可（前提是在 `lv_conf.h` 中已启用对应 Demo）。  

---

如需进一步移植到 STM32 或其他 MCU，只需要保留本工程的 `user_test`（UI + PageManager + HWDataAccess）与相关资源，并在目标平台重用 LVGL 配置与驱动即可；PC 模拟这套工程可以持续作为“本地 UI 调试沙箱”使用。
