# FastUMI 机载相机使用说明

本仓库包含与 **机载相机** 相关的采集与标定工具，分为**视频流采集**和**相机标定**两部分。

---

## 一、SDK 安装（必读）

本仓库中的**视频流采集**（`readimage.py`）和**相机标定**（`demo-api` / `pipe_srv`）均依赖相机驱动与运行环境。若尚未安装 **FastUMI Hardware SDK**，系统无法正确识别相机（如 `/dev/video0` 不可用或标定程序无法工作）。**请先按下列步骤完成 SDK 安装。**

### 1.1 什么是 FastUMI Hardware SDK？

FastUMI Hardware SDK 是 FastUMI 相机官方提供的硬件驱动与接口库，用于在 Ubuntu 20.04 下识别相机、提供 V4L2 设备节点，并为标定等高级功能提供 xvsdk 环境。未安装 SDK 时，本仓库中的脚本与参数识别工具将无法正常使用。

### 1.2 系统要求

- **操作系统**：**Ubuntu 20.04**（官方指南以该版本为准）。

### 1.3 安装步骤（简要）

- **SDK 仓库地址**：<https://github.com/FastUMIData/FastUMI_Hardware_SDK.git>  
- **重要**：仓库内带有完整安装文档。这里不重复贴具体命令，**请严格按照仓库中安装说明完成安装**。

安装完成后，建议做一次基本验证：

- 能正常启动官方示例/launch（例如 `roslaunch xv_sdk xv_sdk.launch`）
- 系统能识别到设备（见下节 `lsusb`）

### 1.4 确认设备已识别

安装完成后，连接相机，在终端执行：

```bash
lsusb
```

若能看到与相机相关的设备信息，说明驱动与连接正常。随后方可使用本仓库的 `readimage.py` 或标定工具。

---

## 二、目录结构

```
FastUMI_Camera/
├── README.md                          # 本说明文档
├── FastUMI_Camera_Steam/              # 相机视频流采集
│   └── readimage.py                   # 基于 V4L2 的实时采集脚本
└── FastUMI_Camera_Calibration-master/ # 相机标定工具
    ├── readme.txt                     # 标定简要说明
    ├── rgb_intrinsic_SEUCM.txt         # RGB 内参文件（SEUCM）
    ├── demo-api                       # 标定 API 演示程序（可执行文件）
    └── pipe_srv                       # 管道服务端（可执行文件）
```

---

## 三、FastUMI_Camera_Steam（视频流采集）

### 功能说明

`readimage.py` 通过 **V4L2** 打开 Linux 下的摄像头设备，以 **YU12（I420）** 原始格式采集帧，在内存中转换为 BGR 后用 OpenCV 窗口实时显示。适用于需要低延迟、可控分辨率和帧率的场景。

### 主要参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `DEV` | `0` | 设备号，对应 `/dev/video0`。若使用其他设备，改为 1、2 等。 |
| `W` | `1280` | 采集画面宽度（像素）。 |
| `H` | `1280` | 采集画面高度（像素）。 |
| `FPS` | `100` | 目标帧率。若设备不支持 100fps，可改为 `60` 等。 |

### 技术要点

- **像素格式**：驱动端使用 `YU12`（即 I420），减少 CPU 转换、降低延迟。
- **不转 RGB**：`cv2.CAP_PROP_CONVERT_RGB, 0` 关闭 OpenCV 自动转 RGB，由脚本内手动做 YUV→BGR。
- **缓冲**：尝试设置 `CAP_PROP_BUFFERSIZE` 为 1，减少延迟（部分驱动可能忽略）。
- **帧数据**：每帧为 `(H*3//2, W)` 的 I420 平面数据，用 `cv2.cvtColor(..., cv2.COLOR_YUV2BGR_I420)` 转为 BGR。

### 依赖

- Python 3
- OpenCV（`cv2`）
- NumPy
- Ubuntu 20.04 系统，且存在 `/dev/video*` 设备（本脚本面向 V4L2）

