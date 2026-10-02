# 家庭健康助理 · APP（uni-app x）

家庭健康助理平台（原 Sisensing 血糖数据管理平台）的移动端，基于 **uni-app x** 开发，一套代码出 **Android / iOS**。

- 平台代码（后端 + 前端）：https://github.com/SimonChenX/Home_Health_Aide （私有：`main` = 后端，`frontend` = 前端）
- APP 代码：本仓库（公开）
- 打包：本仓库 GitHub Actions（`.github/workflows/build.yml`）

## 1. 这个 APP 做什么

| 能力 | 说明 |
| --- | --- |
| 读发射器/探头数据 | 通过 BLE 直连发射器，读实时值与 flash 历史（需协议，见 `docs/ble-protocol.md`） |
| 展示**矫正后**血糖 | 矫正算法只在 API 端（原生 E115M 库），APP 拉「校准曲线」后**本地插值**实时显示 |
| 省电 / 少交互 | 读数本地缓存 + 批量上报；曲线一天一拉；不做定时轮询 |
| 与平台一致 | 同一账号、同一数据、同一套设计令牌（`src/theme.uts` 与平台 `Front/src/styles/index.scss` 对齐） |
| 家庭 | 与平台「家庭管理」一致：查看家庭成员、家庭内共享血糖/设备数据 |

## 2. 关键设计：矫正值怎么在手机上「实时」又「省电」（需求 2.4）

平台侧算法是 x86-64 原生库（`native/libalg_native.so`），**不可能搬到手机上**，也不该搬（唯一事实来源）。
所以：

```
发射器 --BLE--> APP 原始值(raw) --插值--> 显示矫正值(mmol/L)     ← 离线，0 次网络
                     │
                     └──(WiFi/充电/前台) 批量上报 --> API --原生算法回放--> 平台 cal_x10
API --GET /api/calibration/curve--> APP 本地映射表（几十个点，<2KB，一天一拉）
```

- **映射表**由 API 用该探头**自己的真实历史**回放生成（与平台「重新校准」同一口径），
  经中位数去噪 + 等张回归（PAVA）保证单调，返回 `points: [[raw, value], ...]`（value 与平台 `cal_x10` 同口径：mmol/L ×10）。
- APP 端 `src/core/cal-curve.uts` 按 raw 线性插值 → 实时显示，**每条读数 0 次网络请求**。
- 精度实测（真实探头 + 真实历史，见平台 `Scripts/api_cal_curve_check.py`）：
  平均偏差 **0.32 mmol/L**、最大 1.7 mmol/L；独立复核平均 0.21 mmol/L。
  接口自带 `accuracy` 字段把该偏差如实返回，APP 在数值旁展示「本地估算」标记与 ± 区间。
- 上报策略（`src/core/sync.uts`）：仅在 **WiFi + 充电 / 前台手动 / 满 500 条** 时批量提交，
  避免 288 条/天的逐条请求。

## 3. 目录

```
src/
  api/            接口封装（与平台 REST 一致，Bearer token）
  core/           校准曲线拉取与本地插值、缓存、同步策略
  device/         BLE 适配层（协议常量 + 连接/读历史/实时订阅）
  pages/          login / home / history / medical / family / mine
  components/     卡片、曲线、空态
  theme.uts       设计令牌（与平台一致）
docs/
  app-architecture.md   架构与省电设计（含数据流图）
  ble-protocol.md       ⚠ 待补：发射器 BLE 协议清单（当前用模拟实现占位）
```

## 4. 打包（GitHub Actions）

`.github/workflows/build.yml` 用 HBuilderX CLI 云打包（`cli pack`），可出 Android APK / iOS IPA。

需要在仓库 Secrets 里配置（目前**未配置，故流水线跑不出产物**）：

| Secret | 用途 |
| --- | --- |
| `DCLOUD_USER` / `DCLOUD_PASS` | HBuilderX / DCloud 账号（云打包必须） |
| `ANDROID_KEYSTORE_BASE64` / `ANDROID_KEYSTORE_PASS` / `ANDROID_KEY_ALIAS` / `ANDROID_KEY_PASS` | Android 签名证书 |
| `IOS_CERT_BASE64` / `IOS_CERT_PASS` / `IOS_PROVISION_BASE64` | iOS 证书 + 描述文件 |

`src/manifest.json` 里的 `appid` 目前是占位（`__UNI_APPID__`），需在 DCloud 开发者中心申请后替换。

## 5. 本地开发

用 HBuilderX 打开本目录 → 运行到手机/模拟器（uni-app x 仅 HBuilderX 支持编译与真机运行）。
命令行只用于 CI 打包。

## 6. 状态

- [x] 项目骨架、设计令牌、接口封装、校准曲线本地插值、同步策略
- [x] GitHub Actions 打包流水线（缺凭据，未产出过安装包）
- [ ] **BLE 协议**：等平台方提供发射器 BLE 服务/特征 UUID 与 flash 历史读取命令（`docs/ble-protocol.md`）
- [ ] 打包凭据（DCloud 账号 + 证书）与 `appid`
- [ ] 真机联调与 UI 走查
