# FastUMI Onboard Camera User Guide

This repository contains data acquisition and calibration utilities for the **onboard camera**, organized into two main parts: **video stream capture** and **camera calibration**.

---

## 1. SDK Installation (Required)

Both the **video capture** script (`readimage.py`) and the **camera calibration** tools (`demo-api` / `pipe_srv`) in this repository depend on the camera driver and runtime environment. If the **FastUMI Hardware SDK** is not installed, the system will not be able to recognize the camera correctly (for example, `/dev/video0` may be unavailable, or the calibration utilities may fail to run). **Please complete the SDK installation before using this repository.**

### 1.1 What Is the FastUMI Hardware SDK?

The FastUMI Hardware SDK is the official hardware driver and API library for FastUMI cameras. It is used on Ubuntu 20.04 to:

- Make the camera visible to the system
- Provide V4L2 device nodes
- Provide the xvsdk environment for calibration and other advanced features

Without the SDK installed, the scripts and parameter/calibration tools in this repository will not work properly.

### 1.2 System Requirements

- **Operating system**: **Ubuntu 20.04** 

### 1.3 Installation Overview

- **SDK repository**: <https://github.com/lumos-open/FastUMI_Hardware_SDK.git>  
- **Important**: The repository contains complete, detailed installation instructions. This README intentionally does **not** repeat the exact commands. **Always follow the official installation guide in that repository.**

After installation, it is recommended to perform a basic verification:

- You can successfully launch an official example/launch file (for example, `roslaunch xv_sdk xv_sdk.launch`).
- The system can detect the device (see the `lsusb` check below).

### 1.4 Verifying Device Recognition

After the SDK is installed, connect the camera and run the following command in a terminal:

```bash
lsusb
```

If you can see an entry corresponding to the camera, the driver and connection are functioning correctly. You can then proceed to use `readimage.py` and the calibration utilities in this repository.

---

## 2. Directory Layout

```text
FastUMI_Camera/
├── README.md                          # Main usage guide (Chinese)
├── README_en.md                       # Main usage guide (English)
├── FastUMI_Camera_Steam/              # Video stream capture
│   └── readimage.py                   # V4L2-based real-time capture script
└── FastUMI_Camera_Calibration-master/ # Camera calibration tools
    ├── readme.txt                     # Brief calibration description
    ├── rgb_intrinsic_SEUCM.txt         # RGB intrinsic parameters (SEUCM)
    ├── demo-api                       # Calibration API demo (executable)
    └── pipe_srv                       # Pipe server (executable)
```

---

## 3. FastUMI_Camera_Steam (Video Stream Capture)

### 3.1 Overview

`readimage.py` uses **V4L2** to open the camera device on Linux, captures frames in the raw **YU12 (I420)** format, converts them to BGR in memory, and displays them in real time with an OpenCV window. It is suitable for scenarios that require low latency and configurable resolution and frame rate.

### 3.2 Main Parameters

| Parameter | Default | Description |
|----------|---------|-------------|
| `DEV`    | `0`     | Device index, corresponding to `/dev/video0`. Use 1, 2, etc. for other devices. |
| `W`      | `1280`  | Frame width in pixels. |
| `H`      | `1280`  | Frame height in pixels. |
| `FPS`    | `100`   | Target frame rate. If the device does not support 100 fps, change this to `60` or another supported value. |

### 3.3 Technical Notes

- **Pixel format**: The driver is configured to use `YU12` (I420), which reduces CPU overhead for color conversion and lowers latency.
- **No automatic RGB conversion**: The script sets `cv2.CAP_PROP_CONVERT_RGB` to `0` to disable OpenCV's built-in RGB conversion. Instead, it performs the YUV → BGR conversion explicitly in the code.
- **Buffer size**: The script attempts to set `CAP_PROP_BUFFERSIZE` to 1 to reduce latency (this may be ignored by some drivers).
- **Frame data layout**: Each frame is treated as an I420 planar buffer of shape `(H * 3 // 2, W)` and converted using `cv2.cvtColor(..., cv2.COLOR_YUV2BGR_I420)`.

### 3.4 Dependencies

- Python 3
- OpenCV (`cv2`)
- NumPy
- Ubuntu 20.04 with available `/dev/video*` devices (V4L2-based capture)

