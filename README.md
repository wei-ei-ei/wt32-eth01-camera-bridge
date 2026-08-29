# WT32-ETH01 WiFi↔以太网 透明网桥

> 目标：把 WT32-ETH01（ESP32 + LAN8720）做成 WiFi↔以太网 L2 透明网桥，插网线即上网。
> 工程：ESP-IDF v6.0.2 官方示例 `examples/network/sta2eth`
> 硬件：WT32-ETH01（LAN8720 PHY + 25MHz 晶振 + GPIO16 电源使能）

---

## 一、硬件背景：为什么官方原版跑不起来

官方 sta2eth 示例默认假设的是 **ESP32-Ethernet-Kit（IP101 PHY）**，而 WT32-ETH01 硬件不同，导致三个独立故障：

| 故障 | 现象 | 根因 |
|------|------|------|
| WiFi 连上就断 | `0xf00` 4-way 握手超时 | sta2eth 自带 `mbedtls_preset_sta2eth.conf` 禁用了 SHA1，WPA2 握手必需 HMAC-SHA1 |
| 以太网起不来 | `reset timeout` | GPIO16 是 LAN8720 电源使能脚，未拉高 → PHY 没上电 → 25MHz 晶振不起振 |
| 时钟不对 | Link Up 但数据不通 | LAN8720 的 25MHz 晶振倍频 50MHz 输出给 ESP32 GPIO0，ESP32 应是「时钟输入」而非输出 |

---

## 二、源码改动详情

### 2.1 `main/sta2eth_main.c`（2 处改动）

**改动 1：文件顶部加硬编码 WiFi 凭据宏**

```c
static const char *TAG = "example_sta2wired";

/* 【修改】硬编码 WiFi 凭据：
 * NVS 中无已保存凭据时，自动使用下面的 SSID/密码连接，跳过网页配网。 */
#define EXAMPLE_DEFAULT_WIFI_SSID      "HUAWEI"
#define EXAMPLE_DEFAULT_WIFI_PASS      "rxw12345"
```

**改动 2：`app_main()` 里，NVS 空或 SSID 不一致时自动写入硬编码凭据**

官方原版逻辑：
```c
if (do_provision || !is_provisioned()) {
    // 网页配网
    start_provisioning(...);
} else {
    // 桥接（用 NVS 里的凭据）
    connect_wifi(); wired_bridge_init(...);
}
```

修改后逻辑：
```c
if (do_provision) {
    // 按钮长按 → 网页配网（保留作 fallback）
    start_provisioning(...);
} else {
    /* 硬编码凭据优先：NVS 空 或 SSID 与宏不一致 → 用宏覆盖 NVS */
    wifi_config_t wifi_cfg;
    bool update_config = false;
    if (esp_wifi_get_config(WIFI_IF_STA, &wifi_cfg) != ESP_OK) {
        update_config = true;                          // NVS 空
    } else if (strcmp((const char *)wifi_cfg.sta.ssid, EXAMPLE_DEFAULT_WIFI_SSID) != 0) {
        update_config = true;                          // 改了宏
    }
    if (update_config) {
        ESP_LOGI(TAG, "Applying hardcoded WiFi config (SSID=%s)", EXAMPLE_DEFAULT_WIFI_SSID);
        ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));   // 必须先设 STA 模式
        memset(&wifi_cfg, 0, sizeof(wifi_cfg));
        strlcpy((char *)wifi_cfg.sta.ssid, EXAMPLE_DEFAULT_WIFI_SSID, sizeof(wifi_cfg.sta.ssid));
        strlcpy((char *)wifi_cfg.sta.password, EXAMPLE_DEFAULT_WIFI_PASS, sizeof(wifi_cfg.sta.password));
        ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_cfg));
    }
    ESP_LOGI(TAG, "Starting USB-WiFi bridge");
    if (connect_wifi() != ESP_OK) {
        xEventGroupSetBits(s_event_flags, RECONFIGURE_BIT);   // 连不上 → 回退网页配网
    } else {
        wired_bridge_init(wired_recv_callback, wifi_buff_free);
    }
}
```

**行为**：
- 开机 NVS 空 → 自动写硬编码凭据 → 自动连 WiFi → 桥接（无需网页配网）
- 改宏重新烧录 → 检测 SSID 不一致 → 自动覆盖旧凭据 → 连新热点
- 热点没开/密码错 → 连不上 → 自动回退网页配网（不会变砖）

### 2.2 `main/ethernet_iface.c`（1 处遗留改动，与本次打通无关）

```c
// 官方原版：
#define MODIFY_DHCP_MSGS        CONFIG_EXAMPLE_MODIFY_DHCP_MESSAGES
// 当前（早期调试 DHCP 时硬编码为 0）：
// #define MODIFY_DHCP_MSGS        CONFIG_EXAMPLE_MODIFY_DHCP_MESSAGES
#define MODIFY_DHCP_MSGS 0
```

