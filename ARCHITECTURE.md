# Clean My PC Wechat 架构分析文档

这是一个使用 Python 和 PyQt5 开发的 Windows 桌面端工具项目，名为 **Clean My PC Wechat**。它的主要功能是自动识别并清理 PC 端微信长期积累的图片、视频、文件等缓存，以释放磁盘空间。

## 1. 整体技术栈与项目结构
*   **开发语言**：Python 3
*   **GUI 框架**：PyQt5（结合 Qt Designer 生成的 `.ui` 文件分离界面与逻辑）
*   **核心第三方库**：`Send2Trash`（用于将文件移入回收站）、`python_dateutil`（时间计算）
*   **打包工具**：PyInstaller（将应用打包为独立的 Windows `.exe` 可执行文件）

**目录结构解析：**
*   `main.py`：程序的入口文件，包含主窗口逻辑、配置窗口逻辑以及文件扫描的核心业务代码。
*   `utils/`：存放独立的工具模块和线程类（如微信路径自动识别、多线程删除等）。
*   `images/`：存放静态资源，包括应用的图标、SVG 矢量图，以及 Qt Designer 构建的界面布局文件（`main.ui`、`config.ui`）。
*   `requirements.txt`：项目依赖列表。

## 2. 核心模块与工作流程

架构上分为四个主要逻辑层：**UI 交互层**、**路径识别层**、**文件筛选层** 和 **并发清理层**。

### UI 交互层 (View & Controller)
由 `main.py` 中的两个核心类构成：
*   **`MainWindow`**：主窗口类。负责展示应用的清理进度、响应用户的清理操作。它通过重写鼠标事件（`mousePressEvent`、`mouseMoveEvent`）实现了无边框窗口的拖动，并使用 `QPropertyAnimation` 实现了窗口的淡入淡出动效。
*   **`ConfigWindow`**：配置窗口类。允许用户自定义微信数据目录，下拉切换多个微信账号（`wx_id`），并分别设置清理类型（图片、视频、文件等）和保留天数（如保留最近 365 天）。配置会持久化保存到本地的 `config.json` 文件中。

### 路径识别层 (Path Discovery)
由 `utils/selectVersion.py` 提供支持：
*   负责在应用首次启动时，自动寻找微信数据存放目录。
*   它内置了常规 PC 版、Win10 商店版、UWP 版的默认路径字典，同时通过 `winreg` 库读取 Windows 注册表（`software\tencent\wechat` -> `FileSavePath`），以兼容用户自定义存放路径的场景。

### 文件筛选层 (File Filtering)
位于 `main.py` 的 `getPathFileNum` 与相关方法中：
*   程序会根据不同的微信目录结构（如 `Attachment`、`FileStorage/Cache`、`Video` 等）逐一遍历。
*   利用 `os.path.getmtime()` 获取文件最后修改时间，计算与当前时间的差值（天数）。
*   根据 `config.json` 中的用户配置，将超期的目标文件和空文件夹路径分别存入 `file_list` 和 `dir_list` 待清理列表中。

### 并发清理层 (Asynchronous Execution)
由 `utils/multiDeleteThread.py` 处理：
*   **UI 防假死**：文件删除是耗时的 I/O 操作，项目使用了 PyQt 的 `QThread` 将删除操作从主线程抽离，防止界面卡顿。
*   **安全删除**：调用了 `send2trash(file_path)` 方法，将文件放入系统回收站而非直接 `os.remove` 彻底删除，给用户留下了恢复误删文件的余地。
*   **进度同步**：在线程中使用 `QMutex` 互斥锁保护计数器，并通过 `pyqtSignal` 将清理进度实时发送回主线程，更新主界面的进度条。

## 3. 架构设计亮点
*   **多账号独立配置机制**：程序遍历 WeChat Files 目录下的所有用户 ID 文件夹（排除公共文件夹），并以 JSON 格式独立管理每个账号的清理策略，适合家庭公用电脑或多开用户的场景。
*   **UI 表现层与逻辑层适度分离**：界面排版由 `.ui` 静态文件定义，`main.py` 仅使用 `loadUi` 进行动态加载与事件绑定，减少了界面代码对业务逻辑的干扰。
*   **防御性编程（安全优先）**：无论是过滤注册表空值，还是回收站安全删除策略，在涉及到用户本地重要数据时都体现了较高的容错和数据安全意识。