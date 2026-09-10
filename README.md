# WT32-ETH01 无线网桥 · 为 IP 摄像头提供"无感"WiFi 接入

> BLE 蓝牙配网 · WiFi↔以太网 L2 透明桥接 · RTMP 推流实测通过
>
> 让只有**有线网口**的设备（IP 摄像头、NVR、工控板）——**插上网线即接入 WiFi 网络**，无需驱动、无需改设备配置。

---

## 一、应用场景

```
                    ┌──────────────┐  网线   ┌─────────────────┐   WiFi    ┌────────┐
  监控平台/云端  <──│  摄像头模组   │────────│  WT32-ETH01 网桥  │──────────│  路由器  │
  (RTMP拉流+控制)   │ (无WiFi,百兆) │        │ ESP32+LAN8720    │  2.4GHz  └────────┘
                    └──────────────┘        └─────────────────┘
```

- 摄像头只拥有百兆以太网口、**没有 WiFi**，靠网桥接入无线网络；
- 网桥对其完全透明：摄像头**保持自己的 IP**（静态或 DHCP 均可），监控平台**直连**该 IP 拉流/下发控制，无需端口映射（局域网内）；
- 换 WiFi 环境：**长按按键 2 秒** → 蓝牙配网（手机 App），全程不需要接电脑。

## 二、特性

| 特性 | 说明 |
|------|------|
| L2 透明桥接 | 以太网混杂模式 + MAC 改写，同一广播域、端到端会话，IP 层无 NAT |
| **ARP 学习（本仓库核心改动）** | 不发 DHCP 的**静态 IP 设备**也能被自动识别并接入，无需改动网桥配置 |
| BLE 蓝牙配网 | 长按 IO2（2 秒）进入 ESP 统一配网，手机 App（ESP BLE Provisioning）下发 WiFi 凭据 |
| 开机自动回连 | 凭据存 NVS，断电重启自动重连；连不上自动回退配网模式 |
| 拥塞可观测 | WiFi 发送队列丢帧计数告警（`WiFi TX drop #N`），量化空口拥塞 |
| 省电关闭 | `WIFI_PS_NONE`，避免下行流量在休眠窗口丢失 |

## 三、硬件

| 器件 | 型号/要求 | 备注 |
|------|-----------|------|
| 网桥 | WT32-ETH01（ESP32-D0WD-V3 + LAN8720AI，百兆） | 5V 供电 |
| 摄像头 | 海思 HI3516CV610 方案 IPC 模组（无 WiFi、百兆网口） | **DC 12V/2A 独立供电（不能依赖 POE，网桥不供电！）** |
| 路由器 | 任意 2.4GHz WiFi 路由器 | 建议关闭"无线隔离/AP 隔离"；有双频时监控端连 5G 可显著改善延迟 |

> 硬件细节见 [`docs/WT32-ETH01硬件说明.md`](docs/WT32-ETH01硬件说明.md)、接入经验见 [`docs/摄像头模块接入说明.md`](docs/摄像头模块接入说明.md)。

## 四、快速开始

### 1. 环境

- ESP-IDF **v6.0.2**：本工程由官方示例 `examples/network/sta2eth` 改造而来，**v6.0.2 为验证过的编译版本**；
  其他 IDF 版本的示例目录位置与内部 API 均有差异（如 `esp_wifi_internal_reg_rxcb` 等私有接口），
  直接换版本编译不保证通过；
- 依赖组件随 `main/idf_component.yml` 由 Component Manager 自动拉取：`espressif/ethernet_init`、`espressif/network_provisioning`；
- Windows 下若工具链不在默认位置，需设置环境变量（参考文末附录）

### 2. 编译烧录

WT32-ETH01 **没有自动下载电路**，烧录需手动进 bootloader：

1. **IO0 接 GND** → 模块断电再上电（进入下载模式）
2. 烧录：
   ```powershell
   idf.py -p COM15 flash monitor
   ```
3. **拔掉 IO0 接地线** → 断电重新上电，模块自动运行

> 刷机后首次开机（NVS 无凭据）自动进入蓝牙配网；串口第一行会打印固件版本横幅
> `=== fw ... [wt32-eth01-bridge] promiscuous+arp-learn+tx-cnt ===`，用于确认烧的是本仓库代码。

### 3. 蓝牙配网（换 WiFi 也用这一步）

1. 手机安装 **ESP BLE Provisioning** App（Espressif 官方）；
2. **长按网桥 IO2 按键 2 秒**（触发重配网，设备重启进入配网模式）；
3. App 扫描设备 → 发送 WiFi 名称/密码 → 配网成功后设备自动重启进入桥接；
4. 串口看到 `Wi-Fi STA connected` 即成功。

### 4. 接入摄像头

1. 摄像头静态 IP 配置：**与路由器同网段、掩码一致、网关指向路由器**，地址选 DHCP 池之外避免冲突；
2. 网线接入网桥，**给网桥断电重启**（重要！见"已知限制"）；
3. 验证：电脑（连同一 WiFi）`ping 摄像头IP`，真设备的回包特征是 **TTL=64、延迟 >1ms**；
4. 串口出现 `Wired client MAC learned from ARP: xx:xx:...` = 设备已被网桥识别。