作用：关闭桥接时对 DHCP 消息的 MAC 改写。对当前手机热点场景无影响。

---

## 三、配置改动详情（sdkconfig）

### 3.1 `mbedtls_preset_sta2eth.conf`（WiFi 握手修复）

```ini
# 改前：
CONFIG_MBEDTLS_SHA1_C=n
# 改后：
CONFIG_MBEDTLS_SHA1_C=y
```

### 3.2 `sdkconfig` / `sdkconfig.defaults`（PHY + 时钟修复）

```ini
# --- PHY 型号（LAN8720）---
CONFIG_ETHERNET_PHY_USE_LAN87XX=1
CONFIG_ETHERNET_PHY_LAN87XX=y
CONFIG_ETHERNET_PHY_ADDR=1

# --- RMII 接口 ---
CONFIG_ETHERNET_PHY_INTERFACE_RMII=y

# --- 时钟：50MHz 从 GPIO0 输入（LAN8720 晶振倍频后输出）---
CONFIG_ETHERNET_RMII_CLK_INPUT=y
CONFIG_ETHERNET_RMII_CLK_GPIO=0

# --- GPIO16 = LAN8720 电源使能脚（拉高上电）---
CONFIG_ETHERNET_PHY_RST_GPIO=16

# --- SHA1（WiFi 握手）---
CONFIG_MBEDTLS_SHA1_C=y

# --- 配网模式（HTTP 网页配网，作为 fallback）---
CONFIG_EXAMPLE_WIFI_CONFIGURATION_MANUAL=y
CONFIG_EXAMPLE_WIRED_INTERFACE_IS_ETHERNET=y
```

---

## 四、完整操作流程（从 PowerShell 开始）

### 4.1 环境准备

```powershell
cd /d E:\esp-idf\esp-idf-v6.0.2
set IDF_TOOLS_PATH=E:\.espressif
set TEMP=E:\esp_temp
set TMP=E:\esp_temp
call export.bat
```

看到 `Setting IDF_PATH to 'E:\esp-idf\esp-idf-v6.0.2'` 即成功。

### 4.2 进入工程 + 编译

```powershell
cd examples\network\sta2eth
idf.py build
```

成功标志：`sta_to_eth.bin binary size ... Project build complete.`

### 4.3 烧录

硬件操作：
1. 杜邦线把 **IO0 接 GND**（进入下载模式）
2. 模块 **断电再上电**

命令：
```powershell
idf.py -p COM15 flash
```

成功标志：`Writing 'sta_to_eth.bin' ... 100.0%` + `Done`

### 4.4 运行

1. **拔掉 IO0 接地线**（IO0 恢复高电平）
2. 模块 **断电再上电**

模块自动完成配网：写硬编码凭据 → 连 WiFi → 桥接。

### 4.5 验证

- 电脑网线插模块，网卡设「自动获取 IP」
- 电脑应拿到 **热点网段** 的 IP（如 10.89.50.x）
- `ping 223.5.5.5` 通 = 桥接成功

---

## 五、换热点操作（改 SSID/密码）

只需改 `sta2eth_main.c` 顶部两个宏，重新编译烧录：

```c
#define EXAMPLE_DEFAULT_WIFI_SSID      "新热点名"
#define EXAMPLE_DEFAULT_WIFI_PASS      "新密码"
```

```powershell
idf.py build
idf.py -p COM15 flash   # （IO0 接地 + 上电后执行）
```

模块启动时检测到 SSID 不一致，会自动覆盖旧凭据连接新热点，无需手动清 NVS。

---

## 六、常见问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| WiFi 0xf00 握手超时 | SHA1 被禁用 | `CONFIG_MBEDTLS_SHA1_C=y` |
| 以太网 reset timeout | GPIO16 没拉高 | `CONFIG_ETHERNET_PHY_RST_GPIO=16` |
| Link Up 但 ping 不通 | 时钟方向错 | `CONFIG_ETHERNET_RMII_CLK_INPUT=y` |
| 改宏不生效 | NVS 存旧凭据 | 新逻辑已自动覆盖；或擦 NVS |
| 电脑没网 | 模块没连上热点 | 先确认模块连上热点，再插网线 |
| 网页配网打不开 | 以太网没起 | 检查 PHY_RST_GPIO=16 + CLK_INPUT |

---

## 七、一句话总结

**网桥打通 = SHA1 让 WiFi 握手成功 + GPIO16 给 LAN8720 上电 + CLK_INPUT 时钟同源；再加硬编码凭据让开机自动连接、无需网页配网。** 核心功能源码只动了 sta2eth_main.c 的启动逻辑，其余全是 sdkconfig 硬件适配配置。
