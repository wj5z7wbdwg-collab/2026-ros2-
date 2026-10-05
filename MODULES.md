# MODULES.md · 模块边界与独立开发指南

> 目的：让你能**单独开发优化某一个板块**而不误伤其他板块。
> 原则：8 个 ROS 包 = 8 个模块，物理隔离在 `autocar_ws/src/<pkg>/`；
> 模块间只通过**话题/服务/TF（接口契约）**耦合，不通过代码 import 耦合（autocar_msgs 除外）。

## 0. 接口契约总表（改动需全链路评估，独立优化时默认冻结）

| 接口 | 类型 | 生产者 → 消费者 | 定义位置 |
|---|---|---|---|
| `/scan` `/imu/data` `/camera/image_raw` `/camera/camera_info` | 传感器 | 供应商驱动 → 全链 | 外部（不在本仓库） |
| `/scan_filtered` | LaserScan | M1 → M2/M4/M6 | — |
| `/cones` `/boards` | ConeArray/BoardArray | M2 → M3 | `autocar_msgs/msg/` |
| `/race_line` | nav_msgs/Path | M3 → M4 | — |
| `/cmd_vel_raw` `/cmd_vel` | Twist | M4 → M5 | — |
| `/odometry/filtered` `/odom` | Odometry | M5/M6 → M3/M4/M7 | — |
| `/traffic_light/state` `/parking_sign/state` | String | M2 → M7 | — |
| `/race_state` `/race_lap` `/watchdog/status` `/off_track/count` | String/Int32/Bool | M7/M4 → 全链 | — |
| `/map` `/amcl_pose` `/particle_cloud` | 导航 | M6 → M7 | — |
| TF `map→odom→base_link→laser_link/imu_link/camera_link` | TF | M6/M5/M1(bringup) | `autocar_bringup` |

**规则**：独立优化某模块算法 = 只动该模块"算法文件+配置 YAML+测试"三者；上表任何接口名/消息字段不得变（变了先改 `autocar_msgs` 并重编全链，属重操作）。

## 1. 模块一览（独立开发入口）

### M1 感知·激光滤波 — `autocar_perception/scan_filter.py`
- 算法：极坐标距离/强度滤波 → `/scan_filtered`
- 配置：`autocar_perception/config/perception.yaml`（filter 段）
- 独立测试：`python3 autocar_ws/tests/test_perception.py`
- 优化自由度：滤波判据随便改；输出仍是 LaserScan

### M2 感知·锥桶/挡板/视觉 — `cone_detector.py` `dbscan.py` `traffic_light_detector.py` `parking_sign_detector.py`
- 算法：分段 DBSCAN（numpy O(n²)）+ 径向颜色推断 + auto_radial_split 自标定兜底；HSV 阈值 + 连通域形状校验 + 时序滞回
- 备案替代（RUNBOOK §10A / 算法审查文档）：DBSCAN→ABP/RANSAC；HSV→Otsu/YOLOv8n 混合/Hu 矩
- 配置：`perception.yaml`（color/dbscan/boards/traffic/parking 段）
- 独立测试：`tests/test_perception.py`（28 项）+ `tests/test_visual.py`（24 项）
- 优化自由度：聚类与分割算法整体可换，只要 `/cones`（ConeArray）与状态字符串语义不变

### M3 规划 — `autocar_planning/path_generator.py`
- 算法：Voronoi 中轴（scipy）→ 走廊裁剪 [1.0,1.6 m] → 三次样条 → 曲率约束重采样；折线区 `_effective_track_center()`
- 备案：中轴→边界平滑/ESDF 梯度
- 配置：`planning.yaml`
- 独立测试：`tests/test_planning.py`（40 项）
- 优化自由度：路径生成算法可换，输出 `/race_line`（nav_msgs/Path，map 系）不变

### M4 控制 — `autocar_control/pure_pursuit.py` `fuzzy_pid.py` `watchdog.py`
- 算法：PurePursuit（PP-DSC 动态前视）+ 阿克曼逆解 + 模糊 PID（75 规则）+ 静止看门狗（9 s，四态豁免）
- 备案：lookahead 调参→Stanley→MPC(acados)
- 配置：`control.yaml`（steering.convention 与 Q6 联动）+ `watchdog.yaml`
- 独立测试：`tests/test_control.py`（74 项，含 50 Hz 闭环仿真）
- 优化自由度：跟踪控制器可换，输入 `/race_line`+odom、输出 `/cmd_vel_raw` 不变

