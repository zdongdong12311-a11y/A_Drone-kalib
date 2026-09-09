# D455 视觉定位自主导航无人机

基于 **Intel RealSense D455 + VINS-Fusion + Ego-Planner + PX4** 的轻量级自主导航避障无人机系统，支持室内视觉定位、实时避障、航点巡航与仿真验证。

---

## 硬件配置

| 组件 | 型号 / 版本 |
|------|-------------|
| 机载电脑 | OrangePi 5 Max |
| 操作系统 | Ubuntu 20.04 |
| ROS 版本 | ROS Noetic |
| 飞控 | 微控 743 |
| 飞控固件 | PX4 v1.13.3 |
| 深度相机 | Intel RealSense D455（固件 5.15.0.2） |
| 驱动 SDK | librealsense v2.57.7 |

---

## 功能概览

- **视觉定位**：VINS-Fusion 融合 D455 双目红外 + IMU，提供 6DoF 位姿估计
- **飞控融合**：通过 MAVROS 将 VINS 位姿注入 PX4 EKF2，实现室内无 GPS 飞行
- **实时避障**：Ego-Planner 局部路径规划，动态绕障
- **自主导航**：多模式状态机（起飞 / 悬停 / 航点 / 巡逻 / 返航 / 降落）
- **影子跟随**：遥控器与 OFFBOARD 模式无缝切换，无顿挫悬停
- **建图支持**：RTAB-Map 三维建图与地图加载（可选）
- **Gazebo 仿真**：PX4 SITL + 深度相机仿真，仿真环境全流程验证

---

## 系统架构

```
┌──────────────────────────────────────────────────────────────┐
│                      自主导航系统                              │
├──────────────────────────────────────────────────────────────┤
│  命令层   ROS Service  (/nav/takeoff, /nav/go_to, ...)       │
│  状态机   SHADOW → TAKEOFF → HOLD → TRACK / WAYPOINT / PATROL │
│  规划层   Ego-Planner 实时避障轨迹规划                         │
│  控制层   ego_bridge.py → MAVROS → PX4 飞控                  │
├──────────────────────────────────────────────────────────────┤
│  桥接层   vins-to-px4.py  (VINS 坐标系 → PX4 坐标系)          │
├──────────────────────────────────────────────────────────────┤
│  定位层   VINS-Fusion  (D455 双目红外 + IMU)                  │
│  感知层   RealSense D455  (infra1/infra2 + IMU)              │
└──────────────────────────────────────────────────────────────┘
```

**数据流（实机）**

```
D455 相机
  ├─ /camera/infra1/image_rect_raw  ─┐
  ├─ /camera/infra2/image_rect_raw  ─┤→ VINS-Fusion → /vins_estimator/odometry
  └─ /camera/imu                      ─┘         │
                                                  ↓
                                        vins-to-px4.py
                                                  │
                                                  ↓
                              /mavros/vision_pose/pose → PX4 EKF2
                                                  │
Ego-Planner ← 深度点云/里程计              MAVROS ← PX4 飞控
      │
      ↓
planning/pos_cmd → ego_bridge.py → /mavros/setpoint_position/local
```

---

## 项目结构

