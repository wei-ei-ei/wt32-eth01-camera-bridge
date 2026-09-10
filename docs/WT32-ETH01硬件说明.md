# WT32-ETH01 硬件说明（原创整理）

> 本文为项目使用过程中的**原创整理笔记**，用于快速上手；完整规格以原厂资料为准。
> WT32-ETH01 的规格书与资料版权归 **Wireless-Tag（深圳市启明云端科技）** 所有，本仓库不转载原厂 PDF。

## 官方资料入口

- 产品页：<http://www.wireless-tag.com/portfolio/wt32-eth01/>
- 文档站（规格书/用户手册下载）：<http://docs.wireless-tag.com/>
- 规格书直链（V1.1 示例）：<https://docs.wireless-tag.com/wp-content/uploads/2023/08/WT32-ETH01_V1.1-规格书-1.pdf>

## 模块构成

| 组成 | 型号 | 说明 |
|------|------|------|
| 主控 | ESP32-D0WD-V3（位于厂家的 WT32-S1 核心模组内） | WiFi 2.4GHz + BLE，双核 240MHz |
| 以太网 PHY | LAN8720AI | 100BASE-TX RMII 接口 |
| PHY 晶振 | 25 MHz | 由 PHY 内部倍频出 50MHz REF_CLK 供给 ESP32 |
| 网口 | 不含 RJ45 | MDI（TX±/RX±）引出至排针，需外接带网络变压器的 RJ45 座（如 HR911105A 类） |

## 供电

- 主供电 **5V**（板上 LDO 降至 3.3V），另有 3V3 引脚可供电/取电；
- **网桥不给外设供电**：接入设备（如需独立供电的 IPC 模组）必须自行解决电源——
  本项目实测中摄像头"网口不通"的直接原因就是 POE 供电预期落空。

## 关键引脚（本项目用到的）

| 引脚 | 用途 | 说明 |
|------|------|------|
| GPIO0 | RMII REF_CLK **输入** | LAN8720 倍频输出的 50MHz 从这里进 ESP32（时钟方向配置错误 = Link Up 但数据不通） |
| GPIO16 | LAN8720 复位/电源使能 | 需拉高，否则 PHY 初始化超时（`reset timeout`） |
| IO2 | 重配网按键 | 长按 2 秒触发蓝牙重配网（固件内置上拉，按键接 GND） |
| IO39 / IO36 | 预留输入 | 仅输入 |
| TX/RX (U0) | 调试串口 | 115200-8-N1，日志与监控 |
| EN / IO0 | 复位 / 下载模式 | **无自动下载电路**：IO0 接 GND 后断电重上电进入 bootloader |
| LINK | 网络连接指示 | LAN8720 输出，可接 LED |

完整引脚表请以官方规格书为准（见上方链接）。

## 本项目相关的时钟设计要点

```
25MHz 晶振 ──> LAN8720AI ──(内部倍频 50MHz REF_CLK, nINT/REFCLKO 功能)──> ESP32 GPIO0
                                                        │
                              ESP32 RMII 采样时钟 = 该外部输入（CONFIG_ETHERNET_RMII_CLK_INPUT=y）
```

- 若把 ESP32 配成"时钟输出"模式（官方 Kit 的接法），WT32-ETH01 上会 Link Up 但完全不通——
  这是移植时最隐蔽的坑，对应 `sdkconfig.defaults` 的 `CONFIG_ETHERNET_RMII_CLK_INPUT=y`；
- GPIO16 未拉高时 LAN8720 不上电，表现为 `ethernet_init: reset timeout`。

## 实测要点

- 2.4G WiFi：WPA2-PSK 正常，RSSI -44 时 iperf3 单跳 TCP 13~14 Mbps（桥接转发路径的典型水平）；
- 2.4G 与 BLE 共用射频：桥接运行时 BLE 配网服务不启动，无共存损耗；
  若要做"BLE 常驻"版本需注意共存开销（实测吞吐约减半）。
