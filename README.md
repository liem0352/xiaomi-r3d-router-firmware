# 小米 R3D OpenWrt 路由模式固件

## 📋 问题描述

Kwrt 固件默认以 AP 模式（旁路由模式）启动，导致：
- 电脑无法获取 IP 地址
- WiFi 开放且连接到上级路由 IP
- 无法作为独立路由器使用
- 恢复出厂设置后丢失配置

## ✅ 解决方案

本方案提供**预配置为路由模式**的固件，确保：
- 首次启动即为路由模式
- **恢复出厂设置后仍保持路由模式**
- DHCP 服务器默认启用
- WiFi 默认启用并加密

## 📁 文件说明

| 文件/目录 | 说明 |
|-----------|------|
| `docker-build.sh` | Docker 构建脚本（推荐 Windows 用户） |
| `wsl-build.sh` | WSL 构建脚本（推荐 WSL 用户） |
| `.github/workflows/` | GitHub Actions 云构建配置 |
| `r3d_router_config/` | 配置文件目录 |
| `fix_r3d_router_mode.sh` | 快速修复脚本（已刷机设备使用） |

## 🔧 默认配置

- **LAN IP**: 192.168.1.1
- **WAN 模式**: DHCP 客户端
- **DHCP 服务器**: 启用 (地址池: 192.168.1.100-250)
- **WiFi SSID**: Xiaomi_R3D_5G / Xiaomi_R3D_2.4G
- **WiFi 密码**: 12345678
- **管理密码**: 空（首次登录无需密码）
- **NTP 服务器**: 阿里云/腾讯云

## 🚀 构建方法（任选一种）

### 方法一：Docker 构建（最简单，推荐）

