# 室外 ROS 无人车赛（职教组）设计技术文档

| 项 | 内容 |
|---|---|
| 文档版本 | **v2.1**（在 v2.0 基础上完成「ROS 2 专项审计」与第三轮规则逐条复核） |
| 依据规则 | 第二十一届全国大学生智能汽车竞赛《室外 ROS 无人车赛（职教组）比赛规则（初稿）》，2026 年 8 月，全文 13 页 |
| 目标平台 | 北京小豚科技有限公司 智能驾驶竞速车 **HW-Autocar**（昇腾 310 / Ubuntu 22.04 / ROS 2 Humble / ARM64） |
| 文档定位 | 团队内部设计与交付基线；第四部分可直接演进为对外提交的技术手册 |
| 状态 | 需求基线已冻结；总体设计已冻结；接口契约已冻结；参数基线待实车标定回填 |
| 对标方式 | 本版**逐条回查规则原文**，每条需求标注「规则出处」；凡原文未明确者一律标为「待确认」，不做推断性填充 |
| **技术栈声明** | 全系统为 **ROS 2 Humble**（Ubuntu 22.04 / ARM64），**不使用 ROS 1**。规则 2.4.2(9) 的"若系统中存在 ROS 1 则罚时 100 秒"是**反向合规约束**——本项目要做的是「**证明系统中不存在 ROS 1**」，而不是「使用 ROS 1」。本文档所有命令、节点名、接口、参数加载方式均为 **ROS 2 形态**，详见附录 C |

> **版本变更纪律**：第三部分的《TF 树》《ICD 接口表》《参数基线表》是**唯一真值源**。代码与对外手册中的任何话题名、坐标系名、参数名，必须与本表逐字一致。修改必须先改本表，再改代码。

---

## v2.0 修订说明（相对 v1.0，供审查用）

本版是针对「未严格贴合规则原文」这一问题的系统性修订。逐项对照如下：

| # | 修订项 | 规则出处 | 性质 |
|---|---|---|---|
| R1 | 修正图 8「16.8 m」的含义：它是**赛道场地的横向总尺寸**（尺寸线横跨整个虚线外框），**不是单圈周长**。v1.0 据此推算的时间预算（"0.6 m/s 约 28 s"）整体失真，已改为以**实测单圈里程**为变量 | 2.2.1(13)、图 8 | **纠错** |
| R2 | 补「光电传感器」完整机制：传感器设于**红绿灯前 2 个锥桶处**，传感器与红绿灯之间的区域为停车区域；未遮挡传感器 → 灯不亮 → 通行即判「绿灯未亮起就通行」 | 2.4.2(1) | 补全 |
| R3 | 红绿灯处罚则拆为**三档并按严重度排序**：未停车 +50 s ＞ 绿灯未亮通行 +30 s ＞ 车轮不在区域内 +20 s | 2.4.2(1) | **纠错** |
| R4 | 第二圈入库罚则与红绿灯罚则**分列**：入库失败或未停车 +20 s | 2.4.2(2) | 纠错 |
| R5 | 建图评分严格按原文拆为**两档**：+5 s（地图不清晰／挡板模糊但能看出且位置正确／丢失 ≤2 个挡板和锥桶）与 +50 s（提前建图／未构建地图／挡板位置明显不符） | 2.4.2(5) | 纠错 |
| R6 | 新增 FAIL-05/06：视频遥控或造假 → **取消比赛资格**；抽查复现不配合或造假 → **取消成绩、追回奖状** | 3.5.3(4)(5) | 补全 |
| R7 | FAIL-04 表述按原文字面修正为「一经查实**直接取消本场比赛全部成绩**」 | 2.4.2(10) | 纠错 |
| R8 | 补锥桶完整规格（630×280×280 mm、塑料、红/蓝外皮）与**「如有白色反光膜需要去掉」**——影响 DBSCAN 标定与场地准备 | 2.2.1(12) | 补全 |
| R9 | 补三个并列区域（停车检测区域／停车区域／发车区域）及其**几何边界定义**（顶点＝锥桶投影中心，矩形，以胶带内侧为界，带宽 10 cm） | 2.2.1(3) | 补全 |
| R10 | 补线下赛外围**警示线**、红绿灯主体尺寸（60×17×6.7 cm／离地 40 cm）、停车标志牌规格（29.7×42 cm）、折线区为**两根**黄色胶带 | 2.2.1(9)(10)(14)、2.2(3) | 补全 |
| R11 | 新增 §1.8 参赛资格与队伍要求（ELIG）：仅限全日制在校专科生、每队 5 名＋指导教师 2 名、队间抄袭与共用设备取消两队评奖资格、报名/QQ 群/邮箱/网盘链接 | 3.1 / 3.4 / 3.5.2 / 4.1–4.3 | 补全 |
| R12 | 明确设备措辞差异：3.2 称组委会「统一**指定**」，4.4 称「统一**推荐**」——原文不一致，如实标注 | 3.2、4.4 | 如实标注 |
| R13 | 修正开放问题 Q2：**全场只有一个红绿灯**（位置随机，两圈各遇到一次），并新增「停车检测区域与停车区域的关系」问题 | 2.1、2.2.1(6)(7)(8) | 纠错 |
| R14 | 补线上赛视频评分细则全文，并明确「线上赛与线下赛适用罚则不同」（PEN-12~14、FAIL-05/06 仅线上赛适用） | 3.5.1、3.5.3 | 补全 |
| R15 | 显著标注**技术手册官方模板不在规则 PDF 内**，须从 4.2 的百度网盘获取；第四部分为结构建议，以官方模板为准 | 4.2 | 如实标注 |
| R16 | 状态机补「圈次判定」与「行进中模式切换（rolling switch）」——因任意位置静止 >10 s 即判负，切换不能停车 | 2.4.2(8) | 补全 |

---

## v2.1 修订说明（相对 v2.0，供审查用）

本版针对两项诉求：① **明确并强化"本车是 ROS 2 小车"**；② 对规则原文（13 页）做**第三轮逐条复核**。

### A. ROS 2 专项审计

| # | 修订项 | 位置 |
|---|---|---|
| R17 | 新增**「技术栈声明」**：全系统 ROS 2 Humble，**不使用 ROS 1**；并澄清 PEN-10 是**反向合规约束**（要证明"不存在 ROS 1"） | 文档头 |
| R18 | 附录 A.1 由「ROS 1 残留清除」重写为「**ROS 1 残留自查（合规举证）**」：明确"本车是 ROS 2 车，本节只为在裁判面前**举证系统内不存在 ROS 1**"；全部命令为 ROS 2 / shell 形态；新增 `show_ros_version.sh` 与断言脚本 | 附录 A.1 |
| R19 | 新增**附录 C　ROS 2 技术栈与合规声明**：包名清单与安装命令、ROS 1 ↔ ROS 2 术语对照表、生命周期状态名、参数/launch 的 ROS 2 形态 | 附录 C |
| R20 | 节点与工具名全面 ROS 2 化：`map_saver` → **`nav2_map_server`（`map_saver_cli`）**；`amcl` → **`nav2_amcl`**；明确 launch 为 **Python `*.launch.py`**；参数用 **YAML + `--ros-args --params-file`**（ROS 2 无 `rosparam`）；构建用 `colcon`（非 `catkin_make`） | §1.6、§8.2、§9.3、附录 C |
| R21 | FAIL-04 与开发纪律改写：规则原文用词 **`rosbag`** 是 ROS 1 时代工具名，ROS 2 中对应 **`ros2 bag`**；给出**保守执行结论——比赛中完全不使用 `ros2 bag`（连录制也不做）**，行车录像由手机/相机完成，避免任何"回放"嫌疑 | §1.4、§1.5、§1.6、ADR-005 |
| R22 | 补 ROS 2 CLI 自检命令（`ros2 topic hz` / `ros2 run tf2_tools view_frames` / `ros2 param` / `ros2 lifecycle`）与 §13 的 ROS 2 合规自查项 | 附录 A.2、§13 |

### B. 规则对标补充（第三轮逐条回查）

| # | 修订项 | 规则出处 |
|---|---|---|
| R23 | 补「建图成功」的**正面判据**：挡板和锥桶与实际相符并且清晰可见 → **视为建图成功（不罚）** | 2.4.2(5) |
| R24 | 新增开放问题 Q9：丢失**超过 2 个**挡板/锥桶但地图确实存在时如何判罚（原文只规定了"≤2 个"） | 2.4.2(5) |
| R25 | 新增开放问题 Q10、DR-04 标注：3.4 线下赛正文写"技术手册**及视频**"，但其目录结构中**无视频**——原文自相矛盾 | 3.4 |
| R26 | 新增开放问题 Q11：规则 4.4 舵机写作"工作**频率**：1520us/330hz"，1520 µs 实为**脉宽中位值**而非频率（原文表述有误） | 4.4 |
| R27 | 新增开放问题 Q12：规则 2.1 引用"**图 1** 青色框"，2.2.1(6) 引用"**图 2**"——同一物图号不一致 | 2.1、2.2.1(6) |
| R28 | CR-22 明确标注为「**读图推断，需现场核对**」，避免与 §2.1 产生内部矛盾 | 图 2 |
| R29 | CR-15 补规格顺序说明：630 mm 为**高度**、底面 280×280 mm（按行业惯例推断，需与实物核对） | 2.2.1(12) |
| R30 | 新增 DR-15：补 2.3.1 的**案例评判依据原文用词**"创新性、**结构规范、内容深度**"，与四项评分细则并列 | 2.3.1 |

### C. 交付性补充

| # | 修订项 | 位置 |
|---|---|---|
| R31 | 新增**附录 D　向组委会 / 供应商的书面确认清单**：把 Q1 ~ Q12 拆为 **A 组（规则解释类 → 组委会）/ B 组（设备与接口类 → 供应商）** 两封可直接发出的邮件，含邮件正文模板、"**我方当前假设**"与收到答复后的回填动作表 | 附录 D |
| R32 | 修正两处**内部交叉引用错误**：① CR-10 误引 §2.3 **Q8** → 应为 **Q7**（停车区域与标志牌关系）；② §3.2 依据列表补全为 CR-05、CR-07、CR-13 ~ CR-15、CR-17 | §1.2、§3.2 |
| R33 | 新增**附录 E　系统设计卡 E-1：红绿灯识别与停车（含光电触发应对）**：收敛 §1.4/§5.3/§9.5~§9.7/ADR-006 为可独立评审与测试的设计卡（状态机、判据、时序、参数、8 条测试用例、FMEA）。同步在 §8.4 登记 `/traffic/wait_duration`、§9.7 登记 `traffic.approach_distance` | 附录 E、§8.4、§9.7 |
| R34 | **ROS 2 通信与状态机专项核查**（9 处修正）：① `/map_save` 类型由错误的 `std_srvs/srv/Trigger` 改为 **`nav2_msgs/srv/SaveMap`**（`map_saver_cli` 无服务、会阻塞）；② 澄清 **`cartographer_node` 不是 LifecycleNode**，建图栈须进程级停止（ADR-003、§5.2、§7.3、§A.2）；③ `odom→base_link` 唯一发布者为 `ekf_node`，`chassis_driver` 只发话题不发 TF；④ §7.3「任何时刻速度不得为 0」**限定为切换窗口**（否则与红绿灯/入库停车矛盾）；⑤ 新增 `watchdog.exempt_states`，豁免 `TRAFFIC_STOP/SAFE_HOLD/PARKING/DONE` 的静止（红灯倒计时 10 s > 阈值 9 s）；⑥ §5.3 状态图补 `TRAFFIC_GO`，异常分支与附录 E 对齐；⑦ §8 增加**类型书写约定**（`pkg/msg/Type`）与"不使用 Action"说明；⑧ 增加 **QoS 兼容性陷阱**提醒（§8、附录 C.4）；⑨ §A.2 生命周期检查改为 Nav2 节点 | §7、§8、§5.2、§5.3、§9.7、ADR-003、§A.2、附录 C.4 |
| R35 | **全量类型规范化改写**：§8 及全文所有消息 / 服务类型统一为 ROS 2 规范全称 `pkg/msg/Type`、`pkg/srv/Type`（原为 ROS 1 式简写 `pkg/Type`）。同步改写 §8 头部「类型书写约定」，并新增 `.msg` 文件内部字段须写短名的语法差异说明 | §8 全表、附录 C.4 |
| R36 | 新增**附录 F　系统设计卡 E-2：第一圈建图与行进中模式切换（Rolling Switch）**：收敛 §5.1/§5.2/§5.3、§7.2/§7.3 切换时序铁律、§9.2~§9.5/§9.7、ADR-001/ADR-003 为第二张设计卡（头部、边界、接口、状态机、圈次判定与建图成功判据、六步时序、参数表、8 条测试用例 `TC-F01~F08`、FMEA、需求—罚则映射）。**未新增任何接口或参数**（数值全部引自 §9） | 附录 F、§5、§7 |
| R37 | 新增**附录 G　系统设计卡 E-3：第二圈停车标志识别与入库停车**：收敛 §1.2（CR-07/10/11/22）、§1.4（PEN-04）、§1.5（FAIL-02）、§5.3、§9.5~§9.7 为第三张设计卡（两圈分叉、行进中识别、低速入库、8 条测试用例 `TC-G01~G08`、FMEA、映射）。**未新增接口**；新增参数同步登记：§9.6 `hsv.blue_h_range`，§9.7 `parking.approach_distance`、`parking.target_x/y/yaw`、`parking.in_zone_wheels_min` | 附录 G、§9.6、§9.7 |
| R38 | 新增**附录 H　系统设计卡 E-4：锥桶与挡板感知**：收敛 §1.2（CR-02/03/04/06/13/14/15/17）、§1.4（PEN-05/06/09）、§1.5（FAIL-01）、§5.1、§8.1、§9.1 为第四张设计卡（以「处理流水线」替代状态机，含分段 DBSCAN、径向颜色推断、挡板合并后处理、8 条测试用例 `TC-H01~H08`、FMEA、映射）。**未新增接口**；新增参数 `cone.min_confidence` 同步登记到 §9.1 | 附录 H、§9.1 |
| R39 | 新增**附录 I　系统设计卡 E-5：定位（EKF 里程计融合 + 第二圈 AMCL）**：收敛 §3.1、§5.1、§7.1/§7.2/§7.3、§8.2、§9.3 为第五张设计卡（以「定位模式与 TF 权属」替代状态机，含 EKF 融合、里程计标定、AMCL 收敛判据、健康度监控、QoS 核对、8 条测试用例 `TC-I01~I08`、FMEA、映射）。**未新增接口**；**新增 §9.8「里程计融合（`ekf_node`）」参数节**（此前 §9 缺 EKF 基线，含 `wheel.scale_factor`） | 附录 I、§9.8 |
| R40 | 新增**附录 J　系统设计卡 E-6：控制（纯追踪 + 模糊 PID）**：收敛 §1.2（CR-06）、§1.4（PEN-05/06）、§1.5（FAIL-01/03）、§5.1、§8.3、§9.5 为第六张设计卡（以「控制流与限速档」替代状态机，含纯追踪几何、模糊自整定、限幅限速率、限速仲裁、`δ→PWM(1520µs)` 映射、8 条测试用例 `TC-J01~J08`、FMEA、映射）。**未新增接口**；新增控制增益与限速参数（`pid.*`、`fuzzy.*`、`control.max_steering_rate`、`control.max_accel/decel`）同步登记到 §9.5 | 附录 J、§9.5 |
| R41 | 新增**附录 K　系统设计卡 E-7：路径生成（`path_generator`）**：收敛 §1.2（CR-02/03/06/07/17/20）、§1.4（PEN-05/06/09）、§1.5（FAIL-01）、§5.1、§8.1/§8.3、§9.4 为第七张设计卡（以「处理流水线」替代状态机，含内外边界、Voronoi 走廊、折线区收紧、曲率约束、8 条测试用例 `TC-K01~K08`、FMEA、映射）。**未新增接口**；新增参数 `path.safety_margin`、`path.min_turning_radius` 同步登记到 §9.4。**至此 E-1~E-7 七张设计卡覆盖感知/定位/建图切换/控制/路径/红绿灯/入库全链路** | 附录 K、§9.4 |

### D. 算法与工程实证补充（v2.1 续，本轮；依据联网一手资料）

> **本轮改动的取舍原则**：**只采纳可被一手资料支撑的建议**；对来源可疑、无法核实或**自相矛盾**的建议，**降级为"风险标注 / 待实机验证"**，不写成确定结论。逐项依据见各行。

| # | 修订项 | 依据 / 位置 |
|---|---|---|
| R42 | **QoS 由"提醒"升级为"可执行核对"**：§8 新增 **QoS 基线表**（逐话题 Reliability/Durability/Depth）与**兼容性规则**（`Reliable` 发布者可匹配 `Best-Effort` 订阅者，**反之不成立**）；§A.2 新增 3 条核对项；附录新增 **C.2.1 QoS 兼容性自查命令**（可直接粘贴执行） | ROS 2 QoS 官方语义；§8、§A.2、附录 C.2.1 |
| R43 | **AMCL 健康度监控由"建议"升级为"必需"**：§9.3 新增 `amcl.covariance_position_threshold`(0.5 m²)、`amcl.covariance_yaw_threshold`(0.3 rad²)、`amcl.degradation_duration`(2.0 s)；§5.3 新增 `LOCALIZATION_RECOVER` 异常分支（降速 + 调 Nav2 标准服务 `/reinitialize_global_localization`，两级兜底）；附录 I.5.4 / I.7 / I.9 同步 | Nav2 AMCL 标准服务语义；§5.3、§9.3、附录 I |
| R44 | **Voronoi 走廊裁剪补量化判据**：§9.4 新增 `path.corridor_width_min`(**1.0 m**) / `path.corridor_width_max`(**1.6 m**)，由 CR-02(1.5 m) / CR-19(1.25 m) / CR-03 / HR-01(0.56 m) **推导**（**未采用外部建议的 1.8 m**——该值大于赛道宽度，几何上不成立）；附录 K.5.3 补两类反例与桥接规则、K.7 / K.8 补参数与 `TC-K09` | §9.4、附录 K |
| R45 | **纯追踪前视距离改为"转向角驱动"**（PP-DSC 思路）：§9.5 `lookahead.mode` 改为 `steering_adaptive`，新增 `lookahead.steering_gain`(0.5)、`speed.steering_decay`(0.7)，`lookahead.curvature_gain` **降为备选**；附录 J.5.1 换公式并**如实标注边界**——文献（Scientific Reports 2026, DOI `10.1038/s41598-026-38695-1`）在开阔场地改善 **68–82%**，但**同一文献指出小半径（5–9 m）场地标准纯追踪反而更优**，本赛道半径 1.5 m 属小半径，故**增益必须实车标定** | §9.5、附录 J.5.1 |
| R46 | **SLAM 选型补风险标注与 Plan B**：ADR-001 新增「风险标注」（如实呈现文献**不一致**：ACM 2025 支持 Cartographer，但 MDPI *Eng. Proc.* 2025 小尺度赛车评测中 SLAM Toolbox 的 DSC/SSIM 反超或持平；**未采用来源不明且自相矛盾的"IoU 0.4~0.75"数据**）；**新增 ADR-001-B**（SLAM Toolbox 备选，含 3 条量化切换触发条件）；§9.2 补 `submaps.num_range_data` **下调**方向 | ADR-001、ADR-001-B、§9.2 |
| R47 | **EKF 配置改为"实车标定决定"而非直接改值**：§9.8 保留 `imu0_differential: false`（两个独立集成参考均取 `false`，**未采纳"改为 true"的建议**），但补齐**取舍说明**与改 `true` 的触发条件；`imu0_remove_gravitational_acceleration` 补"须与驱动核对、防双重去重力"；§A.2 新增「IMU 静态漂移」核对项；附录 I.9 新增失效模式「IMU 航向零偏注入 EKF → AMCL 低速持续旋转」 | §9.8、§A.2、附录 I.9 |
| R48 | **Cartographer Lifecycle 仅登记为"未来评估项"**：ADR-003 明确"`cartographer_node` 不是 LifecycleNode"这一**事实不变**，Lifecycle 包装器改造**因工程量 / 风险登记为未来评估项**，本周期仍用进程级终止 | ADR-003 |
| R49 | **修正 §9.6 笔误**：`hsv.red_h_ranges` 上界 `180` → **`179`**（OpenCV H 通道为 0–179）——此前该修正只落在 `ros小车.txt`，**本次补入参数唯一来源（§9.6）** | §9.6 |
| R50 | **附录 F.6 补前置检查**：切换时序新增第 ⓪ 步「IMU 自校准完成」前置检查（未收敛不得切换），呼应 R47 | 附录 F.6 |
| R51 | **附录 D 新增 `D.5 确认进度跟踪表`**：把 Q1 ~ Q12 整理为**可逐项打勾的台账**（编号 / 归属 / **是否阻塞编码** / 影响章节 / 发出日期 / 答复日期 / 答复要点 / 已回填），并明确 **Q4、Q5、Q6、Q7 为阻塞编码项**——四项回填完毕方可解除 §3.3、附录 J、附录 K 的"待定 / 不得据此编码"标记 | 附录 D.5 |
| R52 | **新增附录 L「学习类方法（ML/DL）评估与增强路线图」（非基线，仅评估）**：① 对"自主建图"要求做覆盖自检（TR-02 / TR-08 / PEN-07 / PEN-08 / SEC / rolling switch / ADR-001 逐条闭环确认）；② 基于全网络检索的 20 余项代表性工作（End2Race、TinyCenterSpeed、On-Board RL、Dream4Speed、KA-RaceIng、QUT ROS 2 栈、YOLOv8 红绿灯、FALO 边缘部署等）逐模块评估学习类方法的实证数据、赛道适配与风险；③ 给出**三阶段门控式**增强路线图与合规红线（数据来源、影子模式、与 SEC/TR-08/PEN-11 的边界）。**不修改 §9 任何参数、不新增接口、不解除 Q4~Q7 阻塞标记** | 附录 L |
| R53 | **附录 L 新增 `L.10 学习类模型数据采集清单`**：给出 6 个数据集（DS-01 ~ DS-06）的目标模型 / 话题 / 样本量 / 场景覆盖，采集命令（含 `ros2 bag record` 的 QoS 覆盖参数）、标注规范、**按场景分组防数据泄漏**的 train/val/test 划分、存放与 manifest 命名、三段式检查表。全部话题取自 §8 已登记集合（`/scan`、`/camera/image_raw`、`/odometry/filtered`、`/cmd_vel`、`/steering_angle` 等），**仍不修改 §9、不新增接口** | 附录 L.10 |
| R54 | **附录 D 新增 `D.6 Q6 现场核查 SOP、三情形判定与 DS-05 解耦`**：① 明确"哪些被 Q6 阻塞、哪些不被阻塞"（DS-01/02/03/04/06 与阶段 1 训练**均不被阻塞**，不必陪等）；② 给出**现场自查 SOP**（`ros2 topic list/info -v`、驱动包检索、**发单条 `/cmd_vel` 的决定性实验**）把 Q6 从"等供应商"变为"可实测确认"；③ **三情形判定表**（A：`angular.z`=δ、B：`angular.z`=ω、C：不吃 `/cmd_vel` 仅 PWM），每情形明确 §8.3 落笔方式与 `/steering_angle` 可用性；④ **"原始记录优先"原则**——DS-05 只录原始话题，δ 留待 Q6/Q5 答复后离线推导，避免语义变更导致数据集报废；⑤ 供应商**最小答复模板**（勾选+填空）；⑥ 供应商长期不答复时的**实测兜底**（解除阻塞但保留"待确认"标注）。**不修改 §9 参数、不新增接口** | 附录 D.6 |
| R55 | **附录 D 新增 `D.2.1 可直接发送的 B 组邮件正文（致设备供应商 北京小豚科技）`**：把 D.2 的占位式模板与 D.3 的 B 组条目（Q5 / Q6 / Q11）合并为一封**成品邮件**——主题、称呼、必要性说明、**勾选 + 填空**式 5 问（接口方式 / `angular.z` 语义 / 驱动包 / 阿克曼参数 / 舵机脉宽）、"以现场实测为准"的退让条款与附件建议。目的是把供应商的回答成本压到最低，并**直接对应 §8.3 / §9.5 的落笔位置**（判定表见 D.6.3） | 附录 D.2.1 |
| R56 | **代码级深度审计与修复（8 类真实缺陷，使工程"可 build、可启动、可完赛"）**：① `pure_pursuit` 的 `speed_limit_by_state` 原以 **dict** 声明 → rclpy 不支持 dict/嵌套数组参数，节点**启动即崩**，改为 15 个扁平 `speed_limit_by_state.<STATE>` 参数；② `race_manager` 的 `vehicle.wheel_positions` 原为**嵌套数组** `[[...]]` → 同样启动即崩，摊平为 8 元素并两两还原；③ `_on_enter_state('MAPPING')` 原**无条件清零**本圈里程 → 过一次红绿灯里程票归零、`LAP1_COMPLETE` 永不触发，改为仅 `IDLE→MAPPING` 时清零；④ 切换步骤③/⑤ 原用 `can_transform(.., Time())` 判"TF 停发/恢复" → tf2 buffer 缓存最后一帧（cache_time 10 s）使其**恒真**，改为按 **TF 时间戳年龄**判定（新增 `_tf_stamp_age`）；⑤ `switch.timeout` 6.0 s **小于**六步分预算之和（≈15~18 s）→ 切换必被中止，**§9.7 与 §7.3、附录 F/K 同步改为 20.0 s**（本行即该参数变更的记录）；⑥ `_initial_stack_started` 原在 `start()` 前置位 → 失败不重试，改为仅在成功后置位并返回 bool，`initial_lap≥2` 时 IDLE 直接进 `NAVIGATING`；⑦ 看门狗恢复预算原**从不重置** → 3 次后永久无法恢复（静止 >10 s 判负），改为未触发时重置；`LOCALIZATION_RECOVER` 原**无超时退出** → 交通/入库永不可达，改为降级兜底后返回 `NAVIGATING`；⑧ 6 个 `package.xml` 补 `launch`/`launch_ros`/`ament_index_python` 依赖。**其余参数与 §9 保持一致，未新增/删除任何接口或话题。** | 代码（`autocar_ws/src`） |
| R57 | **解除"上车第一步就停死"的致命缺口 + 一键部署入口**：① 定死链路——`color.radial_threshold: 0.0`（未标定）→ `cone_detector` 把所有锥桶置 `unknown` → `classify_boundaries` 返回 `inner=outer=∅` → `build_midaxis` 抛 `InsufficientData` → `/race_line` **永不发布** → `pure_pursuit` 连一条指令都不发（`self._path_pts is None` 时直接 `return`）→ 车辆静止 >10 s **FAIL-03 判负**。② **修复**：`path_generator` 新增模块级 `auto_radial_split()`，在颜色判据不成立时用**锥桶点云质心当中线心 + 到质心距离的中位数为分界**自动分内/外（ADR-005 径向先验的自标定版），新增参数 `boundary.auto_calibrate=true`（默认开）与 `boundary.auto_min_per_side=2`（**已登记 §9.4**）；分侧方向对本流水线**对称**，故无需知道谁是红谁是蓝。③ 折线区收紧原用未标定的 `color.track_center_*(0,0)`，改为 `_effective_track_center()`：已标定用标定值，未标定用本周期边界点云质心。④ 新增 `scripts/deploy.sh`（`build` / `check` / `smoke` / `all`）作为**上车唯一入口**：`build` 校验 8 包 + 4 条自定义消息 + 各包可执行数 + 两个 launch 参数可解析；`smoke` 起全栈后**被动**逐段判定传感器→感知→规划→控制→状态机是否真的通，并对 `/race_line` 缺失给出可执行排查项。⑤ RUNBOOK §4 增"一条命令搞定"、§6.6 标注该标定项**现在不标定也能跑**。**未新增/删除任何话题或服务；`/race_line` 的语义与 1 Hz 频率不变。** | 代码（`autocar_ws/src/autocar_planning`、`autocar_ws/scripts/deploy.sh`）、§9.4、RUNBOOK |
| R58 | **把"必须实车完成的标定"收敛为可判定工具（标定门禁 + 标定向导）**：① 新增 `scripts/calib_check.py`——扫描 10 个 YAML，把散落的 ☐「待标定」按**不标定后果**分三级：**FATAL**（值为 0/占位 → 对应功能根本不存在，如 `lap.lap_length` 使里程票恒 False、`parking.zone_rect` 使 `PARKING→DONE` 永不可达、`extrinsics.*` 使点云整体偏移）、**MAJOR**（能跑但判据/几何失真，如 `wheelbase`/`max_steering_angle`（Q4/Q5 阻塞）、`vehicle.wheel_positions`、`servo.max_pwm_delta`、`protocol.tx_template`）、**MINOR**（纯整定项）；每条给出 `文件:行`、后果与**现成的标定命令**；另有**跨文件一致性**检查（`wheelbase` 与 `max_steering_angle` 须在 control/planning/chassis 三处一致）。退出码 1 = 存在 FATAL（fail-closed）。② 新增 `scripts/calib.sh` + `scripts/calib.py`——把标定做成一条命令：`calib.sh odom` 直行一段 + 卷尺实测 → 自动算 `wheel.scale_factor` 并打印回填行；`calib.sh lap` 手动跑一圈自动累计里程 → 打印 `lap.lap_length` 并连带给出 `lap.start_zone_rect`（起点 ±0.5 m 方框）与 `lap.start_yaw` 建议值。**只读里程、只打印回填值，不写任何配置文件。** ③ `deploy.sh check` 接入标定门禁（新增 `calib` 子命令 = `calib_check.py --all`），并把结论措辞区分为"编译/环境失败"与"仅存在致命未标定项"。④ RUNBOOK 新增 §6.0「标定向导」：分级含义表、**建议标定顺序**（IMU→外参→舵机/轴距→`wheel.scale_factor`→`lap_length`→区域位姿，前项是后项前提）、工具清单；§6.3/§6.7 增向导用法。**未改动任何参数取值、未新增/删除话题或接口。** | 代码（`autocar_ws/scripts/calib_check.py`、`calib.sh`、`calib.py`、`deploy.sh`）、RUNBOOK §5/§6.0/§6.3/§6.7 |
| R59 | **解除"第一圈永不完成"的致命缺口（圈次表决的两道防死锁兜底）**。① 定死链路：`lap.lap_length=0.0` 且 `lap.start_zone_rect=[0,0,0,0]` 时，里程票与发车区票**均恒为 False**，只剩"闭合代理"1 票 < `lap.detect_votes=2` → `LAP1_COMPLETE` 永不触发 → 车永远跑 MAPPING，**比赛无法完赛**。② 审计中进一步发现**该缺口比 `lap_length` 未标定更宽**：闭合代理票要求新 submap 与旧 submap 的**索引相距 ≥ `lap.revisit_index_gap`(8)**，而 cartographer 每约 90 帧才收尾一个 submap → **跑一圈只产生 1~2 个 submap**，故闭合票在第一圈**几乎不可能成立**。这意味着**即使 `lap_length` 已标定、只要 `start_zone_rect` 未标定，仍会卡死**。③ **修复一（标定无关替身）**：新增 `lap.heading_vote_enable`——`lap_length≤0` 时里程票改用"**净航向环绕 ≈ ±2π**"（`ΣΔyaw` 取自 `map→base_link`，与轮径/里程标定**完全无关**）；二者共同占里程票**这一票**，故票数结构未变（仍是 2-of-3），标定后精确里程判据自动恢复。三重门槛：本圈里程 ≥ `lap.heading_min_distance`(5 m)、单 tick `|Δyaw| ≤ lap.heading_max_step`(0.5 rad，超限视为 TF 跳变丢弃且仍推进参考航向以防累加器冻结)、`|ΣΔyaw| ≥ 2π(1−lap.heading_wrap_tol)`。**`:tol` 默认 0.10（非 0.25）**——数值仿真显示 tol=0.25 会在**一圈 76% 处**误触发，0.10 修正到**91%**（真实闭合点前约 1.1 m）。④ **修复二（quorum 自动降级）**：新增 `_quorum()`，按**实际可用票数**取 `max(1, min(detect_votes, 可用票数))`，"可用"仅计里程票（或其替身）与发车区票——**闭合代理票只作加分票、不计入 quorum**（理由见②）。两票都可用时 quorum 仍为 2，**§F.5.1 的三取二语义不变**；降级时打 WARN 明示原因。⑤ 兜底状态 `_hw_sum`/`_hw_last_yaw` 在 `IDLE→MAPPING` 与 `_lap_distance` 同处清零；`LAP1_COMPLETE` 日志与里程票日志均输出实时 `净航向环绕`，便于上车 `deploy.sh smoke` 一眼判定兜底是否生效。⑥ 数值仿真 5 用例全过（圆环 R=2.0 触发于 91%、直道噪声不误触发、TF 跳变被丢弃且累加器不冻结、原地掉头不误触发、最小转弯半径 R=1.5 可触发）。**未新增/删除任何话题或服务；未改动 `lap.detect_votes` 与 §F.5.1 的判据定义。** | 代码（`autocar_race_manager`）、`race_manager.yaml`、§9.7、§F.5.1、`calib_check.py`（`lap.lap_length`/`lap.start_zone_rect` 由 FATAL 降为 MAJOR 并标注"已有兜底"） |
| R60 | **解除"跑完两圈也停不下来"的致命缺口（入库链路的三道防死锁兜底）**。① 定死链路：`parking.target_x/y=0.0` 与 `parking.zone_rect=[0,0,0,0]` 均为占位时——(a) **坐标入口自触发**：`(0,0)` 恰是 `map` 原点=**发车线**（cartographer 以启动处为原点），第二圈一起步 `d≈0 ≤ parking.approach_distance` → **立刻误入 `PARKING_DETECT`**；(b) **`PARKING_DETECT` 原设计没有出口**（只能 → `PARKING`）→ 标志一直识别不到就以 `speed_limit_by_state.PARKING_DETECT=0.30 m/s` 慢速绕圈**到永远**；(c) **`PARKING→DONE` 不可达**：判据 `_wheels_in_zone()` 在 `zone_rect` 退化时 `_rect_valid=False` → 恒返回 0 < `in_zone_wheels_min`(3)，且 `PARKING` 在 `watchdog.exempt_states` 里 → **看门狗也不会救** → 跑完两圈也不结束比赛。② **修复一（视觉入口）**：新增 `parking.entry_by_sign`——第二圈 `/parking_sign/state=detected` 即**直接进 `PARKING`**，跳过坐标门禁；同时新增 `_parking_target_valid()`，把"目标恰在原点"视为**未标定**并**自动停用坐标入口**（否则第二圈起点必自触发），未标定时打 WARN 明示当前走的是哪条兜底。③ **修复二（`PARKING_DETECT` 出口）**：新增 `parking.detect_travel_limit`(8 m)——本态内行驶超过该里程仍无标志 → **放弃入库返回 `NAVIGATING` 继续比赛**；并置 `_parking_abandoned` **锁死坐标入口**，避免"进入→蠕动 8 m→退出→再进入"的反复循环。④ **修复三（`DONE` 兜底）**：新增 `parking.done_travel_fallback`(1 m)、`parking.done_min_travel`(0.3 m)、`parking.done_stall_speed`(0.05 m/s)、`parking.done_stall_duration`(2 s)——`zone_rect` 未标定时，`PARKING` 内"行驶 ≥ 1 m"**或**"已行驶 ≥ 0.3 m 且持续静止 ≥ 2 s"即判 `DONE`（完赛）；`zone_rect` 已标定时**整块不生效**，行为与设计卡 E-3 完全一致。⑤ 新增 `_parking_enter_dist`（入库链路里程基准）、`_parking_stall_since`、`_last_speed`（`_on_odom` 记录），三者使全部兜底判据均**与坐标标定解耦**（只用里程与速度）。⑥ 判据级仿真 6 用例全过（零标定+视觉入口→里程兜底 `DONE`；零标定第二圈起点不误入；零标定全程无标志不卡死；标志识别失败仅进入 1 次且锁生效；**已标定场景"≥3 车轮入区"主判据照常生效**；零标定入库后停稳 0.5 m → 静止兜底 `DONE`）。**未新增/删除任何话题或服务；未改动 `in_zone_wheels_min`、`stop_recognition_limit` 与设计卡 E-3 的判据定义。** | 代码（`autocar_race_manager`）、`race_manager.yaml`、§9.7、附录 G、`calib_check.py`（`parking.zone_rect` 由 FATAL 降为 MAJOR；新增 `parking.target_y` 条目） |
| R61 | **解除"第一圈第一个 tick 就在发车线停死"的致命缺口（红绿灯链路的三道防死锁兜底，采用"分级处理"）**。① 定死链路：`traffic.stop_point_x/y=0.0`（占位）时，(a) `(0,0)` 恰是 `map` 原点=**发车线**，`IDLE→MAPPING` 后**第一个 tick** 车就在原点，`_traffic_distance()≈0 ≤ traffic.approach_distance` → **红绿灯逻辑在发车线就被触发**，与真实灯位无关；(b) `_tl_confirmed` 的初值是**空串**，而 `TRAFFIC_APPROACH` 的三分支只认 `green`/`red`/`off|unknown` → **三者全不匹配** → 车**永久卡在 `TRAFFIC_APPROACH`** 以 0.30 m/s 空跑（该态**不在** `watchdog.exempt_states`，但车"在动"→ `static_time` 不累积 → **看门狗也不会救**）；(c) 灯态 `off`/`unknown` → `SAFE_HOLD`，该态**唯一出口是确认 `green`** 且它在 `watchdog.exempt_states` 内 → 灯不亮 / 相机看不到灯 / `traffic_light_detector` 未启动时**永久停放**。② **修复一（坐标入口自停用 + 视觉入口）**：新增 `_traffic_stop_point_valid()`，把"停车点恰在原点"视为**未标定**并**自动停用坐标入口**（否则第一圈第一个 tick 必误触发）；新增 `traffic.entry_by_light`(+`traffic.visual_trigger_states`，默认只 `red`)——未标定时改由 `/traffic_light/state` **确认红灯**时进入 `TRAFFIC_APPROACH`（位置无关、**边沿触发**，同一红灯光段只触发一次，防"驶离后又被持续红灯拉回"的反复进出）。⚠ 视觉入口**仅在坐标入口不可用时启用**：标定 `stop_point_*` 后只走坐标入口，行为与冻结设计完全一致（否则会在"刚看到红灯"的远处就停车，而非在停车点前停车）。③ **修复二（`TRAFFIC_APPROACH` 出口补齐）**：空串与 `off`/`unknown` **统一按"看不清"处理**并落 `SAFE_HOLD`（`灯不亮 ≠ 通行`，PEN-03），不再有"三分支全不匹配"的悬空态。④ **修复三（等待态分级超时放行）**：新增 `traffic_light.hold_timeout`(60 s，0=禁用) 与 `traffic_light.hold_timeout_states`(默认 `['SAFE_HOLD']`)——**分级处理**：`SAFE_HOLD`（**全程未确认红灯**：灯不亮 / 看不到灯 / 检测节点未起）等待超时后放行，避免停死判负，并打 **PEN-03 风险 WARN**；`TRAFFIC_STOP`（**已确认红灯**）**必停等绿、不放行**（闯红灯的判罚远重于超时等待）。⑤ **更正一处实现与设计不符**：`traffic_light.crawl_speed` 的"超时蠕动"**从未真正生效**——`_traffic_crawl` 只在本节点内部记录、**从未对外发布**，而 `autocar_control` 的限速表按 `/race_state` 取值（`SAFE_HOLD`/`TRAFFIC_STOP` 均为 0.0）→ **蠕动不会发生**。本次仅**如实更正 §9.7 与附录 E 的描述**（保留参数与 ADR-006 的原意），等待态的真实出口由 ④ 提供。⑥ 判据级仿真 10 用例全过（零标定+无灯态帧不卡死；零标定+灯不亮不进状态机；零标定+红转绿 视觉入口→STOP→GO；零标定+红灯持续 入口只触发 1 次且停在红灯前；已标定+灯态空串/灯不亮 → `SAFE_HOLD`→60 s 放行；已标定+确认红灯后灯灭 → **不放行**；已标定+红转绿 正常通行；零标定/已标定发车线静止均**无任何红绿灯迁移**）。**未新增/删除任何话题或服务；未改动 `traffic.approach_distance`、`exit_distance` 与设计卡 E-1/E-4 的判据定义。** | 代码（`autocar_race_manager`）、`race_manager.yaml`、§9.7、附录 E.4/E.7、`calib_check.py`（`traffic.stop_point_x` 由 FATAL 降为 MAJOR；新增 `traffic.stop_point_y` 条目） |
| R62 | **堵住"静态外参假通过"的静默错通道（双源一致性门禁 + 启动周期硬告警）**。① 定死链路：三个静态外参（`base_link`→`laser_link`/`imu_link`/`camera_link`）的**运行值**是 `race.launch.py` 里 `DeclareLaunchArgument` 的默认值（硬编码 `'0.0'`），而 `config/static_transforms.yaml` 只是**人读的标定记录、不参与运行**（`static_transform_publisher` 无 params-file 接口，只能吃命令行参数）。旧版 `calib_check.py` **只读记录文件** → 若只填记录、忘记改 launch，则：运行时 TF 仍是 **identity**（传感器与 `base_link` 完全重合）→ `/scan` 点云整体平移、锥桶与地图在 map 系整体偏移、EKF 姿态投影错误；而**门禁却报"已标定"** → 这是一条典型的"静默的假通过"通道，比"全 0"本身更危险。② **修复一（双源门禁）**：`calib_check.py` 新增 launch 文本解析 `launch_extrinsic_values()`（同时支持 `extrinsics[f'{child}.{dof}'] = '0.0'` 循环写法与 `extrinsics['laser_link.x'] = '0.18'` 逐项覆盖写法，后者优先），对每帧做**双源裁决**：**双源一致且全 0** → `laser_link`/`imu_link` 判 FATAL、`camera_link` 判 MAJOR（纯像素判据不依赖外参）；**双源不一致一律 FATAL**——只填记录 ⇒ "已填的值完全不生效，运行时仍是 identity"，只填 launch ⇒ "记录不可信，后续按记录复核/回填时会把它写回 0，外参在无察觉的情况下丢失"；任一处**无法解析则跳过**（不猜、不假通过）。③ **修复二（启动周期硬告警）**：`race.launch.py` 新增 `_warn_uncalibrated()`，通过 `OpaqueFunction` 读取**已解析的** `context.launch_configurations`（而非生成期的字面默认值——后者在命令行 `laser_link.x:=0.12` 覆盖时会误报，**假告警同样会让人不再相信告警**），三帧中任一是全 0 即打 P4 横幅：点名未标定帧、写明"两处同时回填（缺一即不生效）"、并声明当前走的是兜底。**只告警、不阻断**：identity TF **照常发布**（否则整棵 TF 树缺失、系统根本起不来）——即"**硬告警 + 安全降级**"，把静默错显式化；**刻意不编造几何默认值**（假值比 0 更危险，会被误当成实测值，且若与真实安装不符反而引入新的系统性偏差）。④ **回填方式收口**：RUNBOOK §6.2 取消"启动时命令行覆盖"作为标定手段（每次启动须重敲 18 个数值、漏一个即静默退化为 identity、且不在文件里留痕，双源门禁会**有意地**继续判其未标定），改为唯一路径：**改 `race.launch.py` 默认值（运行值）+ 改 `static_transforms.yaml`（记录值），两处同时回填**。⑤ 判据级仿真 **9 用例全过**（双源一致全 0 分级；只填记录 ⇒ 3× FATAL 不一致；只填 launch ⇒ 3× FATAL 不一致；双源都填且相等 ⇒ 无项；记录全 0 而 launch 仅逐项填 laser ⇒ 不一致 FATAL + imu FATAL + camera MAJOR；launch 仅逐项覆盖 laser ⇒ 其余帧无据可查而跳过；无 launch 文本 ⇒ 全部跳过；循环/逐项解析与覆盖优先级正确）；launch 侧仿真 **8 用例全过**（全 0 打横幅且点名三帧；命令行覆盖 laser 后横幅只点名 imu/camera ——证明**无假告警**；全部填满不打横幅；非法值不崩溃且该帧跳过；`generate_launch_description()` 正常返回）。**未新增/删除任何话题、服务或节点；未改动 §7.4 的外参定义与 TF 权属。** | 代码（`autocar_bringup`：`race.launch.py`、`config/static_transforms.yaml`）、`calib_check.py`、RUNBOOK §0/§6.0/§6.2 |
| R63 | **把"里程标度未标定"从 FATAL 降级为"安全降级 + 启动硬告警"（P5）**。① **定死链路**：`wheel.scale_factor` 的占位值是 **1.0（不是 0）**，且它**只**用于一处——`chassis_driver._tick()` 把编码器回帧速度换算成里程（`self._enc_speed = sample.speed * self.scale_factor`），**不参与任何下行指令**（舵机 PWM 由 δ 与限速决定）。故"值 = 1.0"≠"功能不存在"：若供应商 `ODOM` 回帧单位本就是 m/s，1.0 即为正确值。② **下游影响全部有兜底**：圈次已由"净航向环绕"替身与里程解耦（R59）；入库走视觉入口 + 里程/静止兜底（R60）；红绿灯走视觉入口 + 分级超时（R61）；`pure_pursuit` 是**闭环**跟踪（标度误差只影响精度，不至于失稳）；cartographer 以扫描匹配为主、odom 仅作先验（标度误差表现为"建图鬼影"）。故未标定**不导致功能缺失**，车仍能起步、走完两圈、正常停车。③ **修复（显式开关 + 启动硬告警）**：新增 `wheel.scale_calibrated`（默认 `false`）——`false` 时 `chassis_driver` 启动打 **P5 横幅**（点名 `wheel.scale_factor` 与后果：建图鬼影 / 跟踪精度 / 里程阈值偏移），**但里程照常发布**（**安全降级**，把静默的标度误差显式化）；刻意**不用"值 == 1.0"判未标定**——1.0 可能是对的，按值判会产生**假告警**（假告警会让人不再相信告警，与 R62 同一原则）。④ **门禁同步**：`calib_check.py` 删除 `wheel.scale_factor` 的 FATAL 条目，改为按 `wheel.scale_calibrated` 判的 **MAJOR** 条目（标注"已有安全降级 + 启动硬告警"，并说明若回帧单位即 m/s 则 1.0 可直接置 true）；`calib.sh odom` 的回填片段同时打印 `wheel.scale_calibrated: true`（否则 P5 告警不会消失）。⑤ RUNBOOK §0 / §6.0 / §6.3 / §11 与"已解除的上车即挂项"同步。⑥ 判据级验证：`chassis_common.yaml` 解析通过，`chassis_driver.py` / `calib_check.py` / `calib.py` 均 `py_compile` 通过，`calib_check.py --json` → **FATAL 2（仅 `extrinsics.laser_link` / `extrinsics.imu_link`）/ MAJOR 19 / MINOR 19**。**未新增/删除任何话题、服务或节点；未改动 §8.3 的接口语义与 §9.5/§9.8 的参数取值。** | 代码（`autocar_chassis_driver`：`chassis_driver.py`、`config/chassis_common.yaml`）、`calib_check.py`、`calib.py`、RUNBOOK §0/§6.0/§6.3/§11 |
| R64 | **把 QoS 核对从"模糊 grep"改为"逐端点兼容性判定"（P6）**。① **定死缺陷**：`preflight.sh` 原用 `ros2 topic info -v "$t" \| grep -qi "Reliability: ${want}"` 判定 QoS——但 `ros2 topic info -v` 会**逐个端点**打印 QoS，只要**任意一个端点**命中就通过。于是最致命的组合——**发布者 `BEST_EFFORT`、订阅者要求 `RELIABLE`**（在 ROS 2 中消息是**静默丢弃**、无任何报错）——**照样被判"通过"**；本项目第二圈 `nav2_amcl` 订阅 `/scan` 恰是这条静默失效通道，一旦踩中第二圈定位直接报废。② **修复**：新增 `scripts/qos_check.py`——用 `ros2 topic info -v` 的输出**逐端点解析**（`Endpoint type` / `Reliability` / `Durability`），按 ROS 2 QoS 兼容语义判定：**可靠性**（订阅 `RELIABLE` 要求发布者 ≥ `RELIABLE`；发布 `BEST_EFFORT` + 订阅 `RELIABLE` ⇒ 不兼容）、**持久性**（订阅 `TRANSIENT_LOCAL` 要求发布者亦为 `TRANSIENT_LOCAL`，否则后起的订阅者拿不到历史数据，如 `/map`）；**无发布者时不判定**（返回 `judged=False`，提示"待该话题有发布者后再跑"，绝不当作"已核对"）。默认核对 `/scan` `/scan_filtered` `/imu/data` `/camera/image_raw` `/map`。③ **接入**：`preflight.sh` ④ 段删除原模糊 grep 循环、改调 `qos_check.py`；`deploy.sh smoke` 新增"段 1.5"做同一判定（以"第二圈 AMCL 会废"作为后果提示）。④ **文档**：RUNBOOK §5 增判定规则与"第二圈才可核对的项"（`nav2_amcl` 仅第二圈激活，故 `/scan ↔ amcl` 端点对**只能在第二圈**核对，第一圈无 amcl 订阅者属正常）、§8.4 步骤②、§11 checklist、§1.3 表的 `/amcl_pose` 行；`amcl.yaml` 头注释由笼统的"无法离线验证"改为**给出判定规则 + 工具 + 本工程订阅者全为 Best-Effort 的事实**。⑤ 自检 **4 用例全过**（发布 best_effort↔订阅 reliable ⇒ 不兼容；双双 best_effort ⇒ 兼容；持久性 VOLATILE↔TRANSIENT_LOCAL ⇒ 不兼容；无发布者 ⇒ 不判定），另提供离线 `--text` 模式。**未新增/删除任何话题、服务或节点；未改动任何 QoS 取值。** | 代码（`autocar_ws/scripts/qos_check.py`、`preflight.sh`、`deploy.sh`）、`autocar_slam/config/amcl.yaml`、RUNBOOK §1.3/§5/§8.4/§11 |
| R65 | **堵住"Q6 只改一处"的静默转向尺度错误（Q4~Q7 配置驱动复核 + 双文件耦合门禁，P7）**。① **复核结论**：Q4/Q5/Q6/Q7 的答复**均可只改 YAML、不动代码**——Q5（阿克曼 `wheelbase` / `max_steering_angle` / `servo.neutral_pwm`）在 `chassis_common.yaml` 与 `control.yaml`/`planning.yaml`；Q4/Q7（停车检测区/停车区/标志牌区关系与尺寸）在 `parking.target_*` / `parking.zone_rect` / `parking.approach_distance`；Q6（底盘驱动接口）做成 `scenario_A/B/C` 三情形，由 `chassis.launch.py` / `race.launch.py` 的 `scenario` 启动参数切换（`kinematics.mode` / `drive.output` / `protocol.tx_template` / `protocol.rx_regex` 全在 YAML）。② **复核发现的真实缺口**：Q6 的 `/cmd_vel.angular.z` **语义写在两个文件里**——底盘侧 `scenario_*.yaml` 的 `kinematics.mode`（`delta`/`omega`）与 `drive.output`（`cmd_vel`/`steering_angle`），控制侧 `control.yaml` 的 `steering.convention` 与 `publish_steering_angle`。此前**没有任何一致性检查**：答 Q6 为情形 B 时若只换 `scenario:=B`（底盘按 ω 解释）而漏改 `control.yaml`（控制层仍按 δ 下发），或情形 C 时 `drive.output=steering_angle` 而 `publish_steering_angle=false`，结果是**车会动但转向量全错 / 前轮不转**——与 §8.3 FMEA 标注"极高"风险的"转向尺度误用"完全同源，且**静默无报错**。③ **修复（离线双文件门禁，纯检查、不改运行时行为）**：`calib_check.py` 新增 `q6_coupling()`——从 `race.launch.py` 取其 `scenario` 的 `DeclareLaunchArgument` 默认值（**真正生效的一方**），据此选中对应 `scenario_*.yaml`，与 `control.yaml` 的 `pure_pursuit`/`fuzzy_pid` 两节**自动比对**三条：(a) `kinematics.mode` ↔ `steering.convention` 必须相等；(b) `pure_pursuit` 与 `fuzzy_pid` 的 `steering.convention` 必须相等；(c) `drive.output=steering_angle` 要求 `publish_steering_angle=true`。任一不满足即列入"跨文件一致性冲突"（与既有 `wheelbase`/`max_steering_angle` 三处一致检查同格式）；**任一处解析不到则记为"不可判定"而非静默跳过**（与 R62 同一原则：不制造"假通过"）。④ **验证**：`calib_check.py` `py_compile` 通过；`--json` 结果与 R63 一致（**FATAL 2 / MAJOR 19 / MINOR 19**，冻结状态下 Q6 耦合问题数 = 0，即当前"情形 A + `delta` + `cmd_vel`"三处自洽）；判据级仿真 **5 用例全过**（冻结态 0 问题；只改 `scenario:=B` ⇒ 检出 `omega ↔ delta` 不一致；`scenario:=C` 且 `publish_steering_angle=false` ⇒ 检出"无 δ 通道"；`pure_pursuit` 与 `fuzzy_pid` 语义不一致 ⇒ 检出；解析不到 `scenario` 默认值 ⇒ 记"不可判定"）。⑤ RUNBOOK §0 第 3 条补"Q6 要改两处"、§6.0 跨文件一致性检查新增第 3 条。**未新增/删除任何话题、服务或节点；未改动任何参数取值、未改动 §8.3 的接口语义与 §9.5 的几何参数；未解除 Q4~Q7 的任何"待定"标记。** | 代码（`autocar_ws/scripts/calib_check.py`）、RUNBOOK §0/§6.0 |
| R66 | **把"代码从未在车机上编译过"收敛为可离线判定的静态预检（PZ 收尾：`build_check.py`）**。① **问题**：全工程 8 个包（3 个 `ament_cmake` + 5 个 `ament_python`）的代码**从未在 Ubuntu/ROS 2 上执行过 `colcon build`**，首次编译/首跑可能因若干**机械性**引用错误直接失败（`resource/<pkg>` 标记缺失 → `ament_index` 注册不到包 → `get_package_share_directory` 失败；`setup.py` 入口点指向不存在的模块/`main()`；`data_files` 的 `glob` 目录为空 → 安装后目录空；launch 互引/`config/*.yaml` 引用指向不存在的文件；`msg` 定义未列进 `rosidl_generate_interfaces`；`package.xml` 未声明内部依赖；`Node(executable=...)` 不在该包入口点里）。这类错误与算法正确性无关，**能在装 ROS 之前静态查出来**。② **交付**：新增 `scripts/build_check.py`——**纯标准库**（`ast`/`re`/`pathlib`/`json`，不 import 任何 ROS），逐项核对：**①包结构**（`package.xml`↔目录名、`<build_type>`、`resource/<pkg>`、`CMakeLists.txt` 的 `ament_package()` 与 `install(DIRECTORY ...)` 所列目录、构建类型二义）；**②入口点**（`console_scripts` → 模块文件与 `def main(` 是否真的存在、可执行名跨包唯一）；**③data_files**（`glob('config/*.yaml')` 目录存在且非空、字面路径存在）；**④launch 互引**（`_include('pkg','x.launch.py')` 与 `StackSupervisor.start('pkg','x.launch.py')` 指向的本工作空间 launch 文件存在；**刻意不认裸字面量**——`'imu.launch.py'`/`'camera.launch.py'`/`'lidar.launch.py'` 是外部供应商驱动包的默认名，见 `sensors.launch.py` 的 `_FILE_HINTS`，按字面量全局查会**误报**）；**⑤config 引用**（launch 内 `os.path.join(share,'config','x')` 指向的 YAML 存在）；**⑥消息定义**（`msg/srv` 文件 ↔ `rosidl_generate_interfaces` 双向一致、`<member_of_group>rosidl_interface_packages`、Python 侧 `from autocar_msgs.msg import ...` 的名字存在）；**⑦依赖声明**（代码 `import` 与 launch 引用的 `autocar_*` 是否在 `package.xml` 声明，用 `ast` 只收 import 节点以避开中文注释里的包名误报）；**⑧可执行体**（`Node(package='autocar_x', executable='y')` 的 `y` 是该包已注册入口点，支持 `executable=exe` + `for exe in executables` 变量池解析）；**⑨语法**（全部 `.py` 的 `ast.parse`）。退出码：存在 FAIL → 1（fail-closed），支持 `--json`。③ **接入**：`deploy.sh check` **最前置**执行本步（它不需要 ROS，应在 `preflight.sh` 与标定门禁之前先跑），任一 FAIL 即置 `EXIT=1`；`deploy.sh` 头注释同步。RUNBOOK §4 增"第 0 步：静态预检"、文件索引新增一行。④ **本机验证**：`python scripts/build_check.py` → **8 项 ok / 0 warn / 0 FAIL，exit 0**（包结构 8 个；入口点 10 个可执行名；安装清单 15 条；launch 交叉引用 config 9 条 · launch 8 条；msg 4 个；依赖 0 处缺失；可执行体 10 个；.py 语法 34 个全过）。**诚实声明**：这是**必要条件**检查——全过 ≠ 一定能编译（ROS 依赖是否装齐仍需上位机 `deploy.sh all` 验证），但**任一 FAIL 基本等于首跑必炸**。**未新增/删除任何话题、服务或节点；未改动任何参数取值与接口语义。** | 代码（`autocar_ws/scripts/build_check.py`、`deploy.sh`）、RUNBOOK §4/文件索引 |
| R67 | **按"真实现实比赛要求"深度审计后修复的头号可完赛性缺陷：红绿灯未先越过光电传感器就停车（规则时序违规 → 永久停放 → 判负）**。① **规则原文**（`ros小车.txt` §4.2 时序逻辑注意）："红绿灯前的光电传感器是外部触发装置，倒计时从车辆遮挡传感器时开始。**车辆必须在通过光电传感器之后才开始减速，而不是看到红灯就停，否则倒计时可能不会启动**。"② **定死链路**：未标定 `traffic.stop_point_*` 时坐标入口自动停用、改走**视觉入口**（R61）；视觉入口在 `/traffic_light/state` **首次确认红灯**的那一刻就进 `TRAFFIC_APPROACH`，而**旧实现的 `TRAFFIC_APPROACH` 分支在同一个 tick 内就无条件转 `TRAFFIC_STOP`**。相机看到红灯的距离可能远于停车点（150° 镜头 + 10 m 量级识别距离）→ **车停在光电传感器之前** → 倒计时不启动 → 红灯永不转绿 → `TRAFFIC_STOP` 在产品设计上"**必停等绿、不放行**"（R61 ④，为避 PEN-03）+ 该态在 `watchdog.exempt_states` 内（看门狗不救）→ **车永久停放，比赛无法完成**。③ **修复（新增 `traffic.min_approach_travel`，默认 2.0 m）**：`TRAFFIC_APPROACH` 的 `red` 分支改为"**先前进 ≥ `min_approach_travel` 米（以 `TRAFFIC_APPROACH` 档速 0.30 m/s 前进）才允许转停车态**"，即**先越过光电传感器触发倒计时，再减速停车**；未达门槛时打一次性 WARN 明确说明"暂不减速"的原因。④ **有界时间兜底**：另设 `t_cap = max(10.0, 3 × min_approach_travel / 0.30)`——若里程计失效/空转导致永远走不到该里程，则在 `t_cap` 后按时间转入停车态（否则将永久以 0.30 m/s 空跑）；触发时打 WARN 点名"里程计可能失效"。⑤ **不改冻结态行为**：标定 `traffic.stop_point_*` 后走坐标入口（§R61 ①的"只走坐标入口"），`TRAFFIC_APPROACH` 由坐标距离触发，本分支的"继续前进"会很快满足门槛，行为与冻结设计一致；`min_approach_travel` 置 0 即完全回退旧行为。⑥ 判据级验证：`py_compile` 通过、`build_check.py` 8 项 ok/0 warn、`race_manager.yaml` `yaml.safe_load` 通过。**未新增/删除任何话题、服务或节点；未改动 `traffic.approach_distance`、`exit_distance`、`hold_timeout*` 与设计卡 E-1/E-4 的判据定义。** | 代码（`autocar_race_manager/race_manager.py`）、`race_manager.yaml`、§9.7、附录 E.4 |
| R68 | **修复第二个可完赛性缺陷：停车标志始终识别不到时 `DONE` 不可达（无休止跑圈 → 判负）**。① **定死链路**：`PARKING_DETECT` 内行驶超 `parking.detect_travel_limit`(8 m) 仍无标志 → 放弃入库回 `NAVIGATING` 并置 `_parking_abandoned=True`（R60 ③，为防"进入→蠕动 8 m→退出→再进入"的反复循环）。但该锁**同时锁死了坐标入口**，此后入库**只能**靠视觉入口（`/parking_sign/state=detected`）。② **缺口**：若停车标志识别**始终失败**（光照/角度/标志牌规格不符/`parking_sign_detector` 未起），则坐标入口被锁、视觉入口不成立 → `PARKING` 与 `DONE` **均不可达**，且 `NAVIGATING` 不在 `watchdog.exempt_states` 但车"一直在动"→ 看门狗也不触发 → **车在赛道上无休止跑圈，比赛永不结束（判负）**。③ **修复（新增 `parking.abandon_done_travel`，默认 10.0 m）**：在 `NAVIGATING` 分支新增**兜底完赛出口**——在 `_parking_abandoned` 为真且 `abandon_done_travel > 0` 时，若"放弃入库后又行驶 ≥ `abandon_done_travel` 米"仍未见标志，则**强制 `DONE`**（`_parking_abandon_dist` 在放弃点记录里程基准）。④ **取舍已显式化**：强制 `DONE` 可能被判"停车不在区域内" **+20 s**；但 **DNF/判负的代价远大于 20 s**，故主动取舍为"**优先完赛**"；转 `DONE` 时打 WARN 完整说明该取舍、以及"修复标志识别或标定 `parking.target_*` 后本兜底不再触发"。⑤ `abandon_done_travel = 0` 即**整体禁用**该兜底（仅当确认标志识别 100% 可靠时才建议禁用）。⑥ 判据级验证：`py_compile` 通过、`build_check.py` 8 项 ok/0 warn、`race_manager.yaml` `yaml.safe_load` 通过。**未新增/删除任何话题、服务或节点；未改动 `parking.detect_travel_limit`、`done_*`、`in_zone_wheels_min` 与设计卡 E-3 的判据定义。** | 代码（`autocar_race_manager/race_manager.py`）、`race_manager.yaml`、§9.7、附录 G |
| R69 | **修复第三个可完赛性缺陷：从未收到 `/race_line` 时纯追踪"一条指令都不发"（车静止 >10 s 判负，FAIL-03）**。① **定死链路**：`path_generator` 需 `/cones` ≥ `min_cones`(6) **且** Voronoi 构造成功才发 `/race_line`（`_replan` 失败时"保持上一次路径"——**若从未成功过则一次也不发**）。② **缺口**：发车线附近若锥桶不足（车头朝向开阔区、锥桶被车体自身遮挡、DBSCAN 门限偏严）或 Voronoi 退化，则 `/race_line` **永不发布** → `pure_pursuit._on_timer` 走"路径未收到"分支：旧实现仅打 WARN 并 `return`（`_last_cmd is None` 时**什么也不发**）→ `fuzzy_pid` 收不到 `/cmd_vel_raw`（其 `_on_timer` 在 `_raw is None` 时同样直接 `return`）→ **底盘无任何输入 → 车静止 >10 s 判负**。注意本缺口**不是**"车不知道该往哪走"，而是"**整条链路静默空转**"。③ **修复（新增 `bootstrap.enable`(true) 与 `bootstrap.wait_timeout`(3.0 s)）**：`pure_pursuit` 在"**从未**收到任何 `/race_line`"且距节点启动 ≥ `wait_timeout` 时，发布**直线蠕动**指令（`v = max(speed_min_nonzero, 1e-3)`、`δ = 0`，按 `steering.convention` 换算 `angular.z`），并打 ERROR 点名根因（`/cones` 不足或 Voronoi 失败）；**`/race_line` 一旦到达即自动退出本兜底**、恢复常规纯追踪（`_last_cmd` 会被正常路径覆盖）。④ **参数约束**：`wait_timeout` **必须 < `watchdog.static_timeout`(9 s)**（默认 3.0 s 留有充分余量），否则看门狗会先触发。⑤ 与 R63 同一原则——"**不是发 0，而是发一个明确的最小前进量**"，把"静默不动"变成"可观测的蠕动 + 显式告警"。⑥ 判据级验证：`py_compile` 通过、`build_check.py` 8 项 ok/0 warn、`control.yaml` `yaml.safe_load` 通过。**未新增/删除任何话题、服务或节点；`bootstrap.enable=false` 即完全回退旧行为；未改动 `speed_limit_by_state` 与 §9.5 的任何取值。** | 代码（`autocar_control/pure_pursuit.py`）、`control.yaml`、§9.5、附录 J.6 |
| R70 | **修复第四个可完赛性缺陷（R68 的逻辑缺口）：零标定下 R68 永不触发 → 第二圈无休止跑圈 → 判负（DNF）**。① **定死链路**：R68 的兜底出口挂在 `_parking_abandoned` 上，而该标志**只在 `PARKING_DETECT` 内被置位**（`race_manager.py` `_drive_state_machine` 的 `PARKING_DETECT` 分支），`PARKING_DETECT` 又**只能由坐标入口进入**（`_maybe_enter_parking` ②，且该入口要求 `_parking_target_valid()`，即 `parking.target_*` 非 (0,0)）。② **缺口**：`parking.target_*` 未标定时（零标定，占位 (0,0)，恰为地图原点=发车线）坐标入口被 `_parking_target_valid()` **自动停用** → **永远进不了 `PARKING_DETECT`** → `_parking_abandoned` **恒为 False** → **R68 永不触发**。此时若**视觉入口**（`/parking_sign/state`，蓝底+白字 CV，`confidence_threshold=0.70`）也识别失败（光照/角度/标志规格不符/相机问题），则 `PARKING` 与 `DONE` **均不可达**，`NAVIGATING` 不在 `watchdog.exempt_states` 但车"一直在动"→ 看门狗不触发 → **车在赛道上无休止跑完第二圈又一圈，比赛永不结束（判负）**。**注意：这正是 R68 本要覆盖的场景，却因"入口被停用 ⇒ 放弃标志永不置位"而恰好失效——是纯粹的逻辑缺口，不是参数问题。** ③ **修复（新增 `parking.lap2_force_done`(true) 与 `parking.lap2_force_done_travel`(0.0 m)）**：在 `NAVIGATING` 分支新增**第二圈完赛保障**——用与第一圈里程票**同源且标定无关**的判据"净航向环绕"（闭合回环净转角 ≈ ±2π，与轮径/里程标定无关）判定**第二圈已绕回起点**（新方法 `_lap2_loop_complete`，三重门槛同 `_vote_heading_wrap`：里程下界 `lap.heading_min_distance` + 单 tick 跳变丢弃 `lap.heading_max_step` + 净转角阈值 `2π(1−lap.heading_wrap_tol)`）；若此时**入库仍未被触发**（状态仍为 `NAVIGATING`，即 `PARKING`/`DONE` 均未达成）则**强制 `DONE`**。第二圈进度基准在 `_finish_switch` / `_switch_abort` 里由 `_reset_lap2_progress()` **快照**（否则会把第一圈余额算进来 → 判据提前成立）。④ **附加纯里程退路**：`parking.lap2_force_done_travel > 0` 时，第二圈里程达该值亦强制 `DONE`（供航向不可用时的退路；0 = 只用航向判据，默认）。⑤ **取舍已显式化**：强制 `DONE` 可能被判"停车不在区域内" **+20 s**；但 **DNF 的代价远大于 20 s**，故主动取舍为"**优先完赛**"——与 R68 同一政策。⑥ 与 R68 的**分工**：R68 处理"**已标定（坐标入口可用）→ 进过 PARKING_DETECT → 放弃**"的情形（10 m 触发，更早）；R70 处理"**零标定（坐标入口被停用）→ 从未进过 PARKING_DETECT**"的情形（绕完一圈触发）。两者互不冲突（R68 更早触发时先命中）。⑦ 回退：`parking.lap2_force_done=false` 即完全回退旧行为。⑧ 判据级验证：`py_compile` 通过、`build_check.py` 8 项 ok/0 warn、`race_manager.yaml` `yaml.safe_load` 通过。**未新增/删除任何话题、服务或节点；未改动 `parking.target_*`、`zone_rect`、`detect_travel_limit`、`abandon_done_travel` 与设计卡 E-3 的判据定义。** | 代码（`autocar_race_manager/race_manager.py`）、`race_manager.yaml`、§9.7、附录 G |
| R71 | **修复 R69 兜底的**自身副作用**：bootstrap 蠕动不区分状态 → 在 `IDLE` 会抢跑、在等红灯态会闯红灯（PEN-03 +30 s）**。① **定死链路**：R69 的直线蠕动在"从未收到 `/race_line`"且超 `bootstrap.wait_timeout` 时**无条件**发布（不看 `/race_state`）。② **缺口**：节点启动到建图栈就绪（`IDLE`）通常 > 3 s，此时蠕动 = **抢跑**；若在 `TRAFFIC_STOP` / `SAFE_HOLD`（等红灯）蠕动 = **闯红灯**（PEN-03，+30 s；且规则要求"看清再行"）；在 `PARKING` / `DONE` 蠕动会**撞出停车区**。R69 的本意只是"防 FAIL-03 静止判负"，不应在任何"产品设计上要求停车"的状态下生效。③ **修复（新增 `bootstrap.hold_states` 黑名单）**：默认 `['', 'IDLE', 'TRAFFIC_STOP', 'SAFE_HOLD', 'PARKING', 'DONE']`，命中即**不蠕动**（空串=尚未收到 `/race_state`，不可假设可行驶），并打一次性 WARN 提示"若该状态本应行驶，检查上游是否漏发 `/race_state`"。④ **不削弱 R69 的保护**：蠕动仍在 `MAPPING`/`NAVIGATING` 等**应行驶但无路径**的状态生效（这才是 FAIL-03 的真实风险场景）；黑名单可配置。⑤ 判据级验证：`py_compile` 通过、`build_check.py` 8 项 ok/0 warn、`control.yaml` `yaml.safe_load` 通过。**未新增/删除任何话题、服务或节点；`bootstrap.hold_states: []` 即完全回退 R69 的无状态门控行为；未改动 `speed_limit_by_state` 与 §9.5 的任何取值。** | 代码（`autocar_control/pure_pursuit.py`）、`control.yaml`、§9.5、附录 J.6 |
| R72 | **修复第五个可完赛性缺陷：`TRAFFIC_STOP`（已确认红灯）的唯一出口是绿灯，一旦"倒计时未启动或灯故障"就永久停放 → 判负（DNF）**。① **定死链路**：`TRAFFIC_STOP` 语义为"**已确认红灯** → 必停等绿、**不放行**"（R61 ④，为避 PEN-03），其**唯一出口**是 `_tl_confirmed == 'green'`（`_drive_state_machine` 的 `TRAFFIC_STOP`/`SAFE_HOLD` 分支首个判断）。而 green 只在"车辆**越过光电传感器**触发倒计时"（规则原文，`ros小车.txt` §4.2）后才出现；该态又在 `watchdog.exempt_states` 内（看门狗不救）。② **缺口**：R67 用 `traffic.min_approach_travel` 保证"标定正确时先越过传感器再停"，但该值若**标定偏小**（车仍停在传感器之前），或光电传感器/信号灯本体**故障**，则倒计时永不启动 → 绿灯永不到来 → 车**永久停放 → 判负**。这是零标定/标定错误下**最后一条**DNF 链路。③ **修复（新增 `traffic_light.stop_hard_timeout`，默认 90.0 s，0 = 禁用）**：在 `TRAFFIC_STOP`/`SAFE_HOLD` 分支的**兜底放行之后**新增一条**仅对 `TRAFFIC_STOP` 生效**的最后硬超时——等待 ≥ `stop_hard_timeout` 仍无 green → 强制转 `TRAFFIC_GO` 放行，并打 **ERROR** 级日志完整说明取舍、PEN-03 风险与三条根因排查路径。④ **取值理由**：倒计时一旦触发通常在 10~30 s 内转绿，90 s **远超**该量级 → **正常比赛绝不会误触发**；仅当"倒计时确实未启动/灯故障"才可能命中。⑤ **取舍已显式化**：可能被判"未绿灯通行" **+30 s**（PEN-03）；但 **DNF 的代价远大于 30 s**，故按"**优先完赛**"取舍——与 R68/R70 同一政策。⑥ **不削弱安全语义**：`SAFE_HOLD`（未确认红灯）仍走原 `hold_timeout`(60 s) 分级放行；`TRAFFIC_STOP` 在 60 s 内**仍然必停等绿**，只有超过 90 s 的**硬超时**才放行，两者互不干扰；`stop_hard_timeout=0` 即完全回退旧行为。⑦ 判据级验证：`py_compile` 通过、`build_check.py` 8 项 ok/0 warn、`race_manager.yaml` `yaml.safe_load` 通过。**未新增/删除任何话题、服务或节点；未改动 `traffic.approach_distance`、`exit_distance`、`hold_timeout`、`hold_timeout_states`、`min_approach_travel` 与设计卡 E-1/E-4 的判据定义。** | 代码（`autocar_race_manager/race_manager.py`）、`race_manager.yaml`、§9.7、附录 E.4 |
| R73 | **修复第六个可完赛性缺陷：挡板误合并 → `/cones` 骤降 → 路径饥饿 → 车辆静止/蠕动判负**。① **定死链路**：`path_generator._replan` 的首道门槛是 `len(self._cones) < min_cones(6)` 即**不出路径**（返回 None，"保持上一次路径"；**从未成功过则一条都不发**）。而 `cone_detector._merge_boards` 会把"相邻簇质心间距 ≤ `board.merge_gap`(0.50 m)、簇数 ≥ `board.merge_min_clusters`(3)、且共线 RMS ≤ `board.collinear_rms_max`(0.08 m)"的一组簇判为**挡板**，并**从 `/cones` 移除**（转入 `/boards`）。② **缺口（两层）**：(a) **误判本身** —— 旧实现**完全没有长度下界**，一排**间距 < 0.5 m 的密集/共线锥桶**或**噪点碎块**（跨度可能仅 0.1~0.4 m）恰好满足"多簇 + 共线"即被判为挡板；在赛道宽 1.5 m、折线区锥桶成对/加密的排布下，这是一种**结构性误判**（几何上"等距共线的锥桶排"与"挡板"不可分）。(b) **误判的后果被放大** —— 由于门槛只数 `/cones`，误判直接把锥桶从计数里拿走 → 触发 `min_cones` → **永不发布 `/race_line`** → `pure_pursuit` 无输入（走 R69 蠕动兜底）或静止 > `watchdog.static_timeout`(9 s) → **判负**。**注意：点到其实并未丢失**（同一批点转成了 `/boards` 的 `Board.points`，而 `collect_candidate_points` / `classify_boundaries` **本就同时消费 cones 与 boards**），所以 (b) 是**纯粹的门槛口径错误**。③ **修复（两处，配置驱动）**：**(i) `path_generator` 门槛改口径** —— 新增 `min_boundary_points`(默认 0 = 自动取 `min_cones`)，门槛改为"`锥桶数 < min_cones` **且** `锥桶 + 挡板面点合计 < min_boundary_points`"才拒绝，即**只要几何点够就照常规划**，彻底消除"误合并 ⇒ 路径饥饿"。**(ii) `cone_detector` 加长度下界** —— 新增 `board.min_length`(默认 0.60 m)：`_fit_board` 拟合长度短于此值即**不判挡板**（继续留在 `/cones`），从根因上把"短碎块/密集锥桶排"挡在门外，同时保留对真实长条挡板的识别。④ **取值理由**：`min_boundary_points=0`（= 6）保持"几何充分性"尺度与旧口径一致，**不放松**规划条件；`board.min_length=0.60` 基于本赛挡板尺寸（CR-14 记录挡板长约 0.80 m）留 ~25% 裕度，☐ 现场按实际最短挡板与锥桶间距复核。⑤ **双保险的独立性**：(i) 保证"即使误判也不会饿死路径"，(ii) 降低误判率本身；两者任一单独生效即可解除 DNF，(ii) 还额外改善颜色标定后 `classify_boundaries` 的内/外边界纯度。⑥ **回退**：`board.min_length=0` 关闭 (ii)；`min_boundary_points` 恢复为"必须 ≥ 锥桶数"即可回退 (i) 的口径（但不建议）。⑦ 判据级验证：`py_compile` 通过、`build_check.py` 8 项 ok/0 warn、`planning.yaml` 与 `perception.yaml` `yaml.safe_load` 通过。**未新增/删除任何话题、服务或节点；未改动 `/cones`、`/boards` 的消息结构与发布频率；未改动设计卡 E-7 的 Voronoi/样条/曲率约束判据。** | 代码（`autocar_planning/path_generator.py`、`autocar_perception/cone_detector.py`）、`planning.yaml`、`perception.yaml`、§9.1、§9.4、附录 H.5.4、K.5 |
| R74 | **A4 端到端数据流审计：修复两处"静默全链路失效"类缺陷（坐标系错用 + 外部 QoS 不兼容）**。**(A) 坐标系错用（`pure_pursuit._lookup_pose`）**：① 链路：`/race_line` 的 `frame_id = map`（`path_generator` 第 861 行），`pure_pursuit` 用它做"最近点 / 前视点"匹配，位姿**必须同为 map 系**。② 缺口：旧回退分支在 TF `map←base_link` 瞬时失败（0.1 s 超时、cartographer 回环闭合跳变、AMCL 未收敛）时，**直接把 `/odometry/filtered` 的 odom 系位姿当 map 系使用**，**未叠加 `map←odom` 变换**。而 `map→odom` 的平移可达**数米**、航向可为**任意值** → 前视点整体偏置 → `δ = arctan(2L·sin α / L_d)` 计算完全错误 → 冲出赛道（FAIL-01 / 撞锥桶）。③ 修复：回退时**再取 TF `map←odom` 并做刚体合成** `p_map = R(ψ_map_odom)·p_odom + t_map_odom`、`yaw_map = wrap(yaw_odom + ψ)`；若 `map←odom` 也不可用则**返回 `(None, None)`**，由 `_on_timer` 安全保持上一指令（**宁可保持，不得错系驾驶**）。**(B) 外部 QoS 不兼容（5 个订阅节点）**：① 链路：`/odometry/filtered`（发布者 = **外部** `ekf_node`）被 `pure_pursuit`/`fuzzy_pid`/`watchdog`/`path_generator`/`race_manager` 五处订阅，旧实现**全部用 RELIABLE 订阅**；`/amcl_pose`（外部 `nav2_amcl`）、`/submap_list`（外部 `cartographer`）亦然。② 缺口：DDS 兼容规则下 **BEST_EFFORT 发布者 ↔ RELIABLE 订阅者不兼容，且不报错（订阅端静默零数据）**；而 Reliable/Best-Effort 发布者 ← BEST_EFFORT 订阅者**均兼容**，即 **BEST_EFFORT 订阅的兼容集严格更大**。若上游（Nav2 / robot_localization 历史上都用过 SensorDataQoS）实际为 Best-Effort，则**整条链路静默失效**：无里程 → 不出路径 / 不计圈 / 看门狗失效；`/amcl_pose` 无数据 → `LOCALIZATION_RECOVER` 永不触发。旧文档把这事留作"**必须实机 `ros2 topic info -v` 核对，不得假定**"的隐性前提。③ 修复：新增 `ODOM_QOS`(4 个控制/规划节点) 与 `EXTERNAL_QOS`(race_manager，覆盖 `/odometry/filtered`、`/amcl_pose`、`/submap_list`)，统一 **Best-Effort / Volatile / depth 5**。对**两种上游 QoS 都成立**，从而**消除**"必须先实机核对上游 QoS"的隐性前提（50 Hz 高频流丢几帧对控制无影响）。**内部控制话题**（`/cmd_vel`、`/cmd_vel_raw`、`/race_state`、`/cones`、`/boards`、`/race_line`、`/watchdog/*`——发布者均为本项目节点）**仍用 RELIABLE，不受影响**。**注**：R74 使设计文档中原"`/amcl_pose` 需实机核对 QoS"的**核对项消解**；`/scan` 订阅原即为 `qos_profile_sensor_data`(Best-Effort)，本就无此问题。④ 判据级验证：5 个节点 `py_compile` 通过、`build_check.py` 8 项 ok/0 warn、无残留的旧符号 `AMCL_POSE_QOS` 引用。**未新增/删除任何话题、服务或节点；未改动任何消息类型、发布频率与坐标系命名；未改动 §8.4 中"内部控制话题用 Reliable"的基线。** | 代码（`autocar_control/pure_pursuit.py`、`fuzzy_pid.py`、`watchdog.py`、`autocar_planning/path_generator.py`、`autocar_race_manager/race_manager.py`）、§8.3、§8.4、附录 I.5.4 |
| R75 | **A4 端到端数据流审计（续）：把 R74 的"坐标系错用"修复**补齐到同一缺陷的另外两处**（`fuzzy_pid` / `path_generator`），实现全工程位姿契约统一为 map 系**。① **审计方法与结论**：对全工作区 `create_publisher/subscription`（48 处）与 QoS 定义（69 行）做交叉核对，并逐节点核对"TF `map←base_link` 失败时如何取位姿"。结论：**内部话题 QoS 全部自洽**（`/cones` `OUTPUT_QOS`(Reliable) ↔ `path_generator` Reliable ✓；`/traffic_light/state`、`/parking_sign/state` `STATE_QOS`(Reliable) ↔ `race_manager` Reliable ✓；`/watchdog/*` Reliable ↔ Reliable ✓；`/scan`↔`/scan_filtered` 两端 Best-Effort ✓；`/camera/image_raw` Best-Effort 订阅 ✓），**坐标系错用只剩 R74 已修的 `pure_pursuit` 与此处两处**。② **缺口（与 R74-A 同一根因）**：`fuzzy_pid._lookup_pose` **直接把 `/odometry/filtered` 的 odom 系位置当 map 系**返回 —— 该值喂给 `_lateral_error`（横向误差，用于 PID/模糊纠偏），`map→odom` 非 identity 时误差**整体偏置** → 纠偏方向错误或发散（弯道冲出，FAIL-01）；`path_generator._lookup_pose` 同样直接把 odom 系位姿当 map 系返回 —— 该值喂给 `_align_start`（闭环起点与行进方向选择），`map→odom` 偏移大时**起点选错**（虽为闭环不致断链，但起点/方向非确定，属隐性错）。③ **修复**：两处**同 R74-A 手法**——回退时再取 TF `map←odom` 做刚体合成 `p_map = R(ψ_map_odom)·p_odom + t_map_odom`，`fuzzy_pid` 只需位置、`path_generator` 另合成 `yaw_map = yaw_odom + ψ`；`map←odom` 亦不可用则返回 `None/（None,None）`（**宁可保持上一输出，绝不混用坐标系**）。④ **安全性旁证（审计确认无需改动的节点）**：`race_manager._pose_map` 无帧混用（`_lookup_xyyaw('map','base_link')` 失败返回 `None` → 用缓存 `_last_map_pose`），且其 `/odometry/filtered` 仅用于**标量里程累计与速度**（无帧依赖）；`cone_detector` 直接以 `scan.header.frame_id` 查 TF 且**失败即跳帧**（无 odom 回退）。⑤ 判据级验证：3 个节点（`pure_pursuit`+`fuzzy_pid`+`path_generator`）`py_compile` 通过、`build_check.py` 8 项 ok/0 warn。**未新增/删除任何话题、服务或节点；未改动任何参数取值、消息类型与发布频率；未改动 §8.4 QoS 基线与 §7 TF 权属。** | 代码（`autocar_control/fuzzy_pid.py`、`autocar_planning/path_generator.py`）、§7、§8.3、附录 I.5.4 |
| R76 | **交付物补齐：上车首跑操作单 + 离线预演脚本（不改任何运行代码）**。① 背景：A1~A6 审计完成后，剩余未闭环项全部依赖外部输入（Q1~Q12 答复、实机标定、车机首跑），本机可做的最后一件事是把"第一次上车"从"边查边试"压缩成"照着按、照着判"。② 新增 `autocar_ws/上车首跑操作单.md`：阶段 A→G（环境/编译/驱动/标定/冒烟/首跑/验收），每步含「命令/预期/PASS 判据/失败分支」，置顶两条红线（绿灯才放行、静止 >10 s 判负），含状态流判定表（每阶段期望现象 ↔ R67/R68/R69/R70/R72 兜底触发日志）与一页速查表/排障速查。③ 新增 `scripts/offline_check.py`（纯标准库 + 可选 PyYAML，Windows 本机可跑）：一键串联 5 项离线检查——全量 `.py` `py_compile`、全量 `.yaml` `safe_load`、`build_check.py` 8 项、`calib_check.py`（**FATAL 记 INFO**：属已知待标定基线而非机械错误，退出码不参与 FAIL 判定；退出码 ≥2 才判 FAIL）、自动探测 Git Bash 跑 `bash -n` 全量 `.sh`；报告落盘 `OFFLINE_REPORT.txt`。④ 同步：RUNBOOK 文件索引与核对清单 `.md` 版头部加操作单链接；操作单新增阶段 B0。⑤ 验证：本机实跑 `offline_check.py` → **FAIL=0 / SKIP=0 / INFO=1**（40 py、14 yaml、8 项预检、FATAL 2 基线、5 sh 全过），退出码 0。**未改动任何运行代码、参数、话题与文档既有判据；两份新文件均为交付辅助物。** | `autocar_ws/上车首跑操作单.md`、`scripts/offline_check.py`、`RUNBOOK.md`、`CHECKLIST_实车联调前核对.md` |
| R77 | **交付完整性终审（launch↔代码↔YAML 接线全量核对）：修复唯一"改了没用"类缺陷——`amcl.yaml` 死段 `amcl_health_monitor:`**。① **审计范围**：对 `race.launch.py` include 链、各包 launch 的 `parameters=[...]` 传递、14 个 YAML 顶层节点键与 launch 节点名匹配、5 个 ament_python 包 `setup.py` 的 `data_files`（`share/<pkg>/config`+`launch`）、2 个 ament_cmake 包 `CMakeLists.txt` 的 `install(DIRECTORY ...)`、scenario A/B/C 联动、`map_yaml:=` 传递、`lifecycle_manager_localization` 命名与 race_manager 服务调用一致性、全部 remap 逐项核对；并对 279 个 YAML 叶子键做代码交叉引用核对（49 个"代码找不到"全部落在 `autocar_slam/config` 的 amcl/ekf/map_saver，由**外部 nav2 / robot_localization 节点**消费，属预期）；`wheelbase`(0.30)/`max_steering_angle`(0.45) 在 chassis_common/planning/control 三处一致。**接线核对全部通过。**② **唯一真实缺陷**：`amcl.yaml` 尾部曾有独立顶层段 `amcl_health_monitor:`（含 `covariance_position_threshold`/`covariance_yaw_threshold`/`degradation_duration` 三项），注释声称"由状态机读取"。但全工作区**无任何名为 `amcl_health_monitor` 的节点**——`localization_stack.launch.py` 仅以 `name='amcl'` 加载本文件，ROS 2 按**节点名**取参数段，该段被**静默忽略**；三项参数的**实际唯一来源**是 `race_manager.yaml` 的 `amcl.*` 键（由 `race_manager.py` L286-288 declare、L379-381 读取）。风险与本轮审计要消灭的"静默失效"同类：**整定人改 `amcl.yaml` 这三行改了没用**。③ **修复**：删除 `amcl_health_monitor:` 死段，改为纯注释块，明确"本三项由 race_manager 节点读取，唯一来源 = `autocar_race_manager/config/race_manager.yaml` 的 `amcl.*`；勿改本文件"；同步 §9.3 表格下新增"落地文件区分"注、`autocar_slam/README.md` L19、`RUNBOOK.md` §9 落地映射行。④ 判据级验证：`offline_check.py` 重跑（yaml.safe_load 14 项含 amcl.yaml、build_check 8 项、calib_check 基线不变）。**未改动任何运行代码、话题、服务、参数取值与节点接线；仅删除一个从未生效的死配置段并纠正文档指向。** | `autocar_slam/config/amcl.yaml`、§9.3、`autocar_slam/README.md`、`RUNBOOK.md` |
| R78 | **全板块真实功能测试（第一轮）发现并修复 `path_generator` 两处串联缺陷**。① **测试方法**：无 ROS 环境下以 `tests/ros_stub.py`（rclpy/tf2/消息 mock）+ `tests/harness.py`（YAML 参数注入 + 可编程时钟/TF）直接实例化真实节点并调用回调，逐判据断言；规划板块 10 组 40 项（`tests/test_planning.py`）。② **缺陷 A（裁剪全丢无兜底）**：中轴点稀疏/走廊宽度估计系统性高估时 `clip_and_bridge` 可能丢弃**全部**中轴点（`n_drop=len(idx)`），旧代码 `len(pts)<2` 时直接返回空点集 → `/race_line` 为空 → 下游无路径。修复：新增回退分支——裁剪清空时**回退未裁剪中轴**并打 WARN（走廊约束是软目标，无路径是硬失效，宁给未裁剪路径也不给空路径）。③ **缺陷 B（闭环误判开环）**：`_is_closed` 旧判据 `gap < max(0.5, 0.05·total)` 对稀疏中轴过紧——N=12 点环首末缺口≈8.3% > 5% → 误判开环 → 走 natural 端点样条**外推**，环形路径半径被拉偏（5.69→4.16~5.19，实测复现）。修复：判据改为「首末缺口 ≤ 3× 中位相邻点距」（与 `prune_spurs` 同口径），稀疏环正确识别为闭环，半径恢复 5.5±0.1。④ **验证**：修复后规划板块 40/40 PASS；`offline_check.py` 全量重跑无回归。**未改动任何参数取值、话题、QoS 与 §9 基线；修复均为代码判据级。** | `autocar_planning/path_generator.py`、`autocar_ws/tests/`（新增测试骨架） |
| R79 | **控制板块闭环仿真发现 `fuzzy_pid` 横向纠偏**符号缺陷（正反馈）**并修复**。① **发现方式**：`tests/test_control.py` 第④组用 50 Hz 自行车运动学闭环仿真（`yaw += v·tan(δ)/L·dt`）验证控制律收敛性——纯追踪主环与"纯追踪+模糊 PID 修正"各测 0.3/0.75 m 初始偏移。② **缺陷**：设计卡 J.5.2 公式 `δ = δ_raw + kp·e + …` 的 `e` 按控制学标准约定「目标−实际」（车偏左为**负**）；而实现 `_lateral_error` 约定「车在路径左侧为**正**」（与 δ>0=左转同向），即 `e_code = −e_doc`。实现照抄公式写成 `delta += kp·e + …` → 修正项与纯追踪主纠偏**反向**（正反馈）。实测旧代码"勉强收敛"仅因纯追踪等效增益 `2L/L_d²≈1.67` 恰好压过 kp 上限（0.6+2×0.2=1.0）；**Q4 答复轴距更小（L<0.30）或 kp 整定偏大即发散 → 弯道冲出 FAIL-01**。③ **修复**：改为 `delta -= kp·e + ki·∫e + kd·ė`，并同步头部 docstring 公式为 `δ = δ_raw − (kp·e + ki·∫e dt + kd·ė)` 与约定说明。④ **验证**：修复后闭环仿真 4 工况全部收敛，控制板块 73/73 PASS。**未改动任何参数取值；仅纠正一处符号（等价于把实现拉回设计卡 J.5.2 的原意）。** | `autocar_control/fuzzy_pid.py`、`autocar_ws/tests/test_control.py` |
| R80 | **控制板块测试发现 `chassis_driver` 链路超时保险后的"幽灵里程"缺陷**并修复。① **缺陷**：`_tick` 中链路超时保险把**下行速度归零**（failsafe），但随后 `_publish_odom` 仍读 `self._cmd_speed` 继续积分发布 `/odom` → 车实际停着、里程计却在走（幽灵位移）。污染下游：`race_manager` 计圈里程（R67/R68/R70 兜底判据全部基于 `_lap_distance`）、入库静止兜底（`_last_speed`）、watchdog 静止判定——链路一断，**里程与静止判据同时失真**。② **修复**：`_tick` 把**保险处理后**的 `speed/delta` 传入 `_publish_odom(now, speed, delta)`；签名改为可选参（默认沿用 `_cmd_*`，simulate 模式行为不变）。③ **验证**：新增用例"喂 cmd_vel→断供→超时→odom vx 归零且里程不再增长"，控制板块 73/73 PASS。④ **本轮测试总账**：感知 28/28、视觉 22/22、规划 40/40、控制 73/73、race_manager 110/110（含 R59/R60/R61/R67/R68/R70/R72 兜底全链路、ROLLING_SWITCH 六步时序、AMCL I.5.4 降级、看门狗心跳优先），合计 **273 项断言全过**；race_manager 板块未发现新缺陷。**上限声明：以上为 mock 层+单元测试级验证，不替代实车 `colcon build` 与 `deploy.sh smoke`。** | `autocar_chassis_driver/chassis_driver.py`、`autocar_ws/tests/` |
| R81 | **感知板块深度审查：`cone_detector` 挡板合并与置信度门限顺序颠倒**（PEN-07/08 防线失效）。① 缺陷：旧流水线先按 `cone.min_confidence`(0.5) 丢弃低置信簇、再做挡板合并——远场挡板碎簇（2 点、`radial_threshold` 未标定时 conf≈0.33）**先被丢弃**，合并拿不到原料 → 地图缺挡板。挡板判据本是几何（间距/共线/长度），与单簇置信度无关。② 修复：合并移到门限**之前**，门限只作用于合并后剩余锥桶（`_on_scan` ④⑤ 步序互换）。③ 验证：感知 28/28 PASS（含"长条反射面判为挡板""低置信碎块不误判"两组回归）。 | `autocar_perception/cone_detector.py` |
| R82 | **视觉板块深度审查：`parking_sign_detector` 补齐 CR-11 三重形状门限**（四边形校验 + 透视正视化 + P 字拓扑）。① 缺陷：CR-11 明确牌面 29.7×42 cm 蓝底白字 P，旧实现只做"蓝 blob 面积 + 白像素占比"——圆形蓝色杂物内嵌白块即可误触发 `detected`，入库判据（TR-04/PEN-04）被假阳性劫持。② 修复：`approxPolyDP` 顶点数==4 → 顶边/侧边长宽比 ∈[0.45,1.15]（标称 0.707 留透视压缩余量）→ `getPerspectiveTransform`+`warpPerspective` 正视化到 120×169 标准牌面 → P 字拓扑三判据（左列带竖笔贯穿 ≥0.45H、顶部环区有白、环下右侧白占比 <5%）→ 正视化坐标系内重算白字占比与置信度。③ 验证：`test_visual.py` 合成图改为真实 P 字形（竖笔+顶环），22/22 PASS（含"白字占比不足→absent""纯蓝底→absent"回归）。 | `autocar_perception/parking_sign_detector.py`、`autocar_ws/tests/test_visual.py` |
| R83 | **视觉板块深度审查：`camera.calibration_file` 声明后从未加载**（150° 畸变静默未纠）。① 缺陷：`traffic_light_detector` / `parking_sign_detector` 均 declare 该参数但代码无任何读取路径——`/camera/camera_info` 话题不可达（驱动不发/QoS 不匹配/晚到）时 `_K` 永远 None，`undistort` 静默旁路，ROI/形状判据在 150° 大畸变下整体偏移。② 修复：两节点启动时 `_load_calibration_file()` 兜底加载（兼容 OpenCV FileStorage `K/D` 与 `camera_matrix` 嵌套、普通 YAML `k/d`），camera_info 到达后覆盖（话题优先）；文件缺失/解析失败仅 warn 不致命。③ 验证：视觉 22/22 PASS（stub 环境无标定文件 → 走 warn 分支不崩溃）。 | `autocar_perception/traffic_light_detector.py`、`autocar_perception/parking_sign_detector.py` |
| R84 | **控制板块深度审查：`pure_pursuit` 弯道限速档 `speed.curve` 声明后从未消费**（违反 J.4 限速表）。① 缺陷：J.4:2788 明确弯道 0.9–1.2 m/s 档与"多档并存取最小值"仲裁；旧实现只靠 `steering_decay` 连续减速——本赛道 R=1.5 m 圆弧上 |δ|/δmax≈0.36，v≈1.8×(1−0.7×0.36)≈1.34 m/s **> speed.curve=0.9**，弯道超速威胁 FAIL-01。② 修复：新增 `speed.curve_kappa`(默认 0.5 1/m ≈ R≤2 m) 门限，前视点 Menger 曲率 κ≥门限 → `speed_base = min(状态档, speed.curve)` 再乘 decay（取小仲裁）。③ 同步：`control.yaml` 新增 `speed.curve_kappa: 0.5`。④ 验证：控制 74/74 PASS（闭环仿真收敛回归不变）。 | `autocar_control/pure_pursuit.py`、`autocar_control/config/control.yaml` |
| R85 | **控制板块深度审查：`pure_pursuit`/`fuzzy_pid` TF 帧名硬编码 + 阻塞 lookup**。① 缺陷：planning 走 `frames.*` 参数（planning.yaml），控制两节点却硬编码 `'map'/'base_link'/'odom'`——实车 TF 树帧名不同（如 `base_footprint`）时静默查不到 → 永久保持上一指令；且 `timeout=0.1` 在 50 Hz 定时器上最坏吃掉 5% 占空比。② 修复：两节点新增 `frames.map/odom/base_link` 参数（默认值不变）；lookup timeout 0.1→0（非阻塞，查不到即走回退/保持，下一周期重试）。③ 同步：`control.yaml` 两节点段各加 `frames.*` 三键。 | `autocar_control/pure_pursuit.py`、`autocar_control/fuzzy_pid.py`、`autocar_control/config/control.yaml` |
| R86 | **控制板块深度审查：`watchdog` 静止累计寄生在 odom 回调上**（odom 断流 → 看门狗静默失效 → FAIL-03 判负）。① 缺陷：旧实现只在 `/odometry/filtered` 回调里按回调间隔累计，且 `dt>0.5` 直接清零——odom 断流或降频 <2 Hz 时**永不累计**。最坏场景"车完全不动、里程源也挂"恰是看门狗唯一能救命的时刻，旧实现此时静默。② 修复：新增 10 Hz 墙钟定时器 `_on_tick` 负责累计与发布；odom 回调只记录 |v| 与到达时刻；新增 `odom_stale`(1.0 s) 新鲜度窗口——超过即视为静止（盲速不可控）。③ 同步：`watchdog.yaml` 加 `odom_stale: 1.0`；测试改为驱动定时器并新增断流用例。④ 验证：控制 74/74 PASS（含"odom 断流 12s→触发"新用例）。 | `autocar_control/watchdog.py`、`autocar_control/config/watchdog.yaml`、`autocar_ws/tests/test_control.py` |
| R87 | **规划板块深度审查：`path_generator` 曲率约束与安全裕度顺序颠倒**（推挤重新超 κ_max 且无复检）。① 缺陷：旧顺序 `enforce_curvature` → `enforce_safety_margin`——安全边距推挤（≤0.15 m 法向平移）会在锥桶附近**重新制造** κ>κ_max 尖点，且无任何复检 → 下发的 `/race_line` 局部不可达，FAIL-01 防线被自己绕过。② 修复：顺序改为 margin→curvature（松弛是加权平均，不再引入新边距违规量级）→resample；并加不收敛告警（30 次松弛后 max κ 仍 >105% κ_max → WARN）。③ 验证：规划 40/40 PASS（"曲率约束降低尖角曲率""安全裕度推开至≥margin"回归不变）。 | `autocar_planning/path_generator.py` |
| R88 | **调度板块深度审查：ROLLING_SWITCH ④ 步忽略 `supervisor.start()` 返回值 + abort 后不回启导航栈**（第二圈死路 DNF）。① 缺陷：④ 步 `start()` 返回 False（fork 失败/③ 未停干净拒启）时旧代码照常推进——对着**不存在的导航栈**调 manage_nodes（必然超时跳过），⑤ 永远等不到 map→odom 恢复 → abort；而 `_switch_abort` 在 ③ 之后失败时建图栈已停、导航栈未起，回 WATCHDOG_RECOVER→NAVIGATING 等于**无定位驾驶**，第二圈必然 DNF。② 修复：④ 步检查返回值，失败强制 `stop()` 后重试一次，仍败 → `_switch_abort`；`_switch_abort` 增加补救——supervisor 未在跑且地图 yaml 已落盘则再拉一次导航栈（尽力而为，失败仅 error 日志）。③ 验证：race_manager 110/110 PASS（六步时序回归不变）。 | `autocar_race_manager/race_manager.py` |
| R89 | **调度板块深度审查：`switch.timeout=20 s` 小于最坏分预算之和 ≈22.5 s**（正常慢盘也必被 abort）。① 核算：① map_save 3 s+重试 3 s、② 两服务 2+2 s、③ stop 阻塞最坏 4.5 s+静默确认 1.5 s、④ 1+2+2 s、⑤ 3 s ≈ 22.5 s > 20 s——旧值在 ARM 板慢盘/慢服务响应属常态，**必然超时 abort**，叠加 R88 缺陷直接 DNF。② 修复：20.0→30.0（`race_manager.py` 默认值 + `race_manager.yaml` + README 同步）；测试总超时用例推进量 20.5→30.5 同步。 | `autocar_race_manager/race_manager.py`、`autocar_race_manager/config/race_manager.yaml`、`autocar_ws/tests/test_race_manager.py` |
| R90 | **底盘板块深度审查：`chassis_driver` encoder 模式串口断流后幽灵里程**（R80 同族缺陷的 encoder 分支）。① 缺陷：`serial_link` 读线程异常退出后 `drain()` 永远为空，`_enc_speed` 保持最后一次回帧值——`odom.source=encoder` 时 odom 按**过期速度**继续积分 → 车停/链路死后里程仍在走，污染计圈/停车/看门狗全部判据（与 R80 cmd_integration 分支同一教训）。② 修复：记录 `_last_enc_time`；encoder 分支回帧超 0.5 s 未更新 → 速度/角速度归零再积分。③ 验证：offline_check FAIL=0；控制 74/74、底盘 codec 用例回归不变。**R81~R90 总账：感知 28/28、视觉 22/22、规划 40/40、控制 74/74、race_manager 110/110，合计 274 项断言全过；offline_check FAIL=0/SKIP=0/INFO=1。上限声明不变：mock 层验证，不替代实车。** | `autocar_chassis_driver/chassis_driver.py` |
| R91 | **二次全板块精简/完善复审（行为不变审计）**。① 精简（零行为变更）：AST 扫描 47 个 .py 确认并删除 7 处无用引用——`chassis_driver` 未用 `OdomSample` import、`qos_check` 未用 `sys`、`offline_check` 未用 `os`、`test_visual` 未用 `math`、`ros_stub.make_image` 函数内未用 `import numpy`、`test_planning` 未用 `pg`/`collect_candidate_points`、`test_race_manager` 未用 `SWITCH_STEPS`。核实为**误报不处理**：`from __future__ import annotations`（编译指令）、`use_sim_time`（rclpy 内部消费）、`speed.straight` 等 5 个 declare 未读参数（control.yaml 仍设值、删除触及参数接口，按"不影响功能"保留）、DEAD-FUNC 全为回调经 `self._on_x` 注册。② 完善（行为变更，已测试）：`parking_sign_detector` 图像断流 → `/parking_sign/state` 置 `unknown`（与 traffic_light `_on_stale_timer` 同构；旧实现 15 Hz 无限重发最后一帧判定，相机掉线且末帧恰为 detected 时 race_manager 会在看不见标志下盲目进 PARKING）。新增 `parking.stale_timeout`(1.0 s) 参数 + perception.yaml 同步 + 2 个新断言（断流→unknown、恢复帧→detected）。③ 仅记录不修：cone_detector 断流重发旧 `/cones`——下游 `path_generator` 数据不足时**有意**保持旧路径（测试"数据不足重发旧路径（不打空）"锁定该行为），加失效标记反而破坏"感知短断不慌"设计。④ 验证：感知 28/28、视觉 24/24（+2）、规划 40/40、控制 74/74、race_manager 110/110，合计 276 断言全过；offline_check FAIL=0/SKIP=0/INFO=1。 | `autocar_perception/parking_sign_detector.py`、`autocar_perception/config/perception.yaml`、`autocar_ws/tests/test_visual.py`、`autocar_chassis_driver/chassis_driver.py`、`autocar_ws/scripts/qos_check.py`、`autocar_ws/scripts/offline_check.py`、`autocar_ws/tests/{ros_stub,test_visual,test_planning,test_race_manager}.py` |

---

## 阅读指引

| 读者 | 建议入口 | 关注点 |
|---|---|---|
| 队长 / 指导教师 | 第二部分 → §6 ADR | 架构是否合理、决策理由是否站得住 |
| 算法同学 | 第三部分 → 第四部分 | 接口契约、参数基线、手册案例写法 |
| 电控 / 硬件同学 | §1.3、§1.4、附录 A | 设备能力边界、罚则清单、赛前合规检查 |
| 评委 / 未来学弟 | 第四部分 + §1.5 | 复现步骤、罚则映射 |

> **配套文档与参数唯一来源**：本项目的实施向文档为同目录的 **`ros小车.txt`（从0到1技术方案）**，它侧重零基础落地步骤与代码流程。**两份文档保持独立**，但有一条约定的边界：**本文件 §9「参数基线」是全部参数取值的唯一来源（single source of truth）**。`ros小车.txt` 中出现的任何数值仅为便于阅读的引用；若二者不一致，**一律以本文件 §9 为准**，并应回头修正 `ros小车.txt`。

---

# 第一部分　需求基线

## 1. 规则条目化清单（R0）

规则的原始形态是不可测试的散文。本章把它拆成**带编号、可验证、可追溯**的需求条目，每条注明规则出处。编号规则：

| 前缀 | 含义 | 数量 |
|---|---|---|
| `TR` | 任务需求（Task Requirement） | 9 |
| `CR` | 赛道与场地需求（Course Requirement） | 22 |
| `HR` | 设备需求（Hardware Requirement） | 9 |
| `PEN` | 罚则（Penalty） | 14 |
| `FAIL` | 判负 / 取消资格条款 | 6 |
| `SEC` | 合规与禁止行为 | 7 |
| `ELIG` | 参赛资格、队伍与报名要求 | 9 |
| `DR` | 交付物需求（Deliverable Requirement） | 15 |

每条需求的**验收方式**一栏是设计活动（第四部分的测试计划）的输入。

### 1.1 任务需求（TR）

来源：规则 2.1「赛题内容」。

| ID | 需求原文摘要 | 规则出处 | 验收方式 | 关联罚则 |
|---|---|---|---|---|
| TR-01 | 无人车在赛道中运行**两圈** | 2.1 | 视频中可见完整两圈 | FAIL-01~03 |
| TR-02 | 第一圈：**探索未知环境、自主识别比赛场地，并对场地进行二维建图** | 2.1 | 第一圈结束时产出 `.pgm` + `.yaml` 栅格地图 | PEN-07/08 |
| TR-03 | 第二圈：**基于第一圈构建的赛道地图**进行自主导航并**加速冲刺** | 2.1 | 第二圈速度显著高于第一圈 | — |
| TR-04 | 第二圈：**根据停车检测区域提示，停在相应的停车区域** | 2.1 | 三个及以上车轮停在停车区域内 | PEN-04 |
| TR-05 | 两圈过程中**每次遇到红绿灯**，需在红绿灯停车区域（图 1 青色框包围的六个锥桶围成的矩形区域）内按指示停车，绿灯亮起后继续前进 | 2.1 | 红绿灯区域停车行为正确 | PEN-01/02/03 |
| TR-06 | **等待红灯的时长不计入比赛用时** | 2.1、2.4.1 | 有效时间公式扣减项正确 | — |
| TR-07 | 第二圈需识别停车标志牌进行**停车入库**；第一圈**不需要**在停车区域停车 | 2.1 | 第一圈不得因停车标志而停车 | TR-01 |
| TR-08 | 比赛当天随机在赛道两锥桶间加放**蓝色挡板**（防提前建图），摆放位置当天确定，**最终建图所得地图需要显示出该挡板** | 2.1、2.2.1(11) | 地图中可见 3 处挡板轮廓 | PEN-07/08 |
| TR-09 | **红绿灯停车区域的位置在整个赛道中随机** | 2.1、2.2.1(6)(7) | 不依赖红绿灯的先验里程位置 | PEN-03 |

> **TR-08 的设计含义**：挡板是规则层面的**防提前建图机制**。因此第一圈必须真实执行探索—建图流程，且必须留下因果证据（时间戳、日志、视频）。这决定了架构上**不能**采用"预置赛道先验地图 + 直接定位"的捷径（详见 ADR-005）。

### 1.2 赛道与场地需求（CR）

来源：规则 2.2「比赛赛道」与 2.2.1「赛道搭建注意事项」。

| ID | 需求 | 数值 / 说明 | 规则出处 | 设计含义 |
|---|---|---|---|---|
| CR-01 | 场地形态 | **不规则环形场地**，由红蓝两色锥桶搭建 | 2.2 | 非标准化赛道，无通用先验 |
| CR-02 | 赛道宽度 | **1.5 m** | 2.2(1) | 车宽 0.56 m，单侧余量约 470 mm |
| CR-03 | 锥桶间距 | **小于赛道宽度**（< 1.5 m） | 2.2(2) | 单帧雷达可同时观测内外圈 |
| CR-04 | 锥桶构成 | **大部分**由内圈**红色**锥桶和外圈**蓝色**锥桶构成 | 2.2(3) | 锥桶颜色即行进方向先验（2.2.1(12) 明示"可利用锥桶颜色来识别无人车运行方向"） |
| CR-05 | 折线区铺设 | **小部分赛道由两根黄色胶带铺设而成，胶带宽 10 cm**；视觉识别折线区域由黄色胶带粘贴地面铺设 | 2.2(3)、2.2.1(4) | 两根胶带即折线区两侧边界；压线判定见 PEN-06 |
| CR-06 | 走向与起点 | **发车区域固定**，无人车沿赛道**逆时针**运行 | 2.2.1(5) | 决定内圈即左侧、径向阈值方向 |
| CR-07 | 区域几何定义 | 停车检测区域、停车区域及发车区域：各区域**顶点为锥桶投影到地面的中心**；各区域均为**矩形**；边缘由黄色胶带包围，**胶带宽 10 cm**；**以胶带内侧作为区域的边界** | 2.2.1(3) | 停车判定的几何基准（见下方警示） |
| CR-08 | 红绿灯停车区域 | 为赛道示意图中**青色框区域**（六个锥桶围成的矩形）；红绿灯**位于该矩形区域的右侧（以小车运行方向为参考）** | 2.2.1(6) | 运行方向上的下游侧即红绿灯本体位置 |
| CR-09 | 红绿灯位置随机 | 线下赛摆放位置**比赛当天由裁判决定**；线上赛**赛前一周**提供具体摆放位置 | 2.2.1(7) | 不允许硬编码位置；线上赛需关注 QQ 群公告 |
| CR-10 | 停车识别与入库 | 第二圈需**通过摄像头识别停车检测区域内的标志牌**，并在**进入停车区域后，准确停在标志牌区域内** | 2.2.1(8) | 识别与停车是两个动作、两个区域（详见 §2.3 Q7） |
| CR-11 | 停车标志牌规格 | **29.7 cm × 42 cm**，蓝底白字 P；组委会提供源文件，材质不限 | 2.2.1(9)、图 3 | 摄像头识别的目标尺寸与位姿估计依据 |
| CR-12 | 红绿灯本体 | **60 × 17 × 6.7 cm，离地高度 40 cm** | 2.2.1(10)、图 4 | 决定摄像头的安装高度与俯仰角 |
| CR-13 | 挡板放置规则 | 线下赛**每队上场前**随机在内侧或外侧赛道的**两锥桶之间放置 1 块蓝色挡板**，**整个赛道共计 3 处**；图 6 虚线框为挡板**可能摆放的所有位置** | 2.2.1(11)、图 6 | 挡板位置不可预知，必须在线检测 |
| CR-14 | 挡板尺寸 | **高度 50 cm / 长度 80 cm** | 图 5 | 后处理聚类合并的几何约束（见 §9.1） |
| CR-15 | 锥桶规格 | 名称：锥桶；**材质：塑料**；**规格：630 × 280 × 280 mm**（按行业惯例为 **高 × 底面宽 × 底面长**，即高 630 mm、底面 280×280 mm；**需与实物核对**）；外皮颜色：**红色、蓝色**；**赛道中的锥桶如有白色的反光膜需要去掉** | 2.2.1(12)、图 7 | 反光膜会显著改变雷达回波强度与视觉颜色判定，**必须在赛前场地准备时确认已去除** |
| CR-16 | 搭建测量依据 | 搭建赛道的具体参数**以图 8「赛道测量示意图」为准** | 2.2.1(13)、图 8 | 线上赛须公示赛道尺寸并录制测量视频（DR-09） |
| CR-17 | 警示线 | 为避免无人车失控伤人，**线下赛外围会设有警示线**，具体以比赛现场搭建为准 | 2.2.1(14) | 安全项；影响激光雷达远距离杂波与场地边界保护 |
| CR-18 | 赛道元素构成 | 整体赛道由**直线区域、折线区域、"S"弯、圆形区域**等部分元素构成 | 2.2 | 决定速度规划的分段策略 |
| CR-19 | 图 8 关键尺寸 | 直线段 **3.06 m × 2**；圆弧半径 **1.5 m**；局部收窄 **1.25 m**；折线区顶点角 **135°**；S 弯含 **90° / 45°** | 图 8 | 决定弯道速度上限与通过性瓶颈 |
| CR-20 | 场地横向总尺寸 | 图 8 总尺寸线标注 **16.8 m**，为**赛道场地在图纸横向的总体跨度**；**单圈里程规则未给出，必须现场实测** | 图 8 | 时间预算与里程计标定的输入（详见 §2.1、§4.1） |
| CR-21 | 图纸与实际 | 图中**虚线只为描述锥桶之间相对摆放位置，真实赛道中无虚线痕迹**；具体摆放位置**与图 2 中的示例保持一致** | 2.2.1(1)(2) | 不以虚线为视觉特征 |
| CR-22 | 运行方向上的区域顺序 | 由图 2 箭头（逆时针）**读图推断**：底边直道**自左向右**依次为 **红绿灯停车区 → 停车区 → 发车区**，发车区驶出方向向右（**必须现场核对**） | 图 2（读图推断） | 红绿灯与停车区相邻且在终点前；第二圈将在红绿灯之后**紧接着**执行入库停车 |

> **CR-07 是极易被误读的一条。** 停车区域的实际可用范围从"锥桶中心连线"起算，并以**胶带内侧**为边界。**标定停车点时必须以胶带内侧为界，而不是以锥桶外沿为界**——否则车轮可能被判为不在区域内（PEN-01 的 +20 s）。
>
> **CR-22 的工程含义（重要）**：红绿灯停车区与停车区在运行方向上前后相邻，且都在发车区之前。因此第二圈的实际节奏是：
> `… → TRAFFIC_APPROACH → 红灯停车 → 绿灯起步 → 短距离加速 → PARKING_DETECT → PARKING → 回到发车区完赛`。
> **停车入库前的可用加速距离很短**，速度规划必须在绿灯起步后立即为入库制动预留距离，不能沿用"直道全速"策略。

### 1.3 设备需求（HR）

来源：规则 4.4「参赛设备部分性能要求」。

> **措辞差异（如实标注）**：规则 3.2 表述为"大赛组委会统一**指定**参赛设备，设备供应商为北京小豚科技有限公司"；规则 4.4 表述为"组委会统一**推荐**采用北京小豚科技有限公司的智能驾驶竞速车 HW-Autocar 参赛"。**两处措辞不一致**。本团队按"指定/推荐 HW-Autocar"执行，但需向组委会书面确认底盘驱动接口与运动学模型（见 §2.3 Q5、Q6）。

| ID | 部件 | 关键参数（原文） | 对设计的约束 |
|---|---|---|---|
| HR-01 | 无人车底盘 | 户外越野底盘，**前后轮有差速器**；尺寸 560×350×230 mm | 车体宽 0.56 m；底盘含舵机（见 HR-04），运动学按**阿克曼**处理 |
| HR-02 | 驱动电机 | 有感无刷，Kv 2150–2650，功率 2400 W，最高转速 45000 rpm，最高电压 19 V | 动力远超本赛道需求，速度上限受控制精度而非动力限制 |
| HR-03 | 电调 | 额定电流 120 A／最大电流 760 A，电池节数 2–3S LiPo | 供电与电池管理需留余量 |
| HR-04 | **舵机** | **工作频率：1520 µs／330 Hz**；工作电压 DC 4.8–6.0 V | **车型为阿克曼转向**。控制量为转向角 δ，而非差速的角速度 ω。中位脉宽为 **1520 µs**（非 1500 µs），必须实测标定 |
| HR-05 | 处理器 | 昇腾 310 系列 AI 处理器：1×DaVinciV300 AI core、4×TAISHANV200M 处理器核；AI 算力 12–20 TOPS；内存 12 GB LPDDR4X；存储 SD 卡 64 G；工作电压 12 V；软件 Ubuntu 22.04 + ROS 2 Humble | **ARM64 架构**，Cartographer／Nav2 需 ARM 编译；CANN 生态与 x86 不同，需评估 |
| HR-06 | 激光雷达 | 360° 全方位扫描；测量角 0.22°；扫描频 10–12 Hz；采样频 20000 次/s；测量距 25–30 m；测量量程解析 0.1%；A6 核 ARM64 位处理器、主频 2 GHz、内存 2 G | 单帧间隔位移：以 v m/s 行驶时 = v / f_scan；10 Hz、1.8 m/s 时约 18 cm |
| HR-07 | 姿态传感器 | 动态精度 2.5°（另标注 3° RMS），分辨率 0.1°，陀螺仪 ±2000°/s，零偏稳定性 10°/h，非线性度 0.1% FS | 零偏稳定性尚可，但 2.5° 动态精度对航向贡献有限，**主要依赖激光雷达做航向** |
| HR-08 | 编码器 | 工作电压 5 V；波特率 9600–115200；工作电流 10 mA；通信协议 **Modbus RTU**；内核刷新周期 50 µs；最大机械转速 1000 RPM；最大启动扭矩 0.006 Nm；工作温度 −40~+85 °C | 需独立 Modbus RTU 节点；`scale_factor` 是建图鬼影的头号嫌疑 |
| HR-09 | 摄像头 | 规格 1920×1080P；**视角 150°**；功率 2 W；工作电压 5 V；工作电流 150–200 mA；支持 **UVC 通信协议** | 150° 广角畸变显著，**必须做内参标定**后再用于测距类判断 |

> **HR-04 是本次需求分析中最关键的一条修正。** 原草稿按"前后轮有差速器"把车当差速底盘处理，但设备表明确列出**舵机**。这直接改变控制律的形态：
> - 差速模型：`ω = 2·v·sin(α) / L_d`
> - **阿克曼模型：`δ = arctan(2·L·sin(α) / L_d)`**，其中 `L` 为轴距
>
> 两者的调参逻辑完全不同。设计文档必须在写代码前澄清这一点，并向组委会/供应商确认底盘运动学模型。

### 1.4 罚则矩阵（PEN）

来源：规则 2.4.2「关于比赛加罚时间的处理细则」与 3.5.3「视频评分细则」。

> **适用性说明**：PEN-01 ~ PEN-11 对**线下赛与线上赛同样适用**；PEN-12 ~ PEN-14 与 FAIL-05、FAIL-06 **仅线上赛适用**。本表按"必然性 × 量级"排序。

| ID | 违规行为（原文要点） | 加罚 | 规则出处 | 适用 | 必然性 | 优先级 | 对应设计措施 |
|---|---|---|---|---|---|---|---|
| PEN-01 | 红绿灯停车时，**任一车轮未在停车区域内** | **+20 s** | 2.4.2(1) | 线下/线上 | 中 | **高** | 停车点按胶带内侧标定；减速曲线预留收敛距离 |
| PEN-02 | 红绿灯处**未停车** | **+50 s** | 2.4.2(1) | 线下/线上 | 低 | **极高** | 红灯判定用连续 3 帧确认 + 视觉失效时默认停车 |
| PEN-03 | **绿灯未亮起就通行** | **+30 s** | 2.4.2(1) | 线下/线上 | **高** | **极高** | 见下方「光电触发陷阱」 |
| PEN-04 | 第二圈停车：三及以上车轮停在停车区域内视为成功；**停车失败或未停车** | **+20 s** | 2.4.2(2) | 线下/线上 | 中 | 高 | 标志牌置信度阈值 + 入库低速收敛 |
| PEN-05 | **每次撞击锥桶** +5 s；**若没撞到锥桶、只是压到锥桶的支撑边角且未发生位移，则不加罚** | **+5 s / 次** | 2.4.2(3) | 线下/线上 | 高 | 中 | 轨迹预留横向安全裕度；但**不建议依赖例外条款** |
| PEN-06 | 折线区域运行时，**单圈内只要压线或骑线行驶** | **+10 s / 圈** | 2.4.2(4) | 线下/线上 | 中 | 中 | 折线区独立限速 + 中线跟踪增强 |
| PEN-07 | 在比赛过程中**构建了**地图，但：地图不清晰／挡板模糊但能看出挡板存在且位置正确／**丢失两个及以下挡板和锥桶** | **+5 s** | 2.4.2(5) | 线下/线上 | 中 | 高 | 建图后 RViz 人工复核 + 挡板合并后处理 |
| PEN-08 | **没有在比赛过程中建图**（包含**提前建图、未构建地图、挡板位置明显不符合实际情况**等） | **+50 s** | 2.4.2(5) | 线下/线上 | 低 | **极高** | 保证真实探索流程 + 挡板位置校验 |
| PEN-09 | 偏离轨道运行，**每绕错一个锥桶** | **+5 s / 个** | 2.4.2(6) | 线下/线上 | 中 | 中 | 见 FAIL-01 |
| PEN-10 | 系统中**存在 ROS 1** | **+100 s** | 2.4.2(9) | 线下/线上 | 低 | **极高** | 附录 A.1 的完整残留清除 checklist |
| PEN-11 | **人为干预**（人工修正运行轨迹、人工干预转向等） | **+100 s / 次** | 2.4.2(11) | 线下/线上 | 低 | **极高** | 全程零遥控；参数现场可调但不干预运动 |
| PEN-12 | 线上赛：**未按要求摆放机位** | **+10 s** | 3.5.3(1) | **仅线上** | — | 低 | 按 §11.3 视频脚本布置三机位 |
| PEN-13 | 线上赛：视频中**比赛赛道搭建尺寸不符合要求** | **+10 s** | 3.5.3(2) | **仅线上** | — | 中 | 严格按图 8 尺寸搭建并录制测量视频 |
| PEN-14 | 线上赛：**比赛场地不在室外** | **+5 s** | 3.5.3(3) | **仅线上** | — | 低 | 选择室外合适场地 |

> **建图罚则的两档必须分清（PEN-07 vs PEN-08）**：原文的判据是"**是否在比赛过程中建图**"。
> - 只要**确实建了**，即便质量差（不清晰、挡板模糊、丢失 ≤2 个挡板和锥桶）→ **+5 s**；
> - 只要是"没有在比赛过程中建图"的三种情形之一（提前建图／未构建地图／挡板位置明显不符）→ **+50 s**。
>
> **设计含义**：宁可交一张"模糊但真实"的地图，也绝不能出现"提前建图"的嫌疑。因此第一圈必须**完整跑完探索-建图流程**，且**必须保留挡板在正确位置**（TR-08）。

> **红绿灯三档罚则的严重度排序（PEN-01/02/03）**：
> `未停车 +50 s` ＞ `绿灯未亮通行 +30 s` ＞ `车轮不在停车区域内 +20 s`
>
> **设计含义（与直觉相反）**：现场若对"是否已完全停在区域内"没有把握，**停在区域边缘（最坏 +20 s）优于冲过去（未停车 +50 s，且可能叠加 +30 s）**。因此停车策略应当是**保守制动、宁可停早**。

> **「建图成功」的正面判据（v2.1 补充，规则 2.4.2(5) 原文）**
>
> "比赛任务完成后，需向裁判展示比赛过程中构建的地图，若**挡板和锥桶与实际相符并且清晰可见**，**视为建图成功**。"
>
> 即：**建图成功时没有任何罚时**。罚时只发生在两种情形——① 确实建了但质量不达标（+5 s）；② 没有在比赛过程中建图（+50 s）。
>
> **原文未明确的边界（→ §2.3 Q9）**：原文只写了"丢失**两个及以下**挡板和锥桶 +5 s"，**没有规定丢失超过 2 个、但地图确实存在时的判罚**。

#### 关键风险：光电触发陷阱（PEN-03）

规则 2.4.2(1) 原文：
> 红绿灯前 2 个锥桶处设有光电传感器，传感器与红绿灯中间区域为停车区域；无人车经过光电传感器触发红灯倒计时，红灯倒计时持续时间 3~10 秒不固定，倒计时结束后绿灯亮起；倒计时期间无人车需在红灯前停车，直至绿灯亮起后再继续前进；若绿灯未亮起就通行，则加罚 30 秒；若停车时无人车任一车轮未在停车区域内，加罚 20 秒；若未停车，则加罚 50 秒。

**推演**：红灯并非常亮，而是由**车辆自身遮挡光电传感器**触发的。因此存在一条隐蔽路径：

```
车辆路径偏移 → 未遮挡光电传感器 → 灯不亮（既非红也非绿）
        → 车辆继续前进通过 → 判定「绿灯未亮起就通行」→ +30 s
```

**设计结论**：
1. 红绿灯区域内不能只依赖"看到红灯才停"，必须同时保证**车辆沿赛道中线行驶，以确保遮挡光电传感器**。
2. 红绿灯区域前应主动降速（建议 ≤0.3 m/s），提高遮挡概率与位置精度。
3. 状态机需增加状态 `TRAFFIC_APPROACH`：在进入区域**前**即开始降速，而非检测到红灯后才开始。
4. 视觉上"灯不亮（off）"**不能**解释为"可以通行"，只能解释为"传感器未被触发"或"识别失败"——两种情况的正确动作都是**停车并等待**。

### 1.5 判负与取消资格条款（FAIL）

来源：规则 2.4.2(6)(7)(8)(10)、3.5.3(4)(5)。优先级高于一切性能优化。

| ID | 触发条件（原文要点） | 规则出处 | 后果 | 适用 | 设计措施 |
|---|---|---|---|---|---|
| FAIL-01 | 绕错的锥桶数量**累计多于 3 个**，**或 10 秒内未返回正确轨道** | 2.4.2(6) | **计比赛失败** | 线下/线上 | 偏离检测 + 紧急重新入轨；一旦偏离立即低速回中 |
| FAIL-02 | 在停车检测区域前**停车进行标志牌识别**，**停车时间超过 10 秒** | 2.4.2(7) | **视为本次任务失败** | 线下/线上 | 识别必须在行进中完成，或停车识别严格限制在 10 s 内 |
| FAIL-03 | 无人车**在赛道中任意位置停止运行超过 10 秒** | 2.4.2(8) | **判定本次任务失败** | 线下/线上 | 全局看门狗：任何状态下的"静止"累计 10 s 即触发恢复行为 |
| FAIL-04 | 使用 **rosbag 预录数据、离线回放**等作弊方式模拟赛道行驶。**原文用词 `rosbag` 是 ROS 1 时代的工具名，在 ROS 2 中对应 `ros2 bag`** | 2.4.2(10) | **一经查实直接取消本场比赛全部成绩** | 线下/线上 | **保守执行：比赛中完全不使用 `ros2 bag`（连录制也不做）**；行车录像由手机/相机等外部设备完成；开发期离线调参须在手册中明确标注为"开发期" |
| FAIL-05 | 使用工具对比赛视频修改或造假，**出现视频遥控或造假现象** | 3.5.3(4) | **取消比赛资格** | **仅线上** | 视频须体现自主移动；不使用遥控器 |
| FAIL-06 | 组委会抽查复现时**不配合或造假** | 3.5.3(5) | **取消比赛成绩；若已发放奖状将予以追回** | **仅线上** | 保留可复现的源码与原始数据，提前自测复现流程 |

> **FAIL-03 是全局约束，而非"导航阶段的异常处理"。** 原文覆盖"赛道中**任意位置**"。因此看门狗必须实现为**独立于状态机的全局节点**，且**任何**模式切换、地图保存、重新加载都必须以"不停车"为前提（见 §5.3、§7.3）。
>
> **FAIL-04 与开发纪律（v2.1 改写：ROS 2 语义澄清）**
>
> 规则原文的用词是 **`rosbag`**——这是 **ROS 1** 时代的工具名；在 **ROS 2** 中与之对应的是 **`ros2 bag`**。原文的禁止对象是"用预录数据／离线回放**模拟赛道行驶**"。由于字面表述为"全程禁止使用 rosbag"，存在两种解读：
> - **窄解读**：只禁止"用回放数据驱动车辆"，录制与离线分析不受限；
> - **宽解读**：比赛中不允许出现任何 `ros2 bag` 相关进程。
>
> **本队采用宽解读（保守执行）**：比赛中**不启动任何 `ros2 bag`**，行车录像一律由手机/摄像机等外部设备完成。理由——一旦被认定为"使用 rosbag"，后果是**取消本场比赛全部成绩**，而保守执行的代价仅仅是"少一份车上记录"；且 2 号机位视频（DR-07）本身就要展示上位机终端输出，已能提供赛后复盘所需的证据。
>
> 开发期的离线调参（自录数据、离线回放）属于正常工程手段，但**必须在技术手册中明确标注为"开发期"**，与比赛过程严格区分（见 ADR-005 合规备注）。

### 1.6 合规与禁止行为（SEC）

来源：规则 2.3.2、2.4.2(9)(11)、3.3。

| ID | 要求（原文要点） | 规则出处 | 设计 / 流程措施 |
|---|---|---|---|
| SEC-01 | 比赛中**必须用 ROS 2 实现**（原文："比赛中必须用 ROS2 实现"） | 2.4.2(9) | 全栈基于 **ROS 2 Humble**；`package.xml` 声明 `format="3"`；构建用 **`colcon build`**（非 ROS 1 的 `catkin_make`）；启动用 **Python launch（`*.launch.py`）**（非 ROS 1 的 `.launch` XML）；参数用 **YAML + `--ros-args --params-file`**（**ROS 2 无 `rosparam`**）。详见附录 C |
| SEC-02 | 任务结束后需向裁判**在终端中展示使用的 ROS 版本** | 2.4.2(9) | 脚本 `show_ros_version.sh`：`echo $ROS_DISTRO`（期望 `humble`）→ `echo $ROS_VERSION`（期望 `2`）→ `ros2 --version` → `apt list --installed 2>/dev/null \| grep -c ros-noetic`（期望 `0`）。见附录 A.1、附录 C |
| SEC-03 | 比赛所用的**算法及运行方案需队员自行设计搭建** | 3.3 | 不使用他人未授权的方案；技术手册中说明自研部分是哪些 |
| SEC-04 | 竞速过程中车模需**自主运行，禁止用人工遥控的方式**进行比赛 | 3.3 | 不携带遥控器；赛前检查确认接收机未被使能 |
| SEC-05 | 比赛中**禁止人为干预**（人工修正运行轨迹、人工干预转向等） | 2.4.2(11) | 参数现场可调，但任何涉及运动的干预都禁止（PEN-11） |
| SEC-06 | 两份技术手册**相似度超过 80%** 视为抄袭，**取消技术报告得分** | 2.3.2 | 手册基于本队实验记录撰写，保留原始日志与截图时间戳 |
| SEC-07 | 技术手册中需附带**使用授权声明**（内容参考技术手册模板） | 2.3.2 | 声明文本随官方模板获取，原样附于手册内 |

> **SEC-06 的工程含义**：技术手册的每个案例都必须能追溯到本队的实验记录。这是防止相似度超标的**结构性保障**，而非靠改措辞。建议在 `LOG/` 中按日期保存原始数据（**开发期**的 `ros2 bag` 摘要、截图、终端日志），并在手册中引用其时间戳。

### 1.7 交付物需求（DR）

来源：规则 2.3.1、2.3.2、3.4、3.5.1、3.5.2。

| ID | 交付物 | 命名 / 格式 | 规则出处 | 备注 |
|---|---|---|---|---|
| DR-01 | 压缩包 | `室外ROS无人车赛_学校名称_队伍编号.zip` | 3.4、3.5.2 | 每队**只限提交一次** |
| DR-02 | 技术手册 | `室外ROS无人车赛_学校名称_队伍编号_技术手册.pdf` | 3.4 | 20 分 |
| DR-03 | 源程序文件夹 | `室外ROS无人车赛_学校名称_队伍编号_源程序文件夹` | 3.4 | 复现性评分依据 |
| DR-04 | 线下赛提交内容 | **源码 + 技术手册** | 3.4 | ⚠ 3.4 正文写"将整理好的技术手册**及视频**"，但其目录结构（下方）**只有技术手册与源程序文件夹、无视频**；线上赛（3.5.2）才列出 4 个视频——**原文自相矛盾**，见 §2.3 Q10 |
| DR-05 | 线上赛提交内容 | **3 个不同视角的运行视频 + 赛道测量视频 + 技术手册 + 源码** | 3.5 | — |
| DR-06 | （线上赛）1 号机视频 | `..._1号机.mp4` | 3.5.1 | **移动机位**，跟随无人车；需清晰显示无人车**全部轮廓**以及无人车**周围的跑道信息**；建议用手机跟随录制 |
| DR-07 | （线上赛）2 号机视频 | `..._2号机.mp4` | 3.5.1 | 展示上位机监控：**导航程序运行的终端输出** + **计时秒表**（可用电脑/在线秒表程序/实物秒表）；建议固定机位或电脑共享屏幕画面 |
| DR-08 | （线上赛）3 号机视频 | `..._3号机.mp4` | 3.5.1 | 展示**赛道整体画面**；建议**固定机位** |
| DR-09 | （线上赛）赛道测量视频 | `..._赛道测量视频.mp4` | 3.5.1 | 参考**图 8** 展示**赛道宽度、各弧度区域的半径长度、直线区域长度**；可单独录制 |
| DR-10 | （线上赛）视频长度 | **不得超过 10 分钟** | 3.5.1 | 超长不合规 |
| DR-11 | （线上赛）录制规范 | 禁止使用遥控器控制；建议对无人车正在实现的功能进行**解说**；视频中无人车须自主移动、不得人为干预 | 3.5.1 | 审核时会结合源程序与设计报告评定（FAIL-05） |
| DR-12 | 技术手册案例数量 | **最少不低于 5 个案例**，每个案例最高 4 分；多于 5 个时**取最好的 5 个案例**计算 | 2.3.1 | 手册满分 20 分 |
| DR-13 | 案例评分维度 | ① 案例创新性 0~1.0；② 案例实用性 0~1.0；③ 案例复杂度 0~1.0；④ 案例复现性 **0 或 1**（结合提交源码可成功复现则满分，不可复现则没分） | 2.3.1 | 复现性是二元评分，投入产出比最高 |
| DR-14 | 原创性/授权声明 | 附于技术手册内，内容参考技术手册模板；**组委会将根据声明内容对提交的手册及源码提供保护** | 2.3.2 | 见 SEC-07 |
| DR-15 | 案例评判依据（原文用词） | "依据案例的**创新性、结构规范、内容深度**等几个方面作为案例的评判依据"——与 DR-13 的四项评分细则**并列使用** | 2.3.1 | 写作时既要覆盖 4 项细则，也要照顾"结构规范、内容深度"这三项定性依据 |

### 1.8 参赛资格与队伍要求（ELIG）

来源：规则 3.1、3.2、3.3、3.4、3.5.2、4.1、4.2、4.3。**本条与代码无关，但决定"能否参赛"，属最高优先级合规项。**

| ID | 要求 | 规则出处 | 行动项 |
|---|---|---|---|
| ELIG-01 | 参赛学生**仅限全日制在校专科生** | 3.1 | 报名前核验学籍 |
| ELIG-02 | **每队参赛人数限 5 名，指导教师限 2 名** | 3.1 | 队伍成员清单 |
| ELIG-03 | **禁止参赛队之间相互抄袭**，一旦发现**取消两队评奖资格** | 3.1 | 手册与源码须可溯源（与 SEC-06 呼应） |
| ELIG-04 | **线下比赛，禁止不同队伍用同一设备参赛**，一旦发现**取消两队评奖资格** | 3.1 | 设备归属须明确；跨队借用等于两队同时出局 |
| ELIG-05 | 报名网址 | 4.1 | `https://www.smartcar.zone/` |
| ELIG-06 | 官方技术交流 QQ 群 | 4.3 | **1063631705**；加群备注"学校、姓名"；群昵称改为 `学校-姓名-队伍任职`（如"北京理工大学-李华-队长"）。**线上赛的赛道细节与红绿灯位置通过该群公布**（CR-09） |
| ELIG-07 | 提交邮箱与次数 | 3.4、3.5.2 | `smartcarX@163.com`；**每队只限提交一次** |
| ELIG-08 | 技术手册模板与比赛道具源文件 | 4.2 | 网盘：`https://pan.baidu.com/s/1Vebj2fprITROUD7aCJFXig?pwd=HGHT` 提取码 `HGHT`。含**技术手册模板**与锥桶、停车标志、红绿灯等道具的制作源文件或推荐购买链接 |
| ELIG-09 | 提交参考实例 | 3.4（线下）、3.5.2（线上） | 线下实例 `https://pan.baidu.com/s/19X0M49XfmYhjlqkgVfvc6w?pwd=HGHT`；线上实例 `https://pan.baidu.com/s/1xnyQctSIwRpeiwBWfMQrdA?pwd=HGHT`（提取码均为 `HGHT`） |

**规则 3.3 明确列出的重点考察能力**（技术手册的章节设置应能一一对应）：

| # | 考察能力 | 对应手册章节（见 §11） | 对应本文档章节 |
|---|---|---|---|
| 1 | 程序设计能力 | 第 9 章 源码说明与复现步骤 | 第三部分 ICD |
| 2 | 机器人操作系统（ROS）应用 | 第 7 章 任务调度与安全 | §5.3、§8.4 |
| 3 | SLAM 地图构建与自主导航技术 | 第 4、5 章 建图与定位导航 | §5.2、§9.2、§9.3 |
| 4 | 激光雷达、IMU、摄像头、编码器等机器人传感器应用 | 第 2、3 章 硬件与感知 | §1.3、§3.1、§9.1、§9.6 |

### 1.9 需求追溯矩阵（摘要）

设计活动必须双向可追溯。下表为示例条目，完整矩阵随测试计划维护。

| 需求 ID | 架构组件 | 设计文档章节 | 测试用例 | 手册案例 |
|---|---|---|---|---|
| TR-02 第一圈建图 | Cartographer 建图栈 | §5.2 / §5.3 | ACC-T02 | 案例 2 |
| TR-08 挡板可见 | 挡板后处理模块 | §9.1 | ACC-T08 | 案例 1 |
| CR-05 折线区压线 | 中线跟踪增强 + 折线区限速 | §9.4 / §9.5 | ACC-C05 | — |
| CR-10 停车识别与入库 | 视觉 + 停车状态机 | §5.3 / §9.6 | ACC-C10 | 案例 5 |
| CR-15 锥桶反光膜 | 场地准备 checklist | 附录 A.3 | ACC-C15 | — |
| PEN-01 停车区域 | 停车点按胶带内侧标定 | §9.5 | ACC-P01 | 案例 5 |
| PEN-03 光电触发 | `TRAFFIC_APPROACH` 状态 | §5.3 / §9.5 | ACC-P03 | **案例 6（强烈建议）** |
| PEN-10 ROS 1 残留 | 部署 checklist | 附录 A.1 | ACC-P10 | — |
| FAIL-03 全局看门狗 | 全局监控节点（非状态机内） | §9.7 | ACC-F03 | 案例 5 |
| DR-13 案例复现性 | 复现文档 + 一键脚本 | §11 / §13 | ACC-D13 | 全部案例 |

---

## 2. 赛道几何测绘报告（R1）

### 2.1 图 8 标注值（规则给定）

| 参数 | 标注值 | 位置说明 | 规则出处 |
|---|---|---|---|
| **场地横向总尺寸** | **16.8 m** | 图纸下方总尺寸线，横跨整个虚线外框 | 图 8 |
| 直线段 | **3.06 m × 2** | 主体直线区两臂 | 图 8 |
| 赛道宽度 | **1.5 m**；局部标注 **1.25 m** | 1.25 m 出现在收窄处 | 2.2(1)、图 8 |
| 圆弧半径 | **1.5 m** | 圆弧区 | 图 8 |
| 折线区顶点角 | **135°** | 折线区 | 图 8 |
| S 弯转角 | **90° / 45°** | 两处变向 | 图 8 |
| 折线区胶带 | 两根，**带宽 10 cm** | 折线区两侧边界 | 2.2(3) |
| 发车区 | **固定** | 虚线框；由图 2 读图推断位于**底边直道右端**（见 CR-22，**需现场核对**） | 2.2.1(5)、图 2 |
| 停车区 | 矩形，黄色胶带包围（带宽 10 cm） | 含蓝色停车标志牌（29.7×42 cm）；由图 2 读图推断位于**底边直道**（红绿灯停车区下游、发车区上游），**需现场核对** | 2.2.1(3)(9)、图 2 |
| 红绿灯停车区 | 矩形（青色框，六个锥桶围成） | 红绿灯位于该矩形**右侧（以运行方向为参考）**，本体 60×17×6.7 cm、离地 40 cm | 2.2.1(6)(10) |
| 挡板 | 蓝色，**长 80 cm / 高 50 cm**，共 **3 处** | 每处位于内侧或外侧两锥桶之间 | 2.2.1(11)、图 5、图 6 |
| **单圈里程** | **规则未给出** | — | — |

> **⚠ 关于 16.8 m 的重要更正**：v1.0 曾把 16.8 m 读作"赛道整体长度／单圈周长"，并据此推算"0.6 m/s 约 28 s／1.8 m/s 约 9.3 s"的时间预算。经高清重读图 8 确认：该尺寸线**横跨整个虚线外框**，量的是**场地横向总体跨度**，**不是沿赛道中线累计的单圈里程**；且 16.8 m 与图中另一标注 3.06 m×2 + 1.5 m 半径圆弧的组合在几何上无法构成单圈总长。
>
> **因此：单圈里程必须现场实测**（卷尺沿中线累计，或由第一圈 SLAM 轨迹的累计弧长读出）。§4.1 的时间预算已改为**以实测里程为变量**的公式表，不再使用任何基于 16.8 m 的具体秒数。

### 2.2 现场复核表（待填）

比赛当天 / 线上赛赛道公布后，按下表实测回填。**实测值优先于标注值**。

| 参数 | 标注值 | 实测值 | 测法 | 差异原因 |
|---|---|---|---|---|
| 赛道宽度 | 1.5 m | ☐ | 卷尺量两锥桶中心距 | |
| 收窄段宽度 | 1.25 m | ☐ | 同上 | |
| 直线段长度 | 3.06 m | ☐ | 胶带/激光测距 | |
| 圆弧半径 | 1.5 m | ☐ | 弦长弦高法 | |
| 折线区顶点角 | 135° | ☐ | 角度尺 | |
| 场地横向总尺寸 | 16.8 m | ☐ | 卷尺量外框横向跨度 | |
| **单圈里程** | **未给出** | ☐ | **沿中线累计（卷尺/轮式测距仪/SLAM 轨迹弧长）** | **无标注值可比** |
| 红绿灯位置 | 随机 | ☐ | 记录赛道里程位置 | 线下赛当天由裁判决定；线上赛赛前一周公布 |
| 挡板位置 | 随机 3 处 | ☐ | 记录内侧/外侧 + 里程 | 每队上场前放置 |
| 停车检测区域尺寸 | — | ☐ | 按胶带**内侧**量 | 见 CR-07 |
| 停车区域尺寸 | — | ☐ | 按胶带**内侧**量 | 见 CR-07 |
| 发车区域尺寸 | — | ☐ | 按胶带**内侧**量 | 见 CR-07 |
| 锥桶是否已去除白色反光膜 | — | ☐ 是 / ☐ 否 | 目视检查 | 见 CR-15，**若未去除需当场向裁判提出** |
| 折线区胶带根数与宽度 | 2 根 / 10 cm | ☐ | 目视 + 卷尺 | 见 CR-05 |

### 2.3 需要向组委会确认的开放问题

| # | 问题 | 影响 |
|---|---|---|
| Q1 | 挡板是"每处 1 块、共 3 处"（2.2.1(11) 原文）还是"3 块位置独立随机"？两处表述需统一 | 影响第一圈对挡板数量的校验逻辑 |
| Q2 | **全场是否只有一个红绿灯？** 规则 2.1 与 2.4.2 均以单数描述（"红绿灯停车区域的位置在整个赛道中随机"），且公式扣减项为"**两次**红绿灯时长和"。据此推断：**只有一个红绿灯，位于某固定（当天随机确定）位置，两圈各遇到一次**。请组委会确认 | 影响第二圈是否需要重新搜索红绿灯（若位置相同，第二圈可直接用第一圈的地图/里程先验减速） |
| Q3 | 光电传感器的光束高度与宽度？车辆是否必然遮挡？若未被触发导致灯全程不亮，如何判罚？ | 直接决定 PEN-03 的应对策略（见 §1.4） |
| Q4 | 停车区域的实际可用尺寸（按胶带内侧）？ | 决定停车点的精度要求 |
| Q5 | 底盘运动学模型：阿克曼转向的轴距 L 与最大转向角？舵机中位 1520 µs 对应的实际转角？ | 决定控制律参数（见 HR-04） |
| Q6 | 底盘驱动接口（ROS 话题 / 串口协议 / CAN）？是否已提供现成驱动包？ | 决定 §8 的实际内容（当前为待定） |
| Q7 | "**停车检测区域**"与"**停车区域**"是**两个不同的矩形区域**（2.2.1(3) 并列列出）还是同一区域的两个名称？2.2.1(8) 要求"识别**停车检测区域**内的标志牌，并在进入**停车区域**后准确停在**标志牌区域内**"——请明确三者关系与标志牌的实际安装位置 | 直接决定 ESP/视觉探测范围与停车目标点的标定方式 |
| Q8 | "绿灯未亮起就通行"的判定主体是**视觉识别**还是**红绿灯系统自身的状态记录**？ | 决定 FAIL 风险是否依赖我们自己的识别正确率 |
| Q9 | 原文只规定"丢失**两个及以下**挡板和锥桶 +5 s"，**未规定丢失超过 2 个、但地图确实存在**时如何判罚 | 影响第一圈对建图完整性的自检阈值，以及是否需要现场向裁判确认 |
| Q10 | 3.4（线下赛）正文要求提交"技术手册**及视频**"，但其目录结构中**没有视频**；而 3.5.2（线上赛）明确列出 4 个视频。**线下赛是否也需提交视频？** | 影响提交物清单与是否需要在现场额外录像 |
| Q11 | 4.4 设备表把舵机写作"工作**频率**：1520us/330hz"——1520 µs 实为**脉宽中位值**而非频率（330 Hz 才是频率），原文表述有误。请确认中位脉宽确实为 1520 µs | 直接决定舵机中位标定值（HR-04、`servo.neutral_pwm`） |
| Q12 | 规则 2.1 引用"**图 1** 青色框包围的六个锥桶"，2.2.1(6) 引用"**图 2**"。两处图号不一致，指同一物 | 仅影响内部沟通，不影响设计；但引用时需注明 |

---

## 3. 设备与接口基线（R2）

### 3.1 传感器能力边界（由 HR-05 ~ HR-09 推导）

| 传感器 | 能力 | 工程边界 |
|---|---|---|
| 激光雷达 | 360°、25–30 m、10–12 Hz、0.22° 角分辨率 | 10 m 处锥桶（底部 0.28 m）张角约 1.6°，约 7 个点；**5 m 内约 3–5 个点**。**距离自适应聚类是必需的，不是优化** |
| IMU | 2.5° 动态精度、10°/h 零偏 | 不足以独立提供室外航向；仅作重力参考与短时角速度补偿 |
| 编码器 | Modbus RTU、50 µs 刷新、1000 RPM | 里程精度依赖 `scale_factor` 标定；打滑是鬼影主因 |
| 摄像头 | 1080P、150° FOV | 150° 对角视场 → 边缘畸变极大，**测距类判断必须先去畸变** |
| 处理器 | 昇腾 310、12–20 TOPS、ARM64 | AI 算力充裕，但 **Cartographer + AMCL + 视觉同时运行的 CPU 负载**才是瓶颈；建议顺序加载而非并存 |

### 3.2 场地准备需求（由 CR-05、CR-07、CR-13 ~ CR-15、CR-17 推导，v2.0 新增）

| 项 | 要求 | 依据 | 影响 |
|---|---|---|---|
| 锥桶反光膜 | **确认锥桶上无白色反光膜**；若有须去除 | CR-15 | 反光膜会使激光回波强度异常（易被误判为高反射目标）并干扰颜色判定 → 直接影响 DBSCAN 与颜色分类标定 |
| 锥桶规格 | 630×280×280 mm，塑料，红/蓝 | CR-15 | 底部投影约 280 mm 见方，是聚类间距与横向安全裕度的几何基准 |
| 胶带 | 黄色，宽 10 cm；折线区**两根**；三区域边缘各一圈 | CR-05、CR-07 | 以**胶带内侧**为区域边界（CR-07） |
| 警示线 | 线下赛外围设警示线 | CR-17 | 安全；同时会成为雷达远距离杂波源，需在 `scan_filter` 中考虑 |
| 挡板 | 蓝色，80×50 cm，共 3 处 | CR-13、CR-14 | 每队上场前放置，位置不可预知 |

### 3.3 驱动接口基线（待确认项）

| 部件 | 接口 | 状态 |
|---|---|---|
| 激光雷达 | ROS 2 topic `/scan`（`sensor_msgs/msg/LaserScan`） | 待确认驱动包 |
| IMU | ROS 2 topic `/imu/data`（`sensor_msgs/msg/Imu`） | 待确认驱动包 |
| 摄像头 | `/camera/image_raw`（V4L2 / UVC） | 待确认 |
| 编码器 | Modbus RTU → 自定义节点 | **需自研** |
| 舵机 + 电机 | 底层驱动板 → `/cmd_vel` 或专用协议 | **需确认**（见 §2.3 Q6） |

> **本节是当前文档最大的不确定项。** 在 Q6 得到答复前，第三部分中涉及执行机构的条目标记为「待定」，不得据此编码。

# 第二部分　总体设计

## 4. 评分模型反推与竞速策略

来源：规则 2.4.1「比赛评分规则」。

### 4.1 有效时间与得分公式（原文）

规则原文：

```
有效时间 T有效 = 第一圈耗时 × 30% + 第二圈耗时 × 70% + 加罚时间 − 两次红绿灯时长和

比赛得分 = (Tmax − T有效)² / (Tmax − Tmin)² × 80 + 技术报告得分

Tmax：比赛中完赛用时最长的队伍的有效时间 (s)
Tmin：比赛中完赛用时最短的队伍的有效时间 (s)
T有效：当前队伍有效时间 (s)
最终成绩由比赛成绩和技术手册成绩构成：比赛成绩总分 80 分，技术手册总分 20 分。
```

> **用词说明**：公式中写作"技术**报告**得分"，而 2.3.1 与 2.4.1 正文均称"技术**手册**（分值 20 分）"。二者指同一物，本档统一称"技术手册得分"。

### 4.2 时间预算（以实测单圈里程为变量）

**单圈里程 L_lap 规则未给出，必须实测**（见 CR-20 与 §2.2）。因此预算写成公式：

| 项 | 计算式 | 分值权重 |
|---|---|---|
| T1（第一圈） | `T1 = L_lap / v1` | × 30% |
| T2（第二圈） | `T2 = L_lap / v2` | × 70% |
| T有效（不含罚时） | `T有效 = 0.3·L_lap/v1 + 0.7·L_lap/v2 − 两次红绿灯等待和` | — |

以 v1 = 0.6 m/s、v2 = 1.8 m/s 为例（**仅为量纲示意，实际值待 L_lap 实测后代入**）：

| L_lap（实测填入） | T1 = L_lap/0.6 | T2 = L_lap/1.8 | 0.3·T1 + 0.7·T2 |
|---|---|---|---|
| 20 m | 33.3 s | 11.1 s | 17.8 s |
| 25 m | 41.7 s | 13.9 s | 22.2 s |
| 30 m | 50.0 s | 16.7 s | 28.3 s |

**罚时换算（与 L_lap 无关，可直接用于决策）**：

| 关系 | 推导 | 结论 |
|---|---|---|
| 1 s 加罚 ⟺ 第一圈多跑的秒数 | `ΔT = 0.3·ΔT1` → `ΔT1 = 1 / 0.3 = 3.33 s` | **1 s 罚时 = 第一圈 3.33 s** |
| 1 s 加罚 ⟺ 第二圈多跑的秒数 | `ΔT = 0.7·ΔT2` → `ΔT2 = 1 / 0.7 = 1.43 s` | 1 s 罚时 = 第二圈 1.43 s |
| 红绿灯等待 | 全额扣除 | 等待时间**不亏**，但"绿灯未亮通行"要罚 30 s |

**关键结论：罚时是不可逆的负向杠杆，且远贵于速度。**

- 一次 **PEN-02（未停车 +50 s）** ≈ 第二圈要多跑 71.5 s，等于彻底放弃名次。
- 一次 **PEN-08（未在比赛中建图 +50 s）** 同理。
- 因此策略的第一目标是**零罚时**，而不是刷速度。

### 4.3 得分函数：平方归一化

**这是赢家通吃型分布。**

- 与最快队的时间差被**平方放大**：落后 2 s 的得分惩罚是落后 1 s 的 **4 倍**。
- 竞速部分的 80 分极度集中于头部；技术手册的 20 分是**确定性收益**。
- 最优资源分配：**技术手册拿满（20/20），竞速求稳不判负**。

### 4.4 策略结论

| 优先级 | 目标 | 理由 |
|---|---|---|
| P0 | **零判负** | FAIL-01~06 任一触发即 0 分或取消资格 |
| P0 | **技术手册满分** | 20 分确定性收益；复现性是二元评分（DR-13），投入产出比最高 |
| P1 | 避免 50 s / 100 s 级罚时 | PEN-02/08/10/11 单次即抵消全部速度优势 |
| P2 | 避免 20~30 s 级罚时 | PEN-01/03/04 |
| P3 | 提升第二圈速度 | 平方放大的唯一正向杠杆 |

**第一圈速度建议 0.5–0.7 m/s**：以微小的 30% 权重代价换取 SLAM 质量。这个权衡是划算的——因为 PEN-08（+50 s）远超第一圈降速带来的权重损失。

---

## 5. 系统架构设计（SAD）

### 5.1 分层架构

```
┌──────────────────────────────────────────────────────────┐
│ L4 任务调度层                                             │
│   双圈状态机 · 圈次判定 · LifecycleNode 模式管理            │
│   全局看门狗(FAIL-03) · 偏离恢复(FAIL-01) · 停车计时(FAIL-02)│
├──────────────────────────────────────────────────────────┤
│ L3 决策与控制层                                           │
│   路径生成(Voronoi+样条) · 阿克曼纯追踪 · 模糊 PID          │
├──────────────────────────────────────────────────────────┤
│ L2 定位与建图层（双圈互斥）                                │
│   第一圈: Cartographer 2D SLAM                            │
│   第二圈: AMCL 定位 + EKF 里程计融合                       │
├──────────────────────────────────────────────────────────┤
│ L1 感知层                                                 │
│   锥桶检测(DBSCAN+颜色) · 挡板后处理 · 红绿灯 · 停车标志    │
├──────────────────────────────────────────────────────────┤
│ L0 硬件抽象层                                             │
│   雷达 · IMU · 编码器(Modbus RTU) · 摄像头(UVC) · 舵机电机  │
└──────────────────────────────────────────────────────────┘
```

### 5.2 双圈工作流

| 圈次 | 激活的栈 | 关键动作 | 输出 |
|---|---|---|---|
| 第一圈 | 建图栈（Cartographer） | 低速探索、锥桶跟踪、SLAM、**不停车**通过停车区（TR-07） | `.pbstream` + `.pgm/.yaml` |
| 切换 | 模式管理器 | **行进中**：保存地图（`save_map` 服务）→ **终止建图栈进程**（`cartographer_node` 非 LifecycleNode，见 ADR-003）→ configure+activate 导航栈 | 地图文件 + TF 权属转移（不停车） |
| 第二圈 | 导航栈（AMCL + 纯追踪） | 定位、路径跟踪、高速行驶、红绿灯停车、**识别标志牌并入库** | 完赛 |

**为什么必须互斥**：Cartographer 与 AMCL 都会发布 `map → odom` 变换。若同时激活，TF 树出现双父节点，定位完全失效。此外两者都大量消费 `/scan`，同时运行会导致 CPU 争抢。

### 5.3 任务状态机（v2.0 增补：圈次判定 + 行进中切换）

```
IDLE
 └→ MAPPING              第一圈：建图 + 低速锥桶跟踪（不停车通过停车区）
     ├→ TRAFFIC_APPROACH  红绿灯区域前降速（见 §1.4）→ 红灯停/绿灯走
     └→ LAP1_COMPLETE     圈次判定（见下）→ 触发模式切换
         └→ ROLLING_SWITCH   行进中切换（不停车，防 FAIL-03）
             └→ NAVIGATING      第二圈：AMCL + 纯追踪
                 ├→ TRAFFIC_APPROACH  区域前降速（两圈共用，见附录 E）
                 │   ├→ TRAFFIC_STOP  红灯 → 区域内停车等待
                 │   ├→ SAFE_HOLD     灯不亮 / 识别失效 → 默认停车（"灯不亮"≠通行）
                 │   └→ TRAFFIC_GO    绿灯 → 通行并恢复上一状态
                 ├→ PARKING_DETECT   进入停车检测区域即识别标志牌（行进中识别优先）
                 ├→ PARKING          进入停车区域 → 低速入库（三及以上车轮）
                 └→ DONE             停在停车区域内完赛（任务结束，不计 FAIL-03）
```

**异常分支**（任何状态可进入）

```
ANY → WATCHDOG_RECOVER  静止累计 > 阈值 → 后退 0.3 m → 清局部障碍 → 重规划   （FAIL-03）
                        ⚠ 仅对 MAPPING / NAVIGATING / ROLLING_SWITCH 生效；
                          TRAFFIC_STOP / SAFE_HOLD / PARKING / DONE 的静止被**豁免**
ANY → OFF_TRACK_RECOVER 偏离轨道 → 立即低速回中，10 s 内必须回到正轨         （FAIL-01）
ANY → LOCALIZATION_RECOVER  AMCL 协方差持续超阈值 → 降速 + 调用
                        `/reinitialize_global_localization`（Nav2 AMCL 标准服务）
                        ⚠ 仅第二圈（NAVIGATING）生效；重定位后协方差仍不收敛 →
                          退化为"纯里程计 + 锥桶跟踪"兜底（附录 I.5.4，防 FAIL-01）
ANY → SAFE_HOLD         `/traffic_light/state ∈ {off, unknown}` 且在红绿灯区域内
                        → 默认停车等待（"灯不亮"≠通行，详见附录 E）
```

**圈次判定（v2.0 新增，解决 v1.0 的欠规格）**

| 判据 | 说明 | 可靠性 |
|---|---|---|
| 主判据：里程累计 | 第一圈总行驶里程 ≥ `lap.detect_distance_ratio × L_lap`（`L_lap` 实测填入） | 高（需编码器标定） |
| 辅判据：发车区特征 | 重新进入发车区矩形（地图中已知）+ 航向与发车朝向一致 | 高 |
| 辅判据：地图闭合度 | Cartographer 的闭环约束被触发（`constraint_builder` 产生内部闭环） | 中（依赖闭环成功） |

**采用"三取二"表决**：任两条同时成立即判定第一圈结束并触发切换。单一判据都不足采信（里程受打滑影响、特征受视觉/地图质量影响）。

**为什么必须"行进中切换"（rolling switch）**

规则 2.4.2(8)：无人车在赛道中**任意位置**停止运行超过 10 s → 本次任务失败。而两圈之间必须完成"保存地图 → 卸载建图栈 → 加载导航栈 → 发布初始位姿"这一串操作，**其耗时不可控**（ARM 平台上 Cartographer 退出与 AMCL 收敛都可能超过 10 s）。

因此：
- **不允许**"回到发车区停车 → 保存地图"的静止式切换（v1.0 的方案，存在判负风险）；
- 必须设计为**在行进中完成切换**：车辆在第一圈末段仍以低速行驶，后台并行执行地图保存与节点生命周期转换，切换完成后再提速。
- 兜底：若切换超时，`WATCHDOG_RECOVER` 触发低速蠕动（而非静止），保证"未停止超过 10 s"。

### 5.4 部署拓扑

| 项 | 方案 |
|---|---|
| 计算平台 | 昇腾 310 板载（ARM64 / Ubuntu 22.04 / ROS 2 Humble） |
| 开发机 | x86 Ubuntu 22.04 + ROS 2 Humble（同版本，便于交叉验证） |
| 仿真 | Gazebo，按图 8 关键尺寸复刻赛道（§5.5） |
| 通信 | 无线网桥用于 RViz 远程监控；**不参与运动控制** |

### 5.5 仿真赛道的必要性（原草稿缺失）

实车调试窗口受天气、场地、电池限制。建议在 Gazebo 中按图 8 的**已知尺寸**复刻赛道（1.5 m 宽、1.25 m 收窄、1.5 m 半径圆弧、3.06 m 直线段、135° 折线区、90°/45° S 弯），用于：

1. 状态机全链路联调（不消耗实车时间）
2. 极端工况注入（雷达丢帧、视觉误检、定位发散、静止超时）
3. 参数初值扫描（前视距离、减速曲线、看门狗阈值）
4. **rolling switch 时序演练**（切换耗时是最不可控的一环，必须在仿真中先测）

> **注意**：仿真的单圈里程与真实赛道**不必相等**（16.8 m 是场地横向尺寸，非里程）。仿真只需复刻**几何要素**，用于算法与状态机验证；里程相关的参数以实测回填。

---

## 6. 架构决策记录（ADR）

### ADR-001　SLAM 算法选型

| 项 | 内容 |
|---|---|
| **背景** | 第一圈需在室外锥桶赛道完成 2D 建图，且地图必须清晰显示 3 处挡板（TR-08）。平台为昇腾 310（ARM64），算力受限。 |
| **候选** | ① Cartographer 2D　② SLAM Toolbox　③ Gmapping |
| **决定** | **Cartographer 2D** |
| **理由** | ① 哈工大（威海）以 Cartographer + AMCL + EKF + 模糊 PID + 纯追踪获第二十届该赛项国家一等奖（全国第四），技术栈经同赛事验证；② 2025 年 ACM 论文《Research on SLAM Algorithm for Mobile Robots Based on ROS2》对比三种算法：Cartographer 建图边界平滑、闭环最好、整体精度与稳定性最高，代价是算力需求最大；③ 本赛道为小尺度场地，Cartographer 的算力劣势可接受，精度优势直接转化为建图评分（PEN-07 与 PEN-08 的差别是 5 s 与 50 s）。 |
| **后果** | 需在 ARM64 上编译 Cartographer，并实测建图帧率；需配置 ≥40 个 lua 参数；需保证 `map→odom→base_link→laser` TF 链完整。 |
| **被否决方案** | SLAM Toolbox：算力友好但闭环精度略低，建图评分不容冒险。Gmapping：无闭环，室外易漂移。 |
| **风险标注（v2.1 新增）** | 文献结论**并不一致**，不能只引用支持结论：① 支持 Cartographer——ACM 2025《Research on SLAM Algorithm for Mobile Robots Based on ROS2》：Cartographer 建图**边界平滑、闭环最好、整体精度与稳定性最高**，代价是算力需求最大（已在理由②引用）；② **不完全支持**——MDPI *Eng. Proc.* 2025 对**小尺度赛车平台**的离线评测中，SLAM Toolbox 与 Cartographer 的 DSC 基本持平（**0.608 vs 0.610**）、SSIM 反而略高（**0.758 vs 0.734**，且报告称两者均"存在局部畸变与错位"）；IJAET 2025 用 SSIM 的评测亦显示 SLAM Toolbox 在**宽场景 / 障碍密集场景**最高。→ 说明 **Cartographer 的精度优势并非在所有尺度成立**，小尺度赛道场景存在被 SLAM Toolbox 反超的可能。**缓解**：建图后**必须**在 RViz2 人工复核（锥桶轮廓是否闭合、挡板是否连成连续线段），并按 §9.2 **下调** `submaps.num_range_data` 以保小尺度特征；若仍碎片化 → 执行 **ADR-001-B**。 |

### ADR-001-B　SLAM 备选方案：SLAM Toolbox（Plan B，v2.1 新增）

| 项 | 内容 |
|---|---|
| **背景** | ADR-001 决定用 Cartographer，但小尺度锥桶场景下其地图边界存在碎片化风险（见 ADR-001「风险标注」）。需要一个**代价可控**的退路，避免第一圈建图质量不达标（PEN-07 +5 s / PEN-08 +50 s）。 |
| **候选 / 决定** | 备选 **SLAM Toolbox**（`slam_toolbox`，ROS 2 原生维护）。 |
| **切换触发条件（量化）** | 实车标定后，若出现**任一**：① 连续两次建图中，**≥1 处挡板**无法在地图上形成连续线段；② **≥3 个锥桶**轮廓明显不闭合，或"鬼影"经重标定 `wheel.scale_factor` 后仍无法消除；③ Cartographer 在 ARM64 上建图帧率 < 5 Hz、导致明显漏帧。→ 判定"Cartographer 不达标"，切换 SLAM Toolbox。 |
| **切换代价（低）** | 两者均为**图优化 2D SLAM**，接口一致（都消费 `/scan` + `/odom`、都发布 `map→odom`），TF 权属与 E-2 的切换设计**不需改动**；差异仅在参数文件、launch 与建图栈的**进程级启停方式**（ADR-003 结论复用）。 |
| **接受的损失** | SLAM Toolbox 的全局一致性可能略低（见 MDPI 数据）——因此**建图后的人工 RViz 复核不能省**；`wheel.scale_factor` 标定要求**同样严格**（HR-08，建图鬼影头号嫌疑）。 |
| **被否决** | Gmapping：无回环、室外易漂移，不满足闭环与室外要求。 |

### ADR-002　控制器选型

| 项 | 内容 |
|---|---|
| **背景** | 第二圈需沿地图路径高速跟踪。车型为阿克曼转向（含舵机，见 HR-04）。 |
| **候选** | ① 自研阿克曼纯追踪 + 模糊 PID　② Nav2 Regulated Pure Pursuit　③ DWB / MPPI |
| **决定** | **自研阿克曼纯追踪 + 模糊 PID**，借鉴 Nav2 RPP 的**自适应前视距离**思路 |
| **理由** | ① 纯追踪计算量极小（仅前视距离一个核心参数），适合受限算力；② Nav2 的 RPP/DWB/MPPI 依赖完整 costmap 栈，且引入了与本任务无关的复杂度；③ 赛道宽度 1.5 m、单侧余量 470 mm，几何上并不苛刻（原草稿"窄通道难调参"的说法不成立，真实理由是栈复杂度与算力）；④ Nav2 RPP 已实现"按速度自适应前视距离"，其思路可低成本移植。 |
| **后果** | 需自行实现路径跟踪、速度规划、前视点选取；需自行处理曲率估计（用于动态前视距离）。 |
| **控制律修正** | 必须用阿克曼形式 `δ = arctan(2·L·sin(α)/L_d)`，**不是**差速形式 `ω = 2·v·sin(α)/L_d`。原草稿此处的模型与设备不符。 |

### ADR-003　双圈模式切换机制

| 项 | 内容 |
|---|---|
| **背景** | 建图栈与导航栈都发布 `map→odom`、都消费 `/scan`，不能共存；同时任意位置静止 >10 s 判负（FAIL-03），切换不能靠停车完成。 |
| **候选** | ① LifecycleNode + 自研管理器（行进中切换）　② 两个独立 launch 分两次启动　③ nav2_lifecycle_manager |
| **决定** | **LifecycleNode + 自研管理器，且切换在行进中完成** |
| **理由** | ① LifecycleNode 提供 `unconfigured → inactive → active → finalized` 受控转换，可保证 TF 权属干净转移；② 方案②会丢失进程状态、地图内存与参数上下文，且切换耗时不可控、必然停车（触发 FAIL-03）；③ `nav2_lifecycle_manager` 的 `manage_nodes` 服务模型值得借鉴，但其节点列表绑定 Nav2 组件，无法管理自研节点。 |
| **后果** | 需自行实现管理器节点，按序调用 `change_state`；切换期间车辆保持低速蠕动；切换时序必须写入测试计划（防止 TF 空窗期导致偏离，FAIL-01）。<br>⚠ **重要事实修正**：ROS 2 的 `cartographer_node`（`cartographer_ros`）**不是 LifecycleNode**，没有 `change_state` 服务；`nav2_lifecycle_manager` 也只能管理 Nav2 自身组件。因此对建图栈的"停用"必须改用**进程级管理**（启动/终止节点进程或 launch）+ Cartographer 的 `finish_trajectory` / `write_state` 服务，**不能**依赖生命周期状态机。生命周期机制只适用于**自研节点与 Nav2 组件**。<br>**未来评估项（v2.1 新增，非本周期方案）**：原版 `cartographer_node` **确实不是** LifecycleNode（该事实不变）；若要实现"IMU 自校准完成前不污染建图"这类受控启停，社区有通过改造 `node_main.cc` 或外包一层 Lifecycle 包装器使其可托管的做法。该改造**工程量与风险均高于本竞赛周期可承受范围**，故本设计**仍采用"进程级终止 + `finish_trajectory`/`write_state`"**；Lifecycle 改造仅登记为**未来评估项**，若团队有余力再评估（届时才需修订本 ADR 结论）。 |

### ADR-004　视觉方案选型

| 项 | 内容 |
|---|---|
| **背景** | 需识别红绿灯（红/绿）与停车标志牌（蓝底白字 P，29.7×42 cm）。摄像头 1080P / 150° / UVC。平台有 12–20 TOPS NPU。 |
| **候选** | ① OpenCV 传统 CV（HSV + 形态学 + 轮廓）　② NPU 部署 YOLO　③ 混合 |
| **决定** | **先传统 CV**，NPU 作为备选加速 |
| **理由** | ① 目标仅有 2 类、形态固定、背景可控，传统 CV 足够；② 无训练数据，自建数据集成本高；③ 传统 CV 参数可解释、便于现场标定，符合"现场必须能调"的比赛约束；④ 昇腾 310 的 CANN 生态部署成本显著，收益不匹配。 |
| **后果** | HSV 阈值需现场标定（尤其 S/V）；必须连续 3 帧一致性校验以抑制误检；需先去畸变（150° 视场畸变显著）。 |
| **安全兜底** | 视觉的全部输出都是"**建议**"而非"**许可**"：`traffic_light/state = off/unknown` 时不得通行，只能停车等待（见 §1.4）。 |

### ADR-005　感知层是否使用赛道先验

| 项 | 内容 |
|---|---|
| **背景** | 赛道为不规则环形（CR-01），但走向固定逆时针、发车区固定（CR-06）。是否可预置赛道拓扑先验？ |
| **候选** | ① 完全无先验（纯在线）　② 使用拓扑级先验（内外圈、逆时针）　③ 预置几何地图 |
| **决定** | **② 使用拓扑级先验，禁止③** |
| **理由** | ① 方案③属于"提前建图"，直接命中 PEN-08（+50 s），**绝对禁止**；② 方案①浪费了规则明确赠送的强先验；③ 内外圈颜色与逆时针走向是**规则公开信息**（2.2.1(12) 明示"可利用锥桶颜色来识别无人车运行方向"），与"是否提前建图"无关，可安全使用。 |
| **后果** | 锥桶颜色可直接由径向距离阈值推断（内圈=红，外圈=蓝），无需颜色传感器；但地图本身必须真实构建。 |
| **合规备注** | 开发期用自录 rosbag 离线调参是合法的；但比赛中禁止任何回放（FAIL-04）。技术手册中必须明确区分"开发期离线调参"与"比赛中禁止回放"，避免评委误读。 |

### ADR-006　红绿灯策略：默认停车（v2.0 新增）

| 项 | 内容 |
|---|---|
| **背景** | 红绿灯由**光电传感器**触发（2.4.2(1)）。若车辆未遮挡传感器，灯可能全程不亮；此时通行会被判"绿灯未亮起就通行"（+30 s）。同时"未停车"罚 50 s，"车轮不在区域内"只罚 20 s。 |
| **候选** | ① 只认绿灯（看到绿灯才走）　② 默认停车 + 绿灯才放行　③ 超时（如 30 s）无绿灯信号即通行 |
| **决定** | **② 默认停车 + 绿灯才放行；取消③的"超时即通行"** |
| **理由** | ① 方案①若不遮挡传感器会导致灯不亮、车停死 → 静止 >10 s → **FAIL-03 判负**，是最坏结局；② 方案③的"超时通行"在"绿灯未亮"时正好命中 +30 s，且可能与"未停车 +50 s"叠加；③ 方案②的期望罚时最低：**最坏情况是停在区域边缘（+20 s），不会判负、不会触发 30/50 s 档**。 |
| **后果** | 需实现"停车—等待—超时—低速蠕动重新压传感器"的循环，以保证既不判负（不停死）也不违规（不强行通过）。蠕动速度建议 ≤0.1 m/s，运动范围限制在区域内部。 |
| **数值约束** | 蠕动必须"动"，因为 FAIL-03 判据是"停止运行超过 10 s"。设计上以 `watchdog.static_timeout = 9 s` 为强制动作点。 |

# 第三部分　接口契约

> **本部分是唯一真值源。** 任何代码中的坐标系名、话题名、参数名必须与本部分逐字一致。

## 7. 坐标系与 TF 树

### 7.1 坐标系定义

| frame_id | 物理含义 | 父坐标系 | 发布者 |
|---|---|---|---|
| `map` | 全局地图坐标系（建图起点为原点） | — | Cartographer（第一圈）/ AMCL（第二圈） |
| `odom` | 里程计连续坐标系 | `map` | Cartographer（第一圈）/ AMCL（第二圈） |
| `base_link` | 车体中心（后轴中心投影到地面） | `odom` | **`ekf_node`（唯一发布者）**；`chassis_driver` 只发布 `/odom` **话题**，**不得**再发布 `odom→base_link` TF（双发布者会导致 TF 冲突） |
| `laser_link` | 激光雷达安装中心 | `base_link` | 静态 TF |
| `imu_link` | IMU 安装中心 | `base_link` | 静态 TF |
| `camera_link` | 摄像头光心 | `base_link` | 静态 TF |

### 7.2 TF 权属表（双圈互斥的核心）

| 变换 | 第一圈发布者 | 第二圈发布者 | 备注 |
|---|---|---|---|
| `map → odom` | **cartographer_node** | **amcl** | ⚠ **互斥**：同时存在会导致 TF 树冲突 |
| `odom → base_link` | ekf_node | ekf_node | 两圈一致 |
| `base_link → laser_link` | static_transform_publisher | 同左 | 需实测外参 |
| `base_link → imu_link` | static_transform_publisher | 同左 | 需实测外参 |
| `base_link → camera_link` | static_transform_publisher | 同左 | 需实测外参 |

### 7.3 TF 切换时序（v2.0 修订：行进中切换）

v1.0 的时序是"停车保存地图"，与规则 2.4.2(8)（任意位置静止 >10 s 判负）冲突。修订后：

```
① 圈次判定成立（§5.3 三取二表决）→ 进入 ROLLING_SWITCH（车辆保持低速蠕动 ≥0.15 m/s）
② 后台**异步**调用 `save_map` 服务（`map_saver_server`）落盘 .pgm/.yaml（不停车；不得用阻塞式 `map_saver_cli`）
③ 终止建图栈（`cartographer_node` **非 LifecycleNode**，须进程级停止，先调 `finish_trajectory` / `write_state`）→ 等待 /tf 中 map→odom 停止发布（超时 1.5 s）
④ configure + activate(amcl) →  发布 /initialpose（位姿取当前 odom 位姿 + 地图匹配初值）
⑤ 确认 map→odom 恢复发布（超时 3.0 s）
⑥ 恢复 NAVIGATING，逐步提速至 speed.straight
```

**时序铁律**

| 规则 | 原因 |
|---|---|
| **切换期间**车辆速度不得为 0，最低 0.15 m/s | FAIL-03：静止 >10 s 判负。**该约束仅适用于模式切换窗口**——红绿灯合规停车（`TRAFFIC_STOP` / `SAFE_HOLD`）与入库停车（`PARKING`）**不受此约束**，否则会与规则"等待红灯时长不计入"矛盾（见 §5.3、§9.7） |
| 步骤 ③→⑤ 总耗时必须 < `switch.timeout`（默认 20.0 s） | 六步分预算之和 ≈15~18 s，给蠕动留出余量；超时则进入 `WATCHDOG_RECOVER` 的蠕动分支 |
| `map→odom` 空窗期内**不得提速**、不得进入弯道 | 空窗期定位精度下降，提速会导致偏离（FAIL-01） |
| 若 ② 保存失败，允许继续行驶并在下一圈起点前重试一次 | 地图缺失 = PEN-08（+50 s），必须尽力补救 |

> **禁止**在 `map→odom` 空窗期内让车辆静止或以正常速度通过弯道。若切换发生在弯道，应提前在圈次判定中把切换点安排在**直线段**（用里程先验选择切换窗口）。

### 7.4 静态外参标定表（待实测）

| 变换 | x (m) | y (m) | z (m) | roll | pitch | yaw |
|---|---|---|---|---|---|---|
| `base_link → laser_link` | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| `base_link → imu_link` | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |
| `base_link → camera_link` | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |

> 雷达-IMU 外参错误是建图鬼影的第二大原因（第一大是编码器 `scale_factor`）。
> **摄像头俯仰角需按 CR-12 反推**：红绿灯本体离地 40 cm、高 17 cm，标志牌高 42 cm，需保证在预期识别距离（建议 3–6 m）内目标落在画面有效区域且畸变可接受。

## 8. 话题 / 服务 / Action 接口表（ICD）

> **类型书写约定（ROS 2）**：本表**全部接口类型统一使用 ROS 2 规范全称**——`<包>/msg/<类型>`（消息）、`<包>/srv/<类型>`（服务）；`ros2 topic info` / `ros2 interface show` 输出即为此格式。唯一例外：**自定义 `.msg` 文件内部的字段类型仍写短名**（见 §8.1 末尾说明）。
> **关于 Action**：本设计**不使用 ROS 2 Action**（路径跟踪为自研连续控制，非长任务），故本表实际只含话题与服务；标题保留"Action"仅为章节命名一致。
> **QoS 兼容性提醒（ROS 2 典型陷阱）**：同一话题的发布者与订阅者 **QoS 必须兼容**，否则**收不到数据**（且无报错）。传感器类话题统一 `SensorDataQoS`（best-effort），控制/状态类统一 `Reliable`；若引用第三方节点（Nav2 组件、Cartographer）自带的 QoS，必须**逐话题核对**，不得混用。
>
> **兼容性规则（ROS 2 QoS 官方语义）**：`Reliable` 发布者**可以**匹配 `Best-Effort` 订阅者；**反之不成立**——若发布者 `Best-Effort`、订阅者 `Reliable`，订阅者**收不到任何数据**。因此凡是"我们自己发布、第三方订阅"的话题，必须按第三方的 QoS 来设；"第三方发布、我们订阅"的话题，必须按第三方发布者的 QoS 来设。
>
> **QoS 基线表（本项目内部节点；第三方话题以实机核对为准，见 §A.2、附录 C.2）**
>
> | 话题 | Reliability | Durability | Depth | 说明 |
> |---|---|---|---|---|
> | `/scan`、`/scan_filtered`、`/imu/data`、`/camera/image_raw`、`/camera/camera_info` | `Best-Effort` | `Volatile` | 5 | 即 `SensorDataQoS`；高频传感器数据，允许丢帧 |
> | `/cones`、`/boards`、`/traffic_light/state`、`/parking_sign/state` | `Reliable` | `Volatile` | 10 | 本队节点间；低频、不可丢 |
> | `/cmd_vel_raw`、`/cmd_vel`、`/steering_angle`、`/odom` | `Reliable` | `Volatile` | 10 | 控制链路，不可丢 |
> | `/amcl_pose`、`/particle_cloud`、`/odometry/filtered` | 以 **Nav2 / `robot_localization` 默认值为准** | — | — | 订阅前用 `ros2 topic info -v` 核对，**不得假定为 `Reliable`** |
> | `/map` | `Reliable` | `Transient Local` | 1 | `map_server` 的 `/map` 为瞬态本地（后订阅者也能拿到最后一帧） |
>
> ⚠ **本项目最易踩的坑**：`nav2_amcl` 订阅 `/scan` 的 QoS 由 Nav2 内部决定；若本队 `lidar_driver` 发布 `/scan` 时用了与 Nav2 不兼容的 QoS，AMCL 会**静默无数据**、第二圈直接失效。此点**无法离线验证**，必须在实车上按 §A.2 命令逐条核对。

### 8.1 感知层

| 话题 | 消息类型 | 方向 | 频率 | 发布者 → 订阅者 | 状态 |
|---|---|---|---|---|---|
| `/scan` | `sensor_msgs/msg/LaserScan` | pub | 10–12 Hz | lidar_driver → SLAM / cone_detector | 待确认驱动 |
| `/scan_filtered` | `sensor_msgs/msg/LaserScan` | pub | 10–12 Hz | scan_filter → cone_detector | 待实现（需滤除警示线等远距离杂波，CR-17） |
| `/imu/data` | `sensor_msgs/msg/Imu` | pub | 100 Hz | imu_driver → ekf, cartographer | 待确认驱动 |
| `/camera/image_raw` | `sensor_msgs/msg/Image` | pub | 30 Hz | camera_driver → 视觉节点 | 待确认驱动 |
| `/camera/camera_info` | `sensor_msgs/msg/CameraInfo` | pub | 30 Hz | camera_driver → 视觉节点 | 需内参标定 |
| `/cones` | `autocar_msgs/msg/ConeArray` | pub | 10 Hz | cone_detector → 规划 / 状态机 | 待实现 |
| `/boards` | `autocar_msgs/msg/BoardArray` | pub | 10 Hz | board_detector → 状态机 | 待实现（挡板，CR-13/14） |
| `/traffic_light/state` | `std_msgs/msg/String` | pub | 30 Hz | traffic_light_detector → 状态机 | 值：`red` / `green` / `off` / `unknown` |
| `/parking_sign/state` | `std_msgs/msg/String` | pub | 15 Hz | parking_sign_detector → 状态机 | 值：`detected` / `absent`；对应 CR-10/11 |

**自定义消息 `autocar_msgs/msg/ConeArray`**

```
# autocar_msgs/msg/Cone.msg
float32 x            # 在 map 坐标系下的 x (m)
float32 y            # 在 map 坐标系下的 y (m)
string  color        # "red" | "blue" | "unknown"（由径向阈值推断，见 ADR-005）
float32 confidence   # 0.0 ~ 1.0
uint8   cluster_size # 参与聚类的点数
float32 distance     # 距车体距离 (m)

# autocar_msgs/msg/ConeArray.msg
std_msgs/Header header
Cone[] cones
```

> ⚠ **注意 `.msg` 语法差异**：自定义消息**文件内部**的字段类型必须写**短名**（`std_msgs/Header`、`Cone[]`），**不能**写成 `std_msgs/msg/Header`。上表的 `pkg/msg/Type` 规范全称只用于**话题 / 服务 / 接口**的引用，两者不可混用。

### 8.2 建图与定位

| 话题 / 服务 | 消息类型 | 方向 | 发布者 → 订阅者 | 备注 |
|---|---|---|---|---|
| `/map` | `nav_msgs/msg/OccupancyGrid` | pub | occupancy_grid_node → RViz / 状态机 | 第一圈 |
| `/submap_list` | `cartographer_ros_msgs/msg/SubmapList` | pub | cartographer_node → RViz | 建图调试 |
| `/amcl_pose` | `geometry_msgs/msg/PoseWithCovarianceStamped` | pub | **nav2_amcl**（节点名 `amcl`）→ 状态机 | 第二圈 |
| `/particle_cloud` | `nav2_msgs/msg/ParticleCloud` | pub | nav2_amcl → RViz | 定位健康度 |
| `/initialpose` | `geometry_msgs/msg/PoseWithCovarianceStamped` | pub | mission_manager → nav2_amcl | 切换时自动发布，见 §7.3 |
| `/odometry/filtered` | `nav_msgs/msg/Odometry` | pub | ekf_node（`robot_localization`）→ SLAM / 控制 | — |
| `/map_save` | `nav2_msgs/srv/SaveMap` | srv | mission_manager → **`nav2_map_server` 包的 `map_saver_server` 节点** | 保存地图（行进中触发）。⚠ **注意**：`map_saver_cli` 是**命令行工具、不提供服务**，且同步执行会阻塞；行进中保存必须使用 **`map_saver_server` 的 `save_map` 服务**并**异步调用**（回调组），否则会卡死状态机 |

### 8.3 规划与控制

| 话题 | 消息类型 | 方向 | 频率 | 发布者 → 订阅者 |
|---|---|---|---|---|
| `/race_line` | `nav_msgs/msg/Path` | pub | 1 Hz（或地图加载时一次） | path_generator → pure_pursuit |
| `/cmd_vel_raw` | `geometry_msgs/msg/Twist` | pub | 50 Hz | pure_pursuit → fuzzy_pid |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | pub | 50 Hz | fuzzy_pid → 底盘驱动 |
| `/steering_angle` | `std_msgs/msg/Float32` | pub | 50 Hz | fuzzy_pid → 底盘驱动（若底盘用角度接口） |
| `/odom` | `nav_msgs/msg/Odometry` | pub | 50 Hz | chassis_driver → ekf_node |

> ⚠ `/cmd_vel` 的语义在阿克曼平台上必须澄清：`Twist.angular.z` 表示**航向角速度**还是**舵机转角**？若底盘只接受舵机 PWM，需在 `chassis_driver` 内做 `δ → PWM(µs)` 映射，并在此表中显式声明。**此项依赖 §2.3 Q6 的答复。**

### 8.4 任务调度与状态（v2.0 增补）

| 接口 | 类型 | 说明 |
|---|---|---|
| `/race_state` | `std_msgs/msg/String` (pub) | 状态机当前状态，值取 §5.3 的状态名 |
| `/race_lap` | `std_msgs/msg/UInt8` (pub) | 当前圈次（1/2），供里程累计与日志追溯 |
| `/watchdog/status` | `std_msgs/msg/Bool` (pub) | 全局看门狗心跳（FAIL-03） |
| `/watchdog/static_time` | `std_msgs/msg/Float32` (pub) | 当前"静止"累计时长（s），用于调试与录像佐证 |
| `/off_track/count` | `std_msgs/msg/UInt8` (pub) | 累计绕错锥桶数（FAIL-01，上限 3） |
| `/mode_switch/request` | `std_msgs/msg/Bool` (pub) | 请求进入 `ROLLING_SWITCH` |
| `/<node>/change_state` | `lifecycle_msgs/srv/ChangeState` (srv) | LifecycleNode 标准转换服务 |
| `/<node>/get_state` | `lifecycle_msgs/srv/GetState` (srv) | 查询节点状态 |
| `/traffic/wait_duration` | `std_msgs/msg/Float32` (pub) | 每次等待绿灯的累计时长 (s)；第一/二圈各一次，供 **TR-06 时间扣减**与向裁判举证（**设计卡 E-1**） |

## 9. 参数基线表（PARAM）

> 本表是代码与手册的**共同引用源**。所有参数以 YAML 形式存放于 `autocar_bringup/config/`，节点通过 `declare_parameter` 读取。

### 9.1 锥桶检测（cone_detector）

| 参数 | 默认值 | 范围 | 含义 | 标定方法 |
|---|---|---|---|---|
| `dbscan.eps_near` | 0.15 m | 0.10–0.30 | 距离 < 5 m 时的聚类半径 | 实测锥桶点云间距 |
| `dbscan.min_samples_near` | 3 | 2–6 | 距离 < 5 m 的最小点数 | 同上 |
| `dbscan.eps_far` | 0.25 m | 0.15–0.40 | 5–10 m 时的聚类半径 | 同上 |
| `dbscan.min_samples_far` | 2 | 2–4 | 5–10 m 的最小点数 | 同上 |
| `detection.max_range` | 10.0 m | 6–15 | 超过此距离不做锥桶检测 | 雷达点密度不足 |
| `detection.min_range` | 0.30 m | 0.2–0.5 | 过滤车身附近噪点 | — |
| `color.radial_threshold` | **待标定** | — | 径向距离阈值：小于判为红（内圈） | 赛前标定跑 §9.1.1 |
| `board.merge_gap` | 0.50 m | 0.3–0.8 | 挡板合并的最大聚类间距 | 挡板长 0.80 m（CR-14） |
| `board.merge_min_clusters` | 3 | 2–5 | 触发合并的最少聚类数 | — |
| `cone.min_confidence` | 0.50 | 0.3–0.8 | 锥桶输出置信度下限，低于此值不发布（**设计卡 E-4**） | 按误检率实测调整 |

> **场地前提（CR-15）**：若锥桶保留白色反光膜，回波强度异常会造成聚类半径与点数统计失真，本节所有标定作废。**赛前必须确认反光膜已去除**。

**§9.1.1 径向阈值标定流程**：在赛道上低速跑一圈 → 记录所有锥桶的质心 (x, y) → 计算到赛道中心点的径向距离分布 → 取红/蓝两簇的分界值 → 回填本表。

### 9.2 Cartographer（第一圈建图）

| 参数 | 值 | 含义 | 备注 |
|---|---|---|---|
| `TRAJECTORY_BUILDER_2D.use_imu_data` | `true` | 使用 IMU 提供重力参考 | — |
| `TRAJECTORY_BUILDER_2D.min_range` | 0.30 m | 过滤车身附近噪点 | — |
| `TRAJECTORY_BUILDER_2D.max_range` | 15.0 m | 有效距离 | 室外锥桶场景；需滤除警示线（CR-17） |
| `TRAJECTORY_BUILDER_2D.missing_data_ray_length` | 5.0 m | 缺失数据射线长度 | — |
| `POSE_GRAPH.optimize_every_n_nodes` | 90 | 全局优化频率 | 减小可增加优化次数但耗算力 |
| `TRAJECTORY_BUILDER_2D.submaps.num_range_data` | **待实测（调优方向：下调）** | 每个子图的激光帧数 | 影响子图边界与**小尺度特征保留率**。锥桶场景建议**从默认值（约 90）下调**，使子图更小、数量更多，提高小目标轮廓的保留（ADR-001 风险标注）；具体值实车扫描后确定 |
| `TRAJECTORY_BUILDER_2D.ceres_scan_matcher.translation_weight` | 待调 | 平移先验权重 | 鬼影时优先调此项 |
| `TRAJECTORY_BUILDER_2D.ceres_scan_matcher.rotation_weight` | 待调 | 旋转先验权重 | 同上 |
| `TRAJECTORY_BUILDER_2D.motion_filter.*` | 待实测 | 运动滤波 | 低速时需调小 |

### 9.3 AMCL（第二圈定位）｜ROS 2 实现为 `nav2_amcl`

| 参数 | 值 | 含义 |
|---|---|---|
| `set_initial_pose` | `true` | 切换时自动发布初始位姿（避免人工干预 PEN-11） |
| `initial_pose.x` | **待标定** | 发车区标记点 x |
| `initial_pose.y` | **待标定** | 发车区标记点 y |
| `initial_pose.yaw` | **待标定** | 发车朝向（逆时针，CR-06） |
| `initial_pose.covariance` | 0.25 | 初始不确定度 |
| `min_particles` | 500 | 粒子数下限 |
| `max_particles` | 2000 | 粒子数上限 |
| `laser_model_type` | `likelihood_field` | 激光模型 |
| `update_min_d` | 0.10 m | 最小平移触发更新 |
| `update_min_a` | 0.10 rad | 最小旋转触发更新 |
| `amcl.covariance_position_threshold` | **0.5 m²** | 位置协方差阈值：`/amcl_pose` 超过判**定位退化**（**设计卡 E-5**，v2.1 新增） |
| `amcl.covariance_yaw_threshold` | **0.3 rad²** | 航向协方差阈值（同上） |
| `amcl.degradation_duration` | **2.0 s** | 协方差**持续**超阈值的时长 → 触发 `LOCALIZATION_RECOVER`（§5.3） |

> **⚠ 落地文件区分（v2.1 R77）**：上表前段（`set_initial_pose` ~ `update_min_a`）是 nav2_amcl 原生参数，落在 `autocar_slam/config/amcl.yaml`；后三项 `amcl.covariance_*` / `amcl.degradation_duration` **不是** AMCL 原生参数，由 `race_manager` 节点读取，**唯一落地 = `autocar_race_manager/config/race_manager.yaml` 的 `amcl.*` 键**。整定健康度阈值**勿改 `amcl.yaml`**（该文件只以节点名 `amcl` 加载，任何其它顶层节点段都不会被读取）。

> **发车区标定方法**：在发车区地面用黄色胶带标记起始位姿点；每次上电后将车辆放置于标记点，AMCL 自动发布初始位姿。这解决了原草稿提到的"AMCL 初始位姿错误"风险，且**不构成人为干预**（放置车辆发生在计时开始前）。
>
> **v2.0 修订**：两圈之间的初始位姿**不再依赖发车区标记**，而是取"当前 odom 位姿 + 地图匹配初值"（§7.3 步骤④），因为切换发生时车辆正在行进中。

### 9.4 路径生成（path_generator）

| 参数 | 值 | 含义 |
|---|---|---|
| `spline.smoothing` | 0.5 | 三次样条平滑系数 |
| `voronoi.prune_distance` | 1.0 m | Voronoi 图修剪距离 |
| `path.point_spacing` | 0.05 m | 输出路径点间距 |
| `path.min_segment_length` | 0.30 m | 最短有效段 |
| `fold_zone.margin` | 0.25 m | 折线区中线跟踪的横向收紧量（PEN-06，防压线） |
| `path.safety_margin` | **待标定** | 路径相对锥桶的横向安全裕度（防撞桶 PEN-05，**设计卡 E-7**） |
| `path.min_turning_radius` | **待计算** | 最小转弯半径（阿克曼可达约束：`R = wheelbase / tan(max_steering_angle)`，**设计卡 E-7**） |
| `path.corridor_width_min` | **1.0 m** | Voronoi 中轴走廊宽度**下限**：低于此值的走廊段判为**不可通过并丢弃**（**设计卡 E-7**，v2.1 新增） |
| `path.corridor_width_max` | **1.6 m** | 走廊宽度**上限**：高于此值视为**误并内外圈边界**，该段丢弃并在缺口处用样条**桥接**（同上） |
| `boundary.auto_calibrate` | **true** | **颜色未标定时的自标定径向兜底**：`color.radial_threshold ≤ 0` 时 `Cone.color` 全为 `unknown` → 边界为空 → `/race_line` 永不发布 → **车一步不动**（FAIL-03 静止判负）。开启后改用"**锥桶点云质心当赛道中心 + 中位半径当分界**"自动分内/外（ADR-005 径向先验的自标定版），**无需任何人工标定即可出行走路径**；标定后自动不再触发（**R57 新增，设计卡 E-7**） |
| `boundary.auto_min_per_side` | **2** | 兜底要求的**每侧最少边界点数**；不足则判"径向先验不成立"并按"保持上一次路径"处理（**R57 新增**） |

> **走廊宽度窗口的推导（v2.1 新增，替代原"合理区间"的拍脑袋表述）**：赛道宽度 **1.5 m**（CR-02），局部收窄 **1.25 m**（CR-19），锥桶间距 **< 1.5 m**（CR-03），车宽 **0.56 m**（HR-01）。
> - **下限 1.0 m**：以最窄段 1.25 m 为基准，扣除锥桶检测的横向抖动与聚类质心误差后取保守值；仍保证 0.56 m 车宽通过。
> - **上限 1.6 m**：正常走廊宽度≈赛道宽度 1.5 m；略放宽到 1.6 m 作为容差，**超过即意味着 Voronoi 中轴穿过了一个"本应闭合"的缺口**（把内外圈边界并在了一起），必须裁剪。
> - ⚠ 两值均为**由规则尺寸推导的初始值**，须在实测 `L_lap` 与真实锥桶间距后复核（“待实机复核”，不改变 §9 作为唯一来源的地位）。

### 9.5 纯追踪 + 模糊 PID（控制）

| 参数 | 值 | 含义 |
|---|---|---|
| `wheelbase` | **待实测** | 阿克曼轴距 L（决定 `δ` 计算，HR-04） |
| `max_steering_angle` | **待实测** | 最大转向角（硬件限位） |
| `lookahead.mode` | `steering_adaptive` | 前视距离模式：**转向角驱动**（PP-DSC 思路，v2.1 变更）；备选 `curvature_adaptive` |
| `lookahead.min` | 0.40 m | 前视距离下限（满舵 / 急弯） |
| `lookahead.max` | 0.60 m | 前视距离上限（直道） |
| `lookahead.steering_gain` | **0.5** | 转向角驱动增益（v2.1 新增）：`L_d = lookahead.min + (max − min)·(1 − steering_gain·\|δ\|/δ_max)` |
| `lookahead.curvature_gain` | **待调（备选）** | 曲率驱动增益；**仅在转向角不可得时启用**（`lookahead.mode = curvature_adaptive`）。转向角是控制层自身输出，无需额外估计，故降为备选（详见附录 J.5.1） |
| `speed.straight` | 1.8–2.0 m/s | 直道目标速度 |
| `speed.curve` | 0.9–1.2 m/s | 弯道目标速度 |
| `speed.steering_decay` | **0.7** | 速度随转向角的衰减系数（v2.1 新增）：`v_target = speed.straight·(1 − steering_decay·\|δ\|/δ_max)`，与 `lookahead.steering_gain` 协同（PP-DSC，附录 J.5.1） |
| `speed.fold_zone` | ≤ 0.8 m/s | 折线区限速（PEN-06） |
| `speed.traffic_approach` | ≤ 0.30 m/s | 红绿灯接近区限速（PEN-03） |
| `speed.parking_approach` | ≤ 0.30 m/s | 绿灯起步后至停车区的限速（CR-22：加速距离短） |
| `speed.rolling_switch` | 0.15–0.25 m/s | 切换期间的低速蠕动（防 FAIL-03，§7.3） |
| `control.rate` | 50 Hz | 控制频率 |
| `servo.neutral_pwm` | **1520 µs** | 舵机中位（HR-04，非 1500） |
| `servo.max_pwm_delta` | 待标定 | 满舵脉宽偏移 |
| `pid.kp` / `pid.ki` / `pid.kd` | **待调** | 模糊 PID 基础增益（横向误差 → 转向，**设计卡 E-6**） |
| `pid.integral_limit` | **待调** | 积分限幅（防积分饱和） |
| `fuzzy.enable` | `true` | 启用模糊自整定（±Δkp/Δki/Δkd 动态修正） |
| `fuzzy.rule_scale` | **待调** | 模糊规则输出缩放系数 |
| `control.max_steering_rate` | **待标定** | 转向速率限幅 (rad/s)，防舵机抖动/打滑 |
| `control.max_accel` / `control.max_decel` | **待调** | 加 / 减速度限幅 (m/s²) |

### 9.6 视觉（traffic_light / parking_sign）

| 参数 | 值 | 含义 |
|---|---|---|
| `hsv.red_h_ranges` | `[0,10] ∪ [170,179]` | 红色 H 区间（**OpenCV 的 H 通道为 0–179**，故上界为 179；v2.1 修正原 180 的笔误） |
| `hsv.green_h_range` | `[40,80]` | 绿色 H 区间 |
| `hsv.blue_h_range` | `[100,140]` | 蓝色 H 区间（停车标志牌蓝底，**现场标定**；**设计卡 E-3**） |
| `hsv.s_min` | **现场标定** | 饱和度下限（光照相关） |
| `hsv.v_min` | **现场标定** | 明度下限（光照相关） |
| `traffic.confirm_frames` | 3 | 连续确认帧数（抑制误检） |
| `traffic.min_area` | 待定 | 最小有效面积（按 CR-12：本体 60×17 cm，离地 40 cm，反推像素面积） |
| `traffic.circularity_min` | 0.70 | 圆形度阈值（blob 筛选） |
| `parking.confidence_threshold` | 0.70 | 停车标志置信度阈值（按 CR-11：29.7×42 cm 反推像素面积） |
| `parking.detect_in_motion` | `true` | **优先行进中识别**，避免 FAIL-02 |
| `camera.calibration_file` | `config/camera_info.yaml` | 内参标定文件（必需，150° 畸变） |

### 9.7 状态机、切换与看门狗（v2.0 增补）

| 参数 | 值 | 含义 |
|---|---|---|
| `watchdog.static_timeout` | **9.0 s** | 静止判定阈值（规则限 10 s，留 1 s 余量，FAIL-03） |
| `watchdog.recover_speed` | 0.15 m/s | 恢复阶段的最低蠕动速度（**不得为 0**） |
| `watchdog.recover_back_dist` | 0.30 m | 恢复后退距离 |
| `watchdog.recover_max_attempts` | 3 | 最大恢复尝试次数 |
| `watchdog.exempt_states` | `TRAFFIC_STOP, SAFE_HOLD, PARKING, DONE` | 这些状态下的静止**不触发**看门狗。**必需**：红绿灯红灯倒计时上限 **10 s > `static_timeout`(9 s)**，若不豁免，合规停车会被误判为"静止超时"而触发恢复（逻辑冲突，见 §5.3） |
| `traffic_light.wait_timeout` | 30.0 s | 等待绿灯的**告警**上限；超时后**不通行**，仅打 WARN 并置内部 `_traffic_crawl` 标志（ADR-006 原意是"低速蠕动重新压传感器"）。⚠ **v2.1 R61 更正**：该标志**从未对外发布**，而 `autocar_control` 限速表按 `/race_state` 取值（`SAFE_HOLD`/`TRAFFIC_STOP` 均为 0.0）→ **蠕动实际不会发生**。等待态的**真实出口**见下行 `traffic_light.hold_timeout` |
| `traffic_light.crawl_speed` | ≤ 0.10 m/s | **R61 更正：当前不生效**（本节点只发状态 + `wait_duration`，不限速）。保留以对齐 ADR-006 的原始意图，待 `autocar_control` 接入后再启用 |
| `traffic.entry_by_light` | **`true`** | **R61 新增**：`traffic.stop_point_*` **未标定**时的**视觉入口**总开关（与 `parking.entry_by_sign` 同构）。未标定时坐标入口自动停用，改由 `/traffic_light/state` 确认灯态进入 `TRAFFIC_APPROACH`。⚠ **仅在坐标入口不可用时启用**：标定后只走坐标入口，行为与冻结设计一致 |
| `traffic.visual_trigger_states` | `['red']` | R61 新增：哪些**确认**灯态触发视觉入口（**边沿触发**，同一红灯光段只触发一次，防反复进出）。默认只 `red`——绿灯时车本在正常行驶，进状态机是空转 |
| `traffic_light.hold_timeout` | **60.0 s** | R61 新增：等待态超时放行阈值，**0 = 禁用**。仅对下行列出的状态生效 |
| `traffic_light.hold_timeout_states` | `['SAFE_HOLD']` | R61 新增：允许"超时放行"的等待态。**分级处理**——`SAFE_HOLD`（全程**未确认红灯**：灯不亮 / 相机看不到灯 / 检测节点未起）超时放行，避免停死判负（日志打 **PEN-03 风险 WARN**）；`TRAFFIC_STOP`（**已确认红灯**）**必停等绿、不放行**（闯红灯的判罚远重于超时等待）。若现场确定"宁可冒险也不 DNF"，可把 `'TRAFFIC_STOP'` 一并加入 |
| `traffic.approach_distance` | **待标定** | 距红绿灯停车区域多远进入 `TRAFFIC_APPROACH`（**设计卡 E-1**）；应 ≥ 制动距离 + 区域识别余量 |
| `parking.stop_recognition_limit` | **9.0 s** | 停车识别上限（规则限 10 s，FAIL-02） |
| `parking.approach_distance` | **待标定** | 距停车检测区域多远进入 `PARKING_DETECT`（**设计卡 E-3**）；应 ≥ 识别所需距离 + 减速距离 |
| `parking.target_x` / `parking.target_y` | **待标定** | 入库停车目标点（`map` 坐标系，**设计卡 E-3**）；按 CR-07 **胶带内侧**标定 |
| `parking.target_yaw` | **待标定** | 停车目标航向（应对准入库姿态，CR-10） |
| `parking.in_zone_wheels_min` | **3** | 判定入库成功所需车轮数下限（TR-04、PEN-04） |
| `parking.entry_by_sign` | **`true`** | **v2.1 R60 新增**：第二圈 `/parking_sign/state=detected` 即**直接进 `PARKING`**（视觉入口，与坐标标定无关）。`parking.target_*` 未标定时这是**唯一**的入库入口 |
| `parking.detect_travel_limit` | **8.0 m** | R60 新增：`PARKING_DETECT` 内里程上限——超过仍无标志则**放弃入库返回 `NAVIGATING`**。本态原设计没有出口，否则会以 0.30 m/s 蠕动到永远 |
| `parking.done_travel_fallback` | **1.0 m** | R60 新增：`parking.zone_rect` 未标定时的 `DONE` 里程兜底（入库后行驶 ≥ 该值即完赛） |
| `parking.done_min_travel` | **0.3 m** | R60 新增：静止兜底的**前置里程门槛**，防入库瞬间被误判为"已停稳" |
| `parking.done_stall_speed` | **0.05 m/s** | R60 新增：静止判据速度阈值（取 `_on_odom` 的 `|v|`） |
| `parking.done_stall_duration` | **2.0 s** | R60 新增：已行驶 ≥ `done_min_travel` 且持续静止 ≥ 该时长 → 判 `DONE` |
| `off_track.return_timeout` | **9.0 s** | 偏离回归上限（规则限 10 s，FAIL-01） |
| `off_track.max_wrong_cones` | **3** | 绕错锥桶上限（FAIL-01） |
| `lap.lap_length` | **待实测** | 单圈里程 L_lap（CR-20，规则未给出） |
| `lap.detect_distance_ratio` | 0.95 | 里程判据：累计 ≥ 该比例 × `lap_length` |
| `lap.detect_votes` | 2 | 三取二表决所需票数（§5.3）。实际取 `min(detect_votes, 可用票数)`，见 `lap.heading_vote_enable` 行 |
| `lap.start_zone_rect` | **待标定** | 发车区矩形 `[x_min,y_min,x_max,y_max]`（map 系）；退化（全 0）时"发车区票"恒 False |
| `lap.start_yaw` / `lap.start_yaw_tol` | **待标定** / 0.35 rad | 发车朝向与其容差 |
| `lap.revisit_radius` | 2.0 m | 闭合代理：新旧 submap 中心距离阈值 |
| `lap.revisit_index_gap` | 8 | 闭合代理：新旧 submap 索引最小间隔。⚠ cartographer 每约 90 帧收尾一个 submap，跑一圈只产生 1~2 个 → **该票第一圈几乎不可能成立**，故只作加分票 |
| `lap.closure_vote_enable` | `true` | 闭合代理票开关 |
| `lap.heading_vote_enable` | **`true`** | **v2.1 R59 新增**：`lap.lap_length` 未标定（≤0）时，里程票改用"净航向环绕 ≈ ±2π"这一**标定无关替身**（`ΣΔyaw` 取自 `map→base_link`）。二者共同占里程票**这一票**，故票数结构不变；标定 `lap_length` 后精确里程判据自动恢复 |
| `lap.heading_wrap_tol` | **0.10** | R59 新增：`|ΣΔyaw| ≥ 2π(1−tol)` 判为走完一圈。**tol 越小触发越晚越准**——0.25 会在一圈 76% 处误触发，0.10 为 91%（真实闭合点前约 1.1 m） |
| `lap.heading_min_distance` | **5.0 m** | R59 新增：本圈里程下界门槛，防起步静止/位姿抖动凑出 2π |
| `lap.heading_max_step` | **0.50 rad** | R59 新增：单 tick `|Δyaw|` 上限，超限视为 TF 跳变**丢弃该增量**（但**仍推进参考航向**，否则累加器永久冻结） |
| `switch.timeout` | 20.0 s | `ROLLING_SWITCH` 总超时（§7.3）；须 > 六步分预算之和 |
| `switch.prefer_straight` | `true` | 优先在直线段安排切换窗口 |
| `map.save_dir` | `~/maps/` | 地图保存目录 |
| `map.file_name` | `race_map` | 地图文件名 |

### 9.8 里程计融合（`ekf_node`，`robot_localization`）｜v2.1 新增

> `ekf_node` 是 `odom→base_link` 的**唯一发布者**（§7.1）。本表为 v2.1 新增——此前 §9 只有 Cartographer（§9.2）与 AMCL（§9.3）参数，缺少连接二者的**里程计融合**基线（见**设计卡 E-5**）。配置以 YAML 数组形式书写，顺序为 `[x, y, z, roll, pitch, yaw, vx, vy, vz, vroll, vpitch, vyaw, ax, ay, az]`。

| 参数 | 值 | 含义 |
|---|---|---|
| `frequency` | 50.0 Hz | 滤波输出频率（与 §8.3 `/odometry/filtered` 一致） |
| `two_d_mode` | `true` | 平面运动，锁定 z / roll / pitch |
| `publish_tf` | `true` | 发布 `odom→base_link`（**唯一发布者**） |
| `sensor_timeout` | 0.1 s | 传感器超时 |
| `transform_timeout` | 0.0 s | TF 等待容忍 |
| `odom0` | `/odom` | 轮式里程计输入（`chassis_driver`，§8.3） |
| `odom0_config` | `vx=true, vyaw=true`（其余 false） | 取线速度与航向角速度 |
| `odom0_differential` | `true` | 差分，抑制累积零偏 |
| `odom0_queue_size` | 10 | — |
| `imu0` | `/imu/data` | IMU 输入（100 Hz，§8.1） |
| `imu0_config` | `yaw=true, vyaw=true`（其余 false） | 航向与航向角速度 |
| `imu0_differential` | **`false`（实车标定决定）** | 使用绝对航向。**取舍见下方说明**：`false` 采用 IMU 的重力参考绝对航向（多个集成的默认，如 `imu_pipeline`、`ros2_rover`）；若实测**静止漂移显著**导致 AMCL 在低速时持续旋转，则改 `true`（差分抑制零偏，代价是首测成为零点基准） |
| `imu0_remove_gravitational_acceleration` | **`true`（须与驱动核对）** | 去除重力分量。**若 IMU 驱动已做去重力**（静态 `linear_acceleration.z ≈ 0`），此处必须改 `false`，否则**双重去重力** |
| `imu0_queue_size` | 10 | — |
| `print_diagnostics` | `true` | 输出诊断 |
| `wheel.scale_factor` | **待标定** | 轮式里程计比例因子；**建图鬼影头号嫌疑**（HR-08、§3.1）。标定：直行 10 m 与卷尺实测对比（§12.1） |

> **IMU 精度提示（HR-07）**：IMU 动态精度仅 **2.5°**、零偏稳定性 10°/h，**不足以单独支撑航向**。因此航向的最终真值由**激光扫描匹配**（Cartographer / AMCL）给出，IMU 仅提供短时角速度与重力参考。若实测航向抖动偏大，应下调 `imu0_config` 中 `yaw` 的权重或改用差分。
>
> **`imu0_differential` 与 `imu0_remove_gravitational_acceleration` 的取舍（v2.1 补齐，避免"照抄单一博客"）**：
>
> - **`imu0_differential` 不是"越差分越好"**：设为 `true` 后，EKF 会把**第一帧测量当作零点基准**，之后只融合增量——这能抑制零偏累积，但**丢掉绝对航向**；设为 `false` 则直接融合 IMU 的**重力参考绝对航向**（`robot_localization` 官方文档：绝对航向是"正确去除重力加速度"的前提）。两个独立的集成参考（`imu_pipeline`、`ros2_rover`）均取 **`false`**，故**本设计的默认值保持 `false`**。
> - **什么时候改成 `true`**：只有当**实车实测**表明"IMU 航向零偏注入 → AMCL 在**静止 / 低速**时持续旋转 → 定位发散"（附录 I.9 已登记该失效模式）时，才改 `true` 以抑制漂移。**这是一个实车标定决定项，不是可以先验拍定的值。**
> - **`imu0_remove_gravitational_acceleration` 必须与驱动核对**：该参数为 `true` 表示"由 EKF 去除重力"。若 IMU 驱动**已**去除（静态 `linear_acceleration.z ≈ 0`），EKF 再设 `true` 就是**双重去重力**，会污染 `az` 融合。核对方法见 §A.2「IMU 静态漂移」一行：静置 60 s 看 `linear_acceleration.z` 是 ≈±9.81（未去）还是 ≈0（已去）。

# 第四部分　比赛技术手册模板（20 分直接交付物）

> **⚠ 重要前提（v2.0 新增）**：**技术手册的官方模板不在规则 PDF 内。** 规则 4.2 明确："技术手册模版及比赛道具链接：`https://pan.baidu.com/s/1Vebj2fprITROUD7aCJFXig?pwd=HGHT` 提取码：`HGHT`"。
>
> 因此：
> 1. 本部分给出的是**结构建议**，用于内部写作对齐；
> 2. **最终成稿的章节顺序、封面、声明文本必须以官方模板为准**；
> 3. 原创性/使用授权声明的文本也须从官方模板中取得（SEC-07、DR-14），不得自行拟写。

## 10. 提交物与命名

规则 3.4（线下赛）与 3.5.2（线上赛）原文要求：

```
室外ROS无人车赛_学校名称_队伍编号.zip
├── 室外ROS无人车赛_学校名称_队伍编号_技术手册.pdf
├── 室外ROS无人车赛_学校名称_队伍编号_1号机.mp4        （仅线上赛）
├── 室外ROS无人车赛_学校名称_队伍编号_2号机.mp4        （仅线上赛）
├── 室外ROS无人车赛_学校名称_队伍编号_3号机.mp4        （仅线上赛）
├── 室外ROS无人车赛_学校名称_队伍编号_赛道测量视频.mp4  （仅线上赛）
└── 室外ROS无人车赛_学校名称_队伍编号_源程序文件夹
```

| 纪律项 | 内容 | 依据 |
|---|---|---|
| 提交邮箱 | `smartcarX@163.com` | 3.4 / 3.5.2 |
| 提交次数 | **每队只限提交一次** | 3.4 / 3.5.2 |
| 命名示例 | `室外ROS无人车赛_北京理工大学_NCSC2024HBA26PU.zip` | 3.4 |
| 队伍编号查询 | 大赛报名官网 `https://www.smartcar.zone/` | ELIG-05 |
| 线下赛内容 | 源码 + 技术手册 | 3.4 |
| 线上赛内容 | 3 个视角运行视频 + 赛道测量视频 + 技术手册 + 源码 | 3.5 |

**提交纪律**：提交前完成 §13 的自查清单；参考实例见 ELIG-09（线下/线上各一份网盘实例）。

## 11. 手册章节模板

> 评分项（规则 2.3.1）：案例创新性 0~1、实用性 0~1、复杂度 0~1、复现性 **0 或 1**（二元）。
> 策略：**至少 3 个案例瞄准复现性满分（步骤清晰、源码可跑），2 个瞄准创新性。**

### 11.1 正文章节（结构建议，以官方模板为准）

| 章节 | 页数建议 | 内容 | 对应评分项 / 规则 |
|---|---|---|---|
| 封面与授权声明 | 1 | 队伍信息、原创性/使用授权声明 | SEC-07、DR-14 |
| 1 任务与系统概述 | 2 | 规则理解、系统架构图、技术栈、双圈策略 | 结构规范 |
| 2 硬件平台与传感器标定 | 3 | 设备清单、安装位置、外参标定、相机内参标定、编码器 scale 标定 | **复现性**；考察能力 4 |
| 3 感知方案 | 4 | DBSCAN 距离自适应（案例 1）、径向颜色分类（案例 4）、挡板合并 | 创新 + 复杂度；考察能力 4 |
| 4 建图方案 | 4 | Cartographer lua 逐参数说明、鬼影排查过程与数据 | **复现性**；考察能力 3 |
| 5 定位与导航 | 3 | AMCL 自动初始位姿、Voronoi 中心线生成、样条平滑 | 实用性；考察能力 3 |
| 6 运动控制 | 4 | **阿克曼纯追踪推导**、模糊 PID 隶属度函数与规则表（案例 3） | **复杂度** |
| 7 任务调度与安全 | 4 | 双圈状态机、圈次判定、行进中切换、全局看门狗（案例 5） | **创新性**；考察能力 2 |
| 8 案例集 | 12–16 | ≥5 个案例，按 §12 模板 | 全部四项 |
| 9 源码说明与复现步骤 | 3 | 目录树、依赖安装、编译、运行、参数文件说明 | **复现性**关键；考察能力 1 |
| 附录 | 2 | 参数基线表（引用本文档 §9）、测试记录摘要 | 复现性 |

> **能力覆盖自查**：规则 3.3 列出 4 项重点考察能力（程序设计、ROS 应用、SLAM 与自主导航、传感器应用），上表已逐项映射，确保手册无能力盲区。

### 11.2 案例数量与计分规则（规则原文要点）

- 技术手册**分值 20 分**；以案例形式书写，**最少不低于 5 个案例**；
- **每个案例最高 4 分**；
- **多于 5 个案例时，取所有案例中最好的 5 个案例的成绩计算**；
- 单案例评分细则：创新性 0~1.0、实用性 0~1.0、复杂度 0~1.0、复现性 0 或 1。

> **策略含义**：因为取最好的 5 个，**多写有下界风险为零、上界有增益**——建议提交 6 个案例（5 主 + 1 备），但要保证第 6 个不是凑数（第 6 个若明显低质，虽不扣分，仍会占用评阅读时间并可能影响"结构规范"印象）。

### 11.3 线上赛视频脚本（依规则 3.5.1，v2.0 新增）

| 机位 | 要求（原文） | 现场执行脚本 |
|---|---|---|
| 1 号机（**移动机位**） | 跟随无人车，视角需**清晰显示无人车的全部轮廓以及无人车周围的跑道信息**；建议参赛同学**用手机跟随录制** | 由一名队员手持手机，在车侧后方 2–4 m 处跟随；画面内始终包含整车 + 左右两侧锥桶 |
| 2 号机 | 展示上位机监控信息：**导航程序运行的终端输出** + **计时的秒表**（电脑/在线秒表/实物秒表均可）；建议固定机位或电脑共享屏幕画面 | 一屏内分两个窗口：左为 `ros2 launch` 终端（含 `/race_state`、`/race_lap` 滚动输出），右为秒表；**秒表必须在计时开始时归零** |
| 3 号机（**固定机位**） | 展示**赛道整体画面**；建议固定机位 | 三脚架架高俯拍，覆盖整个赛道 |
| 解说 | **建议**对无人车正在实现的功能和其他功能进行解说 | 按状态机节点讲解（建图 / 切换 / 红绿灯 / 入库） |
| 时长 | **不得超过 10 分钟** | 剪辑前先确认总长；超长属不合规 |
| 测量视频 | 另录一段，参考**图 8** 展示**赛道宽度、各弧度区域的半径长度、直线区域长度**；可单独录制 | 用卷尺逐项测量并读数入镜，与 §2.2 复核表一致 |
| 禁止项 | **禁止使用遥控器控制**；视频中无人车须自主移动、不得人为干预；**禁止使用工具对视频修改或造假** | 出现视频遥控或造假 → **取消比赛资格**（FAIL-05） |

## 12. 案例写作模板

**每个案例必须严格遵循以下 7 段结构**（这是拿满"复杂度"分的关键）：

```
### 案例 N：<一句话标题>

#### N.1 问题背景
（遇到了什么现象？为什么这是个问题？关联哪条规则/需求 ID）

#### N.2 分析过程
（数据、排查步骤、被排除的假设。要有实测数据，不能只有结论）

#### N.3 方案设计
（原理、公式、流程图。说明为什么选它而不是别的）

#### N.4 关键实现
（核心代码片段，标注文件名与函数名。代码必须与源码逐字一致）

#### N.5 实验数据
（对比表格：改进前 vs 改进后的量化指标）

#### N.6 复现步骤
（1. 安装依赖 … 2. 编译 … 3. 运行 … 4. 预期输出 …）

#### N.7 结论与局限
（解决了什么、还剩什么没解决）
```

### 12.1 六个案例的预设骨架（v2.0 扩充为 6 个）

**案例 1：DBSCAN 距离自适应参数策略**（高复现性）
- 问题：固定 `eps`/`min_samples` 导致远距离锥桶漏检或近距离多锥桶误合并
- 数据：实测不同距离下的锥桶点数（5 m ≈ 3–5 点，10 m ≈ 1–2 点）
- 方案：分段参数表（见 §9.1）
- 前置条件：锥桶已去除白色反光膜（CR-15），否则数据不可复现——这一点必须在案例中写明

**案例 2：Cartographer 室外锥桶场景参数调优与鬼影排查**（高复现性）
- 问题：同一锥桶被建到两个位置（鬼影）
- 分析：先查编码器 `scale_factor`（沿直线驶过 10 m 与卷尺实测距离对比），再查雷达-IMU 外参
- 方案：`ceres_scan_matcher` 权重调整 + `motion_filter` 收紧
- 数据：调参前后鬼影数量与地图一致性对比

**案例 3：阿克曼纯追踪的前视距离动态调整**（高复现性 + 复杂度）
- 问题：固定前视距离在直道抖动、弯道切内
- 方案：以激光雷达估计的赛道曲率动态调整 `L_d`（0.40–0.60 m）
- 公式：`δ = arctan(2·L·sin(α)/L_d)`
- 数据：轨迹跟踪误差 RMS 对比

**案例 4：锥桶颜色径向阈值自适应分类**（高创新性）
- 问题：光照变化导致基于 HSV 的颜色分类不稳定
- 方案：改用**几何先验**——依据 CR-04（内圈红、外圈蓝）、CR-06（逆时针）与 2.2.1(12)"可利用锥桶颜色来识别无人车运行方向"，以径向距离阈值分类，完全规避光照
- 数据：分类准确率对比（HSV 方案 vs 径向方案）

**案例 5：双圈模式互斥切换 + 圈次判定 + 异常恢复状态机**（高创新性）
- 问题：建图栈与导航栈 TF 冲突；赛道任意位置静止 >10 s 判负（FAIL-03），因此不能靠停车切换
- 方案：三取二的圈次判定（§5.3）+ LifecycleNode 受控切换时序（§7.3）+ 独立于状态机的全局看门狗
- 数据：连续 N 次**行进中**切换的成功率、TF 空窗时长、切换期间最低速度

**案例 6（强烈建议）：保证光电传感器触发的红绿灯接近与等待策略**（高创新 + 高实用）
- 问题：车辆路径偏移导致未遮挡光电传感器（2.4.2(1)：传感器设于红绿灯前 2 个锥桶处）→ 灯不亮 → 通行被判"绿灯未亮起"（PEN-03，+30 s）；而"停死等待"又触发 FAIL-03
- 分析：这是反向推演规则原文才能发现的因果链；三档罚则的严重度排序（50 ＞ 30 ＞ 20）决定了"宁可停在区域边缘"
- 方案：新增 `TRAFFIC_APPROACH` 状态；默认停车 + 绿灯才放行；超时后以 ≤0.1 m/s 蠕动重新压传感器（ADR-006）
- 数据：区域内轨迹横向偏差分布、触发成功率、等待时长分布

> 案例 6 是全场唯一一个**必须逆向推理规则原文才能发现**的工程点，创新性与实用性都拉满。建议优先撰写。

## 13. 复现性自查清单（提交前逐项打勾）

- [ ] 源码目录结构与手册第 9 章完全一致
- [ ] 手册中出现的**每个参数名**都能在参数文件中找到（与 §9 一致）
- [ ] 手册中出现的**每个话题名 / 坐标系名**都能在代码中找到（与 §8、§7 一致）
- [ ] 手册中出现的**每个函数名**都能在源码中找到
- [ ] 提供完整的依赖安装命令（含 Cartographer、Nav2、OpenCV、PCL 的 ARM64 安装说明），**全部使用 ROS 2 包名（`ros-humble-*`）**，见附录 C.2
- [ ] 提供一键编译说明：**`colcon build`**（非 ROS 1 的 `catkin_make`）
- [ ] 提供一键启动说明：**Python launch（`*.launch.py`）**（非 ROS 1 的 `.launch` XML）
- [ ] 参数以 **YAML + `--ros-args --params-file`** 加载（ROS 2 无 `rosparam` / param server），代码中用 `declare_parameter`
- [ ] 提供示例数据（**开发期** `ros2 bag` 或截图）与预期输出
- [ ] **ROS 2 合规自查**：`echo $ROS_DISTRO` = `humble`、`echo $ROS_VERSION` = `2`、系统内无 ROS 1（附录 A.1、附录 C.3）
- [ ] 说明硬件依赖（无实车时哪些部分可仿真验证）
- [ ] ≥5 个案例均含 §12 的 7 段结构
- [ ] 已附原创性/使用授权声明（DR-14，文本取自官方模板）
- [ ] **章节顺序、封面、声明文本已按规则 4.2 的官方模板校对**
- [ ] 已核对与其他队伍手册的差异（防相似度 >80%，SEC-06；防队间抄袭，ELIG-03）

---

# 附录 A　赛前合规检查清单

## A.1 ROS 1 残留自查（合规举证，PEN-10，+100 s）

> **本节的性质（v2.1 重写）**：本参赛队的比赛用车是 **ROS 2 小车**（Ubuntu 22.04 + ROS 2 Humble），**从设计上就不使用 ROS 1**。
>
> 规则 2.4.2(9) 规定"若系统中存在 ROS1，则罚时 100 秒"——这是一条**反**向合规约束。因此本节的目的**不是"清除 ROS 1"**，而是**在裁判面前举证系统内不存在 ROS 1**，属于 SEC-01、SEC-02 的证据链。

### A.1.1 正面声明脚本 `show_ros_version.sh`（完赛后向裁判展示，SEC-02）

```bash
#!/usr/bin/env bash
# 用途：向裁判展示所用 ROS 版本，并证明系统中不存在 ROS 1
set -u
echo "================ ROS 版本声明 ================"
echo "ROS_DISTRO  = ${ROS_DISTRO:-<未设置>}   # 期望: humble"
echo "ROS_VERSION = ${ROS_VERSION:-<未设置>}  # 期望: 2"
ros2 --version                           # 期望: ros2 cli 的 ROS 2 版本
echo "=============================================="
```

### A.1.2 不存在 ROS 1 的举证（ROS 2 / shell 形态，全部可现场演示）

```bash
# 1) 当前发行版必须是 Humble（ROS 2）
echo $ROS_DISTRO          # 期望输出: humble
echo $ROS_VERSION         # 期望输出: 2

# 2) ROS 1 的安装目录不存在
ls -d /opt/ros/noetic 2>/dev/null || echo "NO ROS1 (no /opt/ros/noetic)"

# 3) 看不到 ROS 1 的发行版（noetic/melodic/kinetic）
ros2 pkg list | grep -c '^ros1_bridge' || true
ls /opt/ros 2>/dev/null                # 期望：仅 humble（可含 ros-base）

# 4) 系统中没有 ROS 1 的 apt 包
apt list --installed 2>/dev/null | grep -ci 'ros-noetic'   # 期望: 0

# 5) 没有 ROS 1 的 Python 模块
python3 -c "import importlib.util as u; print('rospy:', bool(u.find_spec('rospy')), '| rospkg:', bool(u.find_spec('rospkg')))"
# 期望: rospy: False | rospkg: False

# 6) shell 环境变量只指向 ROS 2
env | grep -i '^ROS_' 
grep -n 'noetic\|/opt/ros/.*melodic' ~/.bashrc /etc/bash.bashrc 2>/dev/null || echo "NO ROS1 in bashrc"

# 7) 运行中的进程里没有 ROS 1 主节点
pgrep -a roscore || echo "NO roscore"
pgrep -a roslaunch || echo "NO roslaunch"
```

**判定标准（对裁判的口径）**：以上 7 项全部显示"无 ROS 1"，即满足 SEC-01 与 PEN-10 的合规要求。

> **若确实发现残留**（例如沿用了学长遗留的系统镜像）：删除 `/opt/ros/noetic`，移除 `~/.bashrc` 中所有 `source /opt/ros/noetic/...` 行，清理 `AMENT_PREFIX_PATH` / `PYTHONPATH` 中的 ROS 1 路径，并**重新复跑 A.1.2 全部 7 项**。
>
> **推荐做法**：直接**刷写干净的 ROS 2 Humble 镜像**部署，而不是在原系统上修补——这是最省事、证据最干净的路径。

## A.2 传感器驱动验证

| 检查项 | 命令 | 期望 |
|---|---|---|
| 雷达 | `ros2 topic hz /scan` | 10–12 Hz |
| 雷达数据 | `ros2 topic echo /scan --once` | 含有效 `ranges` |
| IMU | `ros2 topic hz /imu/data` | 稳定频率 |
| 摄像头 | `ros2 topic hz /camera/image_raw` | 约 30 Hz |
| 编码器 | `ros2 topic hz /odom` | 稳定频率 |
| TF 树 | `ros2 run tf2_tools view_frames` | 链完整、无多父节点 |
| 底盘响应 | 发布 `/cmd_vel` | 车辆按预期运动（**此检查必须在计时开始前完成**） |
| 舵机中位 | 下发中位脉宽 | 车辆直行（中位 **1520 µs**，HR-04） |
| 参数加载 | `ros2 param list /pure_pursuit` | 参数名与 §9 逐字一致（ROS 2 参数机制） |
| 节点清单 | `ros2 node list` / `ros2 topic list` | 与 §8 ICD 一致 |
| 生命周期 | `ros2 lifecycle get /<nav2 节点>`（如 `/amcl`、`/map_server`、`/lifecycle_manager`） | 返回 `unconfigured` / `inactive` / `active` 之一。⚠ **`cartographer_node` 不是 LifecycleNode**，对其调用会失败（见 ADR-003） |
| **QoS 兼容性核对（`/scan`）** | `ros2 topic info /scan -v \| grep -A5 Publishers`；`ros2 topic info /scan -v \| grep -A5 Subscriptions`（见附录 C.2 完整命令） | 发布者与订阅者 `Reliability` **兼容**：`Reliable` 发布者可匹配 `Best-Effort` 订阅者，**反之不成立**。若 `nav2_amcl` 收不到 `/scan`，多半是此因（见 §8 QoS 基线表） |
| **QoS 兼容性核对（全话题）** | 对 `/odom`、`/cmd_vel`、`/amcl_pose`、`/map` 同样执行 `ros2 topic info <话题> -v` | 逐话题确认 `Reliability` / `Durability` / `Depth` 与 §8 QoS 基线表一致 |
| **AMCL 粒子云收敛** | `ros2 topic echo /particle_cloud --once` | 粒子云在预期位姿附近**收拢为单峰**（多峰 / 大范围散布 = 定位退化，见 §5.3、附录 I.5.4） |
| **AMCL 协方差** | `ros2 topic echo /amcl_pose --field pose.covariance --once` | 位置协方差 < `amcl.covariance_position_threshold`(0.5 m²)、航向 < `amcl.covariance_yaw_threshold`(0.3 rad²)（§9.3） |
| **IMU 静态漂移** | 静置 60 s：`ros2 topic echo /imu/data --field angular_velocity.z --once` 与 `--field linear_acceleration.z --once` | 静止时 `angular_velocity.z` 接近 0；`linear_acceleration.z` 接近 ±9.81（**未去重力**）或 ≈0（**驱动已去重力**）——决定 §9.8 的 `imu0_remove_gravitational_acceleration` 取值 |
| ROS 2 合规 | 见 A.1.2 的 7 项 | 全部显示"无 ROS 1" |

## A.3 比赛当天流程（v2.0 补充场地准备）

1. 提前 30 分钟完成 A.1 + A.2
2. **确认锥桶已去除白色反光膜**（CR-15）；若未去除，当场向裁判提出并记录
3. 赛道实测回填 §2.2 表格，**特别记录单圈里程 L_lap**（回填 `lap.lap_length`）
4. 观察挡板放置位置（记录内侧/外侧 + 里程），**不修改任何代码逻辑**
5. 红绿灯区域：确认光电传感器位置（红绿灯前 2 个锥桶处），标定 `speed.traffic_approach`
6. HSV 阈值现场重标定（光照）→ 回填 §9.6
7. 发车区位姿标定 → 回填 §9.3 `initial_pose.*`
8. 比赛中：**全程不触碰遥控器**（SEC-04、PEN-11）
9. 完赛：运行 `show_ros_version.sh` 向裁判展示 ROS 版本（SEC-02）
10. 向裁判**展示比赛过程中构建的地图**，确认 3 处挡板清晰可见且位置正确（TR-08、PEN-07/08）

**线上赛额外步骤**

11. 提前一周关注 QQ 群（1063631705）公布的**赛道细节与红绿灯具体位置**（CR-09）
12. 按图 8 尺寸搭建赛道，录制测量视频（DR-09）
13. 布置三机位（1 号移动 / 2 号含秒表 / 3 号固定），总长 ≤10 分钟（DR-06~10）
14. 保留可复现的源码与原始数据，准备应对组委会抽查复现（FAIL-06）

---

# 附录 B　术语表

| 术语 | 含义 |
|---|---|
| AMCL | Adaptive Monte Carlo Localization，基于粒子滤波的蒙特卡洛定位 |
| 阿克曼转向 | Ackermann steering，前轮转向 + 后轮差速的车辆转向方式 |
| 前视距离 L_d | 纯追踪算法中，车辆到目标路径点的前瞻距离 |
| 光电传感器触发 | 设于**红绿灯前 2 个锥桶处**的光束装置，车辆遮挡后触发红灯倒计时（3~10 s） |
| 绿灯未亮起就通行 | 灯未亮（含传感器未被触发）即通过，罚 +30 s |
| 挡板 | 蓝色、**长 80 cm / 高 50 cm** 的挡板，防提前建图机制，全赛道 3 处 |
| 折线区 | 由**两根**黄色胶带（10 cm）贴地铺设的视觉识别区域，压线/骑线罚 10 s/圈 |
| 停车检测区域 | 需通过摄像头识别停车标志牌的矩形区域（CR-10、CR-07） |
| 停车区域 | 第二圈停车入库的目标矩形区域（三及以上车轮在内视为成功） |
| 发车区域 | 固定的起跑矩形区域 |
| 径向阈值 | 判断锥桶属内圈（红）还是外圈（蓝）的径向距离分界值 |
| 鬼影 | SLAM 中同一物体在地图上出现在多个位置的现象 |
| ROS 1 残留 | 系统中仍存在 ROS 1 相关包或环境变量 → 罚时 100 s。**本车为 ROS 2 小车，此项是"证明不存在"的反向合规项**（见附录 C.1） |
| rolling switch | 行进中的模式切换（不停车），用于规避 FAIL-03 |
| T有效 | 有效时间 = T1×30% + T2×70% + 加罚 − 两次红绿灯时长和 |
| ADR | Architecture Decision Record，架构决策记录 |
| ICD | Interface Control Document，接口控制文档 |
| ELIG | 参赛资格与队伍要求（本文档新增编号前缀，非规则原文用词） |

---

# 附录 C　ROS 2 技术栈与合规声明

## C.1 声明

本参赛队比赛用车为 **ROS 2 小车**：软件栈为 **Ubuntu 22.04 + ROS 2 Humble（ARM64）**，**全系统不安装、不使用、不残留 ROS 1**。

规则 2.4.2(9) 规定"比赛中必须用 ROS2 实现……若系统中存在 ROS1，则罚时 100 秒"。这条约束对本项目而言是**反向合规约束**：我们要做的不是"改用 ROS 1"，而是**证明系统中不存在 ROS 1**。举证方式见 C.3 与附录 A.1。

> **⚠ 全文用词勘误指南**：本文档 v1.0 及规则原文中出现的 `rosbag`、`map_saver`、`rosparam` 等名称属 **ROS 1 时代用词**，**实现时必须映射到 C.3 表格右列的 ROS 2 形态**。

## C.2 技术栈清单（全部为 ROS 2 包）

| 用途 | ROS 2 包 / 组件 | 安装命令（Ubuntu 22.04 / Humble） |
|---|---|---|
| 2D SLAM（第一圈） | `cartographer_ros` | `sudo apt install ros-humble-cartographer ros-humble-cartographer-ros` |
| 定位（第二圈） | `nav2_amcl`（节点名 `amcl`） | `sudo apt install ros-humble-nav2-amcl` |
| 地图保存 | `nav2_map_server`（`map_saver_server` 服务 + `map_saver_cli`） | `sudo apt install ros-humble-nav2-map-server` |
| 生命周期管理 | `nav2_lifecycle_manager`（借鉴）+ 自研 `mission_manager` | `sudo apt install ros-humble-nav2-lifecycle-manager` |
| 里程计融合（EKF） | `robot_localization` | `sudo apt install ros-humble-robot-localization` |
| 点云 / 聚类 | `pcl_ros`、`pcl_conversions` | `sudo apt install ros-humble-pcl-ros ros-humble-pcl-conversions` |
| 视觉 | `cv_bridge`、`image_transport`、`camera_calibration` | `sudo apt install ros-humble-cv-bridge ros-humble-image-transport ros-humble-camera-calibration` |
| TF | `tf2_ros`、`tf2_tools` | `sudo apt install ros-humble-tf2-ros ros-humble-tf2-tools` |
| 可视化 | `rviz2` | `sudo apt install ros-humble-rviz2` |
| 数据记录（**仅开发期**） | `ros2 bag`（`rosbag2`） | 随 `ros-humble-ros-base` 提供 |
| 构建 | `colcon` + `ament_cmake` / `ament_python` | `sudo apt install python3-colcon-common-extensions` |

> **ARM64 提示**：昇腾 310 平台上 Cartographer 需要源码编译（`colcon build`），需先确认 `libceres-dev`、`liblua5.3-dev`、`protobuf` 等依赖的 ARM64 版本可用。

### C.2.1 QoS 兼容性自查（实机必做，可直接粘贴执行）

QoS 不兼容会导致**订阅端静默收不到数据、且无任何报错**，是 ROS 2 最隐蔽的陷阱之一（见 §8 QoS 基线表）。装完包、拉通驱动后，按下列命令逐条核对：

```bash
# 1) 查看某话题的发布者/订阅者各自的 QoS
ros2 topic info /scan -v | grep -A5 Publishers
ros2 topic info /scan -v | grep -A5 Subscriptions

# 2) 判定规则：Reliable 发布者可匹配 Best-Effort 订阅者；反之不成立
#    若 /scan 发布者为 Best-Effort、而 nav2_amcl 订阅者要求 Reliable → AMCL 收不到数据

# 3) 逐话题核对（与 §8 QoS 基线表逐行对照）
for t in /scan /scan_filtered /imu/data /camera/image_raw /odom /cmd_vel /amcl_pose /map; do
  echo "=== $t ==="; ros2 topic info "$t" -v | grep -E "Reliability|Durability|Depth|Node name"
done

# 4) 端到端存活验证：若订阅者声明一致但收不到数据，用下面这条看是否有实际回传
ros2 topic hz /scan          # 发布端频率
ros2 topic echo /scan --once  # 能取到一帧即链路通

# 5) 第三方节点的 QoS 只能实测取得（不要臆断为 Reliable）
ros2 node info /amcl | grep -A3 "Subscribers"
```

**常见失败模式**：① 本队 `lidar_driver` 用 `Reliable` 发 `/scan`，`nav2_amcl` 用 `Best-Effort` 订阅 —— 这是**兼容**的（Reliable→Best-Effort 可以）；② 本队驱动用 `Best-Effort`、而某个要求 `Reliable` 的订阅者订阅 —— **不通**，订阅端永远收 0 帧。

## C.3 ROS 1 ↔ ROS 2 术语对照（本文档一律使用右列）

| ROS 1 形态（**本项目不使用**） | ROS 2 形态（**本项目使用**） |
|---|---|
| `roscore` / `roslaunch` | `ros2 run` / `ros2 launch`（**Python `*.launch.py`**） |
| `rostopic` / `rosservice` / `rosnode` | `ros2 topic` / `ros2 service` / `ros2 node` |
| `rosparam`（param server） | **YAML 参数文件 + `--ros-args --params-file`**、`ros2 param` |
| `catkin_make` / `catkin` | **`colcon build`** + `ament_cmake` / `ament_python` |
| `rosbag`（`rosbag record/play`） | **`ros2 bag`（`rosbag2`）** |
| `nodelet` | **可组合节点 `rclcpp_components` / composable node** |
| `tf` / `tf2`（ROS 1 版） | **`tf2`（`tf2_ros` / `tf2_geometry_msgs`）** |
| 私有参数 `~param` | `declare_parameter()` + `get_parameter()` |
| 无生命周期 | **`unconfigured → inactive → active → finalized`**（LifecycleNode） |
| `ros::Time` / `ros::NodeHandle` | **`rclcpp::Clock`（ROS_TIME）/ `rclcpp::Node`** |

## C.4 工程形态要求（代码层面）

| 项 | 要求 |
|---|---|
| `package.xml` | `<package format="3">`，依赖用 `ament_cmake` / `exec_depend` / `build_depend` |
| 构建系统 | `ament_cmake`（C++）/ `ament_python`（Python）；`colcon build --symlink-install` |
| launch | `launch/*.launch.py`，使用 `launch_ros.actions.Node` / `LifecycleNode` |
| 参数 | `config/*.yaml` + `--ros-args --params-file`；节点内 `declare_parameter()` |
| 话题 QoS | 雷达 / 图像用 `SensorDataQoS`（best-effort）；控制与状态用 `Reliable`。**发布者与订阅者的 QoS 必须兼容，否则静默收不到数据**（ROS 2 典型陷阱；引用第三方节点时须逐话题核对） |
| 时间源 | 统一使用 `rclcpp::Clock`（ROS_TIME），**不出现** `ros::Time` |
| 自定义消息 | `autocar_msgs/msg/*.msg` + `rosidl` 生成（见 §8.1） |
| 判据 | 代码中**不得出现** `rospy`、`roscpp`、`roslaunch`、`rosbag` 等 ROS 1 依赖 |

---

# 附录 D　向组委会 / 供应商的书面确认清单（可直接发邮件）

> 本附录由 §2.3 的 **Q1 ~ Q12** 整理而成，目的只有一个：把"文档里标注的待确认项"变成"**一封能直接发出去的邮件**"。发完、答复回来，按 D.4 回填即可。

## D.1 使用说明

| 项 | 说明 |
|---|---|
| 分组 | **A 组（规则解释类）**发大赛组委会；**B 组（设备与接口类）**发设备供应商北京小豚科技 |
| 发送时机 | 建议**同一天发出**；B 组越早越好（Q5、Q6 不答复，第三部分执行机构条目无法编码） |
| 当前假设 | 每组问题均给出「**我方当前假设**」。若赛前未答复，团队按该假设推进，但**必须在技术手册中注明"该假设待确认"** |
| 归档 | 全部邮件与答复归档至 `LOG/`，按日期命名，作为 SEC-06 原创性举证材料的一部分 |

## D.2 邮件正文模板

**主题（A 组）**：第二十一届全国大学生智能汽车竞赛 · 室外 ROS 无人车赛（职教组）· 规则条款确认申请 — [学校名称]-[队伍编号]

**主题（B 组）**：HW-Autocar 底盘驱动接口与运动学参数确认 — [学校名称]-[队伍编号]

```text
尊敬的老师 / 工程师：
您好。
我们是第二十一届全国大学生智能汽车竞赛"室外 ROS 无人车赛（职教组）"参赛队
[学校名称]-[队伍编号]，参赛设备为组委会指定的北京小豚科技 HW-Autocar，
比赛用车软件栈为 Ubuntu 22.04 + ROS 2 Humble。

在依据规则（2026 年 8 月初稿，共 13 页）编写设计与实施方案时，
以下条款需要书面确认。为便于回复，问题已编号列出，可直接逐条批注"是 / 否 / 以现场为准"。

（此处粘贴 D.3 中对应的 A 组或 B 组问题清单）

感谢您的支持，期待回复。

[队长姓名] / [联系电话] / [邮箱]
[学校名称]-[队伍编号]
```

### D.2.1 可直接发送的 B 组邮件正文（致设备供应商 北京小豚科技）

> **说明**：D.2 是通用骨架，本节是**成品**——把 D.3 的 B 组条目（Q5 / Q6 / Q11）合并，并采用 D.6.5 的**勾选 + 填空**格式。复制以下整块内容即可发出，仅需替换 `[ ]` 占位符。

**主题**：HW-Autocar 底盘驱动接口与运动学参数确认 — [学校名称]-[队伍编号]

```text
尊敬的工程师：

您好。

我们是第二十一届全国大学生智能汽车竞赛"室外 ROS 无人车赛（职教组）"参赛队
[学校名称]-[队伍编号]，参赛车辆为贵司 HW-Autocar 智能驾驶竞速车，
比赛软件栈为 Ubuntu 22.04 + ROS 2 Humble。

我方正在编写赛车的控制程序，需要把规划出的转向量换算为底盘可执行的指令。
为把误差控制在比赛允许的范围内（折线区压线 +10 s/圈、撞锥桶 +5 s/次），
以下 5 项参数需要贵司书面确认。为便于回复，已整理为勾选 / 填空形式，
直接在选项上标注或填写即可，无需撰写说明。

────────────────────────────────────────
一、底盘运动指令接口（对应我方文档 Q6）
────────────────────────────────────────
Q1. 底盘通过什么方式接收运动指令？
    ☐ ROS 2 话题（话题名：______________________）
    ☐ 串口协议（波特率：__________；协议文档：☐ 有　☐ 无）
    ☐ CAN
    ☐ 其他：______________________

Q2. 若为 ROS 2 话题，geometry_msgs/msg/Twist 中 angular.z 的物理含义是？
    ☐ 前轮转角 δ（单位：rad）
    ☐ 航向角速度 ω（单位：rad/s）
    ☐ 其他：______________________
    （若为自定义消息类型或其他单位，请一并说明。）

Q3. 贵司是否提供现成的 ROS 2 驱动包？
    ☐ 是
        包名：______________________
        订阅话题：______________________
        发布话题：______________________
        获取方式：☐ 随车提供　☐ GitHub（链接：__________）　☐ 其他：__________
    ☐ 否 —— 我方将自研 chassis_driver，届时需要通信协议文档

────────────────────────────────────────
二、阿克曼运动学参数（对应我方文档 Q5）
────────────────────────────────────────
Q4. 请填写：
    轴距 L（前轴中心 → 后轴中心）= __________ m
    最大前轮转角（单侧）        = __________ °
    内外轮转角是否不等（真阿克曼）？　☐ 是　☐ 否　☐ 不确定

────────────────────────────────────────
三、转向舵机标定（对应我方文档 Q11）
────────────────────────────────────────
Q5. 设备表写作"工作频率：1520us/330hz"。我方理解 1520 应为**脉宽中位值**、
    330 Hz 为工作频率。请确认：
    ☐ 中位脉宽确为 1520 µs（而非常见的 1500 µs）
    ☐ 中位脉宽实为 __________ µs
    中位脉宽对应的前轮转角 = __________ °
    左 / 右满舵脉宽 = __________ µs ~ __________ µs

────────────────────────────────────────
补充说明
────────────────────────────────────────
· 我方具备现场实测条件。若其中部分参数不便书面给出，回复"以现场实测为准"即可，
  我方可自行标定，不会因此反复打扰。
· 若底盘仅支持 PWM / 串口，请直接回复"仅 PWM"，我方将自行实现驱动节点。
· 如可提供现成的 ROS 2 驱动包或示例代码，将显著降低我方适配工作量。

以上问题盼复。如需我方配合（提供话题列表、运行日志）请随时告知，感谢支持。

[队长姓名] / [联系电话] / [邮箱]
[学校名称]-[队伍编号]
```

**发送建议**

| 项 | 建议 |
|---|---|
| 附件 | 附一张本车 `ros2 topic list -t` 与 `ros2 node list` 的截图，便于对方判断是否存在可直接复用的驱动 |
| 抄送 | 若供应商与组委会为同一联系人，可抄送组委会（B 组标题已在 D.2 给出） |
| 归档 | 答复到达后按 **D.4** 回填，并在 **D.5** 台账的 Q5 / Q6 / Q11 行登记日期与答复要点 |
| 兜底 | 若只回复"以现场为准"，按 **D.6.2** 现场实测确定情形（A / B / C） |

## D.3 问题清单

### A 组　规则解释类（发组委会）

| # | 问题（可直接复制到邮件） | 我方当前假设 | 确认后影响 |
|---|---|---|---|
| Q1 | 挡板是"**每处 1 块、全场共 3 处**"（2.2.1(11)），还是"在候选位置中随机放 3 块且位置独立"？两处表述如何统一？ | 每队上场前在赛道内/外侧两锥桶之间各放 1 块，全场共 3 处 | §1.2 CR-13、第一圈挡板数量校验逻辑 |
| Q2 | **全场是否只有一个红绿灯？** 规则 2.1、2.4.2 均以单数描述，且评分公式扣减项为"**两次**红绿灯时长和"。 | 只有一个，位置当天随机确定，两圈各遇到一次 | §5.3 圈次判定、§9.5 减速策略 |
| Q3 | 红绿灯前 2 个锥桶处的**光电传感器**：光束高度与宽度？车辆正常行驶是否必然遮挡？若未被触发导致红灯全程不亮，如何判罚？ | 光束低于车体、正常行驶必然遮挡；未被触发按"灯未亮"处理，我方**默认停车** | §1.4 PEN-03 应对策略 |
| Q4 | 停车区域**按胶带内侧**的实际可用尺寸（长×宽）？ | 矩形，以胶带内侧为界（CR-07）；尺寸以现场实测为准 | §9.5 停车点标定精度 |
| Q7 | "**停车检测区域**""**停车区域**""**标志牌区域**"三者是同一区域还是不同区域？2.2.1(8) 要求"识别**停车检测区域**内标志牌，并在进入**停车区域**后准确停在**标志牌区域**内"，请明确三者关系与标志牌实际安装位置。 | 停车检测区域＝识别标志牌的矩形，停车区域＝入库目标矩形，二者相邻；标志牌立于停车检测区域内 | 视觉探测范围（ESP）、停车目标点标定 |
| Q8 | "绿灯未亮起就通行"的判定主体是**视觉识别结果**还是**红绿灯系统自身记录**？ | 由裁判依据红绿灯系统记录判定；我方仍需自主识别以保证行为正确 | FAIL 风险是否依赖我方识别正确率 |
| Q9 | 原文只规定"丢失**两个及以下**挡板和锥桶 +5 s"，**未规定丢失超过 2 个、但地图确实存在**时如何判罚。 | 按"地图不清晰"档 **+5 s** 处理 | 第一圈建图完整性自检阈值 |
| Q10 | 3.4（线下赛）正文写需提交"技术手册**及视频**"，但其目录结构中**无视频**；3.5.2（线上赛）才列出 4 个视频。**线下赛是否也需提交视频？** | 线下赛仅需"源码＋技术手册"；视频仅线上赛需要，但现场仍自录一份备查 | §10 提交物清单、现场录像安排 |
| Q12 | 规则 2.1 引用"**图 1** 青色框"，2.2.1(6) 引用"**图 2**"，两处图号不一致，请确认指同一"青色框"红绿灯停车区域。 | 同一物，以**图 2** 为准 | 内部沟通引用 |

### B 组　设备与接口类（发供应商 / 组委会）

| # | 问题（可直接复制到邮件） | 我方当前假设 | 确认后影响 |
|---|---|---|---|
| Q5 | 底盘运动学模型：阿克曼转向的**轴距 L** 与**最大转向角**？舵机中位 **1520 µs** 对应的实际前轮转角？ | 阿克曼模型，`δ = arctan(2·L·sin(α)/L_d)`；中位 1520 µs；L 待实测 | §9.5 控制参数、HR-04 |
| Q6 | 底盘驱动接口：**ROS 2 话题 / 串口协议 / CAN**？是否已提供现成 **ROS 2 驱动包**（发布/订阅哪些话题）？ | 提供 ROS 2 驱动节点并订阅 `/cmd_vel`；若仅 PWM 接口则自研 `chassis_driver` | §3.3、§8.3 执行机构接口 |
| Q11 | 4.4 设备表把舵机写作"工作**频率**：1520us/330hz"——1520 µs 实为**脉宽中位值**而非频率（330 Hz 才是频率），原文表述有误。请确认**中位脉宽确为 1520 µs**。 | 中位脉宽 1520 µs（非 1500 µs） | `servo.neutral_pwm` 标定值 |

## D.4 收到答复后的回填动作

| 答复到达 | 立即动作 |
|---|---|
| **Q5、Q6** | 解除 §3.3「待定」标记，据实补全 §8.3 执行机构接口与 §9.5 控制参数。**在此之前，第三部分涉及执行机构的条目不得据此编码** |
| Q2、Q3、Q8 | 校正 §5.3 状态机、§9.5 减速曲线、§1.4 PEN-03 应对策略 |
| Q1、Q9 | 校正 §1.2 CR-13、建图完整性自检阈值 |
| Q4、Q7、Q11 | 回填 §2.2 现场复核表、§9.5 停车点、`servo.neutral_pwm` |
| Q10、Q12 | 校正 §10 提交物清单、§2.1/§2.2 的图号引用 |

## D.5 确认进度跟踪表（打印后逐项打勾）

> 用途：把"12 个问题"从静态清单变成**可追踪的台账**。发出邮件后填"发出日期"，收到答复后填"答复日期 / 答复要点"，按 D.4 完成回填后勾选"已回填"。
> **未收到答复前，一律按 D.3 的「我方当前假设」推进，并在技术手册中注明"该假设待确认"。**
> **"是否阻塞编码"= 是 的 4 项（Q4、Q5、Q6、Q7），在答复前对应章节不得据此编码**（§3.3、§8.3、§9.5、附录 K 执行机构与入库条目）。

| 编号 | 归属 | 是否**阻塞编码** | 影响章节 | 发出日期 | 答复日期 | 答复要点 | 已回填 |
|---|---|---|---|---|---|---|---|
| Q1 | 组委会（A 组） | 否 | §1.2 CR-13、第一圈挡板数量校验 | — | — | — | ☐ |
| Q2 | 组委会（A 组） | 否 | §5.3 圈次判定、§9.5 减速策略 | — | — | — | ☐ |
| Q3 | 组委会（A 组） | 否 | §1.4 PEN-03 应对策略 | — | — | — | ☐ |
| Q4 | 组委会（A 组） | **是** | §9.5 停车点标定精度 | — | — | — | ☐ |
| Q5 | 供应商（B 组） | **是** | §9.5 控制参数、HR-04 | — | — | — | ☐ |
| Q6 | 供应商（B 组） | **是** | §3.3、§8.3 执行机构接口 | — | — | — | ☐ |
| Q7 | 组委会（A 组） | **是** | ESP 探测范围、停车目标点标定（§2.2、§9.5） | — | — | — | ☐ |
| Q8 | 组委会（A 组） | 否 | FAIL 判定是否依赖我方识别正确率 | — | — | — | ☐ |
| Q9 | 组委会（A 组） | 否 | §1.2 建图完整性自检阈值 | — | — | — | ☐ |
| Q10 | 组委会（A 组） | 否 | §10 提交物清单、现场录像安排 | — | — | — | ☐ |
| Q11 | 供应商（B 组） | 否 | `servo.neutral_pwm` 标定值 | — | — | — | ☐ |
| Q12 | 组委会（A 组） | 否 | §2.1 / §2.2 图号引用 | — | — | — | ☐ |

> **阻塞项闭环判定**：Q4、Q5、Q6、Q7 四项全部"已回填"后，方可解除 §3.3、附录 J、附录 K 中标注的"**待定 / 不得据此编码**"标记。

## D.6 Q6 现场核查 SOP、三情形判定与 DS-05 解耦

> **本节目的**：Q6 的答案不必只靠供应商。**"底盘接受什么接口"可以现场实测确认**；而"`/steering_angle` 话题是否可用"更是**本队侧自己决定的事**。本节给出可执行流程，用于**尽快解除 DS-05 采集的阻塞**，同时**不违反"Q6 答复前执行机构代码不得编码"**这条硬约束。

### D.6.1 结论先行：什么被阻塞、什么不被阻塞

| 事项 | 被 Q6 阻塞？ | 说明 |
|---|---|---|
| **DS-01 / 02 / 03 / 06**（视觉数据、负样本） | **否** | 只依赖 `/camera/image_raw`，与底盘接口无关 |
| **DS-04**（雷达强度颜色分类） | **否** | 只依赖 `/scan`，与底盘接口无关 |
| **阶段 1 训练**（红绿灯 YOLO、强度 CNN） | **否** | 用 DS-01/02/04/06 即可开工 |
| **DS-05**（控制残差 RL 轨迹） | **是** | 需经典栈 `pure_pursuit → fuzzy_pid → chassis_driver` 在跑，该链路同时依赖 **Q5 + Q6** |
| §8.3 执行机构接口落笔 / §9.5 控制参数 | **是** | 同 D.4、D.5 |

> **所以：不要让 DS-01~04/06 陪等 Q6。** 可以并行推进的项已经足够跑完阶段 1 的大部分工作。

### D.6.2 现场自查 SOP（先自己查，别只等供应商）

在**非计时、车轮架空或封闭空旷场地、有人监护、可随时断电**的前提下执行：

| 步骤 | 命令 / 动作 | 判读 |
|---|---|---|
| ① 列实体 | `ros2 topic list -t`、`ros2 node list`、`ros2 pkg list \| grep -i -E "autocar\|chassis\|driver"` | 是否已存在厂商 ROS 2 驱动包 / 是否已有 `/cmd_vel` 订阅者 |
| ② 看接口 | `ros2 topic info -v /cmd_vel`、`ros2 interface show geometry_msgs/msg/Twist`、`ros2 topic info -v /steering_angle`（若存在） | 确认订阅者数量与 QoS；`/steering_angle` 是否**已由本队或厂商发布** |
| ③ 查驱动包 | 在驱动包 `launch/` 与 `config/` 中检索 `neutral`、`pwm`、`1520`、`wheelbase`、`steer`、`angle` | 若命中 `δ → PWM` 映射，基本可判为**情形 A** |
| ④ **决定性实验**（直线） | `ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}, angular: {z: 0.0}}"` | 车前进 → `/cmd_vel` 通道**通** |
| ⑤ **决定性实验**（转向） | `ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.0}, angular: {z: 0.3}}"` | 车**转向** → 进入三情形判定；车**完全不动** → **情形 C** |
| ⑥ 观察响应关系 | 依次发 `z = 0.1 / 0.3 / 0.6`，记录**舵机摆角是否近似线性于 z** | 近似线性于 z 且角速度不随 v 变 → 更可能是 **δ**（情形 A） |

> ⚠ **安全铁律**：⑤ 的 `linear.x = 0` 但转向指令会**原地打舵**，若底盘有蠕动逻辑可能移动 —— 必须架空或空旷。
> ⚠ 本 SOP 是**接口勘察**，不是控制实现；勘察结果写入 D.5 台账的"答复要点"栏。

### D.6.3 三情形判定表（答复一到即可落笔）

| 情形 | 现场证据 | `/steering_angle` 可用性 | §8.3 落笔方式 | §9.5 依赖 |
|---|---|---|---|---|
| **A** | 发 `angular.z` 后车转向；驱动包内命中 `δ→PWM` | **可用**（由 `fuzzy_pid` 发布 δ） | 声明 **`/cmd_vel.angular.z = 前轮转角 δ (rad)`**；`chassis_driver` 内 `δ → PWM(µs)` | 仅需 `δ` 标定（Q5 的 L 非必需） |
| **B** | 发 `angular.z` 后车转向；驱动为 `ω → 舵机` | **可用**（`fuzzy_pid` 发布 δ；`/cmd_vel.z = ω`） | 声明 **`/cmd_vel.angular.z = 航向角速度 ω (rad/s)`**；`chassis_driver` 内部折算 | **需 Q5 的 L**：`δ = arctan(ω·L / v)` |
| **C** | 发 `angular.z` **无任何反应**；仅接受 PWM / 串口 | **不可用（须自建）** | **不走 `/cmd_vel`**；`chassis_driver` 定义 `δ → PWM(µs)`，并另建**记录用**话题 | 需 PWM 中位（`servo.neutral_pwm=1520 µs`）与全行程标定 |

> **情形 C 的接口纪律**：用于记录 δ 的新话题**必须先走 §8 的接口变更流程登记**，不得直接编码——它不属本轮基线。

### D.6.4 DS-05 采集的前置与"原始记录优先"原则

**前置（三条同时满足才采）**：① Q5 已答复（`wheelbase`、`max_steering_angle`）；② Q6 已答复或已按 D.6.2 实测确定情形；③ 经典栈已两圈稳定完赛（L.10.1）。

**原始记录优先（本条决定数据集寿命）**：即便已确认为情形 A/B，DS-05 也**只录原始话题**：

```
/scan  /scan_filtered  /odometry/filtered  /imu/data  /cmd_vel  /cmd_vel_raw
/race_state  /race_lap  /tf  /tf_static
```

**不预存已换算的 δ**。理由：若后续 Q5/Q6 改变了语义，**原始数据可离线重新推导**；而预存的 δ 只能报废重采。换算一律放到训练前的预处理脚本里做。

### D.6.5 给供应商的最小答复模板（降低对方答题成本）

开放式提问容易被搁置；改成**勾选 + 填空**：

| # | 请确认 | 形式 |
|---|---|---|
| 1 | 底盘通过什么接收运动指令？ | ☐ ROS 2 话题（话题名：______）　☐ 串口（波特率：______）　☐ CAN　☐ 其他：______ |
| 2 | 若为 ROS 2 话题：`geometry_msgs/msg/Twist.angular.z` 的物理含义？ | ☐ 前轮转角 δ（rad）　☐ 航向角速度 ω（rad/s）　☐ 其他：______ |
| 3 | 是否提供现成 ROS 2 驱动包？ | ☐ 是（包名：______；发布话题：______；订阅话题：______）　☐ 否 |
| 4 | 阿克曼参数填空 | 轴距 L = ______ m；最大前轮转角 = ______ °；舵机中位脉宽 = ______ µs |

### D.6.6 供应商长期不答复时的兜底

按 D.6.2 步骤 ④~⑥ 以**实测**确定情形，然后在 D.5 台账 Q6 行的"答复要点"栏写 **"实测确定：情形 X"**，并：

- **可解除编码阻塞**——理由是接口已由**实验证据**确定，而非猜测；
- **但仍保留"待确认"标注**——在技术手册中注明"该接口由本队实测确定，供应商未书面确认"，并**留存实测录像**以备质询。

---

# 附录 E　系统设计卡 E-1：红绿灯识别与停车（含光电触发应对）

> **设计卡定位**：把散落在 §1.4、§5.3、§9.5 ~ §9.7、ADR-006 的"红绿灯"内容，收敛为一张**可独立评审、可独立实现、可独立测试**的设计卡。
> **一致性约束**：本卡内所有**话题名、参数名、状态名**与 §7 / §8 / §9 逐字一致（第三部分仍是**唯一真值源**）。本卡新增的接口与参数已同步登记到 §8.4 与 §9.7。

## E.0 设计卡头部

| 项 | 内容 |
|---|---|
| 设计卡编号 | **E-1** |
| 子系统名 | 红绿灯识别与停车（含光电触发应对） |
| 覆盖需求 | TR-05、TR-06（等待时长不计入）、CR-08、CR-12 |
| 覆盖罚则 | PEN-01（+20 s）、PEN-02（+50 s）、PEN-03（+30 s）、FAIL-03（判负） |
| 覆盖规则条款 | 2.1、2.4.1、2.4.2(1)、2.4.2(8) |
| 关键前提 | 光电传感器**在赛道上、不在车上**；红灯由**车辆遮挡传感器**触发；等待红灯时长**不计入**有效时间 |
| 技术栈 | **ROS 2 Humble**（节点 / 话题 / QoS 均为 ROS 2 形态，见附录 C） |
| 状态 | 设计已冻结；`speed.traffic_approach`、HSV 阈值、停车点、`traffic.approach_distance` 为**现场标定项**。⚠ **零标定兜底（v2.1 R61）**：`traffic.stop_point_*` 未标定时，本卡 §E.3 的**坐标入口自动停用**（否则第一圈第一个 tick 就在发车线误触发），改由 `/traffic_light/state` **确认红灯**的**视觉入口**进入；灯态 `off`/`unknown`/**未收到灯态帧**统一落 `SAFE_HOLD`，并在 `traffic_light.hold_timeout`(60 s) 后**分级放行**（`TRAFFIC_STOP` 已确认红灯则不放行）。故未标定也能完赛（详见 R61 与 §9.7） |
| 责任人 | 视觉 + 状态机（建议同一人，因判据强耦合） |

## E.1 问题定义

1. **灯并非常亮**：光电传感器设于**红绿灯前 2 个锥桶处**，车辆经过遮挡后触发红灯倒计时（**3~10 s 不固定**）。若车辆路径偏移未遮挡，灯可能**全程不亮**。
2. **"灯不亮"具三义性**：传感器未触发／我方识别失败／确实无灯——三者对车端外观相同，但规则**只承认"绿灯亮"为通行许可**。
3. **罚则严重度与直觉相反**：`未停车 +50 s` ＞ `绿灯未亮通行 +30 s` ＞ `车轮不在区域内 +20 s`。→ 停车策略必须**保守制动、宁可停早**。

## E.2 系统边界与职责划分

| 侧 | 承担 | 不承担 |
|---|---|---|
| **赛道端（组委会道具）** | 光电传感器、红灯倒计时（3~10 s）、绿灯输出、违规判定 | — |
| **车端（本队实现）** | ① 沿赛道中线行驶以**保证遮挡传感器**；② 视觉识别红/绿灯；③ 减速 → 停车 → 等待 → 起步；④ 记录等待时长供 TR-06 扣减 | 不控制灯、**无任何接口**读取灯的内部状态 |

> **关键推论**：车端**拿不到灯的内部状态**，只能依靠"视觉 + 保守策略"。因此 ADR-006 的「默认停车」不是保守选择，而是**唯一可行策略**。

## E.3 接口契约（与 §8 逐字一致）

| 接口 | 类型 | 方向（本卡视角） | 用途 |
|---|---|---|---|
| `/camera/image_raw` | `sensor_msgs/msg/Image` | sub | 视觉输入源（QoS `SensorDataQoS`） |
| `/camera/camera_info` | `sensor_msgs/msg/CameraInfo` | sub | 去畸变内参（150° 视场必需） |
| `/traffic_light/state` | `std_msgs/msg/String` | sub | 值 `red` / `green` / `off` / `unknown`；由 `traffic_light_detector` 发布（§8.1、§9.6） |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | pub | 停车／蠕动的唯一执行通道（QoS `Reliable`） |
| `/race_state` | `std_msgs/msg/String` | pub | 状态名取 E.4 枚举 |
| `/traffic/wait_duration` | `std_msgs/msg/Float32` | pub | **新增（已登记 §8.4）**：本次等待绿灯累计时长 (s)，供 TR-06 与裁判举证 |

## E.4 状态机

```
                 ┌──────────────────────────────────────────────┐
                 │  MAPPING / NAVIGATING （赛道正常行驶）          │
                 └───────────────┬──────────────────────────────┘
      距区域 ≤ traffic.approach_distance（据地图 / 里程预判）│
                                                        ▼
                                         ┌────────────────────────────┐
                                         │ TRAFFIC_APPROACH           │
                                         │ 限速 ≤ speed.traffic_approach│
                                         └─────────────┬──────────────┘
                                     /traffic_light/state
              ┌────────────────┬─────────────────────┬──────────────────┐
           green              red                off / unknown
              │                │                     │
              ▼                ▼                     ▼
      ┌──────────────┐  ┌──────────────┐   ┌────────────────────┐
      │ TRAFFIC_GO    │  │ TRAFFIC_STOP │   │ SAFE_HOLD          │
      │ 直接通行      │  │ 区域内制动    │   │ "灯不亮 ≠ 通行"     │
      │ wait_duration │  │ 开始计时      │   │ 停车等待            │
      │  = 0          │  └──────┬───────┘   └─────────┬──────────┘
      └──────┬───────┘         │ 等待 ≤ wait_timeout   │
             │                 ▼                      │
             │           state = green ───────────────┘
             │                 │
             │                 ▼
             │          TRAFFIC_GO（封存 wait_duration）→ 恢复上一状态
             │
             └──【超时分支】等待 > wait_timeout(30 s) 仍无 green
                     → 不通行；改为 crawl_speed ≤0.1 m/s 在停车区域内前后蠕动
                     → 重新压传感器；蠕动期间不得静止 > static_timeout(9 s)
```

| 状态 | 进入条件 | 动作 | 退出条件 |
|---|---|---|---|
| `TRAFFIC_APPROACH` | 距区域 ≤ `traffic.approach_distance` | 限速至 `speed.traffic_approach`；沿中线行驶 | 采信 `/traffic_light/state` |
| `TRAFFIC_STOP` | `state == red` 连续 `traffic.confirm_frames`(3) 帧 | 在停车点（**胶带内侧**）制动；**开始累计** `/traffic/wait_duration` | 出现 `green`／超时 |
| `SAFE_HOLD` | `state ∈ {off, unknown}` **或未收到任何灯态帧**（区域内） | 停车等待，**绝不通行** | 出现 `green`；或（v2.1 R61 分级兜底）等待 ≥ `traffic_light.hold_timeout` |
| `TRAFFIC_GO` | `state == green` 连续 3 帧 | 加速离开；**封存** `/traffic/wait_duration` | 离开区域 → 恢复上一状态 |
| 超时告警（`TRAFFIC_APPROACH`/`TRAFFIC_STOP`/`SAFE_HOLD` 子行为） | 等待 > `traffic_light.wait_timeout`(30 s) 仍无 `green` | 仅打 WARN 并置内部标志（**不加动作**）；⚠ **R61 更正**：原设计的"以 `traffic_light.crawl_speed` 蠕动"**未真正生效**（`_traffic_crawl` 从未对外发布，`autocar_control` 按 `/race_state` 取值，`SAFE_HOLD`/`TRAFFIC_STOP` 限速均为 0.0） | — |
| **分级超时放行**（v2.1 R61，仅 `SAFE_HOLD` 等 `traffic_light.hold_timeout_states` 列出的态） | 等待 ≥ `traffic_light.hold_timeout`(60 s) 且**全程未确认红灯** | 放行 → `TRAFFIC_GO`；日志打 **PEN-03 风险 WARN** | 立即（下一 tick） |
| **零标定视觉入口**（v2.1 R61，仅 `traffic.stop_point_*` 未标定时启用） | `/traffic_light/state` **确认**为 `traffic.visual_trigger_states`（默认 `red`）且为**边沿**（非触发态→触发态） | 直接进 `TRAFFIC_APPROACH`（位置无关），取代被停用的坐标入口 | 同上三条 |

> **蠕动为何不判负**：FAIL-03 判据是"**停止**运行超过 10 s"。蠕动速度非零 → 不构成"停止"。以 `watchdog.static_timeout = 9 s` 作为**强制动作点**（§9.7）。
> ⚠ **R61 更正**：上句是 ADR-006 的原始推理，但该蠕动**从未真正实现**（见上表）。故 `SAFE_HOLD` 等豁免态的实际出口是 **`traffic_light.hold_timeout` 分级放行**，而非蠕动。**已确认红灯**的 `TRAFFIC_STOP` **不放行**（PEN-03 优先），其静止由 `watchdog.exempt_states` 豁免以免被 FAIL-03 误判。

## E.5 判据与算法

**5.1 视觉判据**（沿用 ADR-004 传统 CV）

| 项 | 做法 |
|---|---|
| ROI | 画面中上部；相机俯仰角按 **CR-12**（灯本体 60×17 cm、离地 40 cm）反推，使目标在 **3~6 m** 有效识别距离内落于画面有效区且畸变可接受 |
| 颜色 | 红 `hsv.red_h_ranges = [0,10] ∪ [170,179]`；绿 `hsv.green_h_range = [40,80]`；`hsv.s_min` / `hsv.v_min` **现场标定** |
| 形状 | 圆形度 ≥ `traffic.circularity_min`(0.70)，面积 ≥ `traffic.min_area` |
| 时序 | **连续 `traffic.confirm_frames`(3) 帧一致**才输出，抑制误检与抖动 |
| 输出 | `red` / `green` / `off`（帧丢失或超时输出 `unknown`） |

**5.2 几何 / 里程判据（保证遮挡）**

- 沿**赛道中线**行驶（CR-08：红绿灯位于区域**右侧**，以小车运行方向为参考）；
- 用第一圈地图 + 里程**预判区域入口**，**提前**进入 `TRAFFIC_APPROACH`（而非"看到红灯才减速"）；
- 停车点为**区域内的固定地图坐标**。

**5.3 停车点标定（PEN-01 的关键）**

- 规则对红绿灯的要求是"**任一车轮**在停车区域内即不罚 +20 s"；
- 但本卡要求按**第二圈入库标准预留更宽裕度**（三及以上车轮在内），以抵御定位漂移；
- 方法：地图上手选停车点 → 现场以**胶带内侧**实测校正（CR-07）→ 回填参数。

## E.6 关键时序

| t | 事件 | 车端动作 | 计时 |
|---|---|---|---|
| t0 | 进入 `TRAFFIC_APPROACH` | 降速至 ≤ `speed.traffic_approach` | 不计（正常行驶） |
| t1 | 遮挡光电传感器（**赛道端**） | 车沿中线通过传感器点 | 赛道端开始倒计时 |
| t2 | 识别 `red` 连续 3 帧 | 进入 `TRAFFIC_STOP` 并制动 | **开始计 `wait_duration`** |
| t3 | 赛道端倒计时结束（3~10 s 随机） | 视觉捕获 `green` 连续 3 帧 | **结束计 `wait_duration`** |
| t4 | 进入 `TRAFFIC_GO` | 加速离开 | — |

> **`wait_duration` 的用途（TR-06）**：有效时间 `T有效 = T1×30% + T2×70% + 加罚 − 两次红绿灯时长和`。规则要求"等待红灯的时长不计入"，故本卡要求**第一、二圈各记录一次** `wait_duration`，完赛后**向裁判出示两次读数之和**。这是全系统唯一需要"向裁判报数"的自研指标。

## E.7 参数表（全部引自 §9，逐字一致）

| 参数 | 值 | 出处 |
|---|---|---|
| `hsv.red_h_ranges` / `hsv.green_h_range` | `[0,10]∪[170,179]` / `[40,80]` | §9.6 |
| `hsv.s_min` / `hsv.v_min` | **现场标定** | §9.6 |
| `traffic.confirm_frames` | 3 | §9.6 |
| `traffic.min_area` | 待定（按 CR-12 反推像素面积） | §9.6 |
| `traffic.circularity_min` | 0.70 | §9.6 |
| `traffic.approach_distance` | **待标定**（新增，已登记 §9.7） | §9.7 |
| `speed.traffic_approach` | ≤ 0.30 m/s | §9.5 |
| `speed.parking_approach` | ≤ 0.30 m/s（绿灯起步后至停车区） | §9.5 |
| `traffic_light.wait_timeout` | 30.0 s | §9.7 |
| `traffic_light.crawl_speed` | ≤ 0.10 m/s（**R61：当前不生效**） | §9.7 |
| `traffic.entry_by_light` / `traffic.visual_trigger_states` | `true` / `['red']`（**R61 新增**，仅坐标入口未标定时启用） | §9.7 |
| `traffic_light.hold_timeout` / `hold_timeout_states` | 60.0 s / `['SAFE_HOLD']`（**R61 新增**，分级超时放行） | §9.7 |
| `watchdog.static_timeout` | 9.0 s | §9.7 |

## E.8 测试用例

| ID | 场景 | 步骤 | 通过判据 |
|---|---|---|---|
| TC-E01 | 正常绿灯 | 置 `/traffic_light/state = green` | 不停车直接通过；`wait_duration` = 0 |
| TC-E02 | 正常红灯 | 触发 `red` 3~10 s 后转 `green` | 区域内停车 → 绿灯后起步；`wait_duration` 与红灯时长一致（±0.3 s） |
| TC-E03 | 灯不亮（未触发） | 置 `state = off` | 进入 `SAFE_HOLD` **停车等待**，**绝不通行**（PEN-03 反例） |
| TC-E04 | 识别抖动 | 逐帧随机切换 `red`/`green` | 连续 3 帧确认生效，**不产生状态跳变** |
| TC-E05 | 超时无绿灯（**R61 更正**） | 持续 `off` > 60 s | 达 `traffic_light.hold_timeout`(60 s) 后**放行**（分级兜底），日志出现 **PEN-03 风险 WARN**；⚠ **不再判"进入蠕动"**——`traffic_light.crawl_speed` 当前不生效 |
| TC-E06 | 停车点精度 | 在停车点停车 | 以**胶带内侧**为界，车轮位于区域内（PEN-01 反例） |
| TC-E07 | 相机断流 | 停止 `/camera/image_raw` | `state → unknown` → `SAFE_HOLD` 停车 |
| TC-E08 | 第一圈通过 | 第一圈进入区域 | **两圈均须停车**（规则 2.1："两圈过程中每次遇到红绿灯需停车"）；第一圈不执行入库停车（TR-07） |
| TC-E09 | **零标定不误触发（R61）** | `traffic.stop_point_*=0` 且车静止在**发车线**，`state` 恒 `off`/无帧 | **不进入任何 `TRAFFIC_*` 状态**（原缺陷为第一 tick 即误入并在发车线停死） |
| TC-E10 | **零标定红灯仍被识别（R61）** | `stop_point_*=0`，在赛道上 `state` 首次确认为 `red` | 经**视觉入口**进 `TRAFFIC_APPROACH` → `TRAFFIC_STOP` 停车等绿；同一红灯段**只触发一次**（不反复进出） |
| TC-E11 | **已确认红灯不放行（R61）** | 已标定 `stop_point_*`，先确认 `red`，随后灯灭 | 停在 `TRAFFIC_STOP`，**超时不放行**（分级处理，PEN-03 优先） |

## E.9 失效模式（简版 FMEA）

| 失效 | 后果 | 严重度 | 检出 / 缓解 |
|---|---|---|---|
| 未遮挡传感器 → 灯不亮 | PEN-03 **+30 s** | 高 | 中线行驶（E.5.2）+ `SAFE_HOLD` 默认停车 |
| 误检为 `green` | 提前通行 | **极高** | 连续 3 帧确认 + 面积 / 圆形度门限 |
| 停车点偏出区域 | PEN-01 **+20 s** | 中 | 按胶带内侧标定 + 低速收敛 + "停早不停晚" |
| 等待中静止 > 10 s | FAIL-03 **判负** | **极高** | `watchdog.exempt_states` 豁免 `TRAFFIC_STOP`/`SAFE_HOLD` + `static_timeout = 9 s`；⚠ **R61**：原"超时蠕动"**未真正实现**，`SAFE_HOLD` 的实际出口改为 `traffic_light.hold_timeout`(60 s) 分级放行 |
| 相机断流 / 曝光异常 | 无法判断 | 高 | `state → unknown` → `SAFE_HOLD` |

## E.10 需求—罚则—设计映射

| 需求 / 罚则 | 本卡对应设计 |
|---|---|
| TR-05 两圈每次遇红绿灯停车 | E.4 状态机（两圈共用同一实现） |
| TR-06 等待时长不计入 | `/traffic/wait_duration` 记录并出示（E.6） |
| CR-08 灯位于区域右侧 | E.5.2 沿中线行驶 + 区域入口预判 |
| CR-12 灯本体尺寸 60×17 cm／离地 40 cm | E.5.1 相机俯仰角反推 + `traffic.min_area` |
| PEN-01 车轮不在区域 +20 s | E.5.3 停车点按胶带内侧标定 |
| PEN-02 未停车 +50 s | E.4 强制 `TRAFFIC_STOP`，不允许"看到灯再决定" |
| PEN-03 绿灯未亮通行 +30 s | `SAFE_HOLD`：`off` / `unknown` 一律停车 |
| FAIL-03 静止 > 10 s 判负 | 超时蠕动 + `static_timeout = 9 s` |

---

# 附录 F　系统设计卡 E-2：第一圈建图与行进中模式切换（Rolling Switch）

> **设计卡定位**：把散落在 §5.1、§5.2、§5.3、§7.2、§7.3、§9.2 ~ §9.5、§9.7、ADR-001、ADR-003 的"第一圈建图 → 行进中切换到导航栈"内容，收敛为一张**可独立评审、可独立实现、可独立测试**的设计卡。这是全系统**风险最高**的一次操作——它同时触及 PEN-07/08（建图）与 FAIL-01/03（判负）三条罚则。
> **一致性约束**：本卡内所有**话题名、服务名、参数名、状态名、TF 名称**与 §7 / §8 / §9 逐字一致（第三部分仍是**唯一真值源**）。本卡未新增任何接口或参数；全部数值均已存在于 §9。

## F.0 设计卡头部

| 项 | 内容 |
|---|---|
| 设计卡编号 | **E-2** |
| 子系统名 | 第一圈建图与行进中模式切换（Rolling Switch） |
| 覆盖需求 | TR-01（两圈）、TR-02（第一圈建图）、TR-03（第二圈基于地图加速）、TR-08（挡板须显示在地图） |
| 覆盖罚则 | PEN-07（+5 s）、PEN-08（+50 s）、FAIL-01（判负）、FAIL-03（判负）、FAIL-04（取消成绩） |
| 覆盖规则条款 | 2.1、2.4.2(5)、2.4.2(6)、2.4.2(8)、2.4.2(10) |
| 关键前提 | ① `cartographer_node` **不是** LifecycleNode，建图栈只能**进程级**停止（ADR-003）；② `map→odom` 在 Cartographer 与 AMCL 之间**互斥**（§7.2）；③ 切换必须在**行进中**完成，否则静止 >10 s 命中 FAIL-03（§5.3、§7.3）；④ 比赛过程中**禁止**任何 `ros2 bag` 回放（FAIL-04）。 |
| 技术栈 | **ROS 2 Humble**（Cartographer ROS 2 移植版 + `nav2_amcl` + `robot_localization`，见附录 C.2） |
| 状态 | 设计已冻结；`lap.lap_length`、`TRAJECTORY_BUILDER_2D.submaps.num_range_data`、切换点里程先验为**待实测项** |
| 责任人 | SLAM + 状态机（建议同一人，因 TF 权属与圈次判定强耦合） |

## F.1 问题定义

1. **两圈用同一套地图，却要两套互斥的定位栈**：第一圈 Cartographer 边跑边建图，第二圈 `nav2_amcl` 在地图上定位。二者**都发布 `map→odom`**（§7.2），同时激活会形成 TF 树双父节点，定位立即失效——切换是**不可避免**的。
2. **切换耗时不可控**：在 ARM64 平台上，Cartographer 退出（`finish_trajectory` + `write_state`）与 AMCL 收敛都可能超过 10 s。而规则 2.4.2(8) 规定**任意位置静止 >10 s 即判负**（FAIL-03）。
3. **因此不能用"停车保存地图"**：v1.0 的方案（回到发车区停车 → 保存 → 换栈）直接命中 FAIL-03，已被 ADR-003 否决。必须**行进中切换**。
4. **建图罚则的落差极大**：只要"确实建了"，即便质量差也只 +5 s（PEN-07）；而"没有在比赛过程中建图"（含提前建图、未构建地图、挡板位置明显不符）则 +50 s（PEN-08）。→ **宁可交一张"模糊但真实"的地图**。
5. **挡板是防作弊机制**：TR-08 要求当天随机放置的 3 处蓝色挡板**必须在地图中显现**。这决定架构上**禁止**预置赛道先验地图（ADR-005）。

## F.2 系统边界与职责划分

| 侧 | 承担 | 不承担 |
|---|---|---|
| **赛道端（组委会）** | 当天随机在内/外侧两锥桶间放置 3 处挡板（CR-13）；提供红绿灯与光电传感器；判定建图质量与是否提前建图 | — |
| **车端（本队实现）** | ① 第一圈**真实**执行探索—建图（低速行驶、占满赛道、保留挡板）；② 圈次判定（"三取二"表决）；③ **行进中**保存地图 → 停用建图栈 → 启用导航栈 → 发布初始位姿；④ 地图落盘合格性自检；⑤ 保留时间戳 / 日志作为"比赛过程中建图"的因果证据 | **不预置地图**、**不硬编码挡板位置**、**比赛中不启动任何 `ros2 bag`**（FAIL-04） |

> **关键推论**：切换窗口是全系统**唯一**允许"定位精度下降"的时段，但**绝不允许**"车辆静止"。因此本卡的设计目标可浓缩为一句：**用"低速蠕动"覆盖整段不确定性，直到 `map→odom` 恢复发布。**

## F.3 接口契约（与 §8 逐字一致）

| 接口 | 类型 | 方向（本卡视角） | 用途 |
|---|---|---|---|
| `/scan` | `sensor_msgs/msg/LaserScan` | sub | 建图输入；Cartographer 与 AMCL **互斥消费**（§8.1） |
| `/imu/data` | `sensor_msgs/msg/Imu` | sub | Cartographer 重力参考（`use_imu_data=true`） |
| `/odometry/filtered` | `nav_msgs/msg/Odometry` | sub | `ekf_node` 输出的融合里程计，建图/定位共用（§8.2） |
| `/map` | `nav_msgs/msg/OccupancyGrid` | sub | 第一圈的栅格地图（`occupancy_grid_node` 发布） |
| `/map_save` | `nav2_msgs/srv/SaveMap` | **srv client** | 由 `mission_manager` **异步**调用 `map_saver_server`，落盘 `.pgm/.yaml`（§8.2）。⚠ 不得用阻塞式 `map_saver_cli` |
| `/initialpose` | `geometry_msgs/msg/PoseWithCovarianceStamped` | pub | 切换完成后向 `nav2_amcl` 发布初始位姿（取当前 odom 位姿 + 地图匹配初值，§7.3 步骤④） |
| `/amcl_pose` | `geometry_msgs/msg/PoseWithCovarianceStamped` | sub | 切换后校验 AMCL 已收敛（§8.2） |
| `/race_state` | `std_msgs/msg/String` | pub | 状态名取 F.4 枚举 |
| `/race_lap` | `std_msgs/msg/UInt8` | pub | 圈次（1/2），供里程累计与日志追溯 |
| `/mode_switch/request` | `std_msgs/msg/Bool` | pub | 请求进入 `ROLLING_SWITCH`（§8.4） |
| `/watchdog/status` | `std_msgs/msg/Bool` | pub | 全局看门狗心跳，切换期间的"不停车"保护（FAIL-03） |
| `/<node>/change_state` | `lifecycle_msgs/srv/ChangeState` | **srv client** | 对**自研节点与 Nav2 组件**做受控转换（§8.4）。⚠ **不可**用于 `cartographer_node` |
| `/<node>/get_state` | `lifecycle_msgs/srv/GetState` | **srv client** | 查询节点状态（同上限制） |

> **两类"停止"的区分（本卡核心）**：对**建图栈**（`cartographer_node`）只能**进程级终止**（先 `finish_trajectory` → `write_state` → 终止进程）；对**自研节点与 Nav2 组件**才使用 `change_state`。混用会静默失败（ADR-003）。`map→odom` 的权属交接通过 **`/tf` 上的实际发布者**观测，而非查询节点状态。

## F.4 状态机

```
IDLE
 └→ MAPPING                     第一圈：Cartographer 建图 + 低速锥桶跟踪
     │                          （不停车通过停车区，TR-07）
     ├→ TRAFFIC_APPROACH       遇红绿灯区域降速（附录 E，两圈共用）
     └→ LAP1_COMPLETE          §5.3「三取二」表决成立
         └→ ROLLING_SWITCH     行进中切换（速度 ≥ speed.rolling_switch，绝不停车）
             │   ① 异步 save_map → 落盘 .pgm/.yaml
             │   ② 终止建图栈进程 → 等待 map→odom 停发（超时 1.5 s）
             │   ③ configure+activate(nav2_amcl) → 发布 /initialpose
             │   ④ 等待 map→odom 恢复发布（超时 3.0 s）
             └→ NAVIGATING     第二圈：AMCL + 纯追踪，逐步提速至 speed.straight
```

**异常分支（切换窗口内）**

```
ROLLING_SWITCH → WATCHDOG_RECOVER   步骤③→⑤ 总耗时 > switch.timeout(20.0 s)
                                    → 转入蠕动分支（速度 ≥ recover_speed 0.15 m/s，
                                      绝不停车），保底防 FAIL-03
MAPPING / ROLLING_SWITCH → OFF_TRACK_RECOVER   偏离轨道 → 10 s 内回中（FAIL-01）
```

| 状态 | 进入条件 | 动作 | 退出条件 |
|---|---|---|---|
| `MAPPING` | 第一圈启动 | Cartographer 建图 + EKF 里程计融合；低速行驶；挡板在线检测 | 三取二表决成立 |
| `LAP1_COMPLETE` | 圈次判据 ≥ 2 票通过 | 触发 `/mode_switch/request = true` | 进入 `ROLLING_SWITCH` |
| `ROLLING_SWITCH` | 收到切换请求 | 见 F.6 六步；**车辆速度恒 ≥ `speed.rolling_switch`** | `map→odom` 恢复发布 |
| `NAVIGATING` | 步骤⑤确认成功 | 启用纯追踪；逐步提速 | 完赛 |

> **为何 `ROLLING_SWITCH` 不计 FAIL-03 静止**：该状态要求速度**下界非零**（蠕动），因此从不满足"停止运行"。看门狗在 `ROLLING_SWITCH` 内**不豁免**（与 `TRAFFIC_STOP` / `SAFE_HOLD` / `PARKING` / `DONE` 不同），反而**主动保障**车辆处于蠕动（§9.7 `watchdog.exempt_states` 不含本状态）。

## F.5 判据与算法

**5.1 圈次判定：三取二表决（§5.3）**

| 判据 | 阈值 / 条件 | 参数 |
|---|---|---|
| 主判据：里程累计 | 累计里程 ≥ `lap.detect_distance_ratio × lap_length` | `0.95 × L_lap`（`L_lap` 待实测） |
| 辅判据：发车区特征 | 重新进入发车区矩形 + 航向与发车朝向一致 | 地图已知矩形 |
| 辅判据：地图闭合度 | Cartographer `constraint_builder` 产生内部闭环 | — |

任两条同时成立即判定第一圈结束。**单一判据都不足采信**（里程受打滑影响、特征受视觉质量影响）。表决票数由 `lap.detect_votes = 2` 参数化。

**5.2 地图落盘判据（PEN-08 的防线）**

- `.pgm` 与 `.yaml` 两文件**均存在**且 `mtime` 晚于本圈开始时间（证明是"比赛过程中"生成）；
- `.yaml` 中 `resolution`、`origin`、`image` 字段完整，`.pgm` 尺寸非零；
- 栅格中**未知率 < 95%**（防止"空地图"被当成已建图）；
- 落盘目录 `map.save_dir = ~/maps/`、文件名 `map.file_name = race_map`。

**5.3 建图成功判据（PEN-07 vs PEN-08，规则 2.4.2(5) 原文）**

> "挡板和锥桶与实际相符并且清晰可见，视为建图成功。"

- **成功**（不罚）：挡板与锥桶清晰、位置正确；
- **+5 s**（PEN-07）：确实建了但质量不达标（不清晰 / 挡板模糊但可见 / 丢失 ≤2 个）；
- **+50 s**（PEN-08）：提前建图 / 未构建地图 / 挡板位置明显不符。

**5.4 切换完成判据**

- **步骤②完成**：`/tf` 上 `map→odom` **暂停发布**，持续 ≥1.5 s 或超时 1.5 s（`switch.timeout` 内的子超时）；
- **步骤④完成**：`/tf` 上 `map→odom` **由 amcl 重新发布**，超时 3.0 s；
- 步骤③→⑤**总耗时 < `switch.timeout`(6.0 s)**；超时即转 `WATCHDOG_RECOVER` 蠕动分支。

**5.5 TF 权属交接顺序（防双发布者）**

```
Cartographer 活动(发布 map→odom) ── ① 异步存图 ── ② 终止进程(停发) ── ③ 启动 amcl
                                    ▲ 空窗期：速度不得为 0、不得提速、不得进弯道
                                    └ ④ amcl 恢复发布 map→odom ── ⑤ 确认 ── ⑥ 提速
```

> `odom→base_link` 全程由 **`ekf_node` 唯一发布**，切换不改变其权属（§7.2）；仅 `map→odom` 发生交接。

## F.6 关键时序（逐字对应 §7.3）

| 步 | 事件 | 车端动作 | 超时 / 约束 |
|---|---|---|---|
| ⓪ | **前置检查：IMU 自校准**（v2.1 新增） | 确认 IMU 静态零偏已收敛（静置时 `angular_velocity.z` 接近 0）；**未收敛则以低速直线行驶等待**，不得带未收敛的 IMU 进入切换 | 依据 §A.2「IMU 静态漂移」；未收敛不得切换 |
| ① | 圈次判定成立 | 进入 `ROLLING_SWITCH`，保持低速蠕动 ≥ `speed.rolling_switch`(0.15–0.25 m/s) | 速度 > 0 |
| ② | 后台**异步**调用 `save_map` | 落盘 `.pgm/.yaml`（不停车，回调组异步） | 禁止阻塞式 `map_saver_cli` |
| ③ | 终止建图栈 | `finish_trajectory` → `write_state` → **进程级终止**；等 `map→odom` 停发 | 等待超时 1.5 s |
| ④ | 启用导航栈 | configure + activate(`nav2_amcl`) → 发布 `/initialpose`（当前 odom 位姿 + 地图匹配初值） | — |
| ⑤ | 确认恢复 | 确认 `map→odom` 由 amcl 恢复发布 | 超时 3.0 s |
| ⑥ | 恢复 `NAVIGATING` | 逐步提速至 `speed.straight` | 空窗期内不得提速 / 不得进弯道 |

**时序铁律（摘自 §7.3，逐字一致）**

| 规则 | 原因 |
|---|---|
| 切换期间速度不得为 0，最低 `speed.rolling_switch` | FAIL-03：静止 >10 s 判负 |
| 步骤③→⑤总耗时 <| `switch.timeout`(20.0 s) 给蠕动留余量；超时进 `WATCHDOG_RECOVER` |
| `map→odom` 空窗期内**不得提速**、不得进弯道 | 空窗期定位精度下降，提速会导致偏离（FAIL-01） |
| 若 ② 保存失败，允许继续行驶并在下一圈起点前重试一次 | 地图缺失 = PEN-08（+50 s），必须补救 |
| 切换窗口优先安排在**直线段** | `switch.prefer_straight = true`；用里程先验选窗 |

## F.7 参数表（全部引自 §9，逐字一致）

| 参数 | 值 | 出处 |
|---|---|---|
| `lap.lap_length` | **待实测** | §9.7 |
| `lap.detect_distance_ratio` | 0.95 | §9.7 |
| `lap.detect_votes` | 2 | §9.7 |
| `switch.timeout` | 20.0 s | §9.7 |
| `switch.prefer_straight` | `true` | §9.7 |
| `map.save_dir` | `~/maps/` | §9.7 |
| `map.file_name` | `race_map` | §9.7 |
| `speed.rolling_switch` | 0.15–0.25 m/s | §9.5 |
| `speed.straight` | 1.8–2.0 m/s | §9.5 |
| `watchdog.static_timeout` | 9.0 s（豁免状态**不含** `ROLLING_SWITCH`） | §9.7 |
| `watchdog.recover_speed` | 0.15 m/s（**不得为 0**） | §9.7 |
| `TRAJECTORY_BUILDER_2D.submaps.num_range_data` | 待实测 | §9.2 |
| `POSE_GRAPH.optimize_every_n_nodes` | 90 | §9.2 |
| `TRAJECTORY_BUILDER_2D.min_range` / `max_range` | 0.30 m / 15.0 m | §9.2 |
| `set_initial_pose` | `true` | §9.3 |
| `initial_pose.covariance` | 0.25 | §9.3 |
| `min_particles` / `max_particles` | 500 / 2000 | §9.3 |
| `laser_model_type` | `likelihood_field` | §9.3 |
| `update_min_d` / `update_min_a` | 0.10 m / 0.10 rad | §9.3 |

## F.8 测试用例

| ID | 场景 | 步骤 | 通过判据 |
|---|---|---|---|
| TC-F01 | 正常切换（直道） | 在直线段触发切换 | 六步全通过；`map→odom` 空窗 < `switch.timeout`(6.0 s)；**全程速度 > 0** |
| TC-F02 | 切换期间零速（反例） | 人为令速度降至 0 持续 >9 s | 看门狗触发 `WATCHDOG_RECOVER` 蠕动；**静止从未 > `static_timeout`(9 s)**（FAIL-03 反例） |
| TC-F03 | 切换超时 | 令步骤③停滞 > 6.0 s | 自动转入蠕动分支，不停车、不判负 |
| TC-F04 | 地图落盘失败 | 令 `save_map` 服务返回失败 | 继续行驶并在下一圈起点前重试一次；`.pgm/.yaml` 最终存在（PEN-08 反例） |
| TC-F05 | 圈次判定表决 | 仅单判据成立 | **不切换**（三取二）；需 ≥ `lap.detect_votes`(2) 票 |
| TC-F06 | TF 双发布者（反例） | 强制同时激活 Cartographer 与 amcl | 应被管理器阻止；`map→odom` 在任一时刻**只有一个发布者** |
| TC-F07 | 挡板显现 | 第一圈经过 3 处挡板 | 地图中可见 3 处挡板轮廓（TR-08、PEN-07/08 反例） |
| TC-F08 | 空窗期进弯道（反例） | 令切换点安排在弯道 | 禁止提速分支生效；偏离检测不触发（FAIL-01 反例） |

## F.9 失效模式（简版 FMEA）

| 失效 | 后果 | 严重度 | 检出 / 缓解 |
|---|---|---|---|
| 切换期间车辆静止 >10 s | FAIL-03 **判负** | **极高** | 速度下界 `speed.rolling_switch` + `watchdog` 主动蠕动保障 |
| Cartographer 与 amcl 同时发布 `map→odom` | 定位失效、TF 树冲突 | **极高** | 严格串行；F.5.5 权属交接 + 步骤②停发确认（1.5 s） |
| 切换耗时 > `switch.timeout` | 空窗期过长、偏离 | 高 | 超时转 `WATCHDOG_RECOVER`；`switch.prefer_straight` 选窗 |
| 地图未落盘 / 落盘为空 | PEN-08 **+50 s** | **极高** | 异步 `save_map` + 落盘合格性自检（F.5.2）+ 重试一次 |
| 被判定"提前建图" | PEN-08 **+50 s** | **极高** | 第一圈真实探索 + 保留时间戳/日志；禁用预置地图（ADR-005） |
| 挡板未在地图中显现 | PEN-07 **+5 s**（严重时 PEN-08） | 高 | 挡板合并后处理（§9.1）+ RViz 人工复核 |
| 使用 `ros2 bag` 回放 | FAIL-04 **取消成绩** | **极高** | 比赛中不启动任何 `ros2 bag`；录像用外部设备 |
| 空窗期进弯道 / 提速 | 偏离轨道 FAIL-01 | 高 | 空窗期限速禁提速；切换点安排于直线段 |

## F.10 需求—罚则—设计映射

| 需求 / 罚则 | 本卡对应设计 |
|---|---|
| TR-01 两圈 | F.4 状态机（`MAPPING → ROLLING_SWITCH → NAVIGATING`） |
| TR-02 第一圈自主探索建图 | `MAPPING` 状态：Cartographer 真实建图 + 落盘 `.pgm/.yaml` |
| TR-03 第二圈基于地图加速 | `NAVIGATING`：`nav2_amcl` 定位 + 逐步提速至 `speed.straight` |
| TR-08 挡板须显示在地图 | F.5.3 建图成功判据 + §9.1 挡板合并后处理 |
| PEN-07 建图质量不达标 +5 s | F.5.2 落盘自检 + 宁可"模糊但真实" |
| PEN-08 未在比赛过程中建图 +50 s | F.5.2 `mtime` + 未知率判据；ADR-005 禁预置地图 |
| FAIL-01 偏离轨道判负 | F.4 异常分支 + 空窗期限速禁入弯道 |
| FAIL-03 静止 >10 s 判负 | F.4 `ROLLING_SWITCH` 速度下界 + F.6 时序铁律 + §9.7 看门狗 |
| FAIL-04 使用 `ros2 bag` 回放取消成绩 | F.2 边界：比赛中禁用 `ros2 bag`，录像用外部设备 |

---

# 附录 G　系统设计卡 E-3：第二圈停车标志识别与入库停车

> **设计卡定位**：把散落在 §1.2（CR-07/10/11/22）、§1.4（PEN-04）、§1.5（FAIL-02）、§5.3（`PARKING_DETECT`/`PARKING`/`DONE`）、§9.5~§9.7 的"第二圈入库停车"内容，收敛为一张**可独立评审、可独立实现、可独立测试**的设计卡。这是第二圈的**收尾动作**，成败直接决定 PEN-04（+20 s）与 FAIL-02（判负）。
> **一致性约束**：本卡内所有**话题名、参数名、状态名**与 §7 / §8 / §9 逐字一致。本卡**未新增接口**（`/parking_sign/state` 等已存在于 §8.1）；**新增的参数已同步登记到 §9.6 与 §9.7**（`hsv.blue_h_range`、`parking.approach_distance`、`parking.target_x/y/yaw`、`parking.in_zone_wheels_min`）。

## G.0 设计卡头部

| 项 | 内容 |
|---|---|
| 设计卡编号 | **E-3** |
| 子系统名 | 第二圈停车标志识别与入库停车（Parking & Docking） |
| 覆盖需求 | TR-04（依据提示停在停车区域，三及以上车轮）、TR-07（第二圈识别标志牌入库；第一圈不停）、CR-10（识别停车检测区域内标志牌并停在标志牌区域内）、CR-11（标志牌 29.7×42 cm 蓝底白字 P）、CR-22（红绿灯区→停车区→发车区的运行顺序） |
| 覆盖罚则 | PEN-04（+20 s）、FAIL-02（判负）、PEN-11（人为干预 +100 s / 次） |
| 覆盖规则条款 | 2.1、2.2.1(3)(8)(9)、2.4.2(2)、2.4.2(7)、2.2.1(11) |
| 关键前提 | ① "停车检测区域"与"停车区域"是否为两个区域**待 Q7 明确**（§2.3）；② 入库成功判据是**三及以上车轮**在停车区域内（TR-04/PEN-04）；③ 在停车检测区域前**停车识别**超 10 s → FAIL-02 **判负**，故识别**优先在行进中**完成；④ 区域边界以**胶带内侧**为准（CR-07）；⑤ 第一圈遇标志牌**不得停车**（TR-07）。 |
| 技术栈 | **ROS 2 Humble**（传统 CV 识别 + 状态机 + 纯追踪/模糊 PID 低速入库，见附录 C.2） |
| 状态 | 设计已冻结；`parking.target_x/y/yaw`、`parking.approach_distance`、`hsv.blue_h_range` 为**现场标定项**；**Q4、Q7 答复前不得据此编码**。⚠ **零标定兜底（v2.1 R60）**：`parking.target_*` / `parking.zone_rect` 未标定时，本卡 §G.3 的坐标入口自动停用、"≥3 车轮入区"改由**里程/静止兜底**判定（详见 R60 与 §9.7），故未标定也能完赛 |
| 责任人 | 视觉 + 状态机 + 控制（入库为三者的交汇点） |

## G.1 问题定义

1. **"三个区域"的关系未完全明确**：规则并列出现"停车检测区域""停车区域""标志牌区域"（CR-10、CR-07），Q7 未答复前三者关系存在歧义。本卡按**最保守解释**设计（识别区 ≠ 入库区，且均以胶带内侧为界）。
2. **罚则严重度与直觉相反**：`FAIL-02 停车识别 >10 s 判负` ＞ `PEN-04 停车失败 +20 s`。→ **宁可"行进中识别+慢速入库"，也绝不在检测区前停死做识别**。
3. **几何精度要求高**：PEN-04 的判据是**三及以上车轮**同时位于停车区域内，而边界是**胶带内侧**（比锥桶中心连线更窄）。→ 入库必须低速收敛，且目标点按胶带内侧标定。
4. **两圈行为必须分叉**：TR-07 要求第二圈识别标志牌入库，**第一圈遇到标志牌不得因它停车**（否则干扰第一圈建图节奏，且无意义）。
5. **零遥控约束**：PEN-11 规定人为干预每次 +100 s。入库是对精度的考验，但**不允许任何人工修正**，只能靠视觉置信度与低速收敛兜底。

## G.2 系统边界与职责划分

| 侧 | 承担 | 不承担 |
|---|---|---|
| **赛道端（组委会）** | 布设停车检测区域、停车区域与标志牌（蓝底白字 P，29.7×42 cm，CR-11）；判定车轮是否在区域内 | — |
| **车端（本队实现）** | ① 第二圈**行进中**识别标志牌；② 判定进入停车区域；③ **低速入库**使 ≥3 车轮位于区域内；④ 停在区域内进入 `DONE`；⑤ 第一圈**抑制**入库行为 | **不依赖标志牌先验位置**、**不硬编码停车点**、**不允许人工修正**（PEN-11） |

> **关键推论**：入库是全系统**对定位精度要求最高**的动作（厘米级），但它发生在**第二圈定位已收敛**之后。因此本卡的设计目标可浓缩为一句：**用"行进中识别 + 低速收敛"换取精度，且全程不因识别而停车超 10 s。**

## G.3 接口契约（与 §8 逐字一致）

| 接口 | 类型 | 方向（本卡视角） | 用途 |
|---|---|---|---|
| `/camera/image_raw` | `sensor_msgs/msg/Image` | sub | 标志牌识别输入源（QoS `SensorDataQoS`） |
| `/camera/camera_info` | `sensor_msgs/msg/CameraInfo` | sub | 去畸变内参（150° 视场必需，§9.6 `camera.calibration_file`） |
| `/parking_sign/state` | `std_msgs/msg/String` | sub | 值 `detected` / `absent`；由 `parking_sign_detector` 发布（§8.1、§9.6） |
| `/amcl_pose` | `geometry_msgs/msg/PoseWithCovarianceStamped` | sub | 入库定位来源（第二圈 `nav2_amcl`，§8.2） |
| `/odometry/filtered` | `nav_msgs/msg/Odometry` | sub | 低速里程辅助（`ekf_node`，§8.2） |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | pub | 入库低速执行的唯一通道（QoS `Reliable`，§8.3） |
| `/race_state` | `std_msgs/msg/String` | pub | 状态名取 G.4 枚举 |
| `/race_lap` | `std_msgs/msg/UInt8` | pub | 圈次（1/2）：第一圈据此**抑制**入库行为（TR-07） |

> **第一圈抑制逻辑**：`/race_lap == 1` 时，即使 `/parking_sign/state == detected`，`mission_manager` 也**不得**进入 `PARKING_DETECT`（TR-07）。该判断是入库子系统的**唯一圈次依赖**。

## G.4 状态机

```
                 ┌──────────────────────────────────────────────┐
                 │  NAVIGATING（第二圈正常行驶，/race_lap == 2）    │
                 └───────────────┬──────────────────────────────┘
    距检测区域 ≤ parking.approach_distance（据地图 / 里程预判）│
                                                        ▼
                                     ┌──────────────────────────────┐
                                     │ PARKING_DETECT               │
                                     │ 限速 ≤ speed.parking_approach │
                                     │ 行进中识别 /parking_sign/state │
                                     └───────┬──────────────┬────────┘
                                  detected   │      超停车识别上限
                                             ▼              ▼
                              ┌──────────────────────┐  ┌────────────────────┐
                              │ PARKING              │  │ SAFE_HOLD（附录 E） │
                              │ 低速入库 → 目标点      │  │ 停死风险 → 蠕动重启 │
                              │ ≥3 车轮入区域          │  └────────────────────┘
                              └──────────┬───────────┘
                                         ▼
                                    ┌─────────┐
                                    │  DONE   │ 停在停车区域内完赛（不计 FAIL-03）
                                    └─────────┘
```

| 状态 | 进入条件 | 动作 | 退出条件 |
|---|---|---|---|
| `PARKING_DETECT` | 第二圈车距 ≤ `parking.approach_distance` | 限速至 `speed.parking_approach`；**行进中**识别 `/parking_sign/state` | `detected` → `PARKING` |
| `PARKING` | `state == detected` | 以目标点 `parking.target_*` 低速收敛入库；持续校验车轮入区数 | ≥ `parking.in_zone_wheels_min`(3) 车轮入区 |
| `DONE` | 入库判据成立 | 停止（**不计 FAIL-03**）；保持静止完赛 | — |
| 超时分支 | `PARKING_DETECT` 内识别耗时 > `parking.stop_recognition_limit`(9.0 s) | 不长时间停车；转入蠕动重新识别（避免 FAIL-02） | 出现 `detected` 或离开区域 |

> **为何 `PARKING` / `DONE` 的静止合法**：二者在 §9.7 `watchdog.exempt_states` 内，静止**不触发**看门狗。`PARKING` 的短暂停稳是入库的必要动作，`DONE` 是任务终止态（规则意义上不再"运行"）。

## G.5 判据与算法

**5.1 标志牌视觉判据（沿用 ADR-004 传统 CV）**

| 项 | 做法 |
|---|---|
| ROI | 画面中下部（标志牌贴地）；相机俯仰角按 **CR-11**（牌面 29.7×42 cm）反推，使其在预期入库距离内落于有效区且畸变可接受 |
| 颜色 | 蓝底 `hsv.blue_h_range` **现场标定**；`hsv.s_min` / `hsv.v_min` **现场标定**（§9.6） |
| 形状 | 白字 "P"（蓝色 blob 内白色字符占比校验）；面积按 CR-11 反推 |
| 置信度 | ≥ `parking.confidence_threshold`(0.70) 才输出 `detected` |
| 去畸变 | 必经 `camera.calibration_file`（150° 视场） |

**5.2 行进中识别优先（FAIL-02 的防线）**

- `parking.detect_in_motion = true`：在 `PARKING_DETECT` 内**边行进边识别**；
- 若确需短暂停车识别，其累计时长**严格 < `parking.stop_recognition_limit`(9.0 s)**；
- 超限即转蠕动重新识别，**绝不**让"停车识别"逼近 10 s 规则上限。

**5.3 入库控制律**

- 限速 `speed.parking_approach` ≤ 0.30 m/s（CR-22：绿灯起步后至停车区距离短，提前减速）；
- 目标点为 `parking.target_x/y/yaw`（`map` 坐标系，**胶带内侧**标定，CR-07）；
- 复用 §9.5 纯追踪 + 模糊 PID，以**低前视距离**（`lookahead.min`）保证末端收敛精度；
- 全程 `/cmd_vel` 输出，**无人工干预**（PEN-11）。

**5.4 入库成功判据**

> 规则 2.4.2(2)：**三及以上车轮**停在停车区域内视为成功；停车失败或未停车 → PEN-04（+20 s）。

- 计数口径：以车体几何外推出四轮在 `map` 坐标系的位置，落在**胶带内侧**矩形内者 ≥ `parking.in_zone_wheels_min`(3)；
- 判据成立即进入 `DONE` 并停稳。

**5.5 两圈分叉（TR-07）**

| 圈次 | 行为 |
|---|---|
| 第一圈（`/race_lap == 1`） | **抑制**入库：忽略 `/parking_sign/state`，不停车（保持建图节奏） |
| 第二圈（`/race_lap == 2`） | 执行 `PARKING_DETECT → PARKING → DONE` |

## G.6 关键时序

| t | 事件 | 车端动作 | 计时 / 约束 |
|---|---|---|---|
| t0 | 车距检测区域 ≤ `parking.approach_distance` | 进入 `PARKING_DETECT`，降速至 ≤ `speed.parking_approach` | 正常行驶 |
| t1 | 视觉识别到标志牌 | `/parking_sign/state = detected`，进入 `PARKING` | **识别耗时 < `stop_recognition_limit`(9.0 s)**（FAIL-02） |
| t2 | 位姿收敛至目标点 | 低速逼近 `parking.target_x/y/yaw` | 限速 ≤ 0.30 m/s |
| t3 | ≥3 车轮入区 | 判定成功 | PEN-04 反例 |
| t4 | 停稳 | 进入 `DONE`（**不计 FAIL-03**） | — |

> **与 E-1/E-2 的衔接（CR-22）**：第二圈节奏为 `TRAFFIC_APPROACH → 红灯停车 → 绿灯起步 → 短距离加速 → PARKING_DETECT → PARKING → DONE`。**绿灯起步后至停车区的加速距离很短**（`speed.parking_approach` 已在 E-1/E-3 中作为衔接限速），速度规划必须提前为入库制动预留距离。

## G.7 参数表（引自 §9，逐字一致）

| 参数 | 值 | 出处 |
|---|---|---|
| `parking.confidence_threshold` | 0.70 | §9.6 |
| `parking.detect_in_motion` | `true` | §9.6 |
| `hsv.blue_h_range` | **现场标定**（新增，已登记 §9.6） | §9.6 |
| `hsv.s_min` / `hsv.v_min` | **现场标定** | §9.6 |
| `camera.calibration_file` | `config/camera_info.yaml` | §9.6 |
| `speed.parking_approach` | ≤ 0.30 m/s | §9.5 |
| `lookahead.min` | 0.40 m | §9.5 |
| `parking.stop_recognition_limit` | **9.0 s** | §9.7 |
| `parking.approach_distance` | **待标定**（新增，已登记 §9.7） | §9.7 |
| `parking.target_x` / `parking.target_y` | **待标定**（新增，已登记 §9.7） | §9.7 |
| `parking.target_yaw` | **待标定**（新增，已登记 §9.7） | §9.7 |
| `parking.in_zone_wheels_min` | **3**（新增，已登记 §9.7） | §9.7 |
| `watchdog.exempt_states` | 含 `PARKING, DONE` | §9.7 |
| `watchdog.static_timeout` | 9.0 s | §9.7 |

## G.8 测试用例

| ID | 场景 | 步骤 | 通过判据 |
|---|---|---|---|
| TC-G01 | 正常入库 | 第二圈识别到标志牌并入库 | ≥3 车轮落在胶带内侧区域内；进入 `DONE`（PEN-04 反例） |
| TC-G02 | 第一圈抑制 | 第一圈经过标志牌 | **不停车**、不进入 `PARKING_DETECT`（TR-07） |
| TC-G03 | 识别超时（反例） | 令 `PARKING_DETECT` 内识别 > 9 s | 转蠕动重新识别；**停车识别从未 ≥ 10 s**（FAIL-02 反例） |
| TC-G04 | 标志牌缺失 | 置 `/parking_sign/state = absent` | 不进入 `PARKING`；按预案低速通过或在区域内安全停稳 |
| TC-G05 | 入库精度 | 在胶带内侧目标点停稳 | ≥3 车轮入区；边界判定以**胶带内侧**为准 |
| TC-G06 | 相机断流 | 停止 `/camera/image_raw` | `state → absent`；不误入 `PARKING` |
| TC-G07 | 停稳合法性 | 入库后静止 > 10 s | **不触发**看门狗（`exempt_states` 含 `PARKING`/`DONE`） |
| TC-G08 | 无人工干预（反例） | 全程不触碰遥控 / 不修正轨迹 | 入库完成且无任何人工干预（PEN-11 反例） |

## G.9 失效模式（简版 FMEA）

| 失效 | 后果 | 严重度 | 检出 / 缓解 |
|---|---|---|---|
| 停车做识别超 10 s | FAIL-02 **判负** | **极高** | `parking.detect_in_motion` + `stop_recognition_limit`(9.0 s) |
| 车轮未全部入区（<3） | PEN-04 **+20 s** | 高 | 低速收敛 + 目标点按胶带内侧标定 + 定位收敛校验 |
| 标志牌漏检 | 无法入库 | 高 | 置信度阈值 0.70 + 行进中多帧识别 + 去畸变 |
| 标志牌误检（非牌面） | 误入入库流程 | 中 | 蓝底 + 白字 "P" 双重校验 + 面积反推 |
| 第一圈误停 | 干扰建图节奏 | 中 | `/race_lap` 门控抑制（G.5.5） |
| 入库中撞锥桶 | PEN-05 **+5 s/次** | 中 | 低速 + 横向安全裕度 |
| 人工修正轨迹 | PEN-11 **+100 s/次** | **极高** | 全程零遥控；参数现场可调但不干预运动 |

## G.10 需求—罚则—设计映射

| 需求 / 罚则 | 本卡对应设计 |
|---|---|
| TR-04 依据提示停在停车区域（≥3 车轮） | G.5.4 入库成功判据 + `parking.in_zone_wheels_min`(3) |
| TR-07 第二圈入库；第一圈不停 | G.5.5 两圈分叉（`/race_lap` 门控） |
| CR-10 识别检测区内标志牌并停在标志牌区域 | G.4 状态机（`PARKING_DETECT → PARKING`） |
| CR-11 标志牌 29.7×42 cm 蓝底白字 P | G.5.1 蓝底 + 白字 "P" 校验 + 面积反推 |
| CR-22 红绿灯区→停车区→发车区顺序 | G.6 与 E-1/E-2 的衔接（短加速距离限速） |
| PEN-04 停车失败 / 未停车 +20 s | G.5.4 精度判据 + 低速收敛 |
| FAIL-02 停车识别 > 10 s 判负 | G.5.2 行进中识别 + `stop_recognition_limit`(9.0 s) |
| PEN-11 人为干预 +100 s/次 | G.2 边界：全程零遥控，仅参数现场可调 |

---

# 附录 H　系统设计卡 E-4：锥桶与挡板感知

> **设计卡定位**：把散落在 §1.2（CR-02/03/04/06/13/14/15/17）、§1.4（PEN-05/06/09）、§1.5（FAIL-01）、§5.1（L1 感知层）、§8.1（`/scan`、`/scan_filtered`、`/cones`、`/boards`）、§9.1（DBSCAN + 颜色 + 挡板）的"锥桶与挡板感知"内容，收敛为一张**可独立评审、可独立实现、可独立测试**的设计卡。感知层是所有罚则的**根因层**——它之上才有建图、路径与状态机。
> **一致性约束**：本卡内所有**话题名、消息类型、参数名**与 §8 / §9 逐字一致。本卡**未新增接口**；**新增参数 1 项已同步登记到 §9.1**（`cone.min_confidence`）。
> **结构适配说明**：感知节点是**连续处理流水线**，不存在离散状态，故本卡以 **H.4「处理流水线」替代其它设计卡的「状态机」**，其余小节结构一致。

## H.0 设计卡头部

| 项 | 内容 |
|---|---|
| 设计卡编号 | **E-4** |
| 子系统名 | 锥桶与挡板感知（Cone & Board Perception） |
| 覆盖需求 | CR-02（赛道宽 1.5 m）、CR-03（锥桶间距 < 1.5 m）、CR-04（内圈红 / 外圈蓝）、CR-06（逆时针、发车区固定）、CR-13（挡板当天随机 3 处）、CR-14（挡板 50×80 cm）、CR-15（锥桶规格 + 去反光膜）、CR-17（外围警示线）；间接支撑 TR-08（挡板须显示在地图） |
| 覆盖罚则 | PEN-05（撞锥桶 +5 s/次）、PEN-06（折线压线 +10 s/圈）、PEN-09（绕错锥桶 +5 s/个）、FAIL-01（绕错 >3 或 10 s 未回轨 → 判负） |
| 覆盖规则条款 | 2.2(1)(2)(3)、2.2.1(4)(11)(12)(14) |
| 关键前提 | ① **不使用颜色传感器**：内红外蓝由**径向距离阈值**推断（ADR-005，规则 2.2.1(12) 明示"可利用锥桶颜色识别运行方向"）；② **CR-15：锥桶白色反光膜必须去除**，否则回波强度异常使 §9.1 全部标定作废；③ 挡板位置**当天随机、不可预知**，必须在线检测（CR-13）；④ `/cones` 发布于 **`map` 坐标系**，依赖 TF 链完整（§7）。 |
| 技术栈 | **ROS 2 Humble**（`scan_filter` → DBSCAN 分段聚类 → 径向颜色推断 → 挡板合并后处理，见附录 C.2） |
| 状态 | 设计已冻结；`color.radial_threshold`、`cone.min_confidence` 为**现场标定项**；**依赖 CR-15 场地前提（反光膜已去除）** |
| 责任人 | 感知 + SLAM（挡板必须落到地图上） |

## H.1 问题定义

1. **颜色不是真颜色**：规则给出的是**拓扑先验**——内圈红、外圈蓝、逆时针（CR-04/CR-06）。因此颜色可由**到赛道中心的径向距离**推断，无需颜色传感器，也无需预置地图（ADR-005）。这是全系统成本最低的一条强先验。
2. **反光膜会摧毁标定**：CR-15 要求去除锥桶的白色反光膜。若未去除，回波强度异常会使**聚类半径与点数统计失真**，§9.1 的所有 DBSCAN 参数与颜色阈值**全部作废**。→ 属**赛前场地准备**的一票否决项。
3. **挡板不是"一堆锥桶"**：挡板是 0.80 m × 0.50 m 的**连续反射面**（CR-14），DBSCAN 会把它切成多个簇。必须做**合并后处理**才能得到"一块挡板"，否则地图上出现一串伪锥桶。
4. **感知是罚则的根因层**：感知偏差会直接传导为撞桶（PEN-05）、折线压线（PEN-06）、绕错锥桶（PEN-09）乃至偏离判负（FAIL-01）。→ 感知的定位精度（`map` 系）是全局安全裕度的来源。
5. **点密度随距离衰减**：雷达 25–30 m 量程（HR-06），但锥桶点数随距离平方衰减。→ 必须**分段 DBSCAN**（近/远两套 `eps` 与 `min_samples`）并设 `detection.max_range` 上限。

## H.2 系统边界与职责划分

| 侧 | 承担 | 不承担 |
|---|---|---|
| **赛道端（组委会）** | 红蓝锥桶（630×280×280，**须去反光膜**）、当天随机 3 处蓝挡板、黄色胶带、外围警示线（CR-17） | — |
| **车端（本队实现）** | ① `/scan` → `/scan_filtered`（滤除警示线等远距离杂波，CR-17）；② 分段 DBSCAN 聚类；③ 径向阈值推颜色；④ 挡板合并后处理 → `/boards`；⑤ 输出 `/cones`（`map` 坐标系） | **不预置锥桶/挡板位置**、**不使用颜色传感器**、**不依赖赛道虚线**（CR-21 明确虚线仅为图纸描述） |

> **关键推论**：感知层输出的是"**建议**"而非"真相"。因此下游（建图、路径生成、状态机）必须保留容错：颜色错误 → 只影响内/外圈判定；位置误差 → 由路径横向裕度吸收。**感知不做决策，只提供带置信度的观测量。**

## H.3 接口契约（与 §8 逐字一致）

| 接口 | 类型 | 方向（本卡视角） | 用途 |
|---|---|---|---|
| `/scan` | `sensor_msgs/msg/LaserScan` | sub | 原始雷达输入（10–12 Hz，QoS `SensorDataQoS`，§8.1） |
| `/scan_filtered` | `sensor_msgs/msg/LaserScan` | pub | 由 `scan_filter` 输出给 `cone_detector`；滤除远距离杂波（CR-17） |
| `/cones` | `autocar_msgs/msg/ConeArray` | pub | 锥桶数组，**`map` 坐标系**，10 Hz（§8.1） |
| `/boards` | `autocar_msgs/msg/BoardArray` | pub | 挡板数组，10 Hz；挡板合并后处理输出（CR-13/14） |
| `/map` | `nav_msgs/msg/OccupancyGrid` | sub | 第一圈参考（`map` 坐标系来源） |
| `/tf`、`/tf_static` | `tf2_msgs/msg/TFMessage` | sub | `map→odom→base_link→laser_link` 链，用于把扫描点变换到 `map` 系（§7） |

**自定义消息（与 §8.1 逐字一致）**

```
# autocar_msgs/msg/Cone.msg
float32 x            # 在 map 坐标系下的 x (m)
float32 y            # 在 map 坐标系下的 y (m)
string  color        # "red" | "blue" | "unknown"（由径向阈值推断，见 ADR-005）
float32 confidence   # 0.0 ~ 1.0
uint8   cluster_size # 参与聚类的点数
float32 distance     # 距车体距离 (m)
```

> ⚠ `.msg` 文件内部字段类型写**短名**（`std_msgs/Header`、`Cone[]`），不得写 `std_msgs/msg/Header`（§8.1 末说明）。

## H.4 处理流水线（本卡以流水线替代「状态机」）

```
/scan ──► [ scan_filter ] ──► /scan_filtered ──► [ 分段 DBSCAN ]
              │ (滤警示线 CR-17,               │
              │  min/max_range)               │ 近/远两套 eps & min_samples
              │                               ▼
              │                        [ 径向颜色推断 ] ──► /cones (map 系)
              │                               │  color.radial_threshold
              │                               └──► [ 挡板合并后处理 ] ──► /boards
              │                                       merge_gap / merge_min_clusters
              └─ 全程依赖 TF：map→odom→base_link→laser_link（§7）
```

| 阶段 | 输入 | 处理 | 输出 |
|---|---|---|---|
| ① 滤波 | `/scan` | 距离门限（`detection.min_range`~`max_range`）；滤除警示线等远距离杂波（CR-17） | `/scan_filtered` |
| ② 聚类 | `/scan_filtered` | 分段 DBSCAN（近/远两套 `eps`、`min_samples`）；质心即锥桶候选 | 锥桶候选簇 |
| ③ 颜色 | 候选簇 + `map` 位姿 | 径向距离 < `color.radial_threshold` → `red`（内圈），否则 `blue`；置信度由 `cluster_size` 与径向裕度合成 | 带 color/confidence 的锥桶 |
| ④ 门限 | 锥桶 | 置信度 < `cone.min_confidence`(0.50) → 丢弃 | 有效锥桶 |
| ⑤ 后处理 | 有效锥桶 | 相距 ≤ `board.merge_gap`(0.50 m) 且簇数 ≥ `board.merge_min_clusters`(3) → 合并为**挡板** | `/boards` 与剩余 `/cones` |

> **为何用流水线而非状态机**：本节点是**每帧独立**的无状态处理（除聚类缓存外），不存在跨帧的状态迁移。唯一"时序性"是帧率（10–12 Hz）与 TF 可用性，属**数据流约束**而非状态机。

## H.5 判据与算法

**5.1 扫描预处理（`scan_filter`）**

- 距离门限：仅保留 `detection.min_range`(0.30 m) ~ `detection.max_range`(10.0 m) 的点；
- 远距离杂波抑制：外围**警示线**（CR-17）会造成大范围回波，须在 `scan_filter` 内剔除（强度/几何联合判据）；
- 输出 `/scan_filtered` 供 `cone_detector` 消费（SLAM 仍直接消费 `/scan`，见 §8.1）。

**5.2 分段 DBSCAN（`cone_detector`）**

| 距离段 | `eps` | `min_samples` | 依据 |
|---|---|---|---|
| < 5 m | `dbscan.eps_near`(0.15 m) | `dbscan.min_samples_near`(3) | 近处点密，锥桶点云间距小 |
| 5–10 m | `dbscan.eps_far`(0.25 m) | `dbscan.min_samples_far`(2) | 远处点稀，需放宽以保持召回 |

**5.3 径向颜色推断（ADR-005）**

- 以**赛道中心点**为参考，计算锥桶候选的径向距离；
- < `color.radial_threshold` → `red`（**内圈**，逆时针运行时的左侧，CR-06）；否则 `blue`（外圈）；
- `color.radial_threshold` **必须赛前标定**（§9.1.1 流程：低速跑一圈 → 记录质心 → 取红/蓝两簇分界）；
- 置信度合成：`confidence = f(cluster_size, |径向距离 − 阈值|)`；无法判定则置 `unknown`。

**5.4 挡板合并后处理（CR-13/14）**

- 挡板是**连续反射面**，会在 DBSCAN 中表现为多个相邻簇；
- 合并判据：相邻簇间距 ≤ `board.merge_gap`(0.50 m) **且**合并簇数 ≥ `board.merge_min_clusters`(3) → 判为**一块挡板**；
- 合并后从 `/cones` 中移除这些簇，单独发布到 `/boards`（避免地图出现伪锥桶，TR-08）。

**5.5 输出门限**

- 锥桶置信度 < `cone.min_confidence`(0.50) → 不发布；
- 该门限用于抑制**反光膜异常回波**（若场地未按 CR-15 去膜）与远距离伪簇。

## H.6 关键时序 / 数据流

| 环节 | 频率 | 延迟约束 | 备注 |
|---|---|---|---|
| `/scan` 输入 | 10–12 Hz | — | HR-06；单帧位移 = v / f_scan（1.8 m/s、10 Hz → 约 18 cm） |
| `/scan_filtered` | 同 `/scan` | < 1 帧 | 逐帧处理 |
| `/cones`、`/boards` 输出 | 10 Hz | < 2 帧 | §8.1 约定 10 Hz |

> **帧率与速度的耦合（安全裕度来源）**：单帧间隔内车辆位移约 `v / f_scan`。速度越高，锥桶观测越"跳"，颜色与位置越易误判。→ 这解释了 §9.5 为何对**折线区**（`speed.fold_zone` ≤ 0.8 m/s）与**红绿灯/入库接近区**（≤ 0.30 m/s）单独限速：限速本身就是感知的**稳定化手段**。

## H.7 参数表（引自 §9.1，逐字一致）

| 参数 | 值 | 出处 |
|---|---|---|
| `dbscan.eps_near` / `dbscan.min_samples_near` | 0.15 m / 3 | §9.1 |
| `dbscan.eps_far` / `dbscan.min_samples_far` | 0.25 m / 2 | §9.1 |
| `detection.max_range` / `detection.min_range` | 10.0 m / 0.30 m | §9.1 |
| `color.radial_threshold` | **待标定**（§9.1.1 流程） | §9.1 |
| `board.merge_gap` | 0.50 m | §9.1 |
| `board.merge_min_clusters` | 3 | §9.1 |
| `cone.min_confidence` | 0.50（新增，已登记 §9.1） | §9.1 |
| `speed.fold_zone` | ≤ 0.8 m/s（感知稳定化） | §9.5 |
| `speed.traffic_approach` / `speed.parking_approach` | ≤ 0.30 m/s（感知稳定化） | §9.5 |

> **场地前提（CR-15）**：上述全部标定以"锥桶**已去除白色反光膜**"为前提；若保留反光膜，回波强度异常会造成聚类半径与点数统计失真，**本节所有标定作废**。

## H.8 测试用例

| ID | 场景 | 步骤 | 通过判据 |
|---|---|---|---|
| TC-H01 | 近场聚类 | 距锥桶 < 5 m 匀速通过 | 每个锥桶稳定检出 1 簇；无碎裂（PEN-05 反例） |
| TC-H02 | 远场聚类 | 距锥桶 5–10 m | 采用 `eps_far`/`min_samples_far`，召回稳定、无误合并 |
| TC-H03 | 颜色推断 | 沿中线低速跑一圈 | 内圈判 `red`、外圈判 `blue`；`color.radial_threshold` 分界清晰（§9.1.1） |
| TC-H04 | 挡板识别 | 经过 3 处挡板 | 每处合并为**一块** `/boards`；不产生伪锥桶（TR-08 反例） |
| TC-H05 | 反光膜异常（反例） | 人为保留反光膜 | 置信度门限 `cone.min_confidence` 抑制异常簇；告警提示场地不达标 |
| TC-H06 | 警示线杂波 | 场地外围有警示线 | `scan_filter` 剔除远距离杂波，`/cones` 不含场外点（CR-17） |
| TC-H07 | 大速度抖动（反例） | 以 `speed.straight`(1.8–2.0 m/s) 直行 | 锥桶 id 不跳变、位置抖动在可接受范围；否则需降速 |
| TC-H08 | TF 缺失（反例） | 断开 `map→odom` | `/cones` 暂停发布（不输出错误坐标）；恢复后自动续发 |

## H.9 失效模式（简版 FMEA）

| 失效 | 后果 | 严重度 | 检出 / 缓解 |
|---|---|---|---|
| 场地未去反光膜 | 标定全废、聚类失真 | **极高** | 赛前按 CR-15 一票否决确认；`cone.min_confidence` 兜底 |
| 远场点稀漏检 | 绕错锥桶 → PEN-09 / FAIL-01 | 高 | 分段 DBSCAN + `detection.max_range` 上限 + 限速 |
| 相邻锥桶误合并 | 路径偏移、压线 | 中 | `dbscan.eps` 收紧；`board.merge_*` 与锥桶分离 |
| 挡板被当锥桶 | 地图伪特征（PEN-07） | 中 | 合并后处理（H.5.4）+ TR-08 复核 |
| 颜色误判（内外圈） | 方向先验错误 | 中 | 径向阈值 + 置信度门限 + 逆时针先验交叉校验 |
| `map` 位姿漂移 | 锥桶坐标错位 | 高 | 依赖 §7 TF 链与 EKF；建图期低速 |
| 警示线杂波混入 | 伪锥桶、路径外扩 | 中 | `scan_filter` 距离/几何联合滤除（CR-17） |

## H.10 需求—罚则—设计映射

| 需求 / 罚则 | 本卡对应设计 |
|---|---|
| CR-02/CR-03 赛道宽 1.5 m、锥桶间距 < 1.5 m | H.5.2 分段 DBSCAN（单帧可同时观测内外圈） |
| CR-04/CR-06 内红外蓝、逆时针 | H.5.3 径向阈值推断（ADR-005，不用颜色传感器） |
| CR-13/CR-14 挡板当天随机 3 处、50×80 cm | H.5.4 挡板合并后处理 → `/boards` |
| CR-15 锥桶规格 + 去反光膜 | H.7 场地前提（一票否决）+ 门限兜底 |
| CR-17 外围警示线 | H.5.1 `scan_filter` 杂波滤除 |
| TR-08 挡板须显示在地图 | H.5.4 挡板单独发布，供建图显现 |
| PEN-05 撞锥桶 +5 s/次 | H.5.2 稳定聚类 + 限速稳定化 |
| PEN-06 折线压线 +10 s/圈 | H.5.1 滤除警示线 + 折线区限速（§9.5） |
| PEN-09 绕错锥桶 +5 s/个 | H.5.2 远场召回 + H.6 帧率/速度耦合控制 |
| FAIL-01 绕错 >3 或 10 s 未回轨判负 | H.8 TC-H07/H08 + 下游偏离恢复 |

---

# 附录 I　系统设计卡 E-5：定位（EKF 里程计融合 + 第二圈 AMCL）

> **设计卡定位**：把散落在 §3.1（传感器能力边界）、§5.1（L2 定位与建图层）、§7.1/§7.2（TF 权属）、§7.3（切换时序）、§8.2（建图与定位接口）、§9.3（AMCL）、§9.8（EKF）的"定位"内容，收敛为一张**可独立评审、可独立实现、可独立测试**的设计卡。定位是**全系统精度链的源头**——感知、路径、入库、停车区域判定的误差都在此层被决定。
> **一致性约束**：本卡内所有**话题名、参数名、TF 名称**与 §7 / §8 / §9 逐字一致。本卡**未新增接口**；**新增 §9.8「里程计融合（`ekf_node`）」参数节**（此前缺失）已同步登记，见 R39。
> **结构适配说明**：定位节点是**连续估计 + 两种全局模式（圈次）**，无逐帧状态迁移，故本卡以 **I.4「定位模式与 TF 权属」替代其它设计卡的「状态机」**，其余小节结构一致。

## I.0 设计卡头部

| 项 | 内容 |
|---|---|
| 设计卡编号 | **E-5** |
| 子系统名 | 定位（EKF 里程计融合 + 第二圈 `nav2_amcl`） |
| 覆盖需求 | TR-03（第二圈基于地图导航并加速）；间接支撑 TR-04（入库 ≥3 车轮）、TR-05（红绿灯区域停车）；CR-06（逆时针、发车区） |
| 覆盖罚则 | FAIL-01（偏离轨道判负）、PEN-09（绕错锥桶 +5 s/个）、PEN-01（红绿灯车轮不在区域 +20 s）、PEN-04（入库失败 +20 s）、PEN-11（人为干预 +100 s/次，靠自动初始位姿规避） |
| 覆盖规则条款 | 2.1、2.4.2(2)、2.4.2(6)、2.4.2(7)、2.4.2(11) |
| 关键前提 | ① `odom→base_link` **唯一发布者**为 `ekf_node`（§7.1）；② `map→odom` 在 Cartographer 与 AMCL 之间**互斥**（§7.2，E-2 已处理切换）；③ 切换时 AMCL 初始位姿**自动发布**（取当前 odom 位姿，§7.3 步骤④），**避免人工干预**（PEN-11）；④ **`nav2_amcl` 订阅 `/scan` 的 QoS 须实机核对**（§8 头部 QoS 陷阱）；⑤ IMU 动态精度仅 2.5°（HR-07），航向**主要依赖激光**。 |
| 技术栈 | **ROS 2 Humble**（`robot_localization` 的 `ekf_node` + `nav2_amcl`，见附录 C.2） |
| 状态 | 设计已冻结；`initial_pose.x/y/yaw`、`wheel.scale_factor` 为**待标定项**；**QoS 兼容性待实机核对**；**Q5/Q6（轴距/驱动接口）答复前控制耦合项不得编码** |
| 责任人 | SLAM + 控制（定位与控制强耦合） |

## I.1 问题定义

1. **定位是"两级"的**：① **局部**（`odom→base_link`，由 `ekf_node` 融合轮速+IMU）；② **全局**（`map→odom`，第一圈 Cartographer、第二圈 AMCL）。两者职责不同，不能互相替代。
2. **`map→odom` 双圈互斥**：Cartographer 与 AMCL 都会发布该变换（§7.2），同存则 TF 树双父、定位失效。切换机制已在 **E-2** 收敛，本卡关注**第二圈 AMCL 的收敛与健康度**。
3. **IMU 精度有限**：HR-07 动态精度 2.5°，若把航向真值交给 IMU 积分，会随时间漂移。→ 航向必须由**激光扫描匹配**锚定（§3.1 已判定"主要依赖激光雷达做航向"）。
4. **里程计标定是鬼影头号原因**：`wheel.scale_factor` 未标定（HR-08、§3.1）会同时劣化**建图**（鬼影）与**定位**（沿行进方向的累积误差）。这是最便宜也最致命的一项。
5. **初始位姿与人工干预的边界**：赛前把车放到发车区标记点**不算**干预（计时前）；行进中切换时若靠人工点 RViz 的"2D Pose Estimate"就**构成干预**（PEN-11，+100 s）。→ 必须**自动**发布 `/initialpose`。
6. **QoS 陷阱**：`nav2_amcl` 若与 `/scan` 发布者 QoS 不兼容，会**静默收不到数据**（§8 头部）。这一项无法靠代码静态分析发现，只能**实机核对**。

## I.2 系统边界与职责划分

| 侧 | 承担 | 不承担 |
|---|---|---|
| **传感器端** | 激光雷达（HR-06）、IMU（HR-07）、编码器（HR-08，Modbus RTU） | — |
| **车端（本队实现）** | ① `ekf_node` 融合 `/odom` + `/imu/data` → `/odometry/filtered` 与 `odom→base_link`；② 第二圈 `nav2_amcl` 提供 `map→odom`；③ 切换时**自动**发布 `/initialpose`；④ 监控 `/amcl_pose` 与 `/particle_cloud` 健康度 | **不预置地图**、**不人工给初始位姿**（除赛前放置）、**不重复发布 `odom→base_link`** |

> **关键推论**：定位层的正确性由**一条唯一的 TF 链**保证：`map → odom → base_link → laser_link`。任何一段出现**双发布者**或**断链**，下游全部失效。因此本卡的第一设计原则是：**权属唯一、链不断裂**。

## I.3 接口契约（与 §8 逐字一致）

| 接口 | 类型 | 方向（本卡视角） | 用途 |
|---|---|---|---|
| `/odom` | `nav_msgs/msg/Odometry` | sub | `chassis_driver` 轮式里程计（50 Hz，§8.3） |
| `/imu/data` | `sensor_msgs/msg/Imu` | sub | IMU 输入（100 Hz，§8.1） |
| `/odometry/filtered` | `nav_msgs/msg/Odometry` | **pub** | `ekf_node` 融合输出 → SLAM / 控制（§8.2） |
| `odom → base_link`（TF） | `tf2_msgs/msg/TFMessage` | **pub** | `ekf_node` **唯一**发布（§7.1、§7.2） |
| `/scan` | `sensor_msgs/msg/LaserScan` | sub | `nav2_amcl` 定位观测（QoS 待核对） |
| `/map` | `nav_msgs/msg/OccupancyGrid` | sub | 第一圈地图，第二圈 AMCL 加载（§8.2） |
| `/initialpose` | `geometry_msgs/msg/PoseWithCovarianceStamped` | **pub** | `mission_manager` → `nav2_amcl`，切换时自动发布（§7.3 步骤④） |
| `/amcl_pose` | `geometry_msgs/msg/PoseWithCovarianceStamped` | sub | AMCL 位姿估计 → 状态机（§8.2） |
| `/particle_cloud` | `nav2_msgs/msg/ParticleCloud` | sub | 粒子云，用于**定位健康度**诊断（§8.2） |

## I.4 定位模式与 TF 权属（本卡以模式表替代「状态机」）

| 圈次 | 局部估计（`odom→base_link`） | 全局估计（`map→odom`） | 说明 |
|---|---|---|---|
| 第一圈 | `ekf_node` | `cartographer_node` | EKF 常开；建图栈提供全局位姿 |
| 切换 | `ekf_node` | **空窗期**（E-2 六步） | 速度不得为 0、不得提速（§7.3） |
| 第二圈 | `ekf_node` | `nav2_amcl`（节点名 `amcl`） | EKF 常开；AMCL 提供全局位姿 |

| 变换 | 发布者（全程） | 备注 |
|---|---|---|
| `odom → base_link` | **`ekf_node`（唯一）** | `chassis_driver` 只发 `/odom` **话题**，**不发 TF**（§7.1） |
| `base_link → laser_link` / `imu_link` / `camera_link` | `static_transform_publisher` | 外参待标定（§7.4） |

> **为何用模式表而非状态机**：`ekf_node` 与传感器输入是**始终在线**的连续估计；"圈次"只切换**全局估计的提供者**，不改变局部估计。真正的状态迁移发生在任务层（E-2 的 `ROLLING_SWITCH`），不在定位层。

## I.5 判据与算法

**5.1 EKF 融合配置（`ekf_node`，§9.8）**

- `odom0 = /odom`：取 `vx`、`vyaw`，**差分模式**（`odom0_differential = true`）抑制零偏累积；
- `imu0 = /imu/data`：取 `yaw`、`vyaw`，绝对航向；
- `two_d_mode = true`：平面运动，锁定 z / roll / pitch；
- `publish_tf = true`：**唯一**发布 `odom→base_link`。

**5.2 里程计标定（`wheel.scale_factor`）**

- 方法：车辆直行 **10 m**，与卷尺实测距离对比 → 修正比例因子；
- 依据：该因子是**建图鬼影的头号嫌疑**（HR-08、§3.1、§12.1 案例骨架）；
- 未标定前，**不得**信任 `/odometry/filtered` 的绝对尺度。

**5.3 AMCL 收敛判据（切换后）**

- `/amcl_pose` 的协方差在 `switch.timeout` 窗口内下降至稳定；
- `/particle_cloud` 粒子云收拢为单峰；
- 与当前 odom 位姿的初值偏差收敛（初值即切换瞬间的 odom 位姿，§7.3 步骤④）。

**5.4 定位健康度监控与恢复（在线，v2.1 由"提醒"升级为"可执行"）**

- **退化判据（量化）**：订阅 `/amcl_pose`，当位置协方差 > `amcl.covariance_position_threshold`(**0.5 m²**) **或** 航向协方差 > `amcl.covariance_yaw_threshold`(**0.3 rad²**)，且**持续** ≥ `amcl.degradation_duration`(**2.0 s**) → 判定定位退化；
- **辅助判据**：`/particle_cloud` 粒子云**多峰 / 大范围散布** → 与上条为 **"或"** 关系；
- **恢复动作（两级）**：
  1. **一级**：控制层立即降速至 ≤ `speed.curve`，同时 `mission_manager` 调用 Nav2 AMCL 标准服务 **`/reinitialize_global_localization`**（全局重定位），随后在行进中采集多样化扫描验证收敛；
  2. **二级**：若重定位后协方差仍不收敛，退化为**纯里程计 + 锥桶跟踪**兜底（不依赖 AMCL），保持低速沿锥桶中点行驶，避免 FAIL-01；
- **禁止**：退化与恢复全程**速度不得为 0**（否则触发 FAIL-03）；不得靠停车等人为干预（PEN-11）。
- ⚠ 本项**依赖 Q1~Q12 之外的一项实机标定**：协方差阈值 0.5 m² / 0.3 rad² 为**初始值**，须在真实场地上按"静止 + 直行 + 急弯"三组工况实测后修正。

**5.5 QoS 核对（实机必做）**

- 核对 `nav2_amcl` 订阅 `/scan` 的 QoS 与雷达发布者是否**兼容**；
- 传感器类话题统一 `SensorDataQoS`（best-effort），控制/状态类 `Reliable`（§8 头部）；
- **此项无法离线验证，须在实机上以 `ros2 topic info -v` 核对**。

## I.6 关键时序

| 阶段 | 事件 | 定位层动作 | 约束 |
|---|---|---|---|
| 上电 | 传感器就绪 | `ekf_node` 启动；第一圈 `cartographer_node` 提供 `map→odom` | 静态外参先标定（§7.4） |
| 第一圈 | 建图行驶 | EKF 融合；局部尺度由 `wheel.scale_factor` 决定 | 低速（提升扫描匹配质量） |
| 切换 | `ROLLING_SWITCH`（E-2） | 停用建图栈 → 启用 `nav2_amcl` → **自动**发布 `/initialpose` | 速度 ≥ `speed.rolling_switch`；窗口 < `switch.timeout` |
| 第二圈 | 导航行驶 | AMCL 收敛；`map→odom` 由 `amcl` 发布 | 收敛后逐步提速至 `speed.straight` |

> **初始位姿的取值**：切换时不使用发车区标记，而取**切换瞬间的 odom 位姿**作为 AMCL 初值（§7.3 步骤④、§9.3 v2.0 修订）。这既保证连续（车在动），又**避免人工干预**（PEN-11）。

## I.7 参数表（引自 §9.3 与 §9.8，逐字一致）

| 参数 | 值 | 出处 |
|---|---|---|
| `set_initial_pose` | `true` | §9.3 |
| `initial_pose.x` / `initial_pose.y` / `initial_pose.yaw` | **待标定** | §9.3 |
| `initial_pose.covariance` | 0.25 | §9.3 |
| `min_particles` / `max_particles` | 500 / 2000 | §9.3 |
| `laser_model_type` | `likelihood_field` | §9.3 |
| `update_min_d` / `update_min_a` | 0.10 m / 0.10 rad | §9.3 |
| `amcl.covariance_position_threshold` / `amcl.covariance_yaw_threshold` | 0.5 m² / 0.3 rad²（新增，已登记 §9.3） | §9.3 |
| `amcl.degradation_duration` | 2.0 s（新增，已登记 §9.3） | §9.3 |
| `frequency`（EKF） | 50.0 Hz | §9.8 |
| `two_d_mode` | `true` | §9.8 |
| `publish_tf` | `true` | §9.8 |
| `odom0` / `odom0_differential` | `/odom` / `true` | §9.8 |
| `imu0` / `imu0_differential` | `/imu/data` / `false` | §9.8 |
| `wheel.scale_factor` | **待标定** | §9.8 |
| `switch.timeout` | 20.0 s | §9.7 |
| `speed.rolling_switch` | 0.15–0.25 m/s | §9.5 |

## I.8 测试用例

| ID | 场景 | 步骤 | 通过判据 |
|---|---|---|---|
| TC-I01 | 里程计标定 | 直行 10 m，卷尺实测 | `wheel.scale_factor` 修正后误差 ≤ 1%（鬼影抑制） |
| TC-I02 | EKF 融合 | 同时喂 `/odom` 与 `/imu/data` | `/odometry/filtered` 平滑；`odom→base_link` **唯一发布者** |
| TC-I03 | 双发布者（反例） | 令 `chassis_driver` 也发 TF | 应被检出（TF 冲突告警）；配置层禁止 |
| TC-I04 | AMCL 收敛 | 切换后观察 `/amcl_pose` | `switch.timeout` 内协方差下降、粒子云单峰 |
| TC-I05 | 自动初始位姿 | 行进中切换 | `/initialpose` 自动发布 = 当时 odom 位姿；**无人工操作**（PEN-11 反例） |
| TC-I06 | 定位退化 | 注入雷达丢帧/遮挡 | `/particle_cloud` 告警 → 降速；不触发 FAIL-01 |
| TC-I07 | QoS 核对 | `ros2 topic info -v /scan` | `nav2_amcl` 与发布者 QoS **兼容**（实机必做） |
| TC-I08 | TF 断链（反例） | 断开 `odom→base_link` | 下游（感知/控制）检出并安全降速，不输出错误坐标 |

## I.9 失效模式（简版 FMEA）

| 失效 | 后果 | 严重度 | 检出 / 缓解 |
|---|---|---|---|
| `wheel.scale_factor` 未标定 | 建图鬼影 + 定位累积误差 | **极高** | 直行 10 m 标定（H.5.2、TC-I01） |
| `map→odom` 双发布者 | 定位完全失效 | **极高** | 权属唯一（E-2 串行切换，TC-I03） |
| AMCL 不收敛 / 退化 | 第二圈偏离 → FAIL-01 | 高 | 自动初始位姿 + **协方差阈值监控**（`amcl.covariance_*`）+ `LOCALIZATION_RECOVER` 调 `/reinitialize_global_localization`（I.5.4） |
| QoS 不兼容 | AMCL **静默无数据** | 高 | 实机核对（TC-I07、§8 QoS 基线表、附录 C.2.1） |
| **IMU 航向零偏注入 EKF** | AMCL 在**静止/低速**时因 IMU 偏航漂移而**持续旋转** → 定位发散 → FAIL-01 | 中 | ① 航向**以激光扫描匹配为准**，IMU 仅作短时角速度；② 若实测静止漂移显著，**改用 `imu0_differential: true`** 并复核 `imu0_remove_gravitational_acceleration`（§9.8 说明 + §A.2 IMU 静态漂移核对） |
| IMU 航向漂移 | 短时航向偏差 | 中 | 航向以激光为准；下调 `imu0_config` yaw 权重，或改用差分（见上行） |
| 定位跳变 | 路径/入库错位 | 中 | `/amcl_pose` 跳变检测 + 降速 + 触发 `LOCALIZATION_RECOVER` |
| 人工给初始位姿 | PEN-11 **+100 s** | 高 | 初始位姿全自动（TC-I05） |

## I.10 需求—罚则—设计映射

| 需求 / 罚则 | 本卡对应设计 |
|---|---|
| TR-03 第二圈基于地图导航并加速 | I.4 第二圈 AMCL + I.6 收敛后提速 |
| CR-06 逆时针、发车区 | I.5.3 初始位姿 + §9.3 标定 |
| FAIL-01 偏离轨道判负 | I.5.4 健康度监控 + 退化降速 |
| PEN-09 绕错锥桶 +5 s/个 | I.5.1 EKF 尺度正确（路径精度来源） |
| PEN-01 红绿灯车轮不在区域 +20 s | I.5.3 AMCL 收敛（区域判定精度来源） |
| PEN-04 入库失败 +20 s | I.5.3 收敛 + E-3 目标点标定 |
| PEN-11 人为干预 +100 s/次 | I.5.3/I.6 初始位姿全自动 |

---

# 附录 J　系统设计卡 E-6：控制（纯追踪 + 模糊 PID）

> **设计卡定位**：把散落在 §1.2（CR-06 逆时针）、§1.4（PEN-05/06）、§1.5（FAIL-01/03）、§5.1（L4 控制层）、§8.3（控制接口）、§9.5（纯追踪 + 模糊 PID）的"控制"内容，收敛为一张**可独立评审、可独立实现、可独立测试**的设计卡。控制是**精度链的末端执行**——定位再好，若控制抖动，一样会撞桶、压线、偏离。
> **一致性约束**：本卡内所有**话题名、参数名**与 §8 / §9 逐字一致。本卡**未新增接口**；**新增控制增益与限速参数已同步登记到 §9.5**（`pid.*`、`fuzzy.*`、`control.max_steering_rate`、`control.max_accel/decel`）。
> **结构适配说明**：控制是**连续两级流水线 + 分区限速**，无逐帧状态迁移，故本卡以 **J.4「控制流与限速档」替代其它设计卡的「状态机」**，其余小节结构一致。

## J.0 设计卡头部

| 项 | 内容 |
|---|---|
| 设计卡编号 | **E-6** |
| 子系统名 | 控制（纯追踪 `pure_pursuit` + 模糊 PID `fuzzy_pid`） |
| 覆盖需求 | TR-01（两圈稳定行驶）、CR-06（逆时针绕行）、CR-20（里程）；间接支撑所有"位置相关"罚则 |
| 覆盖罚则 | PEN-05（撞锥桶 +5 s/次）、PEN-06（折线压线 +10 s/圈）、FAIL-01（偏离轨道判负）、FAIL-03（静止 >10 s 判负）、PEN-11（人为干预 +100 s/次） |
| 覆盖规则条款 | 2.1、2.4.2(4)、2.4.2(6)、2.4.2(8)、2.4.2(11) |
| 关键前提 | ① **`/cmd_vel` 语义未定**——阿克曼平台上 `Twist.angular.z` 是**航向角速度**还是**舵机转角**？**依赖 Q6 答复**（§8.3）；② 舵机中位为 **1520 µs（非 1500）**（HR-04）；③ `wheelbase` / `max_steering_angle` **待实测**；④ 阿克曼为**非完整约束**，不能横移；⑤ 控制频率 **50 Hz**（`control.rate`）。 |
| 技术栈 | **ROS 2 Humble**（`pure_pursuit` + `fuzzy_pid` + `chassis_driver`，见附录 C.2） |
| 状态 | 设计已冻结；`pid.*`、`fuzzy.*`、`control.max_*`、`wheelbase` 为**待调项**；**Q6 答复前 `/cmd_vel` 语义与 PWM 映射不得编码** |
| 责任人 | 控制 + 底盘（转向标定为二者交汇点） |

## J.1 问题定义

1. **`/cmd_vel` 语义歧义（最高优先级风险）**：阿克曼底盘通常只接受**舵机 PWM**，而 `/cmd_vel` 是差速语义的 `Twist`。若把 `Twist.angular.z` 直接当舵机转角用，会造成**转向尺度错误**（增益失配）。→ 必须在 `chassis_driver` 内显式声明 `δ → PWM(µs)` 映射，**Q6 未答复前不得编码**。
2. **舵机中位不是 1500 µs**：HR-04 明确中位为 **1520 µs**。用 1500 会让车**持续偏转**，等价于给赛道注入一个恒定的横向偏差。
3. **非完整约束**：阿克曼车不能横移，横向纠偏靠**转向 + 前进**，因此对**前视距离**与**速度**极敏感。弯道高速 + 大前视 = 切内线撞桶。
4. **限速是安全的核心手段**：折线区、红绿灯接近区、切换期都靠"降速"换取余量（`speed.fold_zone`、`speed.traffic_approach`、`speed.rolling_switch`）。→ 控制必须实现**多档限速的"取最小"仲裁**。
5. **不能静止**：`FAIL-03 静止 >10 s 判负`。控制层在"等待/异常"时必须维持**蠕动下界**（`watchdog.recover_speed`），这是控制与看门狗的接口约束。

## J.2 系统边界与职责划分

| 侧 | 承担 | 不承担 |
|---|---|---|
| **上层（规划）** | 提供 `/race_line`（`nav_msgs/msg/Path`）与状态（`/race_state`） | 不直接控制底盘 |
| **车端（本队实现）** | ① `pure_pursuit` 把路径转为转向角与基础速度 → `/cmd_vel_raw`；② `fuzzy_pid` 整定增益 → `/cmd_vel` 与 `/steering_angle`；③ `chassis_driver` 把 `δ` 映射为舵机 PWM 并驱动底盘 | **不横移**、**不人工干预**（PEN-11）、**不违反限速档** |

> **关键推论**：控制层是**纯执行**——它不感知、不定位、不决策，只把"路径 + 速度档"变成"舵机 PWM + 电机转速"。因此它的正确性判据是**跟踪误差**与**约束满足**（限速、限幅、限速率）。

## J.3 接口契约（与 §8.3 逐字一致）

| 接口 | 类型 | 方向（本卡视角） | 用途 |
|---|---|---|---|
| `/race_line` | `nav_msgs/msg/Path` | sub | `path_generator` → `pure_pursuit`（1 Hz，§8.1） |
| `/cmd_vel_raw` | `geometry_msgs/msg/Twist` | **pub** | `pure_pursuit` → `fuzzy_pid`（50 Hz） |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | **pub** | `fuzzy_pid` → `chassis_driver`（50 Hz） |
| `/steering_angle` | `std_msgs/msg/Float32` | **pub** | `fuzzy_pid` → `chassis_driver`（若底盘用角度接口） |
| `/odometry/filtered` | `nav_msgs/msg/Odometry` | sub | 位姿反馈（横向误差计算，§8.2） |
| `/race_state` | `std_msgs/msg/String` | sub | 当前状态 → **限速档来源**（§8.4） |
| `/race_lap` | `std_msgs/msg/UInt8` | sub | 圈次（影响直道提速上限） |

> ⚠ **语义澄清（§8.3）**：`/cmd_vel` 的 `Twist.angular.z` 在阿克曼平台上可能是**航向角速度**或**舵机转角**；若底盘只接受 PWM，需在 `chassis_driver` 内做 `δ → PWM(µs)` 映射并在该表显式声明。**依赖 Q6。**

## J.4 控制流与限速档（本卡以流程 + 档位表替代「状态机」）

```
/race_line ──► [ pure_pursuit ] ──► /cmd_vel_raw ──► [ fuzzy_pid ] ──► /cmd_vel
  (路径)          δ = atan(2L·sinα/l_d)    (转向+基础速度)   (模糊整定增益)   /steering_angle
                    ▲                                                   │
                    └── /odometry/filtered（横向误差）                    ▼
                                                              [ chassis_driver ]
                                                               δ → PWM(µs)，中位 1520 µs
```

| 场景 | 限速参数 | 上限 | 依据 |
|---|---|---|---|
| 直道 | `speed.straight` | 1.8–2.0 m/s | 时间预算 |
| 弯道 | `speed.curve` | 0.9–1.2 m/s | 非完整约束稳定裕度 |
| 折线区 | `speed.fold_zone` | ≤ 0.8 m/s | PEN-06 防压线 |
| 红绿灯接近区 | `speed.traffic_approach` | ≤ 0.30 m/s | PEN-03 停车精度 |
| 绿灯后至停车区 | `speed.parking_approach` | ≤ 0.30 m/s | CR-22 加速距离短 |
| 模式切换期 | `speed.rolling_switch` | 0.15–0.25 m/s | FAIL-03 蠕动下界 |
| 异常恢复 | `watchdog.recover_speed` | 0.15 m/s | FAIL-03 不可为 0 |

> **仲裁规则**：同一时刻多个档位并存时，**取最小值**（安全优先）。例如"折线区 + 切换期"→ 取 `min(speed.fold_zone, speed.rolling_switch)`。

## J.5 判据与算法

**5.1 纯追踪（`pure_pursuit`，v2.1 改为转向角驱动的动态前视距离）**

- 转向角：`δ = arctan(2·L·sin(α) / L_d)`，其中 `L = wheelbase`（待实测）、`α` 为车体指向与目标点连线的夹角、`L_d` 为前视距离；
- **动态前视距离（主方案，`lookahead.mode = steering_adaptive`）**：`L_d = lookahead.min + (lookahead.max − lookahead.min)·(1 − lookahead.steering_gain·\|δ\|/max_steering_angle)`。直道 `\|δ\|≈0` → `L_d → lookahead.max`(0.60 m)；满舵 → `L_d → lookahead.min`(0.40 m)；
- **速度协同（`speed.steering_decay`）**：`v_target = speed.straight·(1 − speed.steering_decay·\|δ\|/max_steering_angle)`，直道提速、弯道自动降速；
- **备选方案（`lookahead.mode = curvature_adaptive`）**：`L_d = clamp(lookahead.min + lookahead.curvature_gain·κ, min, max)`；**仅在转向角不可得时**使用；
- 输出 `/cmd_vel_raw`（转向 + 基础速度）。

> **为什么主方案改用转向角驱动（v2.1）**：① `δ` 是控制层**自身的输出**，直接可得，无需像 `κ` 那样额外估计；② 速度与转向角同步衰减，可在弯道同时"看近一点、开慢一点"，缓解固定前视距离在弯道切内 / 切外的问题。
>
> **依据与边界（诚实标注）**：转向角驱动的动态前视距离 + 速度协同借鉴 **PP-DSC**（*Enhanced pure pursuit with dynamic steering control for autonomous mobile robots…*, **Scientific Reports, 2026-02, DOI `10.1038/s41598-026-38695-1`**；ROS 2 + 阿克曼平台实测）。该文在**开阔场地**（直线 / 圆 / 8 字，1.0–5.0 m/s）报告平均横向偏差 **0.05 / 0.07 / 0.08 m**，较固定前视距离纯追踪（**0.19 / 0.40 / 0.27 m**）改善 **68–82%**。
> ⚠ **同一文献同时指出**：在**转弯半径很小（5–9 m）**的场地里，标准固定前视距离纯追踪**反而优于**安全约束版 PP-DSC（+15.6%）。本赛道圆弧半径 **1.5 m**（CR-19）、局部收窄 **1.25 m**，属**小半径**工况。因此本设计**只借用其"按转向角同时调 `L_d` 与 `v`"的机制**，增益 `lookahead.steering_gain` / `speed.steering_decay` **必须实车标定**（以横向偏差 RMS 最小为目标、0.05 为步长扫描），**不直接照搬文献数值**；若实车发现小半径下变差，可回退至 `curvature_adaptive` 或固定 `L_d`。

**5.2 模糊 PID（`fuzzy_pid`）**

- 控制量：**横向跟踪误差** `e` 与误差变化率 `ė`；
- 基础增益 `pid.kp/ki/kd`（待调）+ **模糊自整定** `±Δkp/Δki/Δkd`（`fuzzy.enable = true`，规则输出按 `fuzzy.rule_scale` 缩放）；
- 积分限幅 `pid.integral_limit` 防饱和；
- 输出 `/cmd_vel`（或 `/steering_angle`）。

**5.3 转向限幅与速率限制**

- 幅度：`|δ| ≤ max_steering_angle`（硬件限位，待实测）；
- 速率：`|dδ/dt| ≤ control.max_steering_rate`（防舵机抖动/打滑）；
- 加速度：`|a| ≤ control.max_accel`、`|a| ≤ control.max_decel`。

**5.4 PWM 映射与中位（HR-04）**

- `δ = 0` ↔ PWM `= servo.neutral_pwm`（**1520 µs**，非 1500）；
- 满舵 ↔ `servo.neutral_pwm ± servo.max_pwm_delta`（待标定）；
- 映射在 `chassis_driver` 内实现，**Q6 答复后**确定是否走 `/cmd_vel` 或 `/steering_angle`。

**5.5 限速仲裁与静止保护**

- 多档限速取**最小值**（J.4）；
- 任何"等待"状态不得让速度降为 0（`FAIL-03`）；异常恢复期维持 `watchdog.recover_speed`(0.15 m/s)。

## J.6 关键时序

| 阶段 | 事件 | 控制层动作 | 约束 |
|---|---|---|---|
| 正常行驶 | 收到 `/race_line` | 纯追踪算 `δ` → 模糊 PID → `/cmd_vel` | 50 Hz；限速按档 |
| 进入受限区 | `/race_state` 变化 | 切换限速档（取最小） | 提前减速（`control.max_decel`） |
| 模式切换 | `ROLLING_SWITCH` | 维持蠕动 ≥ `speed.rolling_switch` | 不得为 0（FAIL-03） |
| 异常恢复 | 定位退化 / 偏离 | 降速 + 维持蠕动下界 | `watchdog.recover_speed` |

> **控制与看门狗的接口**：控制层**不自行决定停车**；"能不能停"由状态机（`watchdog.exempt_states`）决定。控制层的职责是**在允许的限速档内平滑执行**。

## J.7 参数表（引自 §9.5，逐字一致）

| 参数 | 值 | 出处 |
|---|---|---|
| `wheelbase` | **待实测** | §9.5 |
| `max_steering_angle` | **待实测** | §9.5 |
| `lookahead.mode` | `steering_adaptive`（备选 `curvature_adaptive`） | §9.5 |
| `lookahead.min` / `lookahead.max` | 0.40 m / 0.60 m | §9.5 |
| `lookahead.steering_gain` | 0.5（新增，已登记 §9.5） | §9.5 |
| `lookahead.curvature_gain` | **待调（备选）** | §9.5 |
| `control.rate` | 50 Hz | §9.5 |
| `servo.neutral_pwm` | **1520 µs** | §9.5（HR-04） |
| `servo.max_pwm_delta` | **待标定** | §9.5 |
| `pid.kp` / `pid.ki` / `pid.kd` | **待调**（新增，已登记 §9.5） | §9.5 |
| `pid.integral_limit` | **待调**（新增，已登记 §9.5） | §9.5 |
| `fuzzy.enable` | `true`（新增，已登记 §9.5） | §9.5 |
| `fuzzy.rule_scale` | **待调**（新增，已登记 §9.5） | §9.5 |
| `control.max_steering_rate` | **待标定**（新增，已登记 §9.5） | §9.5 |
| `control.max_accel` / `control.max_decel` | **待调**（新增，已登记 §9.5） | §9.5 |
| `speed.straight` / `speed.curve` | 1.8–2.0 / 0.9–1.2 m/s | §9.5 |
| `speed.steering_decay` | 0.7（新增，已登记 §9.5） | §9.5 |
| `speed.fold_zone` | ≤ 0.8 m/s | §9.5 |
| `speed.traffic_approach` / `speed.parking_approach` | ≤ 0.30 m/s | §9.5 |
| `speed.rolling_switch` | 0.15–0.25 m/s | §9.5 |
| `watchdog.recover_speed` | 0.15 m/s | §9.7 |

## J.8 测试用例

| ID | 场景 | 步骤 | 通过判据 |
|---|---|---|---|
| TC-J01 | `δ → PWM` 映射 | 给 `δ = 0` | PWM = **1520 µs**；直行不偏（HR-04） |
| TC-J02 | 直道跟踪 | 以 `speed.straight` 直行 | 横向误差在阈值内；不压线（PEN-06 反例） |
| TC-J03 | 弯道跟踪 | 以 `speed.curve` 过弯 | 不切内线撞桶（PEN-05 反例） |
| TC-J04 | 折线区 | 进入折线区 | 自动降至 `speed.fold_zone`(≤0.8)，不压线 |
| TC-J05 | 限速仲裁 | 折线区 + 切换期并存 | 取**最小值**；速度实际值合规 |
| TC-J06 | 静止保护（反例） | 令控制层持续输出 0 速度 | 被看门狗纠正；静止从未 > `static_timeout`(9 s)（FAIL-03 反例） |
| TC-J07 | 转向速率限幅 | 阶跃横向误差 | \|dδ/dt\| ≤ `control.max_steering_rate`；无舵机抖动 |
| TC-J08 | `/cmd_vel` 语义 | 按 Q6 答复的接口下发 | 底盘动作与指令一致（依赖 Q6） |

## J.9 失效模式（简版 FMEA）

| 失效 | 后果 | 严重度 | 检出 / 缓解 |
|---|---|---|---|
| `/cmd_vel` 语义误用 | 转向尺度错误、失控 | **极高** | Q6 澄清 + `chassis_driver` 内显式映射（TC-J08） |
| 舵机中位用 1500 µs | 持续偏转、压线 | **极高** | `servo.neutral_pwm = 1520 µs`（TC-J01） |
| 前视距离过大 | 弯道切内线撞桶 | 高 | 自适应前视 + 弯道限速 |
| 增益过大 | 蛇形、舵机抖动 | 高 | 模糊自整定 + 速率限幅（TC-J07） |
| 限速仲裁失效 | 超速 → 偏离 | 中 | 多档取最小（TC-J05） |
| 静止保护缺失 | FAIL-03 判负 | 高 | 蠕动下界 + 看门狗（TC-J06） |
| 积分饱和 | 稳态偏差/超调 | 中 | `pid.integral_limit` |

## J.10 需求—罚则—设计映射

| 需求 / 罚则 | 本卡对应设计 |
|---|---|
| TR-01 两圈稳定行驶 | J.4 控制流 + J.5 纯追踪/模糊 PID |
| CR-06 逆时针绕行 | J.5.1 前视自适应（持续左转跟踪） |
| PEN-05 撞锥桶 +5 s/次 | J.5.1 弯道保守前视 + J.4 弯道限速 |
| PEN-06 折线压线 +10 s/圈 | J.4 折线区限速 + J.5.2 横向误差抑制 |
| FAIL-01 偏离轨道判负 | J.5.1 跟踪误差 + J.5.3 限幅限速率 |
| FAIL-03 静止 >10 s 判负 | J.5.5 蠕动下界 + 限速仲裁 |
| PEN-11 人为干预 +100 s/次 | J.2 边界：纯自动执行，无人工修正 |

---

# 附录 K　系统设计卡 E-7：路径生成（`path_generator`）

> **设计卡定位**：把散落在 §1.2（CR-02/03/06/07/17/20）、§1.4（PEN-05/06/09）、§1.5（FAIL-01）、§5.1（L3 规划层）、§8.1/§8.3（`/cones`、`/boards`、`/race_line`）、§9.4（路径生成）的"路径生成"内容，收敛为一张**可独立评审、可独立实现、可独立测试**的设计卡。它是**感知（E-4）与控制（E-6）之间的桥梁**——上游是稀疏锥桶点，下游是连续可跟踪路径。
> **一致性约束**：本卡内所有**话题名、参数名**与 §8 / §9 逐字一致。本卡**未新增接口**；**新增路径参数已同步登记到 §9.4**（`path.safety_margin`、`path.min_turning_radius`）。
> **结构适配说明**：路径生成是**每次输入触发的流水线**（1 Hz 重规划），无逐帧状态，故本卡以 **K.4「处理流水线」替代其它设计卡的「状态机」**，其余小节结构一致。

## K.0 设计卡头部

| 项 | 内容 |
|---|---|
| 设计卡编号 | **E-7** |
| 子系统名 | 路径生成（`path_generator`：锥桶 → 可行驶走廊 → 三次样条路径） |
| 覆盖需求 | TR-03（第二圈基于地图导航）、CR-02（赛道宽 1.5 m）、CR-03（锥桶间距 < 1.5 m）、CR-06（逆时针）、CR-07（区域几何，胶带内侧）、CR-17（警示线）、CR-20（里程） |
| 覆盖罚则 | PEN-05（撞锥桶 +5 s/次）、PEN-06（折线压线 +10 s/圈）、PEN-09（绕错锥桶 +5 s/个）、FAIL-01（偏离轨道判负） |
| 覆盖规则条款 | 2.1、2.2.1(4)、2.4.2(4)、2.4.2(6) |
| 关键前提 | ① 输入是 **E-4 输出的稀疏锥桶点**，**不是**赛道先验中心线（禁用预置地图，ADR-005）；② 必须先构造**可行走廊**再平滑，否则样条会穿过锥桶；③ 折线区必须**收紧**（`fold_zone.margin`，PEN-06）；④ 阿克曼为非完整约束 → 路径**曲率有上限**（`path.min_turning_radius`）；⑤ 输出频率仅 **1 Hz**（§8.1），两次更新之间控制层沿用旧路径。 |
| 技术栈 | **ROS 2 Humble**（Voronoi 走廊 + 三次样条，见附录 C.2） |
| 状态 | 设计已冻结；`path.safety_margin` **待标定**、`path.min_turning_radius` **待计算**（依赖 §9.5 `wheelbase`/`max_steering_angle`） |
| 责任人 | 规划 + 感知（走廊质量取决于 E-4 的稳定度） |

## K.1 问题定义

1. **输入是"点"不是"线"**：E-4 给出的是锥桶**离散点**（`map` 系），赛道中心线**必须被构造**。→ 这是本子系统存在的根本理由。
2. **直接样条会穿锥桶**：若把锥桶点直接当控制点做样条，曲线会**穿过锥桶**（PEN-05）。→ 必须先构造**可行域**（内外边界 → 中轴走廊），再在走廊内平滑。
3. **折线区是压线高发区**：CR-17 的折线/警示线区域若沿中线跟踪过宽，会压线（PEN-06）。→ 需在折线区**收紧**横向量（`fold_zone.margin = 0.25 m`）。
4. **阿克曼曲率约束**：路径曲率 κ 必须满足 `κ ≤ tan(max_steering_angle) / wheelbase`，否则控制层**跟不上** → 偏离 → FAIL-01。→ 样条平滑后必须做**曲率裁剪**。
5. **两圈差异**：第一圈在建图中，`/cones` 不完整、地图在生长；第二圈基于完整地图，走廊更稳定、可提速（TR-03）。
6. **1 Hz 的稳定性要求**：路径每秒才更新一次，两次之间控制层用旧路径。→ 路径必须**平滑连续**，禁止在更新瞬间发生大跳变（否则控制抖动 → 撞桶）。

## K.2 系统边界与职责划分

| 侧 | 承担 | 不承担 |
|---|---|---|
| **上游（感知 E-4）** | 提供 `/cones`、`/boards`（`map` 系） | 不生成路径 |
| **车端（本队实现）** | ① 按颜色构造内/外边界；② Voronoi 中轴走廊；③ 走廊裁剪；④ 三次样条平滑；⑤ 曲率约束 + 重采样 → `/race_line` | **不控制底盘**、**不预置赛道**、**不输出穿过锥桶的路径** |

> **关键推论**：路径生成本质是一次**几何优化**——在"锥桶约束 + 阿克曼曲率约束 + 折线区收紧"三者之间求一条**平滑可行**的曲线。它的正确性判据是**不碰锥桶、不压线、可被跟踪**。

## K.3 接口契约（与 §8 逐字一致）

| 接口 | 类型 | 方向（本卡视角） | 用途 |
|---|---|---|---|
| `/cones` | `autocar_msgs/msg/ConeArray` | **sub** | 锥桶点（`map` 系），含 `color`/`confidence`（§8.1，E-4 输出） |
| `/boards` | `autocar_msgs/msg/BoardArray` | **sub** | 挡板（并入可行边界，§8.1、E-4 输出） |
| `/map` | `nav_msgs/msg/OccupancyGrid` | **sub** | 第一圈地图（可选：与锥桶联合约束） |
| `/odometry/filtered` | `nav_msgs/msg/Odometry` | **sub** | 当前位姿（起终点、走廊裁剪参考，§8.2） |
| `/race_state` | `std_msgs/msg/String` | **sub** | 状态（折线区/受限区切换，§8.4） |
| `/race_lap` | `std_msgs/msg/UInt8` | **sub** | 圈次（第一/二圈路径策略差异） |
| `/race_line` | `nav_msgs/msg/Path` | **pub** | 输出路径 → `pure_pursuit`（**1 Hz**，§8.1/§8.3） |

## K.4 处理流水线（本卡以流水线替代「状态机」）

```
/cones + /boards ──► [ 内外边界构造 ] ──► [ Voronoi 中轴走廊 ] ──► [ 走廊裁剪 ]
  (E-4, map 系)        按 Cone.color 分内/外        voronoi.prune_distance    按赛道宽 1.5m(CR-02)
                            │                                                    │
                            ▼                                                    ▼
                     [ 三次样条平滑 ] ◄──────────────────────────────────── [ 折线区收紧 ]
                       spline.smoothing                                   fold_zone.margin(0.25m)
                            │
                            ▼
                    [ 曲率约束 + 重采样 ] ──► /race_line (1 Hz)
                     min_turning_radius / point_spacing / safety_margin
```

| 阶段 | 输入 | 处理 | 参数 |
|---|---|---|---|
| ① 边界构造 | `/cones`、`/boards` | 按 `color` 分**内圈（红）/ 外圈（蓝）**；挡板并入边界 | ADR-005 径向先验 |
| ② 中轴走廊 | 内外边界 | Voronoi 骨架 → 取中轴作为**候选路径** | `voronoi.prune_distance` |
| ③ 走廊裁剪 | 中轴 + 赛道宽 | 剔除过窄/过宽段（CR-02：宽 1.5 m；CR-03：间距 < 1.5 m） | — |
| ④ 折线区收紧 | 走廊 + `/race_state` | 折线区横向收紧，靠中线跟踪（防压线） | `fold_zone.margin` |
| ⑤ 样条平滑 | 有效中轴点 | 三次样条拟合 | `spline.smoothing` |
| ⑥ 曲率约束 + 重采样 | 平滑曲线 | 曲率裁剪至阿克曼可达；等间距重采样 | `path.min_turning_radius`、`path.point_spacing`、`path.min_segment_length`、`path.safety_margin` |

> **为何用流水线而非状态机**：本节点每收到一次输入即重算一次路径（1 Hz），无跨帧状态迁移；唯一"时序性"是更新频率与地图加载时机。

## K.5 判据与算法

**5.1 内外边界构造（ADR-005 径向先验）**

- 依据 E-4 给出的 `Cone.color`（内圈红 / 外圈蓝）分出两条边界；
- 挡板（`/boards`）作为**连续反射面**并入边界（不可当单点）；
- 若颜色为 `unknown`，退回按**径向距离**临时归类（不外扩边界）。

**5.2 Voronoi 中轴走廊**

- 对边界点构造 Voronoi 图，取其**中轴**作为候选路径；
- 按 `voronoi.prune_distance`(1.0 m) 修剪**悬挂分支**（避免路径指向边界外的死枝）。

**5.3 走廊裁剪（CR-02/CR-03，v2.1 补量化判据）**

- 对每条 Voronoi 中轴走廊段计算其**局部宽度** `w`（等于该处到左右边界最近点的距离之和）；
- **保留判据**：`path.corridor_width_min`(**1.0 m**) ≤ `w` ≤ `path.corridor_width_max`(**1.6 m**)；
- **丢弃并桥接**：`w` 超出窗口的走廊段整段丢弃，缺口两端用三次样条**桥接**（不是直接相连，需满足 §5.5 曲率约束）；
- **几何依据**：赛道宽 1.5 m（CR-02）、局部收窄 1.25 m（CR-19）、锥桶间距 < 1.5 m（CR-03）、车宽 0.56 m（HR-01）；
- **两类典型反例**：
  1. `w < 1.0 m`：多为**漏检锥桶**导致左右边界贴得过近 → 该段不可信，丢弃；
  2. `w > 1.6 m`：多为**锥桶漏检造成的缺口**，使 Voronoi 中轴"抄近路"穿过缺口、把内外圈并在了一起 → 必须丢弃；若直接使用会导致路径穿越锥桶（PEN-05）或偏离赛道（FAIL-01）。

**5.4 折线区收紧（PEN-06 的防线）**

- `/race_state` 标识折线区时，横向收紧 `fold_zone.margin`(0.25 m)，**贴近中线**；
- 目的：在折线/警示线区域预留**压线余量**（CR-17）。

**5.5 曲率约束与重采样**

- 曲率裁剪：`κ ≤ 1 / path.min_turning_radius`，其中 `path.min_turning_radius = wheelbase / tan(max_steering_angle)`；
- 横向安全裕度：路径与锥桶保持 ≥ `path.safety_margin`（防撞桶 PEN-05）；
- 重采样：按 `path.point_spacing`(0.05 m) 等间距输出；丢弃 < `path.min_segment_length`(0.30 m) 的碎段。

## K.6 关键时序

| 阶段 | 事件 | 路径层动作 | 约束 |
|---|---|---|---|
| 第一圈 | 建图中 | 用不完整的 `/cones` 生成路径 | 保守（外扩 `safety_margin`），不追速度 |
| 稳态 | 每次 `/cones` 更新（10 Hz）后重规划 | 输出 `/race_line`（1 Hz） | 两次更新间控制沿用旧路径 |
| 第二圈 | 地图完整 | 走廊更稳定，可支撑提速至 `speed.straight` | TR-03 |
| 进入折线区 | `/race_state` 变化 | 收紧（`fold_zone.margin`） | PEN-06 反例 |

> **1 Hz 的平滑性要求**：路径在更新瞬间的**首点/末点**必须与旧路径**连续**，避免控制层出现阶跃（否则转向抖动 → 撞桶）。→ 样条需做**边界连续**处理。

## K.7 参数表（引自 §9.4，逐字一致）

| 参数 | 值 | 出处 |
|---|---|---|
| `spline.smoothing` | 0.5 | §9.4 |
| `voronoi.prune_distance` | 1.0 m | §9.4 |
| `path.point_spacing` | 0.05 m | §9.4 |
| `path.min_segment_length` | 0.30 m | §9.4 |
| `fold_zone.margin` | 0.25 m | §9.4 |
| `path.safety_margin` | **待标定**（新增，已登记 §9.4） | §9.4 |
| `path.min_turning_radius` | **待计算**（新增，已登记 §9.4） | §9.4 |
| `path.corridor_width_min` / `path.corridor_width_max` | 1.0 m / 1.6 m（新增，已登记 §9.4） | §9.4 |
| `wheelbase` | **待实测**（用于 `min_turning_radius`） | §9.5 |
| `max_steering_angle` | **待实测**（用于 `min_turning_radius`） | §9.5 |

## K.8 测试用例

| ID | 场景 | 步骤 | 通过判据 |
|---|---|---|---|
| TC-K01 | 基本路径生成 | 给定一圈锥桶点 | 生成连续路径；不穿过任何锥桶（PEN-05 反例） |
| TC-K02 | 内外圈分类 | 混合红/蓝锥桶 | 内边界=红、外边界=蓝；路径居中（ADR-005） |
| TC-K03 | 折线区收紧 | 进入折线区 | 路径贴近中线；横向量 ≤ 收紧值（PEN-06 反例） |
| TC-K04 | 曲率约束 | 生成含急弯的走廊 | 路径曲率 ≤ `1/min_turning_radius`（阿克曼可达） |
| TC-K05 | 挡板并入 | 走廊含挡板 | 挡板作为边界，路径不穿越（CR-13） |
| TC-K06 | 更新连续性 | 连续两次 1 Hz 更新 | 路径首尾连续，无阶跃（控制不抖动） |
| TC-K07 | 稀疏点鲁棒 | 人为丢弃部分锥桶 | 路径仍可行；不外扩至锥桶外（FAIL-01 反例） |
| TC-K08 | 碎段过滤 | 输入含孤立点 | 丢弃 < `path.min_segment_length`(0.30 m) 的碎段 |
| TC-K09 | 走廊宽度异常注入（v2.1 新增） | 在内外圈之间人为插入一个"假锥桶"，使某段走廊宽度降至 0.8 m；另一组使其抬升至 1.8 m | 两种情形该段均被**丢弃**；缺口由样条**桥接**绕行；路径**不穿越**假锥桶（PEN-05 反例）、不越过内外圈（FAIL-01 反例） |

## K.9 失效模式（简版 FMEA）

| 失效 | 后果 | 严重度 | 检出 / 缓解 |
|---|---|---|---|
| 路径穿越锥桶 | PEN-05 **+5 s/次**，严重时 FAIL-01 | **极高** | 先走廊后样条 + `path.safety_margin`（TC-K01） |
| 折线区未收紧 | PEN-06 **+10 s/圈** | 高 | `fold_zone.margin` 收紧（TC-K03） |
| 曲率超阿克曼可达 | 跟踪不上 → 偏离 FAIL-01 | 高 | `path.min_turning_radius` 裁剪（TC-K04） |
| 内外圈误分类 | 路径外扩/内缩 → 绕错锥桶 | 高 | 依赖 E-4 径向先验 + `unknown` 兜底（TC-K02） |
| 路径更新阶跃 | 转向抖动 → 撞桶 | 中 | 样条边界连续（TC-K06） |
| 悬挂死枝 | 路径指向边界外 | 中 | `voronoi.prune_distance` 修剪 |
| 挡板当单点 | 路径穿越挡板 | 中 | 挡板并入边界（TC-K05） |

## K.10 需求—罚则—设计映射

| 需求 / 罚则 | 本卡对应设计 |
|---|---|
| TR-03 第二圈基于地图导航 | K.6 第二圈走廊稳定 → 支撑提速 |
| CR-02/CR-03 赛道宽 1.5 m、间距 < 1.5 m | K.5.3 走廊裁剪（宽度/间距约束） |
| CR-06 逆时针 | K.5.1 内外边界按颜色（内红/外蓝，ADR-005） |
| CR-07 区域几何 | K.5.5 `path.safety_margin`（相对胶带内侧留量） |
| CR-17 警示线 | K.5.4 折线区收紧 |
| PEN-05 撞锥桶 +5 s/次 | K.5.5 安全裕度 + K.5.2 中轴走廊 |
| PEN-06 折线压线 +10 s/圈 | K.5.4 `fold_zone.margin` 收紧 |
| PEN-09 绕错锥桶 +5 s/个 | K.5.1 颜色分类正确（路径居中） |
| FAIL-01 偏离轨道判负 | K.5.5 曲率约束 + K.6 更新连续性 |

---

# 附录 L　学习类方法（ML/DL）评估与增强路线图（非基线，仅评估）

> **本附录回答两个问题**：
> ① 规则要求的"**自主建图**"在本设计中是否已闭环？
> ② 能否用**模型训练（ML/DL）**让激光雷达"智能识别"并覆盖**全部比赛项目**（感知 / 建图 / 定位 / 规划 / 控制）？
>
> **硬边界（必读）**：本附录是**评估与路线图**，**不修改 §9 任何参数**、**不新增接口或话题**、**不解除 Q4~Q7 的阻塞编码标记**。凡本附录讨论的学习类模块，若其落地依赖 Q4/Q5/Q6/Q7 的答复（执行机构接口、入库条目），则同样**"不得据此编码"**（见附录 D.5）。学习类方法在本项目中一律定位为**增强（enhancement）而非替换（replacement）**。

## L.1 「自主建图」要求覆盖自检（回答第①问）

| 规则依据 | 要求 | v2.1 覆盖位置 | 状态 |
|---|---|---|---|
| **TR-02**（规则 2.1） | 第一圈探索未知环境、自主识别场地、对场地做**二维建图** | §5.1 Cartographer 配置 / §5.2 行驶策略 / §5.3 保存与加载；§7.2 状态机 `MAPPING` 态 | 已覆盖 |
| **TR-08** | 挡板是**防提前建图**机制，禁止使用预置先验地图走捷径 | ADR-005；§5.3「地图必须含挡板信息」 | 已覆盖 |
| **PEN-07**（+5 s） | 建图不清晰（挡板缺失 / 轮廓模糊） | §5.2 建图质量检查；附录 F「建图成功判据」 | 已覆盖 |
| **PEN-08**（+50 s） | **完全未建图** | §5.3 保存成功 → 加载地图 的状态迁移；附录 F `TC-F` 组用例 | 已覆盖 |
| **SEC**（取消成绩） | 禁止用 `ros2 bag`（预录数据）回放替代实时感知 | §3.3 合规红线；附录 C.2 | 已覆盖 |
| 圈次切换铁律 | 切换**不得停车**，否则任意位置静止 >10 s 判负（FAIL-03） | §7.2 rolling switch；**附录 F 设计卡 E-2**（六步时序） | 已覆盖 |
| SLAM 选型风险 | 小尺度锥桶场景 Cartographer 地图边界**碎片化** | **ADR-001 风险标注** + **ADR-001-B**（SLAM Toolbox 备选 + 3 条量化切换触发条件） | 已覆盖（含 Plan B） |

**结论**：**"自主建图"在需求编号、架构分层、状态机、设计卡、ADR、罚则映射六个层面均已闭环**，不存在设计缺口。未闭环的是**实机项**而非设计项：

- `lap.lap_length`（单圈里程，决定 §5.3 圈次判定阈值）；
- `wheel.scale_factor`（里程计标定，建图"鬼影"头号嫌疑）；
- `submaps.num_range_data`（实机下调取值）；
- ADR-001-B 的 3 条切换触发条件的实机阈值。

**本附录提出的两项工程加固**（属加固、非新需求）：

1. **建图质量量化自检**：第一圈结束前自动判据化——锥桶聚类数、挡板段数、地图有效栅格占比是否达阈值。不达标**不直接进入第二圈**，而是走 ADR-001-B 的备选栈重跑（避免带着 PEN-07/PEN-08 风险进第二圈）。
2. **建图证据链留存**：第一圈的 `/map`、`/tf`、`/scan` 与关键日志**带时间戳留存**，用于佐证 TR-08 的"真实执行探索-建图"。**该留存仅用于赛后举证，绝不可作为实时感知输入**（否则触碰 SEC 禁预录回放红线）。

> **注**：QUT 的 "Racing With ROS 2"（arXiv:2311.14276）示范了用现成 ROS 2 栈（`slam_toolbox` + `robot_localization` EKF + DCNN 锥桶检测）替代自研组件；本项目**不采用**该路线，理由是 TR-08 的合规举证与 ADR-001 的实测结论（详见 §6 ADR）。

## L.2 学习类方法总览：模块地图（回答第②问的"能不能全做"）

**答案：不是"全部替换"，而是"分模块、分风险、门控式增强"。** 下表是全部可行模块的一览：

| 环节 | 现基线（v2.1 唯一来源） | 学习类候选 | 代表工作（含实证数据） | 适配判断 |
|---|---|---|---|---|
| 锥桶**颜色**分类 | 径向位置推断（附录 H / E-4） | LiDAR **强度**模式浅层 1D CNN | Morgado 等：用强度均值 + 点数，**浅层 CNN 可 CPU 实时** | **低风险增量**，可做 |
| 锥桶**检测/分割** | DBSCAN 聚类（附录 H） | PointNet++ / PTv3 语义分割；PointPillars / CenterPoint 3D 检测 | TinyCenterSpeed（arXiv:2504.08655，ETH）：推理 **7.88 ms**、检测提升 **61.38%**、CPU 占用降 **8.3×**（需外部 TPU）；Jetson Orin 上 25 FPS | **高算力**，昇腾 310(ARM64) 受限，暂缓 |
| 红绿灯 / 停车标志 | HSV + 圆度 + 3 帧一致性（附录 E / E-1、E-3） | YOLOv8 检测；双分类头（状态 + 图标） | YOLOv8：Gazebo-ROS2 下 mAP **98.5%**（IJACSA 2023）；TLD-READY（KIT）双头；Autoware 7 层流水线（地图先验 ROI + 时序滤波 + 冗余） | **最值得做的学习模块**，见 L.3.2 |
| 建图（SLAM） | Cartographer 2D（附录 F / E-2） | 无成熟学习类替换；可借**图优化** GraphSLAM | KA-RaceIng（arXiv:2210.10933，FS 冠军栈）：GraphSLAM 建图 **RMSE < 15 cm**，35 m 识别锥桶 | **不替换**；GraphSLAM 仅作 §9.2 上游参考 |
| 定位 | EKF 里程计融合 + 第二圈 AMCL（附录 I / E-5） | 学习类里程计（需大量标定数据） | IIT Bombay（arXiv:2408.06113）：DBSCAN + 高度阈值，**EKF-SLAM** | **不替换**，AMCL 已够用 |
| 路径生成 | Voronoi 中轴 + 参数样条（附录 K / E-7） | 端到端策略直接出控制（跳过显式路径） | End2Race（arXiv:2509.16894）：LiDAR → 转向/速度，推理 **<1 ms** | **保留显式路径**（可解释/可调试），学习类只做**前视/代价的残差调整** |
| 横向 + 速度控制 | PP-DSC + 模糊 PID（附录 J / E-6） | ① **残差 RL**（在经典控制器上学小残差）；② 端到端模仿学习 BC+PPO；③ 域随机化 RL | ① On-Board RL（RLJ 2025，ETH）：SAC 残差，**20 min 训练圈速 −11.5%**；② End2Race：BC 预训练 + PPO 微调，仿真**安全率 94.2%**；③ arXiv:2504.02420：域随机化**零样本 sim2real**，17 m 赛道超人类 **0.286 s**；Dream4Speed（CMU）：DreamerV3 模型式 RL 零样本 sim2real | **残差 RL 优先级最高**，见 L.3.5 |
| 仿真 / sim2real | 无（v2.1 未固化仿真环境） | F1TENTH Gym + ROS 2 bridge；CARLA + ROS 2 | `ros2 launch f1tenth_gym_ros gym_bridge_launch.py`（ROS 2 Humble 兼容）；FS-AI 2025 用 CARLA+ROS 2（arXiv:2511.11310） | **建议引入**，见 L.3.6 |
| 边缘部署 | 原生 ROS 2 节点（CPU） | ONNX / TensorRT / NPU 编译 | FALO（arXiv:2506.04499）：Jetson Orin / Qualcomm NPU 上 ONNX 部署；EMC2（MoE 边缘协同） | 昇腾 310 需 **CANN/ATC** 转换，见 L.3.7 |

> **一句话结论**：学习类方法**不可能也不应该**"一键接管所有比赛项目"。真正低风险、高收益的落点是**红绿灯视觉检测（L.3.2）**与**控制层残差 RL（L.3.5）**；锥桶的 LiDAR 强度颜色分类（L.3.1）是低成本增量；其余环节（SLAM / 定位 / 路径）**保持经典栈**。

## L.3 逐模块深度评估

### L.3.1 感知—锥桶检测与颜色分类

**现有基线**：DBSCAN 聚类（`eps≈0.15, min_samples=3`，距离自适应）+ **径向位置推断颜色**（附录 H / E-4）。该法零训练、可解释、CPU 实时。

**学习类候选与实证**：

- **LiDAR 强度分类（推荐）**：Morgado 等研究表明，锥桶颜色可由激光**回波强度**区分；**浅层 1D CNN** 仅用「强度均值 + 点数」即可在 **CPU 上实时**分类。对昇腾 310（ARM64、无独立 GPU）**算力友好**。
- **点云语义分割（暂缓）**：PointNet / PointNet++ / PTv3（Point Transformer v3），数据集 SemanticKITTI / nuScenes。精度高但**参数与延迟大**，需要 GPU/NPU；本项目 12–20 TOPS 的昇腾 310 上**不达实时余量**。
- **3D 检测（暂缓）**：PointPillars / CenterPoint 在 Jetson Orin 上约 25 FPS；**TinyCenterSpeed** 用外部 TPU 把推理压到 7.88 ms，但引入了额外硬件（TR 之外的 BOM 变更）。
- **视觉锥桶检测（可选，作为冗余）**：TU/e 的 "Perception-Enabled Pure Pursuit" 用 **YOLO 锥桶检测 + 自适应前视距离**，并与 DDPG 对比——结论是**检测增强的纯追踪在可解释性上更优**。BIT "Smart Shark II"（arXiv:1909.00119，FSAC 2018 冠军）用 **LiDAR + 视觉冗余** + CNN 检测红蓝锥分布，是"冗余而非替换"的经典范式。

**本项目建议**：**保留 DBSCAN 为主检测器**，把「LiDAR 强度浅层 CNN 颜色分类」作为**第二表决源**（与径向先验"三取二"），冲突时输出 `unknown` 交由 E-4 兜底逻辑处理。**不引入 3D 检测 / 语义分割。**

### L.3.2 感知—红绿灯与停车标志（最值得做的学习模块）

**现有基线**：HSV 阈值 + 圆度筛选 + **连续 3 帧一致性**（附录 E / E-1）；停车标志用蓝色底 + 四边形轮廓 + 透视校正 + "P" 字形检测（附录 G / E-3）。

**学习类候选与实证**：

- **YOLOv8 红绿灯检测**：在 Gazebo-ROS2 仿真数据上 **mAP 98.5%**（IJACSA 2023），训练数据易采集（本车摄像头录制）。
- **双分类头设计**：KIT 的 **TLD-READY** 把「灯态分类」与「图标分类」解耦，避免单一 softmax 混淆。
- **Autoware 的 7 层红绿灯流水线**：**地图先验裁剪 ROI → 分类 → 时序滤波 → 冗余安全架构**。其中"地图先验 ROI"对本项目有直接价值——第二圈已有第一圈地图，可**用地图先验把红绿灯 ROI 限定在已知区域**，大幅降低误检。

**为什么它风险最低**：
1. **失败可回退**：HSV 通道可与之并行运行做"三取二"；
2. **罚则方向明确**：绿灯未亮通行 +30 s（PEN-03），误检的代价远大于漏检代价→ 阈值应向"宁可多等"倾斜；
3. **数据可采集**：无需真值标注硬件，本车摄像头自录即可。

**与现有设计的整合点**：附录 E.1 的「3 帧一致性」判据可升级为「YOLO 置信度 ≥ 阈值 **且** HSV 通道一致 **且** 连续 3 帧同态」的**三取二**表决，**参数取值仍以 §9.6 为准**。

### L.3.3 建图与定位（建议不替换）

**现有基线**：Cartographer 2D（第一圈）+ AMCL（第二圈）+ EKF 融合（附录 F、I）。

- **KA-RaceIng**（FS 无人驾驶冠军栈，arXiv:2210.10933）的 **GraphSLAM** 达到建图 **RMSE < 15 cm**，并且其**Delaunay 三角剖分 + 参数三次样条**路径生成思路与附录 K 的 Voronoi 中轴法**互为对偶**，可作为 E-7 的**独立交叉验证**（不作为替换）。
- **"Racing With ROS 2"（QUT）** 用 `slam_toolbox` + `robot_localization` EKF + DCNN 替代自研——与本项目 ADR-001-B（SLAM Toolbox 备选）方向一致，但其 DCNN 部分**不引入**。
- **学习类里程计**需要大量成对标注数据（LiDAR/IMU → 真值轨迹），在没有高精度真值定位源的场地上**不具备可行性**。

**本项目建议**：**保持 Cartographer + AMCL + EKF 不变**；仅把 KA-RaceIng 的 GraphSLAM 与样条方案作为 §9.2 / 附录 K 的**参考文献**登记。

### L.3.4 规划—路径生成（保留显式路径）

**现有基线**：锥桶点 → 内外边界 → Voronoi 中轴走廊 → 折线区收紧 → 三次样条 + 曲率约束（附录 K / E-7）。

- **端到端策略**（如 End2Race）会**绕过显式路径**，直接输出控制量。其优点是推理 < 1 ms；缺点是**不可解释、不可调试**——一旦撞桶（PEN-05 +5 s/次）无法定位是"感知错"还是"规划错"。
- KA-RaceIng 的 **Delaunay + 参数样条** 与 **2D 欧氏聚类提取锥心（~32 ms）** 是**非学习类**的工业级做法，与本项目路线同族。

**本项目建议**：**保留显式路径生成**（可解释性对竞赛至关重要）。学习类在此环节的唯一入口是**为前视距离/路径代价提供一个学习到的修正项**（与 L.3.5 的残差 RL 合流），而**不接管路径本身**。

### L.3.5 控制—横向与速度（残差 RL 优先级最高）

**现有基线**：PP-DSC 转向角驱动前视 + 模糊 PID（附录 J / E-6，参数见 §9.5）。

三种学习类路线的风险/收益排序：

| 路线 | 机理 | 实证 | 风险 |
|---|---|---|---|
| **① 残差 RL（首选）** | 在**经典控制器输出之上**学一个小残差 `Δδ, Δv`；经典保证安全下界，学习只做优化 | On-Board RL（RLJ 2025，ETH）：SAC 残差，**20 min 训练圈速 −11.5%**；异步训练可在线 | **低**——经典控制器始终在环，残差可置零即回退 |
| **② 端到端模仿学习（第二阶段）** | LiDAR 扫描 → 直接出转向/速度；BC 预训练 + PPO 微调 | End2Race（arXiv:2509.16894）：GRU + BC/PPO，推理 **<1 ms**，F1Tenth 仿真**安全率 94.2%** | **中高**——训练依赖专家轨迹；实车数据分布偏移会失效 |
| **③ 域随机化 RL（探索性）** | 在大规模随机化仿真中训练，零样本迁移 | arXiv:2504.02420：仿真训练**零样本部署** F1TENTH，17 m 赛道**超人类 0.286 s**；Dream4Speed：DreamerV3 模型式 RL 零样本 sim2real | **高**——sim2real gap；本赛道为阿克曼 + 舵机，与 F1TENTH 差速平台动力学差异显著 |

**本项目建议**：**先做路线①**。具体形态是"PP-DSC 输出 + 学习残差"，且残差**设硬限幅**（如 |Δδ| ≤ 满舵的 20%）；训练与部署初期采用**影子模式**（见 L.5）——模型在线推理但**输出不接入 `/cmd_vel`**，只做记录与离线比对，确认优于基线后再切换。

### L.3.6 仿真与 sim2real（v2.1 未固化的缺口）

- **F1TENTH Gym + ROS 2 bridge**：`ros2 launch f1tenth_gym_ros gym_bridge_launch.py`，**ROS 2 Humble 兼容**，是上述 ①②③ 路线的共同训练场。注意其默认平台为**差速**，本车是**阿克曼 + HR-04 舵机**（中位 1520 µs），需在仿真的执行器模型中显式加入**舵机死区、延迟与中位偏置**，否则 sim2real 会系统性偏移。
- **CARLA + ROS 2**：FS-AI 2025 使用（arXiv:2511.11310），场景更真实但**资源开销大**，昇腾 310 上不适合作为主训练环境（可离线在 x86 工作站上训练）。
- "Drive Fast, Learn Faster" 提出的**执行器动力学建模 + 域随机化**是缩小 sim2real gap 的关键，建议在仿真中列为必做项。

> **合规提醒**：仿真仅用于**训练与回归测试**。**任何仿真数据/预录数据都不得在比赛计时期间作为实时感知输入**（SEC，取消成绩）。

### L.3.7 边缘部署（昇腾 310 / ARM64）

| 项 | 说明 |
|---|---|
| 训练侧 | 在 x86 + GPU 工作站上用 PyTorch 训练（昇腾 310 不适合训练） |
| 转换链路 | PyTorch → **ONNX** → 昇腾 **CANN / ATC** 转 `.om`。**必须先验证算子支持**（浅层 1D CNN / YOLO 骨干通常支持良好；PointNet 系算子需逐一验证） |
| 参考 | FALO（arXiv:2506.04499）展示了 Jetson Orin / Qualcomm NPU 上的 ONNX 部署可行性；EMC2 展示 MoE 边缘-云协同。均非昇腾，仅作**工程方法学**参考 |
| 实时预算 | 感知节点应控制在 **≤ 30 ms**（保证 10–12 Hz 雷达周期内不丢帧）；控制残差推理应 **≤ 5 ms** |

## L.4 推荐路线图（三阶段，门控式）

> **总原则**：**经典栈永远是可靠下界**；每一阶段都以"不降低现有完赛概率"为进入前提。

| 阶段 | 内容 | 进入判据 | 退出判据（回退条件） |
|---|---|---|---|
| **阶段 0（当前基线）** | 维持 v2.1 经典栈，完成全部实机项回填与两次完整实跑 | — | 两圈稳定完赛、无 PEN-07/08 |
| **阶段 1（低风险增量）** | ① 红绿灯 YOLO + HSV 三取二（L.3.2）；② LiDAR 强度 1D CNN 颜色分类（L.3.1）；③ 搭建 F1TENTH Gym 仿真（L.3.6） | 阶段 0 退出；训练数据已采集（本车实时传感器，非计时时段） | 任一模块在影子模式下误检率高于基线 → 立即回退该模块 |
| **阶段 2（中等风险增强）** | 控制层**残差 RL**（L.3.5 路线①）+ 学习类前视修正（L.3.4） | 阶段 1 影子模式连续 N 次实跑优于基线 | 残差导致撞桶或压线增多 → 残差置零回退纯 PP-DSC |
| **阶段 3（探索，不接管）** | 端到端 BC+PPO（L.3.5 路线②）作为**对比/记录**运行 | 阶段 2 稳定；已具备专家轨迹数据集 | 永不直接接管 `/cmd_vel` |

## L.5 数据与合规红线（与 SEC / TR-08 / PEN-11 的边界）

| 项 | 规则 | 本设计约束 |
|---|---|---|
| **训练数据来源** | SEC：禁止预录回放替代实时感知（取消成绩） | 训练数据必须来自**本车实时传感器**，且采集须在**非计时练习时段**；数据集标注来源与时间戳 |
| **模型推理 vs 回放** | 同上 | 计时期内模型**只做在线推理**（输入来自实时 `/scan`、`/camera`），**绝不读取预录文件** |
| **影子模式** | PEN-11：人工干预 +100 s | 学习类模块初期**只记录、不接入 `/cmd_vel`**；切换为接管必须有判据与一键回退 |
| **防提前建图** | TR-08 | 训练不得使用赛前赛道先验地图；地图严格来自第一圈实时 SLAM |
| **可追溯** | TR-08 举证 | 模型文件入版本库，附训练数据来源、时间戳、评测记录 |

## L.6 风险登记（简版 FMEA）

| 失效 | 后果 | 严重度 | 缓解 |
|---|---|---|---|
| 学习类模型误检红绿灯 | 绿灯未亮通行 **+30 s**（PEN-03） | 极高 | 三取二表决；阈值向"多等"倾斜；HSV 兜底 |
| 学习类模型介入控制后撞桶 | **+5 s/次**（PEN-05），严重偏离 **FAIL-01** | 极高 | 残差硬限幅；影子模式；一键置零回退 |
| 仿真相差导致 sim2real 失效 | 实车行为与仿真不符，偏离轨道 FAIL-01 | 高 | 仿真中显式建模舵机死区/延迟/中位（1520 µs） |
| 模型推理超时丢帧 | 感知延迟 → 控制滞后 → 压线 | 中 | 推理预算 ≤ 30 ms；超时退回经典通道 |
| 昇腾算子不支持 | 模型无法部署 | 中 | 先验证 ONNX→ATC 算子支持；浅层 CNN 优先 |
| 误用预录数据 | **取消成绩** | 极高 | L.5 全部约束；代码审查 + 上线检查清单 |

## L.7 未来评估项登记（不进入本期基线）

| ID | 项 | 触发条件 | 影响章节 |
|---|---|---|---|
| FE-01 | 红绿灯 YOLO + HSV 三取二表决 | 阶段 1 进入 | §9.6 定稿后登记新参数 |
| FE-02 | LiDAR 强度浅层 1D CNN 颜色分类 | 阶段 1 进入 | 附录 H / E-4 接口不变，仅增表决源 |
| FE-03 | F1TENTH Gym 仿真环境固化 | 阶段 1 进入 | 新增仿真章节（本轮不写） |
| FE-04 | 控制层残差 RL（含硬限幅与影子模式） | 阶段 2 进入 | 附录 J / E-6 输出不变，仅加残差项 |
| FE-05 | 昇腾 CANN/ATC 转换链验证 | 任一模型部署前 | 部署章节 |
| FE-06 | 建图质量量化自检阈值 | 阶段 0 实测后 | §5.2、附录 F |
| FE-07 | GraphSLAM / Delaunay+样条作为 E-7 交叉验证 | 可选 | 附录 K、ADR-001 |

## L.8 对 §9 参数基线的影响

**无。** 本附录**未新增、未修改任何 §9 参数**。学习类模块一旦进入落地阶段（FE-01/FE-02/FE-04），其超参（置信度阈值、残差限幅比、推理预算）应**按 §9 的既有流程登记为新增参数**，并遵守"§9 为参数唯一来源"这条跨文档约定（见「阅读指引」）。

## L.9 参考资料（本轮全网检索代表文献）

| 方向 | 文献 |
|---|---|
| 端到端模仿学习竞速 | End2Race, arXiv:2509.16894 |
| 轻量 3D 检测 + 边缘 TPU | TinyCenterSpeed, arXiv:2504.08655（IEEE IV 2025） |
| 残差 RL 增强经典控制器 | Drive Fast / Learn Faster（On-Board RL），RLJ 2025, ETH Zurich |
| 模型式 RL 零样本 sim2real | Dream4Speed, CMU（DreamerV3） |
| 域随机化 RL 竞速 | arXiv:2504.02420 |
| FS 无人驾驶冠军软件栈 | KA-RaceIng, arXiv:2210.10933 |
| 现成 ROS 2 导航栈（QUT） | Racing With ROS 2, arXiv:2311.14276 |
| FSAC 2018 冠军（LiDAR+视觉冗余） | BIT Smart Shark II, arXiv:1909.00119 |
| DBSCAN + EKF-SLAM（IIT Bombay） | arXiv:2408.06113 |
| 感知增强的纯追踪（TU/e） | Perception-Enabled Pure Pursuit |
| LiDAR 强度 CNN 锥桶分类 | Morgado 等 |
| 红绿灯 YOLOv8 / 双头检测 | IJACSA 2023；TLD-READY（KIT）；Autoware 红绿灯流水线 |
| 边缘部署 | FALO, arXiv:2506.04499；EMC2 |
| 仿真平台 | F1TENTH Gym（`f1tenth_gym_ros`）；CARLA+ROS 2, arXiv:2511.11310（FS-AI 2025） |

## L.10 学习类模型数据采集清单（阶段 1 前置，非基线）

> **用途**：本清单是 L.4「阶段 1」的入场券。**数据未按本清单采集完成，不得进入阶段 1**。所有话题名取自 §8 已登记集合，**本清单不新增任何话题或接口**。

### L.10.0 合规门（采集前必须同时满足，缺一不可）

| # | 条件 | 依据 |
|---|---|---|
| G1 | 仅在**非计时练习 / 调试时段**采集，不得在官方计时时段录制 | 附录 L.5；SEC |
| G2 | 数据源必须是**本车实时传感器**；采集过程中**不得把任何历史 bag 作为感知输入** | SEC（禁预录回放 → 取消成绩） |
| G3 | **训练不得使用赛前赛道先验地图**；地图类数据只能来自第一圈实时 SLAM | TR-08（防提前建图） |
| G4 | 采集时车辆由**人遥控/人工建图模式**驱动即可，但该时段**不计入任何成绩**，且采集结束后必须清除遥控痕迹、恢复程序自动模式 | PEN-11（人工干预 +100 s） |
| G5 | 录制的 bag 仅用于**离线训练与回归**；计时期内一律**只做在线推理** | 附录 L.5 |

### L.10.1 数据集清单总表

| ID | 目标模型 | 传感器 / 话题 | 数据类型 | 目标样本量 | 必须覆盖的场景 |
|---|---|---|---|---|---|
| **DS-01** | 红绿灯检测（YOLO） | `/camera/image_raw` + `/camera/camera_info` | 图像帧（含时间戳） | ≥ 3000 帧 | 红灯 / 绿灯 / 熄灭三态；远（>15 m）·中·近三档；正午强光 / 阴影 / 逆光 / 黄昏；单帧与连续帧（供 3 帧一致性回归） |
| **DS-02** | 红绿灯 HSV 对照集 | 同 DS-01（**同源复用，不单独采**） | 图像帧 | ≥ 500 帧 | 作为 L.3.2「三取二表决」的**回归基准**，用于量化 YOLO 相对 HSV 的增益 |
| **DS-03** | 停车标志识别 | `/camera/image_raw` | 图像帧 | ≥ 800 帧 | 斜视 / 正视；远 / 近；部分遮挡；反射干扰；标志牌规格 29.7×42 cm（CR-10/11） |
| **DS-04** | 锥桶 LiDAR 强度分类 | `/scan`（**必须含 `intensity` 字段**）、`/scan_filtered`、`/tf`、`/tf_static` | LaserScan 帧 | ≥ 2000 帧 | 红 / 蓝两色；2 m / 5 m / 10 m 三档距离；单桶 / 成对 / 成排；含挡板混入的近场帧 |
| **DS-05** | 控制残差 RL 专家轨迹 | `/scan` + `/odometry/filtered` + `/imu/data` + `/cmd_vel` + `/cmd_vel_raw` + `/steering_angle` + `/race_state` + `/race_lap` + `/tf` | 同步时序轨迹 | ≥ 30 圈（**仅经典栈已验证稳定时采集**） | 直道 / 弯道 / 折线区（CR-17）/ 红绿灯停车起步 / 入库停车 |
| **DS-06** | 负样本集（降误检） | `/camera/image_raw` + `/scan` | 图像帧 + 点云帧 | ≥ 1000 帧 | 无锥桶空旷段 / 纯挡板段 / 行人 / 其他车辆 / 地面反光 / 警示线（CR-17）/ 阴影与光斑 |

> **为什么必须有 DS-06**：竞赛罚则对**误检**的惩罚远大于漏检（绿灯未亮通行 +30 s；撞桶 +5 s/次）。负样本直接决定误报率，不可省略。

### L.10.2 采集命令（示例）

| 数据集 | 命令 |
|---|---|
| DS-01 / DS-02 / DS-03 | `ros2 bag record /camera/image_raw /camera/camera_info -o ds_camera_$(date +%Y%m%d_%H%M)` |
| DS-04 | `ros2 bag record /scan /scan_filtered /tf /tf_static -o ds_lidar_$(date +%Y%m%d_%H%M)` |
| DS-05 | `ros2 bag record /scan /odometry/filtered /imu/data /cmd_vel /cmd_vel_raw /steering_angle /race_state /race_lap /tf -o ds_traj_$(date +%Y%m%d_%H%M)` |
| DS-06 | 同 DS-01 + DS-04 合并录制 |

**关键注意（否则会静默丢数据）**：

- `ros2 bag record` 的**默认订阅 QoS 未必匹配**本项目的传感器话题（§8 基线：`/scan`、`/camera/image_raw` 等为 `SensorDataQoS` = Best-Effort/depth 5）。若录制时**零帧落盘**，用 `--qos-profile-overrides-path` 显式指定 QoS（best-effort, depth 5）后重录。
- DS-05 的 `/steering_angle` 是**阿克曼舵机转角**通道；若底盘仅接受 PWM，则该话题当前不存在，须先解决 §2.3 **Q6**（阻塞编码项）后才可采集——**Q6 未答复前，DS-05 采集暂缓**。**核查流程、三情形判定、以及"只录原始话题、δ 离线推导"的原则见附录 D.6**（该节同时说明：DS-01/02/03/04/06 与阶段 1 训练**不受 Q6 阻塞**，不必一起等）。
- 录制前先 `ros2 topic hz <topic>` 确认频率符合 §8 表（`/scan` 10–12 Hz、`/imu/data` 100 Hz、`/camera/image_raw` 30 Hz）。

### L.10.3 标注规范

| 数据集 | 标注形式 | 类别定义 |
|---|---|---|
| DS-01 / DS-03 | **YOLO txt**：每行 `cls cx cy w h`（归一化 0–1），与 `image` 同名 `.txt` | `red_light` / `green_light` / `off_light` / `p_sign` |
| DS-04 | **按聚类簇标注**：用 §9.4 的 DBSCAN 参数产出簇后，给每个簇打颜色标签 | `red_cone` / `blue_cone` / `unknown`（挡板等非锥桶归入 `unknown`，不参与颜色分类损失） |
| DS-05 | **无需人工标注**：监督目标即**当时实际执行的 `δ` 与 `v`**（取自 `/steering_angle` 与 `/cmd_vel`），并须与 `/scan` 逐帧对齐 | —（回归任务） |
| DS-06 | 负样本：图像标为**无目标**（空 label 文件），点云帧标为**无锥桶簇** | —（空标签必须显式建文件，防止被当成"漏标"丢弃） |

**标注质量要求**：抽 **10%** 做双人交叉复核；边界框与锥桶簇的交并比争议 > 10% 的样本直接废弃。**不做"凑数式"加标**——错误标签对误检的伤害大于样本量不足。

### L.10.4 数据划分与防泄漏（易被忽视，但决定评估是否可信）

| 规则 | 说明 |
|---|---|
| 划分比例 | train / val / test = **7 / 2 / 1** |
| **按场景分组划分** | **同一次采集（同一时段、同一光照、同一场地）不得跨 split**。必须整段进 train 或整段进 val/test，否则同一光照下的相邻帧会"泄漏"到验证集，导致指标虚高 |
| 测试集独立性 | test 集至少包含 **1 个训练中完全未出现的场景**（如"黄昏逆光"或"成排锥桶"） |
| 禁止跨集增广 | 数据增广（翻转、亮度扰动）只能在**训练集内部**做，不得先生成再随机划分 |

### L.10.5 存放与命名

```
~/datasets/
├── ds01_traffic_light/
│   ├── images/            # 000001.jpg ...
│   ├── labels/            # 000001.txt ...
│   └── manifest.yaml      # 见下表
├── ds02_tl_hsv_baseline/
├── ds03_parking_sign/
├── ds04_cone_intensity/
│   ├── scans/             # 逐帧 .npy 或 PCD
│   └── labels/
├── ds05_residual_traj/    # 按圈次建子目录 lap1/ lap2/
└── ds06_negative/
```

- 原始 bag 命名：`YYYYMMDD_HHMM_<场景>_<序号>`（如 `20261015_1430_noon_01`）
- 每个数据集**必须**带 `manifest.yaml`，字段：`采集日期 / 时段 / 光照 / 天气 / 场地 / ROS 版本 / 话题列表 / 各话题 QoS / 时长 / 帧数 / 标注人 / 复核人 / train-val-test 划分文件`。

### L.10.6 检查表

**采集前**

- [ ] 已确认处于**非计时时段**（L.10.0 G1）
- [ ] 已确认**不读取任何历史 bag** 作为感知输入（G2）
- [ ] 已确认赛场未使用先验地图（G3）
- [ ] `ros2 topic hz` 各话题频率符合 §8 表
- [ ] **QoS 覆盖参数已就位**，试录 30 s 验证帧数非零
- [ ] 摄像**内参标定文件**已加载（150° 视场，§9.6 `camera.calibration_file`）
- [ ] 若采 DS-05：已按 **附录 D.6** 现场核查并确定接口情形（A / B / C），且已按"**只录原始话题**"原则配置录制列表

**采集中**

- [ ] 每次采集记录时段 / 光照 / 天气 / 场地到 manifest
- [ ] 同一场景连续采集，不中途换场地（保证 L.10.4 的分组划分成立）
- [ ] 覆盖该数据集的**全部必采场景**（见 L.10.1 末列）
- [ ] 采集时长与帧数与目标量对齐，实时核对

**采集后**

- [ ] 落盘帧数 > 0，且与 manifest 记录一致
- [ ] 完成标注 + 10% 双人复核
- [ ] 按 L.10.4 完成**按场景分组**的 train/val/test 划分并落划分文件
- [ ] 数据与模型文件入版本库（或数据盘 + 索引），附 manifest
- [ ] **确认 bag 未被接入任何实时通路**（复核节点订阅列表）

---

**文档结束**

> **待办**：§2.2 现场复核表、§7.4 静态外参表、§8.3 执行机构接口（依赖 §2.3 Q6）、§9 中标注"待标定/待实测"的参数项，需在实车调试阶段逐项回填。其中 **`lap.lap_length`（单圈里程）为 v2.1 的关键待测量**，它决定了 §4.2 的全部时间预算与 §5.3 的圈次判定阈值。
>
> **第三轮复核遗留的确认项**：§2.3 的 Q1 ~ Q12（其中 Q9 ~ Q12 为 v2.1 新增），共 12 项需向组委会/供应商书面确认。**可直接发出的邮件见附录 D**（A 组发组委会 / B 组发供应商，含正文模板与回填动作）。**在 Q5、Q6 答复前，第三部分涉及执行机构的条目不得据此编码。**
>
> **ROS 2 合规提醒**：本车为 ROS 2 小车，全文涉及的 `rosbag` 等 ROS 1 用词均须按附录 C.3 映射为 ROS 2 形态；赛前按附录 A.1.2 完成"无 ROS 1"举证。
>
> **设计卡**：附录 E~K 的 **E-1（红绿灯与光电触发）**、**E-2（第一圈建图与行进中模式切换）**、**E-3（第二圈停车标志识别与入库停车）**、**E-4（锥桶与挡板感知）**、**E-5（定位：EKF + AMCL）**、**E-6（控制：纯追踪 + 模糊 PID）**、**E-7（路径生成）** 已按同一结构完成（头部 → 边界 → 接口 → 流水线/模式/状态机 → 判据 → 时序 → 参数 → 测试 → FMEA → 映射），其中 E-4/E-5/E-6/E-7 依节点特性分别以「处理流水线」「定位模式表」「控制流与限速档」「处理流水线」替代「状态机」。**七张卡已覆盖感知（E-4）、定位（E-5）、建图切换（E-2）、路径（E-7）、控制（E-6）、红绿灯（E-1）、入库（E-3）全链路**；尚余**任务调度状态机**（§5.3，可复用 E-1~E-7 的接口与判据）。