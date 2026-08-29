# WT32-ETH01 WiFi↔以太网 透明网桥

> 把 WT32-ETH01（ESP32 + LAN8720）做成 WiFi↔以太网的 L2 透明网桥。
> 本质：为只有有线网口的设备提供 WiFi 接入，作用类似路由器/无线网卡——设备插网线即可接入 WiFi 网络。

---

## 使用场景

- 对于不具备WIFI能力的板卡，提供一种基于有线网口的无感免驱联网的解决方案。

---

## 为什么官方原版跑不起来（三个硬件适配点）

官方 sta2eth 示例默认针对 **ESP32-Ethernet-Kit（IP101 PHY）**，而 WT32-ETH01 硬件不同，导致三个独立故障：

| 故障 | 现象 | 原因 |
|------|------|------|
| WiFi 连上就断 | `0xf00` 握手超时 | sta2eth 禁用了 SHA1，WPA2 握手必需 HMAC-SHA1 |
| 以太网起不来 | `reset timeout` | GPIO16 是 LAN8720 电源使能脚，未拉高 → PHY 没上电 |
| 时钟不对 | Link Up 但数据不通 | LAN8720 的 25MHz 晶振倍频 50MHz 输出给 GPIO0，ESP32 应「时钟输入」而非输出 |

对应配置（`sdkconfig.defaults`）：

```ini
CONFIG_MBEDTLS_SHA1_C=y              # WiFi 握手
CONFIG_ETHERNET_PHY_RST_GPIO=16      # LAN8720 电源使能
CONFIG_ETHERNET_RMII_CLK_INPUT=y     # 50MHz 从 GPIO0 输入
CONFIG_ETHERNET_RMII_CLK_GPIO=0
CONFIG_ETHERNET_PHY_USE_LAN87XX=1    # PHY 型号
CONFIG_ETHERNET_PHY_ADDR=1
CONFIG_ETHERNET_PHY_INTERFACE_RMII=y
```

---

## 改动步骤（仅 `main/sta2eth_main.c`）

### 改动 1：顶部加硬编码 WiFi 凭据

```c
#define EXAMPLE_DEFAULT_WIFI_SSID      "wifi名"     
#define EXAMPLE_DEFAULT_WIFI_PASS      "wifi密码" 
```

### 改动 2：`app_main()` 里，NVS 空或 SSID 不一致时自动写入硬编码凭据

**为什么改**：官方原版只在 NVS 里没有凭据时才进网页配网；一旦配过一次，改宏就再也不生效。改成"启动时对比 NVS 里的 SSID 和宏，不一致就用宏覆盖"，这样改宏重新烧录就能直接连新 WiFi。

```c
if (do_provision) {
    start_provisioning(...);   // 按钮长按 → 网页配网（保留作 fallback）
} else {
    wifi_config_t wifi_cfg;
    bool update_config = false;
    if (esp_wifi_get_config(WIFI_IF_STA, &wifi_cfg) != ESP_OK) {
        update_config = true;                       // NVS 空
    } else if (strcmp((const char *)wifi_cfg.sta.ssid, EXAMPLE_DEFAULT_WIFI_SSID) != 0) {
        update_config = true;                       // 改了宏
    }
    if (update_config) {
        ESP_LOGI(TAG, "Applying hardcoded WiFi config (SSID=%s)", EXAMPLE_DEFAULT_WIFI_SSID);
        ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
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
- 开机自动连硬编码 WiFi，无需网页配网
- 改宏重新烧录 → 自动覆盖旧凭据 → 连新 WiFi
- WiFi 没开/密码错 → 自动回退网页配网

---

## 实行步骤（操作流程）

### 步骤 1：打开 ESP-IDF 编译环境

**通用做法（适合大多数人）**：

- 从 Windows 开始菜单打开 **「ESP-IDF 6.0 PowerShell」**（版本号可能不同，认准带 ESP-IDF 字样的 PowerShell 快捷方式），它会自动配置好所有环境变量。

- 或者用 **VS Code**：安装 "ESP-IDF" 插件，用它打开本工程文件夹，直接编译烧录。

> 只有当工具链装在**非默认位置**（不在 `C:\Users\你的用户名\.espressif`）时，才需要手动配置环境变量，见文末「附录：特殊环境配置」。

### 步骤 2：进入工程 + 编译

```powershell
cd <你的ESP-IDF目录>\project\wt32-eth01-bridge
idf.py build
```

成功标志：`sta_to_eth.bin binary size ... Project build complete.`

### 步骤 3：烧录

1. 杜邦线把 **IO0 接 GND**（进入下载模式），模块**断电再上电**
2. 执行（`COM15` 换成你电脑实际识别的串口号）：

```powershell
idf.py -p COM15 flash
```

成功标志：`Writing 'sta_to_eth.bin' ... 100.0%` + `Done`

### 步骤 4：运行

1. **拔掉 IO0 接地线**（IO0 恢复高电平），模块**断电再上电**
2. 模块自动连 WiFi 进入桥接

### 步骤 5：验证

- 电脑网线插模块，网卡设「自动获取 IP」
- 电脑拿到 WiFi 网段 IP，`ping 223.5.5.5` 通 = 成功

---

## 换 WiFi

改 `sta2eth_main.c` 顶部两个宏，重新编译烧录即可，无需清 NVS：

```c
#define EXAMPLE_DEFAULT_WIFI_SSID      "新WiFi名"
#define EXAMPLE_DEFAULT_WIFI_PASS      "新密码"
```

---

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| WiFi 0xf00 握手超时 | SHA1 被禁用 | `CONFIG_MBEDTLS_SHA1_C=y` |
| 以太网 reset timeout | GPIO16 没拉高 | `CONFIG_ETHERNET_PHY_RST_GPIO=16` |
| Link Up 但 ping 不通 | 时钟方向错 | `CONFIG_ETHERNET_RMII_CLK_INPUT=y` |
| 改宏不生效 | NVS 存旧凭据 | 新逻辑已自动覆盖 |
| 电脑没网 | 模块没连上 WiFi | 先确认模块连上 WiFi 再插网线 |

---

## 总结

**网桥打通 = SHA1 让 WiFi 握手成功 + GPIO16 给 LAN8720 上电 + CLK_INPUT 时钟同源；再加硬编码凭据让开机自动连接。** 源码只改了 sta2eth_main.c 的启动逻辑，其余是 sdkconfig 硬件适配。


## 附录：特殊环境配置（工具链装在非默认位置）

本仓库原作者把 ESP-IDF 工具链装在了 E 盘。如果你也遇到工具链不在默认位置的情况，参考以下手动配置（把路径换成你的实际路径）：

```powershell
cd /d E:\esp-idf\esp-idf-v6.0.2
set IDF_TOOLS_PATH=E:\.espressif
set TEMP=E:\esp_temp
set TMP=E:\esp_temp
call export.bat
cd project\wt32-eth01-bridge
idf.py build
```

说明：

- `IDF_TOOLS_PATH` 指向工具链目录（默认是 `C:\Users\你的用户名\.espressif`，只有改了位置才需要 set）
- `TEMP` / `TMP` 指向不含中文的临时目录（避免中文用户名路径导致编译报错）
- `call export.bat` 加载 ESP-IDF 环境，之后才能用 `idf.py`

