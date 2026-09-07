# AI 协力构建的 Quantum Fiber / Gemtek W1700K OpenWrt 固件

适用于 **Quantum Fiber / Gemtek W1700K** 路由器的定制 OpenWrt 固件构建项目。

基于 [W1700K OpenWrt Builds](https://github.com/w1700k/builds) 构建框架，源码基线为 [OpenW1700k](https://github.com/OpenWRT-fanboy/OpenW1700k)（ubi2 / ubi2-oc 分支）。

> ⚠️ **仅适用于 Quantum Fiber / Gemtek W1700K，请勿刷入其他型号设备。**

---

## ✨ 主要特性

- 🌐 LuCI 及内置应用默认中文
- 🏪 整合 iStoreOS（istoreos-25.12）：iStoreOS feeds、files 包与 Argon 风格界面，首次启动默认启用
- 🎨 Aurora LuCI 主题保留，可随时切换
- 🕐 系统时区：香港（UTC+8）
- 🌡️ LuCI 首页增加设备温度及风扇转速显示

---

## 🏪 iStoreOS 整合

本项目已整合 [iStoreOS](https://github.com/istoreos/istoreos)（`istoreos-25.12` 分支）：

| 组件 | 来源 | 说明 |
| --- | --- | --- |
| `user/default/feeds.conf` | jjm2473/packages、jjm2473/luci（`istoreos-25.12`）、openwrt/routing、jjm2473/openwrt-third | CI 中替换基线仓库默认 feeds，提供完整 iStoreOS 软件包生态（含 `luci-theme-argon`、`luci-app-argon-config`） |
| iStoreOS 内置包 | istoreos/istoreos（`istoreos-25.12` 分支）的 `package/istoreos-files` 与 `package/diy/{dkml,luci-app-ota}` | 构建时由 `custom.sh` 克隆并拷入源码树（与 Aurora / OpenClash / iStore 商店相同方式，版本锁定在 istoreos-25.12） |

首次启动行为：

- 默认界面切换为 Argon（iStoreOS 风格 + easepi 图标）；Aurora 主题仍可在 LuCI 中切换
- 时区保持 Asia/Hong_Kong（`99-w1700k-defaults` 在 iStoreOS `09_istoreos` 之后执行，覆盖其默认值）
- 启用 USB 自动挂载（blockmount）、fstab LuCI 菜单、dkml 内核模块管理、OTA 刷机页面

应用商店（linkease/istore）沿用 `custom.sh` 既有的 package 克隆方式接入，未以 feed 形式重复添加。刷入后在 LuCI 侧边栏可见「应用商店」入口；其运行于 apk 模式（固件内置 `/etc/apk/arch`），软件源来自固件自带的 `distfeeds.list`（OpenWrt snapshot feeds）。

---

## 📦 固件版本

| 固件 | 说明 |
| --- | --- |
| `ubi2` | 常规版本，使用标准 CPU 工作参数 |
| `ubi2-oc` | 超频版本，使用项目提供的超频配置 |

---

## 默认访问

- 管理地址：`192.168.8.1`
- 管理密码：无
- Wi-Fi SSID：`W1700K`
- Wi-Fi 密码：`12345678`

---

## 📡 默认无线配置

| 项目 | 2.4 GHz | 5 GHz | 6 GHz |
| --- | --- | --- | --- |
| 状态 | 开启 | 开启 | **关闭** |
| 区域 | US | US | US |
| 信道 | 1 | 36 | 37 |
| 频宽 / 模式 | Wi‑Fi 7（EHT20） | Wi‑Fi 7（EHT160） | Wi‑Fi 7（EHT320） |
| SSID | `W1700K` | `W1700K` | `W1700K-6G` |
| 加密 | WPA2-PSK | WPA2-PSK | WPA3-SAE |
| 密码 | `12345678` | `12345678` | `12345678` |
| 发射功率 | 23 dBm | 25 dBm | 25 dBm |

---

## 🌡️ 温度监控

LuCI 状态首页显示 CPU、主板、10G WAN/LAN PHY、2.4/5/6 GHz WiFi 温度及风扇转速/占空比，随温度区间变色提示。

---

## 🔄 自动构建

GitHub Actions 每日 **香港时间 14:00** 自动构建：

```text
W1700K-OpenWrt_<构建时间>_r<版本号>
W1700K-OpenWrt-OC_<构建时间>_r<版本号>
```