```
A_Drone-kalib-main/
├── README.md                          ← 本文件（项目总览）
│
├── kalib_标定/                         ← 传感器标定流程与配置文件
│   ├── README.md                      ← IMU / 双目 / IMU+双目 标定详细步骤
│   ├── 棋盘格_yaml/april_129_A4.yaml  ← 标定板参数
│   ├── imu标定/d455_imu_param.yaml    ← IMU 噪声标定结果
│   ├── 左右双目标定/                   ← 多相机标定结果
│   ├── imu+双目标定/                   ← IMU-相机联合标定结果
│   └── vins_配置文件-修改后/           ← 标定后填入的 VINS 配置
│       ├── realsense_stereo_imu_config.yaml
│       ├── left.yaml
│       └── right.yaml
│
├── real/                              ← 实机部署文档
│   ├── README.md                      ← 环境安装、驱动、VINS、PX4 参数、测试流程
│   └── fuctions_ws/src/fuctions/
│       ├── scripts/
│       │   ├── vins-to-px4.py         ← VINS → PX4 位姿桥接（坐标系转换）
│       │   ├── ego_bridge.py          ← Ego-Planner → PX4 控制桥接（影子跟随）
│       │   ├── autonomous_navigator.py← 自主导航状态机（实机版）
│       │   └── v888_basic.py          ← 基础测试脚本
│       ├── launch/
│       │   ├── ego/                   ← Ego-Planner 启动配置
│       │   └── build&load-map/        ← RTAB-Map 建图/加载
│       ├── docs/
│       │   ├── state_machine.md       ← 状态机与 ROS 命令说明
│       │   ├── ego.md                 ← Ego-Planner 避障测试流程
│       │   └── build&load_map.md      ← 三维建图说明
│       └── shfiles/
│           └── d455_to_px4.sh         ← 一键启动脚本（相机+VINS+MAVROS+桥接）
│
└── gazebo/                            ← Gazebo 仿真
    ├── README.md                      ← PX4 SITL + Gazebo 仿真环境搭建
    ├── bridge.py                      ← 仿真版 Ego → PX4 桥接
    ├── autonomous_navigator_sim.py    ← 自主导航（仿真版，用 Gazebo Odom 替代 VINS）
    └── gazebo_ego.launch              ← 仿真 Ego-Planner 启动文件
```

---

## 快速开始

### 整体流程

```
1. 环境搭建        →  real/README.md
2. 传感器标定      →  kalib_标定/README.md
3. VINS 定位测试   →  real/README.md 第七节
4. VINS→PX4 融合   →  real/README.md 第七、八节
5. 避障测试        →  real/fuctions_ws/.../docs/ego.md
6. 自主导航        →  real/fuctions_ws/.../docs/state_machine.md
7. 仿真验证（可选）→  gazebo/README.md
```

### 实机：6 终端启动

```bash
# 终端 1：RealSense 相机
roslaunch realsense2_camera rs_camera.launch

# 终端 2：MAVROS 飞控通信
roslaunch mavros px4.launch

# 终端 3：VINS-Fusion 定位
rosrun vins vins_node ~/catkin_ws/src/VINS-Fusion/config/xxx/realsense_stereo_imu_config.yaml

# 终端 4：VINS → PX4 位姿桥接
python3 ~/A_Drone-kalib-main/real/fuctions_ws/src/fuctions/scripts/vins-to-px4.py

# 终端 5：Ego-Planner 避障规划
roslaunch fuctions run_in_sim.launch   # 见 launch/ego/ 目录

# 终端 6：自主导航
python3 ~/A_Drone-kalib-main/real/fuctions_ws/src/fuctions/scripts/autonomous_navigator.py
```

**飞行流程**：遥控器手动起飞 → 悬停 → 切换 OFFBOARD → 通过 ROS 命令控制

### 仿真：Gazebo 快速验证

```bash
# 终端 1：启动 PX4 SITL + Gazebo（深度相机模型）
cd ~/PX4-Autopilot
source Tools/setup_gazebo.bash $(pwd) $(pwd)/build/px4_sitl_default
roslaunch px4 mavros_posix_sitl.launch vehicle:=iris \
  sdf:=$(pwd)/Tools/sitl_gazebo/models/iris_depth_camera/iris_depth_camera.sdf

# 终端 2：Ego-Planner
roslaunch ego_planner gazebo_ego.launch

# 终端 3：PX4 桥接
python3 gazebo/bridge.py

# 终端 4：自主导航（仿真版）
python3 gazebo/autonomous_navigator_sim.py

# 终端 5：解锁并起飞
rosservice call /mavros/set_mode 0 "OFFBOARD"
rosservice call /mavros/cmd/arming True
rosservice call /nav/takeoff "{}"
```

