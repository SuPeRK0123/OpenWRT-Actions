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

## 自定义软件包

### 在哪里查找官方包

**官方包搜索页面**（推荐）：
- 包列表：https://openwrt.org/packages/table/start
- 按名称/功能搜索，可以看到包的描述、依赖、大小

**官方 packages 源码仓库**（查看 Makefile 和依赖关系）：
- https://github.com/openwrt/packages
- https://github.com/openwrt/luci

**OpenWrt Wiki**（按功能查找包名）：
- https://openwrt.org/docs/guide-user/start

### LuCI 包的命名规律

LuCI 相关的包名看起来繁杂，但有固定规律：

| 包名格式 | 说明 | 是否需要手动添加 |
|---|---|---|
| `luci-app-xxx` | 某功能的 LuCI 应用主体，提供菜单和配置页面 | ✅ 需要 |
| `luci-i18n-xxx-zh-cn` | 对应应用的中文翻译 | ✅ 按需添加 |
| `luci-mod-xxx` | LuCI 框架级模块，如 `luci-mod-admin-full` | ⛔ 通常由 `luci` 自动依赖，不需要手动写 |
| `luci-base` | LuCI 核心库 | ⛔ 自动依赖 |
| `luci-theme-xxx` | 主题 | ✅ 需要时手动添加 |

**实际规则**：编译配置里只需要写 `luci-app-xxx` 和你想要的 `luci-i18n-xxx-zh-cn`，其余 `luci-mod-*` 和 `luci-base` 会通过依赖关系自动拉入，不需要手动填写。

例如要添加 DDNS 功能：

```
CONFIG_PACKAGE_luci-app-ddns=y          # 应用主体，必须
CONFIG_PACKAGE_luci-i18n-ddns-zh-cn=y   # 中文翻译，可选
# luci-mod-admin-full、luci-base 等不需要写，自动依赖
```

### 如何添加想要编译进去的包

**第一步**：在上面的官方包搜索页面找到包名，确认包存在于当前 OpenWrt 版本。

**第二步**：编辑对应平台的 config 文件（`config/x86_64.config` 或 `config/mt7981.config`），添加一行：

```
CONFIG_PACKAGE_包名=y
```

一个包有三种状态：

```bash
CONFIG_PACKAGE_xxx=y            # 编译进固件
# CONFIG_PACKAGE_xxx is not set  # 明确排除
                                 # 不写 = 使用默认值（通常不包含）
```

**第三步**：提交后手动触发编译即可。

**示例**：添加 vnStat 流量统计

```
# 在 config/x86_64.config 末尾添加
CONFIG_PACKAGE_luci-app-vnstat2=y
CONFIG_PACKAGE_luci-i18n-vnstat2-zh-cn=y
```

**管理可选包的推荐方式**：用注释标注，需要时取消注释：

```
# ── 可选包，按需取消注释 ────────────────────────

# 流量统计
# CONFIG_PACKAGE_luci-app-vnstat2=y
# CONFIG_PACKAGE_luci-i18n-vnstat2-zh-cn=y

# NAT 穿透
# CONFIG_PACKAGE_luci-app-natmap=y

# SQM QoS 队列管理
# CONFIG_PACKAGE_luci-app-sqm=y
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

## 内核补丁与驱动

### 背景知识

OpenWrt 的补丁分两层，作用完全不同：

| 类型 | 路径 | 用途 |
|---|---|---|
| 设备树补丁（DTS） | `target/linux/{平台}/dts/` | 描述硬件布局，如分区大小、GPIO 定义 |
| 内核驱动补丁（.patch） | `target/linux/{平台}/patches-{内核版本}/` | 修改或新增内核功能、驱动支持 |

### 为什么嵌入式设备需要专用驱动

x86 平台有几十年的标准化积累，Linux 内核主线已内置几乎所有主流网卡驱动。嵌入式 SoC（如 MT7981、MT7986）把 CPU、交换芯片、Wi-Fi 射频全部集成在一块芯片上，这些外设走芯片内部私有总线，驱动必须由芯片厂商（如 MediaTek）单独提供并推进合并进内核主线。因此编译嵌入式固件时必须明确指定设备 profile，由 OpenWrt 的构建系统自动引入对应的驱动和固件文件。

Wi-Fi 驱动分两种：

| 驱动类型 | 说明 | 代表 |
|---|---|---|
| 开源驱动 | 已合并进 Linux 主线，OpenWrt 官方支持 | `mt76`（MT7915/MT7981 等） |
| 闭源私有驱动 | MediaTek 内部维护，性能更强但不公开源码 | padavanonly 等社区维护的补丁 |

ImmortalWrt 等第三方固件使用的私有驱动以**内核补丁**形式打入源码树一起编译，不是直接给二进制 `.ko` 文件，但补丁内容依赖 MediaTek 提供的私有代码，因此本仓库不引入，使用官方开源驱动。

### 如何在编译时应用补丁

在 `.github/workflows/build-openwrt.yml` 的 `Add custom packages` 步骤之后，`Load config fragment` 之前，加入一个独立步骤：

```yaml
      - name: Apply patches
        working-directory: openwrt
        run: |
          # 示例一：用 sed 直接修改设备树文件（修改分区大小）
          sed -i 's/reg = <0x5c0000 0x7000000>/reg = <0x5c0000 0x7a40000>/' \
            target/linux/mediatek/dts/mt7981b-cudy-tr3000-v1.dts

          # 示例二：把仓库里的 .patch 文件复制到内核补丁目录
          # cp ../patches/0001-fix-something.patch \
          #   target/linux/mediatek/patches-6.6/

          # 示例三：用 patch 命令直接应用（适合修改非内核文件）
          # patch -p1 < ../patches/my-fix.patch
```

如果补丁较多，建议在仓库根目录新建 `patches/` 目录统一存放 `.patch` 文件，workflow 里循环应用：

```yaml
          # 批量应用 patches/ 目录下所有补丁
          for f in ../patches/*.patch; do
            patch -p1 < "$f" && echo "Applied: $f" || echo "Failed: $f"
          done
```

### .patch 文件从哪里来

**方法一：从社区仓库直接取**

在 GitHub 上搜索对应平台的补丁仓库，直接下载 `.patch` 文件放入 `patches/` 目录。

**方法二：自己生成**

在本地修改源码后，用 `git diff` 生成：

```bash
# 在 openwrt 源码目录里
git diff > ../patches/0001-my-change.patch
```

**方法三：从提交历史生成**

```bash
# 把某个 commit 导出为 patch 文件
git format-patch -1 <commit-hash>
```

### 补丁应用顺序的注意事项

多个补丁之间可能有依赖关系，文件名通常以数字开头保证顺序（`0001-`、`0002-`）。如果补丁 B 修改的代码依赖补丁 A 的改动，必须先应用 A。应用失败时 `patch` 命令会输出 `FAILED` 并生成 `.rej` 文件，说明源码和补丁不匹配，通常是 OpenWrt 版本升级后源码变动导致的，需要手动调整补丁内容。

## 参考

- [OpenWrt 官方](https://openwrt.org)
- [sing-box 官方文档](https://sing-box.sagernet.org)
- [argon 主题](https://github.com/jerrykuku/luci-theme-argon)
