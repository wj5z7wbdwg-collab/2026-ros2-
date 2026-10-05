# 室外 ROS 无人车赛（职教组）· 参赛工程

北京小豚 HW-Autocar（昇腾 310 / ARM64 / Ubuntu 22.04 / ROS 2 Humble）
两圈制：第一圈建图 → 滚动切换 → 第二圈 AMCL 导航 → 红绿灯 → P 牌入库 → 完赛。

## 目录结构

```
├── README.md                  ← 本文件（导航）
├── MODULES.md                 ← 模块边界与独立开发指南（改算法先读这个）
├── 赛项一-室外ROS无人车赛（职教组）比赛规则（初稿）(46).pdf   # 规则原文
├── 室外ROS无人车赛_设计技术文档_v2.1.md                      # 参数唯一来源（§9 基线）
├── ros小车.txt / 实车联调前核对清单.txt                       # 引用版（与 v2.1 冲突以 v2.1 为准）
└── autocar_ws/                ← ROS 2 工作区（colcon 工作区根）
    ├── src/                   ← 8 个 ROS 包 = 8 个模块（见 MODULES.md）
    ├── tests/                 ← mock 层功能测试（276 项断言，Windows 本机可跑）
    ├── scripts/               ← 工具链（deploy/run_race/race_test/calib*/offline_check 等）
    ├── maps/                  ← 建图产物落位（上车生成，不入库）
    ├── RUNBOOK.md             ← 操作手册（§10A 算法备案表）
    ├── 技术手册.md             ← DR-02 交付物（20 分）
    └── 上车首跑操作单.md / 代码部署上车手册.md / 标定验证总结报告.md /
        实车联调测试用例清单.md / 算法思路与替代方案审查.md   ← 联调与交付支撑文档
```

## 常用入口

| 想做什么 | 命令（车机） |
|---|---|
| 编译+自检 | `bash autocar_ws/scripts/deploy.sh all` |
| 拉起全栈联调 | `bash autocar_ws/scripts/run_race.sh up [simulate:=true]` |
| 跑测试用例 | `bash autocar_ws/scripts/race_test.sh a` → `b` → `c1..c7` → `report` |
| 标定门禁 | `python3 autocar_ws/scripts/calib_check.py`（FATAL=0 才可发车） |
| 本机离线预演 | `python3 autocar_ws/scripts/offline_check.py`（Windows 可跑） |
| 单板块功能测试 | `python3 autocar_ws/tests/test_perception.py`（等 5 个，见 MODULES.md） |

## 模块独立开发约定（摘要，全文见 MODULES.md）

1. **一个板块 = 一个包**：算法文件、配置 YAML、mock 测试三者同包/同名对应，改哪个板块只动哪个目录。
2. **接口冻结**：话题名与消息定义（autocar_msgs）是模块间契约，独立优化算法时**不得改接口**，只换实现。
3. **改算法必过三件套**：该板块 mock 测试 → offline_check.py → race_test.sh 对应组用例。
4. **参数唯一来源**：v2.1 §9；代码只读 YAML，禁止硬编码新参数。
5. **比赛日冻结**：只改 YAML 数值，不改 Python。
