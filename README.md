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
| iStoreOS 内置包 | istoreos/istoreos（`istoreos-25.12` 分支）的 `package/istoreos-files` 与 `package/diy/{dkml,luci-app-ota}` | 构建时由 `custom.sh` 克隆并拷入源码树（与 Aurora 主题相同方式，版本锁定在 istoreos-25.12） |

首次启动行为：

- 默认界面切换为 Argon（iStoreOS 风格 + easepi 图标）；Aurora 主题仍可在 LuCI 中切换
- 时区保持 Asia/Hong_Kong（`99-w1700k-defaults` 在 iStoreOS `09_istoreos` 之后执行，覆盖其默认值）
- 启用 USB 自动挂载（blockmount）、fstab LuCI 菜单、dkml 内核模块管理、OTA 刷机页面

应用商店（linkease/istore）沿用 `custom.sh` 既有的 package 克隆方式接入，未以 feed 形式重复添加。刷入后在 LuCI 侧边栏可见「应用商店」入口；其运行于 apk 模式（固件内置 `/etc/apk/arch`），软件源来自固件自带的 `distfeeds.list`（OpenWrt snapshot feeds）。

---

## 📦 固件版本

| 固件 | 说明 |
| --- | --- |
| `ubi2` | 常规版本，使用标准 CPU 工作参数。**推荐日常使用。** |
| `ubi2-oc` | 超频版本。上游实现为「每个频率比标称电压低 200 MHz 档位」的全程欠压，可能出现随机重启；本仓库已强制 `ondemand` 并把上限收到 1.2 GHz 以降低风险。详见 [BUG_REPORT.md](BUG_REPORT.md) |

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

## 🌐 透明代理：nikki + mihomo（Clash.Meta）

