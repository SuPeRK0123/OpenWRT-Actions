# OpenWRT-Actions

基于 GitHub Actions 的 OpenWrt 自动编译仓库，支持 x86_64 和 MT7981（Cudy TR3000）平台，每日自动跟踪官方最新版本。

## 特性

- 自动检测 OpenWrt 官方新版本并触发编译
- 支持主路由 / 旁路由模式参数化
- 内置 sing-box、ddns-scripts、ttyd、argon 主题
- 编译产物自动发布到 GitHub Releases

## 支持的平台

| 平台 | 设备 | 架构 |
|---|---|---|
| x86_64 | 通用 x86 软路由 / 虚拟机 | x86_64 |
| mt7981 | Cudy TR3000 | aarch64（MediaTek Filogic） |

## 内置软件包

| 软件包 | 说明 |
|---|---|
| sing-box | 通用代理平台（编译进固件，apk 源已预配置） |
| ddns-scripts + ddns-scripts-cloudflare | 动态 DNS，支持 Cloudflare |
| ttyd | 浏览器终端 |
| luci-theme-argon | Argon 主题 |

## 手动触发编译

进入 Actions → Build OpenWrt → Run workflow，填写参数：

| 参数 | 说明 | 示例 |
|---|---|---|
| `target` | 编译目标 | `x86_64` / `mt7981` |
| `version` | OpenWrt 版本 tag | `v25.12.2` |
| `mode` | 路由模式 | `main`（主路由）/ `bypass`（旁路由） |
| `lan_ip` | LAN IP 地址 | `192.168.1.1` |
| `gateway` | 上游网关（旁路由模式填写） | `192.168.1.1` |
| `rootfs_size` | Root 分区大小 MB，仅 x86_64 | `512` |

## 自动更新

每天北京时间 10:00 自动检测 OpenWrt 官方最新 release，有新版本时自动触发 x86_64 和 mt7981 两个平台的编译并发布 Release。

## 固件下载

前往 [Releases](../../releases) 页面下载。

### x86_64 文件说明

| 文件名 | 启动方式 | 文件系统 | 推荐场景 |
|---|---|---|---|
| `*-ext4-combined-efi.img.gz` | UEFI | ext4 | 日常使用首选 |
| `*-squashfs-combined-efi.img.gz` | UEFI | squashfs | 折腾期支持恢复出厂 |
| `*-ext4-combined.img.gz` | Legacy BIOS | ext4 | 旧主机 |

### MT7981（TR3000）文件说明

| 文件名 | 用途 |
|---|---|
| `*-squashfs-sysupgrade.bin` | 刷机 / 升级，通过 LuCI 或 sysupgrade 命令使用 |

## 刷机后配置 sing-box

固件内已预配置 sing-box 官方 apk 源，联网后执行：

```bash
apk update
apk add sing-box
/etc/init.d/sing-box enable
/etc/init.d/sing-box start
```

后续升级：

```bash
apk upgrade sing-box
```

## 自定义软件包

编辑 `config/x86_64.config` 或 `config/mt7981.config`，取消注释或添加需要的包，提交后手动触发编译即可。

## 添加第三方包

在 `.github/workflows/build-openwrt.yml` 的 `Add custom packages` 步骤中添加 `git clone`，然后在对应 config 里启用：

```yaml
- name: Add custom packages
  working-directory: openwrt
  run: |
    git clone --depth=1 https://github.com/xxx/yyy package/yyy
```

## 目录结构

```
.
├── .github/
│   └── workflows/
│       ├── build-openwrt.yml   # 编译 workflow
│       └── check-update.yml    # 自动检测新版本
├── config/
│   ├── x86_64.config           # x86_64 配置片段
│   └── mt7981.config           # MT7981 配置片段
├── version.txt                 # 当前跟踪的版本号
└── README.md
```

## 参考

- [OpenWrt 官方](https://openwrt.org)
- [sing-box 官方文档](https://sing-box.sagernet.org)
- [argon 主题](https://github.com/jerrykuku/luci-theme-argon)