### 使用方法（即使用指南中的「启动此脚本」→「运行显示」）

1. 确认相机已连接，并已完成 **SDK 安装**（见上文**第一节**），设备已被识别（`lsusb` 可见）。
2. 确认有访问 `/dev/video0` 的权限（如将用户加入 `video` 组）。
3. 进入目录并运行：

```bash
cd FastUMI_Camera_Steam
python readimage.py
```

4. 窗口会显示实时画面，按 **`q`** 退出。
5. 若无法打开设备，检查：
   - 设备是否存在：`ls /dev/video*`
   - 权限或改用 `sudo` 测试
   - 若 100fps 无法打开，在脚本中将 `FPS` 改为 `60` 再试。

---

## 四、FastUMI_Camera_Calibration-master（相机标定）

### 功能说明

用于在 **xvsdk** 环境下对 FastUMI 相机进行标定。通过 **pipe_srv**（管道服务）与 **demo-api**（标定 API 演示）配合，在终端输入指令即可获取 RGB 标定参数。

### 前置条件

- 已安装并配置好 **xvsdk** 环境。

### 目录内文件说明

| 文件/程序 | 说明 |
|-----------|------|
| `demo-api` | 标定 API 演示程序，运行后会通过管道接收指令并输出 RGB 标定参数。 |
| `pipe_srv` | 管道服务端，接收用户输入的指令并转发给 demo-api。 |
| `readme.txt` | 标定步骤的简要说明。 |
| `rgb_intrinsic_SEUCM.txt` | RGB 内参文件（SEUCM）。 |

### 使用方法

1. **进入标定目录**

```bash
cd FastUMI_Camera_Calibration-master
```

2. **启动两个程序（需要两个终端）**

   - **终端 1**：启动标定 API 演示程序  
     ```bash
     ./demo-api
     ```
   - **终端 2**：启动管道服务  
     ```bash
     ./pipe_srv
     ```

3. **在 pipe_srv 所在终端输入指令**

   输入：**`1-0-37`**（按实际标定流程或文档要求可调整）。

4. **查看结果**

   在 **demo-api** 所在的终端中会打印出 **RGB 标定参数**。

### 使用注意

- 必须先启动 `demo-api`，再启动 `pipe_srv`，以便管道连接正常。
- 若提示权限不足，可为可执行文件添加执行权限：  
  `chmod +x demo-api pipe_srv`
- 具体标定步骤、参数含义及更多指令格式请以 xvsdk 和厂商标定文档为准。

---

## 常见问题

1. **lsusb 看不到相机设备**  
   检查数据线是否插在**最小的那个接口**（不是大接口）；确认转接器、USB 线连接牢固；换 USB 口或换线再试。

2. **readimage.py 报错“无法打开 /dev/video0”**  
   检查设备节点是否存在（`ls /dev/video*`）、用户是否有权限，或修改脚本中的 `DEV` 使用其他 `/dev/video*`。若尚未安装 SDK，请先按「一、SDK 安装」完成安装。

3. **帧率 100 无法打开**  
   在 `readimage.py` 中将 `FPS = 100` 改为 `FPS = 60`（使用指南中注明：若 100 启动不了可改为 60）。

4. **标定程序无输出**  
   确认 xvsdk 已正确安装，且先运行 `./demo-api` 再运行 `./pipe_srv`，并在 pipe_srv 终端输入指令（如 `1-0-37`）。

---

## 依赖汇总

| 组件 | 依赖 |
|------|------|
| 整体环境（按使用指南） | Ubuntu 20.04、FastUMI Hardware SDK（含 ROS1 Noetic）、设备通过 lsusb 可见 |
| readimage.py | Python 3、OpenCV (cv2)、NumPy  |


更多细节请参考：**FastUMI Hardware SDK** 仓库内安装文档，以及各子目录内说明文档。