详细仿真步骤见 [gazebo/README.md](gazebo/README.md)。

---

## 自主导航 ROS 命令

| 命令 | 功能 |
|------|------|
| `rosservice call /nav/takeoff "{}"` | 一键起飞（默认高度） |
| `rosservice call /nav/go_to "{}"` | 飞向下一个预置航点 |
| `rosservice call /nav/start_patrol "{}"` | 开始自主巡逻 |
| `rosservice call /nav/enable_tracking "{}"` | 启用 Ego-Planner 避障追踪 |
| `rosservice call /nav/return_home "{}"` | 返航 |
| `rosservice call /nav/land "{}"` | 降落并自动停桨 |
| `rosservice call /nav/hold "{}"` | 悬停当前位置 |
| `rosservice call /nav/stop "{}"` | 停止控制，切回影子跟随 |
| `rosservice call /nav/status "{}"` | 查看当前状态 |

完整说明见 [state_machine.md](real/fuctions_ws/src/fuctions/docs/state_machine.md)。

---

## 传感器标定

标定是使用 VINS-Fusion 的前提，按以下顺序进行：

| 步骤 | 内容 | 工具 | 输出文件 |
|------|------|------|----------|
| 1 | IMU 内参标定 | imu_utils | `d455_imu_param.yaml` |
| 2 | 双目 + RGB 外参标定 | Kalibr | `multicameras_calibration-camchain.yaml` |
| 3 | IMU + 双目联合标定 | Kalibr | `imu_stereo-camchain-imucam.yaml` |
| 4 | 填入 VINS 配置 | 手动 | `realsense_stereo_imu_config.yaml` 等 |

标定完成后还需进行 **VINS 外参在线收敛**（手持飞机绕场飞行，将 `extrinsic_parameter.csv` 结果写回配置文件）。

详细步骤见 [kalib_标定/README.md](kalib_标定/README.md)。

---

## PX4 飞控关键参数

室内视觉定位飞行前，需在 QGroundControl 中修改以下 EKF2 参数：

| 参数 | 设置 | 说明 |
|------|------|------|
| `EKF2_EV_CTRL` | 勾选 Horizontal position + Yaw | 开启外部视觉融合 |
| `EKF2_HGT_MODE` | Vision | 用视觉定高，废弃气压计（室内空调干扰大） |
| `EKF2_GPS_CTRL` | 0（关闭全部） | 彻底关闭 GPS 融合 |
| `EKF2_EV_DELAY` | 50 | 视觉延迟补偿（毫秒） |

---

## 重要注意事项

### 结构光必须关闭

D455 的红外结构光投影会干扰 VINS 轨迹精度。标定和飞行前务必关闭：

```bash
roslaunch realsense2_camera rs_camera.launch
rosrun rqt_reconfigure rqt_reconfigure
# 将 camera → stereo_module → emitter_enabled 设为 Off (0)
```

或用黑胶带物理遮挡结构光发射器。

### 不建议使用 D455 内置 IMU 驱动飞控

D455 的 IMU 主要用于 VINS 融合，**不要**将其作为 PX4 的主 IMU 源。飞控应使用自身板载 IMU，VINS 通过 `/mavros/vision_pose/pose` 提供外部视觉定位。

