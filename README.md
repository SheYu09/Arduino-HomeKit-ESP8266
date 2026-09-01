# Arduino HomeKit ESP8266（中文说明）

基于 [Mixiaoxiao/Arduino-HomeKit-ESP8266](https://github.com/Mixiaoxiao/Arduino-HomeKit-ESP8266) v1.4.0 的 ESP8266 原生 Apple HomeKit 配件服务器库，集成本项目（DC1 / esp12f_bl0942 智能插座固件）所需的定制适配。无需任何桥接，即可将设备接入 Apple「家庭」App。

## 特性

- 原生 HomeKit 配件实现（源自 [esp-homekit](https://github.com/maximkulkin/esp-homekit)），无桥接依赖
- 兼容 ESP8266 Arduino core 2.4.0（PlatformIO `espressif8266@2.4.0`）
- 配对（Pair Setup）、密钥校验（Pair Verify）、SRP / ED25519 / WolfSSL 加密均在板内完成
- 周期 MDNS 通告（每 5 秒）保活，提升 iOS 家庭 App 发现稳定性
- 日志可通过 `on_homekit_log` 回调接入固件统一日志系统
- 配对数据与固件框架配置共用 EEPROM 扇区，互不覆盖

## v1.5 适配说明（相对上游 v1.4.0）

1. **存储偏移 +2048**：HomeKit 配对数据移至 EEPROM 扇区 `[2048, 3456)`，扇区头 `[0, 2048)` 保留给固件框架配置；格式化时先读取备份整个扇区，擦除后整扇区写回，避免破坏框架配置
2. **`on_homekit_log` 日志回调**：`HOMEKIT_PRINTF` 优先调用回调函数指针，未赋值时回退 `printf_P` 输出
3. **MDNS 周期 `announce()`**（每 5 秒）：在 `arduino_homekit_loop()` 中加入保活通告，解决部分 iOS 环境初次发现不到配件的问题
4. **`base64.h/.c` 更名为 `homekit_base64.h/.c`**：避免与 ESP8266 core 自带 `base64.h` 冲突（该冲突会导致框架库 `ESP8266HTTPClient` 编译报 `'base64' has not been declared`）
5. **移除 `cJSON_Print` 调试输出**：省内存、防泄漏

## 快速开始（PlatformIO）

在 `platformio.ini` 的 `lib_deps` 中引用：

```ini
lib_deps =
  HomeKit-ESP8266=https://github.com/SheYu09/Arduino-HomeKit-ESP8266.git
```

或在工程内使用本地路径：

```ini
lib_deps =
  /workspace/HomeKit-ESP8266
```

代码示例：

```c
#include <arduino_homekit_server.h>

extern "C" homekit_server_config_t config;

void setup() {
    WiFi.begin(ssid, password);
    arduino_homekit_setup(&config);
}

void loop() {
    arduino_homekit_loop();
}
```

配件定义（C 文件，宏风格声明）：

```c
homekit_accessory_t *accessories[] = { ... };
homekit_server_config_t config = {
    .accessories = accessories,
    .password = "111-11-111",
    //.on_event = on_homekit_event, // 可选
    //.setupId = "ABCD"              // 可选
};
```

## 存储布局

- HomeKit 配对数据直接读写 ESP8266 内部 Flash 的 EEPROM 仿真扇区（4096 B），不使用带缓存的 `EEPROM` 库
- 本适配版偏移 +2048（地址 = 扇区基址 + 偏移 + 2048）：

| 数据 | 偏移 |
|------|------|
| MAGIC | 0 + 2048 |
| ACCESSORY_ID | 4 + 2048 |
| ACCESSORY_KEY | 32 + 2048 |
| PAIRINGS | 128 + 2048 |

- 扇区头 `[0, 2048)` 为固件框架配置区（如 esp_framework 的 EEPROM 配置）
- 地址布局与 v1.2.0（qlwz 版）完全一致，**升级固件无需重新配对**
- 本库不使用文件系统（FS），FS 可自由使用

## 日志回调

分配 `on_homekit_log` 函数指针即可将 HomeKit 库日志接入统一日志系统：

```c
extern "C" void (*on_homekit_log)(PGM_P formatP, ...);

void hk_log_callback(PGM_P formatP, ...) {
    // 转发到 Debug::AddInfo 等
}

void setup() {
    on_homekit_log = hk_log_callback;
}
```

## 性能与内存

- CPU 频率需设为 160 MHz（配对过程尤其必要，避免 iOS 因超时断开 TCP 连接）
- 配对耗时（上游参考数据）：Preinit 约 9.1 s，Pair Setup 全程约 14 s，Pair Verify 约 1 s
- 空闲堆内存（上游 v1.1.0 优化后参考数据）：Boot 约 46000，Preinit 结束后约 41000

## 推荐 IDE/构建设置

- 模块：Generic ESP8266 Module
- FlashSize：固件需至少 470 KB（WolfSSL 占用较大）
- LwIP Variant：v2 Lower Memory
- CPU Frequency：160 MHz（必须）
- 首次烧录选择「All Flash Contents」擦除

## 版本历史

- **v1.5（本分支）**：基于上游 v1.4.0，加入上述 5 项适配
- v1.4.0（上游）：crypto 计算中加入 `yield()` 防止 WiFi 断开；新增示例
- v1.3.0（上游）：小幅改进
- v1.2.0（上游）：新增示例
- v1.1.0（上游）：内存优化，字符串常量尽可能放入 Flash；附带 `ESP8266WiFi_nossl_noleak` 定制库
- v1.0.1（上游）：降低配对所需堆内存；MDNS 绑定 STA IP；重命名 `HTTP_METHOD` 避免多定义冲突

## 致谢

- [esp-homekit](https://github.com/maximkulkin/esp-homekit) / [esp-homekit-demo](https://github.com/maximkulkin/esp-homekit-demo)
- [esp_hw_wdt](https://github.com/ComSuite/esp_hw_wdt)
- [WolfSSL/WolfCrypt](https://www.wolfssl.com/products/wolfcrypt-2/)
- [cJSON](https://github.com/DaveGamble/cJSON)
- [cQueue](https://github.com/SMFSW/cQueue)