## 五、性能实测（供参考）

| 项目 | 数据 | 条件 |
|------|------|------|
| iperf3 TCP（单跳） | 13~14 Mbps | 对端接路由器有线侧 |
| iperf3 TCP（两跳同频） | ~9.6 Mbps | 对端与网桥同为路由器 2.4G 无线终端 |
| ping 延迟（两跳） | 5~11 ms | 信号 RSSI -44 时 |
| 摄像头 RTMP 推流 | 可用 | 两跳场景建议码率 ≤2~3 Mbps，并观察串口 `WiFi TX drop` 计数 |

> 同频两跳的吞吐约为单跳的 5~7 折（空口时间共享所致），是物理规律而非故障。改善：路由器开双频，监控端走 5G。

## 六、与官方示例的差异

本工程基于 Espressif 官方 `sta2eth` 示例（ESP-IDF）改造。**逐项改动说明（含原因与验证方法）见 [`docs/官方示例改动说明.md`](docs/官方示例改动说明.md)**，概要：

1. **硬件适配三项**：WPA2 握手需要 SHA1（默认预设被禁）；LAN8720 复位脚 GPIO16 需拉高；RMII 50MHz 时钟为 GPIO0 **输入**（LAN8720 晶振倍频输出）；
2. **配网方案**：网页配网 → **BLE 蓝牙配网**（`network_provisioning` + NimBLE）；
3. **开启以太网混杂模式**：官方默认关闭，关闭时 EMAC 硬件过滤会丢弃"目的 MAC 非本机"的单播帧——这是"能上网但无法与 WiFi 侧设备互通"的直接根因；
4. **ARP 学习（原创补丁）**：官方 MAC 槽位只认 DHCP DISCOVER，静态 IP 设备（不发 DHCP）永远无法接入——现从首个上行 ARP 帧学习，串口打印学习结果；
5. **TX 丢包计数**：官方对 WiFi 发送失败只打 LOGD（不可见），现改为 WARN 级计数告警。

## 七、已知限制与排查口诀

| 限制/现象 | 原因 | 处理 |
|-----------|------|------|
| 电脑拿到 192.168.4.x | WiFi 没连上，掉进了配网模式（内置 DHCP） | 查 WiFi 环境后**断电**重启（软重启会停留在配网模式） |
| ping 通但 TTL=128 / 0ms | 应答的是电脑自己（IP 撞车），不是目标设备 | 检查本机 IP 配置，`arp -d *` 后用正确 IP 重测 |
| 能上网但无法与 WiFi 侧设备互通 | 混杂模式未开启（本仓库已默认开启） | 确认 `CONFIG_EXAMPLE_ETHERNET_USE_PROMISCUOUS=y` |
| 换了有线设备后新设备不通 | MAC 槽位是开机后学习的，只认第一个设备 | **网桥断电重启** |
| 推流延迟持续升高不回落 | 空口容量不足 → 看串口 `WiFi TX drop` 计数 | 降码率（≤3 Mbps）、双频分流、换干净信道 |

## 八、目录结构

```
├── main/
│   ├── sta2eth_main.c        # 主逻辑：桥接/配网分支、WiFi 事件、收发回调
│   ├── ethernet_iface.c      # 以太网侧：MAC 改写（mac_spoof）、ARP 学习、桥接收发
│   ├── provisioning.c        # BLE 配网（network_provisioning + NimBLE）
│   ├── usb_ncm_iface.c       # (备用) USB NCM 有线接口，以太网方案下不参与编译
│   └── manual_config.c       # (备用) 手动配置方案
├── docs/
│   ├── 官方示例改动说明.md     # 逐项改动、原因、验证方法
│   ├── WT32-ETH01硬件说明.md  # 模块构成、引脚、时钟设计、官方资料链接
│   └── 摄像头模块接入说明.md   # IPC 模组接入经验（供电/静态IP/推流验证）
├── mbedtls_preset_sta2eth.conf  # mbedtls 裁剪预设（已修正 SHA1）
└── sdkconfig.defaults        # 本仓库全部配置差异（混杂模式、BLE、PHY、时钟…）
```

## 九、许可与致谢

- 代码基于 **Espressif 官方示例 `examples/network/sta2eth`**（ESP-IDF v6.0.2）改造，原文件保留 Espressif 的 SPDX 标识（`Unlicense OR CC0-1.0`）；本仓库的修改同样以该许可发布；
- 依赖组件（`espressif/ethernet_init`、`espressif/network_provisioning` 等）许可随 IDF Component Manager 声明；
- WT32-ETH01 规格资料版权归原厂（Wireless-Tag / 启明云端）所有，本仓库**不转载原厂 PDF**，只提供原创整理笔记与官方链接；
- 各商标归其所有者所有。

---

## 附录：特殊环境配置（工具链非默认位置）

```powershell
cd E:\esp-idf\esp-idf-v6.0.2
set IDF_TOOLS_PATH=E:\.espressif
set TEMP=E:\esp_temp
set TMP=E:\esp_temp
.\export.ps1
cd project\wt32-eth01-bridge
idf.py build
```
