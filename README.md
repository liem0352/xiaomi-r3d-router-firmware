<img src="assets/readme/hero.svg" alt="小米 R3D OpenWrt 路由模式固件 — 预配置路由模式，恢复出厂保持，LAN 192.168.1.1，DHCP 启用，WiFi 加密" width="100%" />

## 默认配置与拓扑

小米 R3D 出厂刷入 Kwrt 后默认以 AP 模式（旁路由）启动。本固件在镜像构建阶段预配置为路由模式，首次启动即为独立路由器，恢复出厂后仍保持。

### 物理拓扑

```
   光猫 / 入户网线
          │
          │ WAN (eth0) · DHCP 客户端
     ┌────┴────┐
     │   R3D   │  LAN 192.168.1.1
     │  路由器  │  DHCP 192.168.1.100-250
     └──┬──┬──┬┘
        │  │  │  LAN (eth1 / eth2 / eth3)
        │  │  └─ 设备
        │  └──── 交换机
        └─────── PC
```

### 默认配置

| 项 | 值 |
| --- | --- |
| LAN IP | 192.168.1.1 |
| WAN | eth0，DHCP 客户端 |
| DHCP 服务器 | 启用，192.168.1.100 - 192.168.1.250 |
| WiFi 5G SSID | Xiaomi_R3D_5G |
| WiFi 2.4G SSID | Xiaomi_R3D_2.4G |
| WiFi 密码 | 12345678（WPA2-PSK） |
| 管理密码 | 空（首次登录后请设置） |
| NTP | ntp.aliyun.com / time1.cloud.tencent.com |

### 构建方法

| 方法 | 适用环境 | 入口 |
| --- | --- | --- |
| Docker 构建 | 任意装了 Docker 的系统 | `docker-build.sh` |
| WSL 构建 | Windows（推荐） | `wsl-build.sh` |
| GitHub Actions 云构建 | Fork 后免费云构建 | `.github/workflows/` |
| Linux 本地构建 | Debian/Ubuntu | ImageBuilder + `make image` |

## 它解决什么

Kwrt 固件默认 AP 模式带来四个问题：电脑无法获取 IP、WiFi 开放且连上级路由 IP、无法作为独立路由器使用、恢复出厂后丢失路由配置。本固件在镜像层解决这四个问题。

## 为什么不同

- **预配置路由模式** — 在 ImageBuilder 阶段把 `network` / `firewall` / `dhcp` 配置打包进镜像，首次启动即路由模式，不是 AP 模式。
- **恢复出厂仍保持** — 通过 `uci-defaults` 首次启动脚本（`r3d_router_config/etc/uci-defaults/99-router-mode`），每次出厂复位后重新写入路由模式配置并写入标记文件避免重复执行。
- **DHCP 默认启用** — LAN 桥接 `br-lan`，地址池 192.168.1.100-250，电脑接上即获 IP。
- **WiFi 默认加密** — 5G / 2.4G 双频 SSID 默认 WPA2-PSK 加密，不再开放。

## 工作原理

### 首次启动脚本

`r3d_router_config/etc/uci-defaults/99-router-mode` 在系统首次启动或恢复出厂后执行一次：

1. 配置 WAN 为 eth0 DHCP 客户端
2. 配置 LAN 桥接 `br-lan`，地址 192.168.1.1/24
3. 启用 DHCP 服务器，地址池 100-250
4. 启用 WiFi 5G / 2.4G，设置 SSID 与 WPA2 密码
5. 清空 root 密码
6. 写入标记文件，避免重复执行

### 网络配置

`r3d_router_config/etc/config/network`：

```
WAN: eth0, proto dhcp
LAN: br-lan, 192.168.1.1/24
```

### 防火墙

`r3d_router_config/etc/config/firewall`：

```
NAT: 启用
LAN → WAN: 允许转发
```

## 如何使用

<img src="assets/readme/section-build.svg" alt="构建方法 — 四条路径：Docker、WSL、GitHub Actions、Linux" width="100%" />

### 1. 构建固件

**Docker 构建（最简单）**

```bash
sh docker-build.sh
```

**WSL 构建（Windows 推荐）**

```bash
sh wsl-build.sh
```

**GitHub Actions 云构建**

1. Fork 本仓库
2. 进入仓库 Actions 页
3. 选择对应 workflow
4. Run workflow
5. 构建完成后在 Artifacts 下载固件

**Linux 本地构建**

```bash
sudo apt install build-essential libncurses-dev libssl-dev
# 下载 OpenWrt ImageBuilder
# 复制 r3d_router_config/ 到 ImageBuilder 目录
make image
```

### 2. 刷机步骤

1. 网线连接电脑到 R3D LAN 口
2. SSH 登录路由器：
   ```bash
   ssh root@192.168.1.1
   ```
3. 上传固件：
   ```bash
   scp firmware.bin root@192.168.1.1:/tmp/
   ```
4. 刷机（`-n` 不保留旧配置）：
   ```bash
   sysupgrade -n /tmp/firmware.bin
   ```
5. 等待重启完成，约 2-3 分钟

### 3. 系统内升级

```bash
cd /tmp
wget http://your-server/firmware.bin
sysupgrade -n firmware.bin
```

### 4. 修复已刷入的 Kwrt 固件

若已刷入 AP 模式的 Kwrt，无需重刷固件，用快速修复脚本：

```bash
scp fix_r3d_router_mode.sh root@192.168.1.1:/tmp/
ssh root@192.168.1.1 'sh /tmp/fix_r3d_router_mode.sh'
```

## 故障排除

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| 电脑无法获取 IP | DHCP 未启用 / 接错 WAN 口 | 确认网线接 LAN 口，检查 DHCP 服务 |
| 无法访问 192.168.1.1 | 电脑 IP 不在同段 | 手动设静态 IP 192.168.1.x |
| 无法上网 | WAN 未获取地址 | 检查光猫到 R3D WAN 口网线 |
| WiFi 无法连接 | 密码错误 | 默认密码 12345678 |
| SSH 连接失败 | root 密码已改 | 恢复出厂后默认空密码 |
| 恢复出厂后变 AP 模式 | 未使用本固件 | 刷入本固件或运行修复脚本 |

## 配置文件说明

| 路径 | 作用 |
| --- | --- |
| `r3d_router_config/etc/config/network` | WAN eth0 DHCP / LAN br-lan 192.168.1.1 |
| `r3d_router_config/etc/config/firewall` | NAT 启用 / LAN→WAN 允许 |
| `r3d_router_config/etc/config/dhcp` | DHCP 服务器启用 |
| `r3d_router_config/etc/uci-defaults/99-router-mode` | 首次启动自动配置路由模式 |
| `docker-build.sh` | Docker 构建入口 |
| `wsl-build.sh` | WSL 构建入口 |
| `.github/workflows/` | GitHub Actions 云构建 |
| `fix_r3d_router_mode.sh` | 已刷固件快速修复脚本 |

## 安全建议

- 设 root 密码：`passwd`
- 修改 WiFi 密码（默认 12345678 仅为占位）
- 启用 HTTPS 管理

## 技术支持

- [OpenWrt 官方文档](https://openwrt.org/docs/start)
- [OpenWrt 中文文档](https://openwrt.org/zh/docs/start)
- [恩山无线论坛](https://www.right.com.cn/forum/)
- [GitHub Issues](https://github.com/liem0352/xiaomi-r3d-router-firmware/issues)

## License

GPL v2，基于 OpenWrt。

## 致谢

- [OpenWrt](https://openwrt.org/)
- Kwrt
- 恩山论坛