### 3.5 Usage (Corresponding to “Start This Script” → “Display Output” in the Chinese Guide)

1. Ensure the camera is connected and **the SDK has been installed** (see **Section 1** above), and confirm that the device is recognized (`lsusb`).
2. Confirm that the current user has permission to access `/dev/video0` (for example, by adding the user to the `video` group).
3. Run the following commands:

```bash
cd FastUMI_Camera_Steam
python readimage.py
```

4. A window will display the live video stream. Press **`q`** to exit.
5. If the script cannot open the device, check the following:
   - Whether the device node exists: `ls /dev/video*`
   - User permissions (or test with `sudo`)
   - If 100 fps fails to start, change `FPS = 100` to `FPS = 60` in the script and try again.

---

## 4. FastUMI_Camera_Calibration-master (Camera Calibration)

### 4.1 Overview

This directory contains utilities for calibrating the FastUMI camera in an **xvsdk** environment. Calibration is performed by running **`demo-api`** (calibration API demo) together with **`pipe_srv`** (pipe server). After starting both programs, you can send commands in a terminal to obtain RGB calibration parameters.

### 4.2 Prerequisites

- The **xvsdk** environment has been correctly installed and configured.

### 4.3 Files in This Directory

| File / Program | Description |
|----------------|-------------|
| `demo-api`     | Calibration API demo program. It receives commands through a pipe and prints RGB calibration parameters. |
| `pipe_srv`     | Pipe server that accepts user input and forwards commands to `demo-api`. |
| `readme.txt`   | Brief description of the calibration steps. |
| `rgb_intrinsic_SEUCM.txt` | RGB intrinsic parameters file (SEUCM). |

### 4.4 Usage

1. **Enter the calibration directory**

```bash
cd FastUMI_Camera_Calibration-master
```

2. **Start the two programs (in two separate terminals)**

   - **Terminal 1**: Start the calibration API demo

     ```bash
     ./demo-api
     ```

   - **Terminal 2**: Start the pipe server

     ```bash
     ./pipe_srv
     ```

3. **Enter the command in the terminal running `pipe_srv`**

   Type the following (or another command as specified by your calibration workflow or documentation):

   ```text
   1-0-37
   ```

4. **View the results**

   The terminal running **`demo-api`** will print out the **RGB calibration parameters**.

### 4.5 Notes

- Always start `demo-api` **before** starting `pipe_srv` to ensure that the pipe connection is established correctly.
- If you encounter a “permission denied” error, grant execute permission to the binaries:

  ```bash
  chmod +x demo-api pipe_srv
  ```

- For detailed calibration procedures, parameter definitions, and additional command formats, refer to the xvsdk documentation and any vendor-provided calibration manuals.

---

## 5. FAQ

1. **`lsusb` does not show the camera**  
   - Check that the data cable is firmly connected.  
   - Try a different USB port or cable.  
   - Make sure the SDK has been installed correctly and that the OS is Ubuntu 20.04.

2. **`readimage.py` reports “Cannot open /dev/video0”**  
   - Check whether the device node exists (`ls /dev/video*`).  
   - Confirm that the current user has sufficient permissions, or test with `sudo`.  
   - If multiple video devices are present, try changing the `DEV` index (for example, from 0 to 1 or 2).  
   - If the SDK is not yet installed, complete **Section 1: SDK Installation** first.

3. **The camera cannot start at 100 fps**  
   - In `readimage.py`, change `FPS = 100` to `FPS = 60` (or another frame rate supported by your device), then run the script again.

4. **No output from the calibration programs**  
   - Verify that the xvsdk environment is installed and configured correctly.  
   - Ensure that `./demo-api` is started **before** `./pipe_srv`.  
   - Make sure a valid command (such as `1-0-37`) is entered in the terminal running `pipe_srv`.

---

## 6. Dependency Summary

| Component                         | Dependencies |
|-----------------------------------|--------------|
| Overall environment (per SDK doc) | Ubuntu 20.04, FastUMI Hardware SDK (with ROS1 Noetic), camera visible via `lsusb` |
| `readimage.py`                    | Python 3, OpenCV (`cv2`), NumPy |


For more details, please refer to the documentation included in the **FastUMI Hardware SDK** repository and the individual documents in each subdirectory of this project.