### M5 底盘 — `autocar_chassis_driver/`（`chassis_driver.py` `codec.py` `serial_link.py`）
- 算法：阿克曼正/逆解 + PWM 线性映射（HR-04 中位 1520 µs）+ 串口帧协议 + failsafe 幽灵里程防护
- 三情形：`scenario_A_angle/B_yawrate/C_pwm.yaml`（Q6 答复选一个生效）
- 配置：`chassis_common.yaml`（wheel.scale_factor、tx_template）
- 独立测试：无独立 test 文件（串口依赖重）——由 `deploy.sh smoke` + 实车 B2 用例覆盖
- 优化自由度：协议编解码在 codec.py 内改；运动学参数只进 YAML

### M6 定位建图 — `autocar_slam/`（配置+launch 封装，算法在外部包）
- 内容：cartographer_2d.lua（子图 SLAM）、amcl.yaml（KLD 粒子滤波）、ekf_config.yaml（EKF 融合）、map_saver.yaml + 三个 stack launch
- 备案：AMCL→KLD+全局重定位→SynPF
- 优化自由度：只调参数（lua/yaml）；换算法=换外部包，launch 接口保持发布 map→odom 唯一性（ADR-003 幽灵发布者红线）

### M7 赛事调度 — `autocar_race_manager/race_manager.py` `stack_supervisor.py`
- 算法：有限状态机（IDLE→MAPPING→ROLLING_SWITCH 六步→NAVIGATING→PARKING→DONE）+ 圈次三取二 quorum + R59~R72 完赛兜底链 + AMCL 健康度监控
- 配置：`race_manager.yaml`（68 参数，含 initial_pose.covariance）
- 独立测试：`tests/test_race_manager.py`（110 项，全时序仿真）
- 优化自由度：状态机逻辑可增强；对外服务接口（/reinitialize_global_localization 等）不变

### M8 消息契约 — `autocar_msgs/`（Cone/Board/ConeArray/BoardArray.msg）
- **改动 = 重操作**：字段增删会破坏 M2/M3 与全部测试；只加字段不删改，改后全链 colcon build

### M0 集成 — `autocar_bringup/`（race.launch.py、sensors.launch.py、static_transforms.yaml、autocar_race.rviz）
- 双源外参之一（另一处=launch 默认值）；改外参必须两处同改（calib_check 门禁会查）

## 2. 单模块独立开发工作流

```bash
# ① 只改这个模块的算法+配置（src/autocar_<pkg>/ 与 tests/test_<pkg>.py）
# ② 本机预演（Windows 可跑，不依赖 ROS）：
python3 autocar_ws/tests/test_<pkg>.py          # 该板块断言全过
python3 autocar_ws/scripts/offline_check.py     # 全量静态回归 FAIL=0
# ③ 车机：
colcon build --symlink-install --packages-select autocar_<pkg>   # 只编该包
bash autocar_ws/scripts/run_race.sh up simulate:=true
bash autocar_ws/scripts/race_test.sh <对应组>   # M2→c1/c2, M3→c4, M4→c5, M5→c6, M7→c7
# ④ 连续 2 次 FAIL 才启用备案替代（清单 §D）；切换后回归三件套+登记 R92+
```

## 3. 测试↔模块映射速查

| 测试文件 | 覆盖模块 | 断言 |
|---|---|---|
| tests/test_perception.py | M1 M2(激光) | 28 |
| tests/test_visual.py | M2(视觉) | 24 |
| tests/test_planning.py | M3 | 40 |
| tests/test_control.py | M4 | 74 |
| tests/test_race_manager.py | M7 | 110 |
| （M5 由 deploy.sh smoke + 实车覆盖；M6 由实车 B3/C6 覆盖） | | |

公共桩：`tests/harness.py` + `tests/ros_stub.py`（rclpy/mock 层，测试专用，不进运行链）。