参考：[PX4 与 RealSense IMU 说明](https://blog.csdn.net/qq_40186909/article/details/113104595)

### 安全保护机制

实机版 `autonomous_navigator.py` 内置以下安全策略：

- VINS 数据超时（0.5s）自动触发返航
- 低电量（< 20%）自动返航，严重低电量（< 18%）强制降落
- 地理围栏限制最大飞行半径（默认 8m）
- 最低飞行高度保护（0.5m，防止触地）
- 遥控器随时可切出 OFFBOARD，进入影子跟随模式

---

## 依赖项汇总

| 组件 | 版本 / 来源 |
|------|-------------|
| ROS Noetic | Ubuntu 20.04 官方 |
| MAVROS | `ros-noetic-mavros` + GeographicLib 数据集 |
| librealsense | v2.57.7（`-DFORCE_RSUSB_BACKEND=true` 编译） |
| realsense-ros | `ros1-legacy` 分支 |
| VINS-Fusion | [HKUST-Aerial-Robotics/VINS-Fusion](https://github.com/HKUST-Aerial-Robotics/VINS-Fusion) 或社区修复版 |
| Ego-Planner | [ZJU-FAST-Lab/ego-planner](https://github.com/ZJU-FAST-Lab/ego-planner) |
| Kalibr | [ethz-asl/kalibr](https://github.com/ethz-asl/kalibr) |
| imu_utils | [gaowenliang/imu_utils](https://github.com/gaowenliang/imu_utils) |
| PX4 Autopilot | v1.13.3（仿真） |
| RTAB-Map | `ros-noetic-rtabmap-ros`（建图，可选） |

各组件安装细节见 [real/README.md](real/README.md)。

---

## 子目录文档索引

| 文档 | 内容 |
|------|------|
| [real/README.md](real/README.md) | 完整环境搭建：MAVROS、RealSense 驱动、VINS-Fusion 编译、PX4 参数、定位测试、外参收敛、悬停测试 |
| [kalib_标定/README.md](kalib_标定/README.md) | IMU 标定、双目标定、IMU+双目联合标定、VINS 配置文件填写 |
| [gazebo/README.md](gazebo/README.md) | PX4 SITL + Gazebo 仿真环境、Ego-Planner 仿真避障、自主导航仿真 |
| [state_machine.md](real/fuctions_ws/src/fuctions/docs/state_machine.md) | 自主导航状态机、ROS 命令、Python API、故障排除 |
| [ego.md](real/fuctions_ws/src/fuctions/docs/ego.md) | Ego-Planner 实机避障测试流程 |
| [build&load_map.md](real/fuctions_ws/src/fuctions/docs/build&load_map.md) | RTAB-Map 三维建图与地图加载 |

---

## 常见问题

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| RealSense 识别不到设备 | SDK 版本与固件不匹配 | 检查固件版本，尝试插拔，确认 udev 规则 |
| RealSense 无 IMU 数据 | 未用 USB 后端编译 | cmake 时加 `-DFORCE_RSUSB_BACKEND=true` |
| VINS 编译报错 `CV_LOAD_IMAGE_GRAYSCALE` | OpenCV 4 API 变更 | 替换为 `cv::IMREAD_GRAYSCALE` 等，或使用社区修复版 |
| VINS 轨迹漂移严重 | 结构光未关闭 / 标定不准 | 关闭 emitter，重新标定并做外参收敛 |
| PX4 位姿与 VINS 不一致 | 坐标系映射错误 | 调整 `vins-to-px4.py` 中的轴映射与偏移参数 |
| 切 OFFBOARD 后飞机失控 | 未持续发送 setpoint 心跳 | 确认 `ego_bridge.py` 在运行（30Hz 心跳） |
| 避障不生效 | 未启用 tracking 模式 | 先调用 `/nav/enable_tracking` |
| RViz 中机器人坐标不动 | VINS `child_frame_id` 设为 "world" | 修改为 `vins_body` 并重新编译 VINS |

---

## 许可证

本项目基于多个开源项目整合，使用时请遵循各上游项目的许可证：

- [VINS-Fusion](https://github.com/HKUST-Aerial-Robotics/VINS-Fusion) — GPL-3.0
- [Ego-Planner](https://github.com/ZJU-FAST-Lab/ego-planner) — GPL-3.0
- [PX4 Autopilot](https://github.com/PX4/PX4-Autopilot) — BSD-3-Clause
- [Kalibr](https://github.com/ethz-asl/kalibr) — Apache-2.0
