# WIFIRefSensSearch

> 基于 **R&S CMW500** 综测仪的 WiFi 接收灵敏度自动寻值上位机

一款 Windows 桌面工具，用于自动测量 WiFi 设备（DUT）的**接收灵敏度**——即在给定丢包率（PER）门限下，DUT 仍能正常收包的**最低下行射频电平（dBm）**。软件通过 SCPI（TCP）控制 CMW500 发包并测量 PER，同时通过串口（AT 指令）驱动 DUT 关联与断连恢复，按内置算法逐步逼近灵敏度点，并支持一键导出 Excel 报表。

![platform](https://img.shields.io/badge/platform-Windows-blue)
![framework](https://img.shields.io/badge/.NET-10.0--windows-512BD4)
![ui](https://img.shields.io/badge/UI-WPF-2B579A)
![instrument](https://img.shields.io/badge/instrument-R%26S%20CMW500-green)

---

## 目录

- [功能特性](#功能特性)
- [工作原理](#工作原理)
  - [整体流程](#整体流程)
  - [灵敏度寻值算法](#灵敏度寻值算法)
  - [关联 / 断连恢复](#关联--断连恢复)
- [测试用例（Test Profiles）](#测试用例test-profiles)
- [硬件与连接](#硬件与连接)
- [环境要求](#环境要求)
- [构建与运行](#构建与运行)
- [发布（单文件 EXE）](#发布单文件-exe)
- [使用说明](#使用说明)
- [导出报表](#导出报表)
- [配置持久化](#配置持久化)
- [项目结构](#项目结构)
- [关键实现说明](#关键实现说明)

---

## 功能特性

- **全自动灵敏度寻值**：粗调（Probe，±0.5 dB / 200 包）+ 细调（Seek，+0.1 dB / 1000 包）两阶段逼近，输出满足 PER 门限的最低电平。
- **多制式覆盖**：11b / 11g(OFDM) / 11g/n / 11a / 11a/n，涵盖 20M / 40M 带宽、2.4G / 5G 频段，每用例 3 个信道，共 **7 个 Profile × 3 信道 = 21 个测试点**。
- **CMW500 SCPI 控制**：原始 TCP（默认 5025 端口）直连，无需安装 VISA 运行库；支持 `TCPIP0::<ip>::INSTR` 或 `ip:port` 两种地址写法。
- **DUT 串口驱动**：通过 AT 指令（`AT+WLCONN` 关联、`AT+WLPS` 关省电）控制 DUT，实时解析 `CONNECTION STATUS`。
- **强关联 / 断连恢复**：cell OFF/ON 循环、安全参数重写、`AT+WLCONN` 重发、串口固件卡死自动重连，最大限度保证测试连续性。
- **随时暂停 / 恢复 / 停止**，逐行可勾选执行，起始电平（Start）可在界面直接编辑并持久化。
- **一键导出 Excel**（.xlsx），通道结果自动转为纯数值。
- **深色仪表风 UI**，实时日志窗口。
- **GitHub Actions 自动发布**：打 `v*` / `V*` tag 即自动构建自包含单文件 EXE 并发布到 Releases。

---

## 工作原理

### 整体流程

```
连接 CMW500 (SCPI/TCP)  ──┐
连接 DUT (串口/AT)      ──┤
                          ▼
      勾选用例 + 设置 Cable Loss / RX Expected PEP
                          ▼
            对每个 Profile 的每个信道：
   ┌──────────────────────────────────────────────┐
   │ 1. 电平回安全位 (-45 dBm)                       │
   │ 2. 结构化配置：cell OFF → 写 制式/带宽/信道/安全 │
   │    → cell ON（等待 beacon 稳定）                │
   │ 3. 配置 PER：FDEF / PayloadSize / PN9 数据      │
   │ 4. 强关联（轮询 PSWitched=ASS）                 │
   │ 5. 电平梯度下降（Ramp-down，防止链路突断）      │
   │ 6. 灵敏度寻值算法（Probe + Seek）               │
   │ 7. 记录结果，回安全位，进入下一信道             │
   └──────────────────────────────────────────────┘
                          ▼
                    导出 Excel
```

### 灵敏度寻值算法

灵敏度定义为**使 PER ≤ 门限的最低电平**。算法从 Profile 的探测起始电平（可被 UI 覆盖）开始：

1. **初测**：在起始电平发 200 包，测一次 PER，据此分支：
   - `PER < 门限` → **Probe-1**：每次 **−0.5 dB**（200 包），直到 `PER ≥ 门限`；
   - `PER > 20%`（灾难性丢包）→ **Probe-2**：每次 **+0.5 dB**（200 包），直到 `PER ≤ 20%`；
   - `门限 ≤ PER ≤ 20%` → 跳过粗调，直接进入 Seek。
2. **Seek（细调，统一）**：每次 **+0.1 dB**（1000 包），直到 `PER ≤ 门限`。**第一个满足条件的电平即为灵敏度结果**。

> 门限（PerLimitPercent）随 Profile 不同：11b 为 8%，其余为 10%。全局 CMW `PER:LIMit` 固定设为 20%（仅为防止 CMW 提前终止测量），真正的判定门限由软件算法执行。

各阶段包数与步进（见 `Services/SensitivityTester.cs`）：

| 阶段 | 步进 | 包数 | 结束条件 |
|------|------|------|----------|
| Probe-1（粗调下降） | −0.5 dB | 200 | PER ≥ 门限 |
| Probe-2（粗调恢复） | +0.5 dB | 200 | PER ≤ 20% |
| Seek（细调） | +0.1 dB | 1000 | PER ≤ 门限 |

### 关联 / 断连恢复

WiFi 信令测试中 DUT 极易掉线，软件内置多层恢复机制（详见 `docs/` 下的两份 Word 文档）：

- **关联轮询**：进入即 cell OFF/ON + `AT+WLCONN`，之后每 10 s 检测一次 `PSWitched`；失败则重发关联命令，每累计 4 次（第 4/8/12/16 次）重启一次 cell；最多 16 次 + 末次仍未关联则跳过该信道。
- **安全参数重写**：cell OFF 会把 SSID/安全重置为 DIS，每次重启前按带宽重写——**40M 用 WPA2-Personal + AES(CCMP)**（AES 不像 TKIP 那样禁用 11n/MCS 速率），**20M 用 WPA-Personal + TKIP**，写后用 `SECurity:TYPE?` 校验并重试。
- **梯度降电平（Ramp-down）**：关联后从 −45 dBm 逐级下降到探测起始电平，每级校验链路并做 100 包粗测，避免 −45→−83 的突跳导致掉线。
- **省电关闭**：关联成功后发 `AT+WLPS=lps,0,ips,0`，防止 DUT 进入休眠导致下行全丢。
- **串口固件卡死自愈**：当 DUT 回显 `MsgSend wait inic ipc done` 时，自动关闭串口 → 等 1 s → 重开 → 重发 `AT+WLCONN`。

---

## 测试用例（Test Profiles）

定义于 `Models/TestProfile.cs`。UI 的 **Test Mode** 下拉可按带宽 / 制式过滤，行首复选框控制是否执行。

| # | 名称（DisplayName） | 制式 | 带宽 | 速率 | Payload | PER 门限 | 探测起始 | 信道 A/B/C |
|---|---------------------|------|------|------|---------|----------|----------|------------|
| 1 | 11b 20M 2.4G CCK_11M | 11b | BW20 | CCK 11M | 1024 | 8% | −83 dBm | 3 / 7 / 11 |
| 2 | 11g OFDM 20M 2.4G OFDM_54M | 11g | BW20 | OFDM 54M | 1000 | 10% | −73 dBm | 3 / 7 / 13 |
| 3 | 11g/n 20M 2.4G MCS7 | 11gn | BW20 | MCS7 | 1024 | 10% | −68 dBm | 3 / 7 / 13 |
| 4 | 11g/n 40M 2.4G MCS7 | 11gn | BW40 | MCS7 | 1024 | 10% | −63 dBm | 3 / 7 / 11 |
| 5 | 11a 20M 5G OFDM_54M | 11a | BW20 | OFDM 54M | 1000 | 10% | −73 dBm | 36 / 100 / 149 |
| 6 | 11a/n 20M 5G MCS7 | 11an | BW20 | MCS7 | 1024 | 10% | −68 dBm | 36 / 100 / 149 |
| 7 | 11a/n 40M 5G MCS7 | 11an | BW40 | MCS7 | 1024 | 10% | −63 dBm | 38 / 102 / 159 |

---

## 硬件与连接

```
┌──────────────┐   SCPI over TCP (5025)   ┌──────────────┐
│  上位机 PC    │ ───────────────────────► │  R&S CMW500  │
│ (本软件)      │                          │   综测仪      │
│              │   串口 AT (1500000 8N1)   └──────┬───────┘
│              │ ───────────────────────►        │ RF (含线损)
└──────────────┘        ┌──────────────┐         │
                        │     DUT       │◄────────┘
                        │  (被测 WiFi)  │
                        └──────────────┘
```

- **CMW500**：RF 路由为 `SUU1,RF1C,RX1,RF1C,TX1`；Cable Loss 写入 `EATTenuation:INPut/OUTPut`。
- **DUT**：默认串口 `COM19`，波特率 `1500000`，8N1；AT 固件示例为 AmebaDPlus（`AT+WLCONN` / `AT+WLPS`）。
- **AP 配置**：SSID `CMW-AP`，密码 `12345678`。

---

## 环境要求

- **操作系统**：Windows 10 / 11（x64）
- **运行发布版**：无需额外依赖（自包含单文件 EXE 已内嵌 .NET 运行时）
- **从源码构建**：[.NET SDK 10.0](https://dotnet.microsoft.com/)（`net10.0-windows`，需启用 WPF）
- **NuGet 依赖**（自动还原）：
  - [`System.IO.Ports`](https://www.nuget.org/packages/System.IO.Ports) 9.0.0 —— 串口通信
  - [`ClosedXML`](https://www.nuget.org/packages/ClosedXML) 0.104.2 —— Excel 导出

---

## 构建与运行

源码位于 `src/` 目录。

```bash
cd src

# 还原依赖
dotnet restore

# 构建
dotnet build -c Debug

# 直接运行（Debug 不生成 exe 启动器，经 dotnet 宿主启动，绕过 Smart App Control 对未签名 exe 的拦截）
dotnet run
```

> `research/build-console.cmd` 提供了一个交互式构建控制台，内置 `r`/`b`/`p`/`rd`/`rp` 等快捷命令（restore/build/publish/run），可双击打开。

在 Visual Studio 中：打开 `src/WIFISensitivitySearch.slnx`，F5 启动。

---

## 发布（单文件 EXE）

### 方式一：本地发布

使用内置发布配置 `Properties/PublishProfiles/win-x64.pubxml`（ReadyToRun + 自包含 + 单文件 + 压缩）：

```bash
cd src
dotnet publish -p:PublishProfile=win-x64
```

> 注意：该 pubxml 的 `PublishDir` 指向固定输出目录，如需改路径请自行编辑。

### 方式二：GitHub Actions 自动发布

`.github/workflows/release.yml` 会在推送 `v*` / `V*` tag 时自动构建自包含单文件 EXE 并发布到 Releases（产物形如 `WIFISensitivitySearchV3.1.exe`）：

```bash
git tag v3.1
git push origin v3.1
```

也可在 Actions 页面手动触发（`workflow_dispatch`，仅上传 artifact，不创建 Release）。

---

## 使用说明

1. **连接 CMW500**：在 *CMW500 Connection* 填入 VISA 地址（如 `TCPIP0::172.29.0.3::INSTR`），点 **Connect**。连接成功后会读取 `*IDN?` 并一次性写入 RX Expected PEP。
2. **连接 DUT**：在 *DUT Connection* 选择串口与波特率（默认 `COM19 @ 1500000`），点 **Connect**。
3. **配置测试**：
   - **Test Mode**：选择要跑的用例集合（全部 / 按带宽 / 按制式）；
   - **Cable Loss (dB)**：射频线缆损耗；
   - **RX Expected PEP (dBm)**：默认 30，仅在 Connect 时下发一次，测试中不再改动；
   - 在结果表中勾选/取消要执行的行，必要时直接编辑 **Start (dBm)** 起始电平。
4. **开始测试**：点 **Start**。测试中可 **Pause / Resume**、**Stop**。
5. **查看结果**：每个信道列显示 `信道号 : 灵敏度 dBm`，异常状态显示 `NOT ASSOC` / `RAMP FAIL` / `FAIL`。
6. **导出**：点 **Export Excel** 保存报表。

---

## 导出报表

`Services/ResultExporter.cs` 使用 ClosedXML 生成 `.xlsx`：

- 列：`Type` / `Threshold` / `Start` / `Channel A` / `Channel B` / `Channel C`；
- 通道结果 `"36 : -74.1 dBm"` 自动解析为纯数值 `-74.1`（Excel 数字单元格）；空值 / `NOT ASSOC` / `FAIL` 等状态保留原文本；
- 默认文件名 `WIFI_Sensitivity_yyyyMMdd_HHmmss.xlsx`。

---

## 配置持久化

`Services/SettingsStore.cs` 将用户配置保存到
`%LocalAppData%\WIFISensitivitySearch\`：

| 文件 | 内容 |
|------|------|
| `start_overrides.json` | 各 Profile 被手动修改过的起始电平（仅存与默认值不同的行） |
| `rx_expected_pep.json` | RX Expected PEP 值（默认 30 dBm） |

I/O 均为 best-effort，出错静默忽略。

---

## 项目结构

```
WIFISensitivitySearch/
├─ .github/workflows/release.yml   # 打 tag 自动构建 + 发布单文件 EXE
├─ docs/                            # 断连恢复机制说明（Word）
│  ├─ WIFI信令断连恢复.docx
│  └─ WIFI40M信令断连恢复.docx
├─ research/build-console.cmd       # 交互式构建控制台（doskey 快捷命令）
└─ src/
   ├─ App.xaml(.cs)                 # WPF 应用入口
   ├─ MainWindow.xaml(.cs)          # 主界面 + 交互逻辑（连接/开始/暂停/导出）
   ├─ Models/
   │  ├─ TestProfile.cs             # 7 个测试用例定义（制式/带宽/速率/信道/门限…）
   │  └─ TestRow.cs                 # 表格行 ViewModel（INotifyPropertyChanged）
   ├─ Services/
   │  ├─ ScpiClient.cs              # CMW500 SCPI over TCP 客户端
   │  ├─ DutSerialClient.cs         # DUT 串口 AT 客户端 + 固件卡死自愈
   │  ├─ SensitivityTester.cs       # 核心：结构化配置 + 关联恢复 + 寻值算法
   │  ├─ ResultExporter.cs          # Excel 导出（ClosedXML）
   │  ├─ SettingsStore.cs           # 配置持久化（LocalAppData/JSON）
   │  └─ LicenseGuard.cs            # 构建有效期校验
   ├─ Properties/PublishProfiles/   # win-x64 单文件发布配置
   └─ WIFISensitivitySearch.csproj  # net10.0-windows / WPF
```

---

## 关键实现说明

- **SCPI 传输**（`ScpiClient`）：基于 `TcpClient` 原始套接字，命令以 `\n` 结尾，`\r\n` 逐字节读回；`SemaphoreSlim` 串行化收发；每条写命令后 `SYSTem:ERRor?` 校验。**无需安装厂商 VISA 库**。
- **枚举硬编码**：制式（BSTD/ASTD/GOST/GNST/ANST）、格式（NHT/HTM）、速率（C11Mbits/Q6M34/MCS7）按 CMW 文档直接硬编码，不做 SCPI 探测，提高稳定性。
- **PER 测量**（`MeasurePerAsync`）：`ABORt → 设包数 → INITiate → 轮询 PER:STATe? 至 RDY → FETCh PER?`，校验 reliability（0=OK，26=INV），最多重试 3 次。
- **暂停闸门**：`PauseGate`（`TaskCompletionSource`）+ `CancellationToken` 贯穿整个异步测试流程，可在任意 `GateAsync()` 点暂停/取消。
- **数据模式**：PER 测量使用协议规定的 **PN9** 伪随机序列（`PER:DPATtern PN9`）。

---


<sub>本项目为 WiFi 射频接收灵敏度测试内部工具，依赖 R&S CMW500 综测仪及对应 AT 固件的 DUT。</sub>
