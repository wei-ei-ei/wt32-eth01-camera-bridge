# WT32-ETH01 WiFi↔以太网 透明网桥

> 把 WT32-ETH01（ESP32 + LAN8720）做成 WiFi↔以太网的 L2 透明网桥。
> 本质：为只有有线网口的设备提供 WiFi 接入，作用类似路由器/无线网卡——设备插网线即可接入 WiFi 网络。
> 手机热点本质就是 WiFi（和无线路由器一样），本网桥对二者无区别，都可直接使用。

---

## 使用场景

- 台式机、工控设备、NAS 等**只有有线网口**的设备，需要接入 WiFi 网络
- WiFi 网络可以是**无线路由器**，也可以是**手机热点**（本质都是 WiFi）
- 需要"无线转有线"、让设备无感上网的场合

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

## 源码改动（仅 `main/sta2eth_main.c`）

### 改动 1：顶部加硬编码 WiFi 凭据

```c
#define EXAMPLE_DEFAULT_WIFI_SSID      "HUAWEI"
#define EXAMPLE_DEFAULT_WIFI_PASS      "rxw12345"
```

### 改动 2：`app_main()` 里，NVS 空或 SSID 不一致时自动写入硬编码凭据

**为什么改**：官方原版只在 NVS 里没有凭据时才进网页配网；一旦配过一次，改宏就再也不生效。改成"启动时对比 NVS 里的 SSID 和宏，不一致就用宏覆盖"，这样改宏重新烧录就能直接连新热点。

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
- 开机自动连硬编码热点，无需网页配网
- 改宏重新烧录 → 自动覆盖旧凭据 → 连新热点
- 热点没开/密码错 → 自动回退网页配网（不会变砖）

---

## 操作流程

### 编译

```powershell
cd /d E:\esp-idf\esp-idf-v6.0.2
set IDF_TOOLS_PATH=E:\.espressif
set TEMP=E:\esp_temp
set TMP=E:\esp_temp
call export.bat
cd project\wt32-eth01-bridge
idf.py build
```

### 烧录

1. 杜邦线把 **IO0 接 GND**，模块**断电再上电**
2. `idf.py -p COM15 flash`

### 运行

1. **拔掉 IO0 接地线**，模块**断电再上电**
2. 模块自动连 WiFi 进入桥接

### 验证

- 电脑网线插模块，网卡设「自动获取 IP」
- 电脑拿到热点网段 IP，`ping 223.5.5.5` 通 = 成功

---

## 换热点

改 `sta2eth_main.c` 顶部两个宏，重新编译烧录即可，无需清 NVS：

```c
#define EXAMPLE_DEFAULT_WIFI_SSID      "新热点名"
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
| 电脑没网 | 模块没连上热点 | 先确认模块连上热点再插网线 |

---

## 一句话总结

**网桥打通 = SHA1 让 WiFi 握手成功 + GPIO16 给 LAN8720 上电 + CLK_INPUT 时钟同源；再加硬编码凭据让开机自动连接。** 源码只改了 sta2eth_main.c 的启动逻辑，其余是 sdkconfig 硬件适配。