**前提条件**: 安装 [Docker Desktop](https://www.docker.com/products/docker-desktop)

```bash
# 在 PowerShell 或 Git Bash 中运行
bash docker-build.sh
```

构建完成后，固件会保存在 `output_时间戳/` 目录。

### 方法二：WSL 构建（推荐 Windows 用户）

**前提条件**: 安装 WSL2 + Ubuntu

```bash
# 在 WSL 终端中运行
bash wsl-build.sh
```

构建完成后会自动打开 Windows 资源管理器显示输出目录。

### 方法三：GitHub Actions 云构建（无需本地环境）

1. Fork 本仓库到你的 GitHub 账号
2. 进入 Actions 页面
3. 选择 "Build Xiaomi R3D Router Mode Firmware"
4. 点击 "Run workflow"
5. 等待构建完成，下载固件

### 方法四：Linux 本地构建

```bash
# 1. 安装依赖
sudo apt-get update
sudo apt-get install -y build-essential git wget python3

# 2. 下载 ImageBuilder
wget https://downloads.openwrt.org/releases/23.05.5/targets/ipq806x/generic/openwrt-imagebuilder-23.05.5-ipq806x-generic.Linux-x86_64.tar.xz
tar -xJf openwrt-imagebuilder-23.05.5-ipq806x-generic.Linux-x86_64.tar.xz
cd openwrt-imagebuilder-23.05.5-ipq806x-generic.Linux-x86_64

# 3. 复制配置文件
mkdir -p files/etc/config files/etc/uci-defaults
cp ../r3d_router_config/etc/config/* files/etc/config/
cp ../r3d_router_config/etc/uci-defaults/* files/etc/uci-defaults/

# 4. 构建固件
make image PROFILE="xiaomi_r3d" PACKAGES="luci luci-i18n-base-zh-cn" FILES="files/"
```

## 📝 刷机步骤

### 首次刷机

1. **连接路由器**
   - 电脑网线连接到 R3D 的 LAN 口
   - 如果无法获取 IP，手动设置电脑 IP 为 192.168.1.10/24

2. **SSH 登录**
   ```bash
   ssh root@192.168.1.1
   ```

3. **上传固件**
   ```bash
   scp openwrt-23.05.5-r3d-router-mode-sysupgrade.bin root@192.168.1.1:/tmp/
   ```

4. **执行刷机**
   ```bash
   sysupgrade -n /tmp/openwrt-23.05.5-r3d-router-mode-sysupgrade.bin
   ```

5. **等待重启**（约 2-3 分钟）

### 系统内升级

如果已经在运行 OpenWrt：

```bash
# 下载固件到 /tmp
wget -O /tmp/firmware.bin [固件URL]

# 执行升级
sysupgrade -n /tmp/firmware.bin
```

## 🔧 修复已刷入的 Kwrt 固件

如果你已经刷入了 Kwrt 但配置不正确：

```bash
# 1. SSH 登录路由器
ssh root@192.168.1.1

# 2. 上传修复脚本
scp fix_r3d_router_mode.sh root@192.168.1.1:/tmp/

# 3. 执行修复
sh /tmp/fix_r3d_router_mode.sh

# 4. 等待 30 秒后重新连接
```

## 🔌 物理连接

**路由模式连接方式：**
```
光猫/入户网线 → R3D WAN 口 (eth0)
R3D LAN 口 (eth1/eth2/eth3) → 电脑/交换机
```

## ⚠️ 故障排除

| 问题 | 解决方案 |
|------|----------|
| 无法获取 IP | 检查网线是否插在 LAN 口；手动设置电脑 IP 为 192.168.1.10/24 |
| 无法访问后台 | 确保电脑和路由器在同一网段；尝试 http://192.168.1.1 |
| 无法上网 | 检查 WAN 口是否连接到上级网络；检查 WAN 口是否获取到 IP |
| WiFi 无法连接 | 检查无线是否已启用；尝试重新配置无线 |
| SSH 连接失败 | 检查是否开启 SSH；尝试 telnet |
| 恢复出厂设置后变 AP 模式 | 使用本固件，已修复此问题 |

## 📂 配置文件说明

### 关键配置文件

- `r3d_router_config/etc/config/network` - 网络接口配置
  - WAN 口: eth0, DHCP 客户端
  - LAN 口: br-lan (eth1+eth2+eth3), 192.168.1.1
  - DHCP: 启用

- `r3d_router_config/etc/config/firewall` - 防火墙配置
  - NAT: 启用
  - 转发: LAN → WAN 允许

- `r3d_router_config/etc/uci-defaults/99-router-mode` - 首次启动脚本
  - 自动配置为路由模式
  - 配置 WiFi
  - 清空 root 密码
  - 创建标记文件防止重复配置

## 💡 固件特点

- ✅ 预配置为路由模式
- ✅ 恢复出厂设置后**仍保持路由模式**
- ✅ 首次启动自动配置
- ✅ 内置中文语言包 (luci-i18n-base-zh-cn)
- ✅ WiFi 默认启用并加密
- ✅ 默认密码为空（方便首次登录）
- ✅ 使用国内 NTP 服务器

## 🔒 安全建议

首次登录后请立即：

1. **设置 root 密码**
   ```bash
   passwd
   ```

2. **修改 WiFi 密码**
   - 进入 网络 → 无线 → 修改

3. **启用 HTTPS 管理**
   ```bash
   opkg update
   opkg install luci-ssl
   ```

## 📞 技术支持

- OpenWrt 官方文档: https://openwrt.org/docs/start
- OpenWrt 中文文档: https://openwrt.org/zh/docs/start
- 恩山无线论坛: https://www.right.com.cn/forum/
- GitHub Issues: [提交问题](https://github.com/your-repo/issues)

## 📄 许可证

本项目基于 OpenWrt 开源项目，遵循 GPL v2 许可证。

## 🙏 致谢

- [OpenWrt](https://openwrt.org/) 开源项目
- [Kwrt](https://op.supes.top/) 固件源
- 恩山无线论坛的各位开发者