**OpenClash 与 honk 已从本固件中移除**，替换为 [nikki](https://github.com/nikkinikki-org/OpenWrt-nikki)
驱动的 [mihomo](https://github.com/MetaCubeX/mihomo)（Clash.Meta）—— 也就是 Clash Verge Rev /
ClashX Meta / mihomo-party 等客户端所使用的同一个内核。

- **接入方式**：`src-git nikki` 写在 `user/default/feeds.conf` 里。该文件在 CI 中被复制成
  `feeds.conf.default`，**早于** `scripts/feeds update/install` 执行，所以 feed 能被正常解析；
  如果在 `custom.sh` 阶段才 clone 到 `package/` 已经太晚（这正是过去 OpenClash / Aurora 必须
  走 `package/` 的原因）。
- **LuCI 界面**：`服务 → Nikki`（中文界面包 `luci-i18n-nikki-zh-cn` 已勾选）
- **内核**：`/usr/libexec/mihomo`（`/usr/bin/mihomo` 为 alternatives 符号链接），
  由 `mihomo-meta` 用 OpenWrt Go 工具链从源码编译，`GO_PKG_TAGS=with_gvisor`，TUN 模式可用。
  与 `mihomo-alpha` 互斥，二选一。
- **配置**：`/etc/config/nikki`（UCI）；配置文件在 `/etc/nikki/`，订阅落在
  `/etc/nikki/subscriptions/`
- **默认行为**：开机自启但**不改变任何流量路径**，需自行添加节点/订阅后才生效
- **服务控制**：`/etc/init.d/nikki {start|stop|restart|reload|enable|disable}`

### 依赖与内核要求

nikki 需要 **firewall4（fw4）**（本固件 OpenWrt 25.12 + 内核 6.18 满足）、`kmod-nft-tproxy`
（TPROXY）、`kmod-tun`（TUN）、`kmod-nft-socket`、`kmod-dummy`、`kmod-inet-diag`。这些都在两个
`config.diff` 的 nikki 段落里显式勾选。

随着 honk 一起移除的还有它所需的 `CONFIG_KERNEL_DEBUG_INFO` / `DEBUG_INFO_BTF` / CGROUP_BPF /
NET_NS 以及 `kmod-sched-*` / `kmod-veth` / `kmod-nft-queue` —— eBPF 已不再是固件能力的一部分，
内核随之变小。

### 升级 nikki / mihomo

`src-git nikki` 跟踪 `main` 分支，`feeds update` 会自动拉到新版本；mihomo 内核版本则由
`mihomo-meta/Makefile` 的 `PKG_VERSION` 锁定。要升级内核就 fork 该 feed 并改 `PKG_VERSION`
与 `PKG_SOURCE_VERSION`，把 `feeds.conf` 指向自己的 fork。若希望内核像 OpenClash 那样在
**运行时下载**（不烤进固件），需要另行改造 `mihomo-meta` 的 Makefile，去掉 `nikki` 对 `+mihomo`
的强依赖 —— 当前实现是编译期静态链接进镜像。

---

## 🐞 Bug Investigation Report

详见 [BUG_REPORT.md](BUG_REPORT.md)。结论摘要：

- **CI 编译失败**：已修复 — 上游 `OpenW1700k@ubi2` force-push 导致 `998-flowsense-i18n.patch` 无法应用；已在 `c3b007a` 重新适配，CI run #22 全绿，两个固件 Release 已发布。
- **随机重启**：**根因已确认并已修复** — `ubi2-oc` 超频分支并非单纯提高频率，而是把 PM-domain 的 PLL 公式从 `500 + state*50` 改成 `700 + state*50`，同时把 DTS 的 OPP 表整体上移 200 MHz，**但 ATF 的电压表没有同步提高**。结果是**每一个频率都在跑「低 200 MHz 档位的电压」= 全程欠压**；又叠加该分支把默认 governor 改成 `performance`，把 4 核永久钉在 1.4 GHz / 欠压上。欠压在这种最边缘状态下产生随机单比特内存翻转，内核在 arch timer tick 路径上读到被破坏的指针后 panic — 即「无故重启」。详见 [BUG_REPORT.md](BUG_REPORT.md) 的 2026-10-15 更新。
- **设备连不上、拿不到 IP（但老设备正常）**：已修复 — 不是 WiFi 问题。LuCI 的「网络 → DHCP/DNS」页面里的 `recordtypes` 列表含 `ALIAS`，而这个类型 dnsmasq 2.93 不接受（`bad RR type`），于是 dnsmasq 起不来、进入 procd 崩溃循环，**整个 LAN 的 DHCP 与 DNS 全挂**。已持有租约的老设备照常工作，新连上来的设备就卡在「正在获取 IP 地址」。详见 [BUG_REPORT.md](BUG_REPORT.md) 的 Bug 5。
- 次要问题：`system info` localtime 偏差 +8 小时（时区被应用两次）；lan2 链路约每 5–6 分钟闪断一次。

---

## 🛡️ 稳定性守护（本次新增）

针对上述根因新增了两个开机强制执行的守护脚本，**不修改上游任何文件**（避免再次被上游 force-push 打断），只通过本仓库的 overlay 在运行时覆盖：

| 组件 | 作用 |
| --- | --- |
| `/etc/init.d/w1700k-cpufreq` | 每次开机强制 governor 与频率限制；结果写入 syslog，`logread -e w1700k-cpufreq` 可查 |
| `/etc/config/w1700k` | 参数配置。标准版 `governor=ondemand`、不限频 |
| `ubi2-oc` 专属覆盖 | `governor=ondemand` + `max_freq=1200000`，不再停留在最激进的两个超频档位 |
| `/etc/init.d/w1700k-crashlog` | 每次开机把 `/sys/fs/pstore` 里幸存的 panic 转储复制到**持久化**的 `/etc/w1700k-crashlog/`，并在 `events.log` 追加一行记录上一轮开机撑到多少秒才崩 |
| `/usr/sbin/w1700k-heartbeat` | 每分钟心跳，写在 `/tmp`（内存），几乎不增加 flash 擦写 |
| `/etc/init.d/w1700k-dnsmasq-guard` | 开机（S18，早于 dnsmasq 的 S19）校验 dnsmasq 配置，自动移除它不接受的 `cache_rr` 类型 |

想恢复完整 1.4 GHz 超频：`uci set w1700k.cpufreq.max_freq=1400000 && uci commit w1700k && /etc/init.d/w1700k-cpufreq restart`

> ⚠️ 由于 `ubi2-oc` 的每个频率都处于欠压状态（并非只有最高频），限频只能降低边缘程度、无法完全消除风险。**已出现过 panic 的设备建议改用标准版 `ubi2` 固件**作为日常固件。

---

## 🔎 复现与取证

崩溃后无需人工干预即可取证：

```sh
logread -e w1700k-cpufreq          # 查看当前 governor / 频率是否已生效
cat /etc/w1700k-crashlog/events.log # 每次重启一行：CRASH(上一轮撑了多久) / BOOT
ls /etc/w1700k-crashlog/            # 完整 panic 转储（含寄存器与调用栈）
tail -20 /tmp/w1700k-heartbeat.log  # 最近一分钟级心跳
```

---

## 🚑 常见故障速查

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 新设备连上 WiFi 但一直「正在获取 IP 地址」，老设备正常 | dnsmasq 配置非法 → 崩溃循环 → 无 DHCP/DNS | `dnsmasq --test -C /var/etc/dnsmasq.conf.cfg01411c`；`/etc/init.d/w1700k-dnsmasq-guard start`；`/etc/init.d/dnsmasq restart` |
| 网页保存「网络 → DHCP/DNS」后就断网 | 同上（`cache_rr` 被写入 `ALIAS`） | 同上；本固件已从 LuCI 列表移除该类型 |
| 重启过于频繁 | 见上文调频部分 | `logread -e w1700k-cpufreq` 确认 governor 为 `ondemand` |

排查 dnsmasq 是否活着，一条命令即可：

```sh
dnsmasq --test -C /var/etc/dnsmasq.conf.cfg01411c   # 配置是否合法
netstat -lnup | grep ':67'                          # DHCP 是否在监听
wc -l /tmp/dhcp.leases                              # 有没有在下发租约
```
