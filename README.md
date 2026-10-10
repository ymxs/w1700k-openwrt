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

---

## 🧪 honk（eBPF 透明代理，实验性）

固件已内置 [honk](https://github.com/daeuniverse/honk) —— 基于 eBPF 的透明代理（dae 血统，sing-box 风格出站组 + Clash 兼容 API），版本 **v0.0.1.beta.84**（上游预编译 aarch64 musl 二进制，已内嵌 eBPF object）。

- **默认状态**：开机自启；但默认配置为 `direct-only`（`fallback: direct`），**不改变任何流量路径**，仅用于验证 datapath 能正常起来。
- **配置文件**：`/etc/honk/config.dae`（OpenWrt conffile，sysupgrade 会保留本地修改）
- **服务控制**：`/etc/init.d/honk {start|stop|restart|reload|enable|disable}`
- **Clash API**：`127.0.0.1:9091`，external-ui 为 `yacd`
  - 端口特意避让已内置的 OpenClash（9090）
  - ⚠️ honk 与 OpenClash **同为透明代理，请勿同时运行**，切换前请先停用另一个
- **运行时**：数据目录 `/var/lib/honk`；eBPF map 挂载点 `/sys/fs/bpf`（init 脚本会自动挂载 bpffs）

### 内核依赖

honk 真实 datapath 需要内核 ≥ 6.12（本固件为 6.18）以及 BTF / cgroup-bpf / netns / sched-bpf / nft-queue 支持。这些已在两个 `config.diff` 中通过 `CONFIG_KERNEL_*` 打开（含 `CONFIG_KERNEL_DEBUG_INFO_BTF=y` 并显式 `# CONFIG_KERNEL_DEBUG_INFO_REDUCED is not set`，否则 BTF 无法启用），同时选中对应 `kmod-*`。

### 升级 honk

编辑 `user/default/honk/Makefile` 的 `PKG_VERSION`（不带 `v` 前缀）与 `PKG_HASH`，同步为新版本号的官方 release `sha256`；两者不一致时会在下载校验阶段直接失败。

> honk 仍属 beta（v0.0.1.beta.84），上游 README 明确不建议用于生产环境，请自行评估风险。

---

## 🐞 Bug Investigation Report

详见 [BUG_REPORT.md](BUG_REPORT.md)。结论摘要：

- **CI 编译失败**：已修复 — 上游 `OpenW1700k@ubi2` force-push 导致 `998-flowsense-i18n.patch` 无法应用；已在 `c3b007a` 重新适配，CI run #22 全绿，两个固件 Release 已发布。
- **随机重启**：已定位根因 — DRAM 中存储的 per-CPU 指针 `desc->kstat_irqs` 发生单比特（bit 63）损坏引发的内核 panic；疑似 DRAM 弱位或 OOT 驱动（`mt7996e` WiFi / `airoha_npu` NPU offload）野 DMA 写入。完整寄存器/指令级证据与缓解建议见报告。
- 次要问题：`system info` localtime 偏差 +8 小时（时区被应用两次）；lan2 链路约每 5–6 分钟闪断一次。
