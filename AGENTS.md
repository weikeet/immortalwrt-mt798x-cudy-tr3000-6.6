# AGENTS.md — Cudy TR3000 ImmortalWrt 固件项目

本文件是 AI 编程助手的规范参考。它记录了项目的架构、关键约定、文件语义和工作流，
帮助助手准确理解代码库并做出安全、符合上下文的修改。

---

## 1. 项目概述

**目标：** 为 Cudy TR3000 v1 路由器（MediaTek MT7981 / Filogic 820，Linux 6.6 内核）
构建一个精简、保持上游更新的 ImmortalWrt 24.10 固件，集成闭源硬件加速，并通过
GitHub Release 发布。

整个构建在 GitHub Actions 中完成。本仓库只包含**构建配方** —— 配置、自定义脚本、
文件覆写和工作流，不含 OpenWrt 源码树。

| 属性               | 值                                                                      |
|--------------------|-------------------------------------------------------------------------|
| 目标设备           | Cudy TR3000 v1                                                          |
| 上游源码           | [padavanonly/immortalwrt-mt798x-24.10](https://github.com/padavanonly/immortalwrt-mt798x-24.10) |
| 上游分支           | `openwrt-24.10-6.6`                                                     |
| 内核               | Linux 6.6                                                               |
| 目标三元组         | `mediatek/filogic`                                                      |
| 子目标架构         | `aarch64_cortex-a53`                                                    |
| 硬件加速           | MTK 闭源 MT Wi-Fi + 有线加速                                            |
| 镜像文件名前缀     | `{YYYYMMDD}-24.10-6.6-{device}-{version}-{hash}-rootfs`                |
| 许可证             | MIT                                                                     |

---

## 2. 仓库布局

```
.
├── .github/workflows/
│   ├── openwrt-builder.yml     # 主入口 — 触发 image-builder.yml
│   ├── image-builder.yml       # 可复用的编译工作流（由 openwrt-builder 调用）
│   ├── update-checker.yml      # 定时检查上游源码变更的 Cron 工作流
│   └── clean-up.yml            # 清理工作流运行记录和 Release
├── files/                      # 文件系统覆写 → 打包进固件镜像
│   └── etc/opkg/distfeeds.conf # 自定义 opkg 软件源地址
├── cudy-tr3000.config          # 完整的 .config（由 `make menuconfig` 生成）
├── cudy-tr3000.sh              # DIY part 2 — feeds install 之后的自定义脚本
├── cudy-tr3000.md              # GitHub Release 说明 / 变更日志
├── diy-part1.sh                # DIY part 1 — feeds update 之前的自定义脚本
├── diy-part2.sh                # 占位模板（未使用；cudy-tr3000.sh 替代了它）
├── FM350-GL.md                 # Fibocom FM350-GL 5G 模组文档与 Q&A
├── README.md                   # 项目级别的用户文档
├── .config                     # 空的占位文件；构建时会覆盖
├── AGENTS.md                   # 本文件
└── LICENSE                     # MIT
```

### 2.1 工作流语义

CI 采用**两层编排模式**：

1. **`openwrt-builder.yml`** — 编排器。`setup` job 从 env 变量读取各项参数（源码 URL、分支、
   配置文件路径等），通过 workflow outputs 传递给 `image-builder.yml`。构建完成后，
   `update-cache` job 将上游仓库的 commit hash 写入 GitHub Actions 缓存键，
   供 `update-checker.yml` 检测变更。

2. **`image-builder.yml`** — 可复用的构建引擎。通过 `workflow_call` inputs 接收所有参数，
   执行标准 OpenWrt 构建流程（clone → feeds update → feeds install → 配置覆写 →
   `make download` → `make`），并将固件产物发布为 CI artifact 和/或 GitHub Release。
   输出上游源码 commit hash（`repo_hash`）供缓存键链式使用。

3. **`update-checker.yml`** — 按 Cron 调度执行（`0 0 */5 * *`，每 5 天一次），也可通过
   `workflow_dispatch` 手动触发。查询上游仓库（`padavanonly/immortalwrt-mt798x-24.10`，
   分支 `openwrt-24.10-6.6`）的最新 commit hash。若与缓存值不同，则触发主构建工作流。
   同时监控上游 `defconfig` 路径（`defconfig/mt7981-ax3000.config`），检测到变更时创建 Issue。

4. **`clean-up.yml`** — 在 `update-checker.yml` 结束时调用。删除旧的工作流运行记录
   （保留最新 2 个），并可选择清理旧的 GitHub Release（保留 1 个）。

### 2.2 构建脚本

| 脚本             | 阶段                        | 用途                                                                 |
|------------------|-----------------------------|----------------------------------------------------------------------|
| `diy-part1.sh`   | 在 `feeds update` 之前      | 当前为存根（注释掉了未使用的 feed）。在此添加 `src-git` feed 覆写。     |
| `cudy-tr3000.sh` | 在 `feeds install` 之后     | 真正的自定义脚本，替代上游的 `diy-part2.sh`。                          |
| `diy-part2.sh`   | 占位模板                    | 来自 P3TERX 上游的模板文件；本仓库未使用。                             |

核心文件是 **`cudy-tr3000.sh`**，按顺序执行以下操作：

1. **主题**：从 feeds 中移除 `luci-theme-argon`，将 `jerrykuku/luci-theme-argon` +
   `jerrykuku/luci-app-argon-config` 克隆到 `package/` 目录。
2. **Golang**：将 feeds 中的 `golang` 包替换为 `sbwml/packages_lang_golang`（26.x 分支）。
3. **Passwall 相关包**：移除多个 feed 中的网络包，克隆 `Openwrt-Passwall/openwrt-passwall-packages`。
4. **Passwall2**：移除 feed 中的 `luci-app-passwall`，克隆 `Openwrt-Passwall/openwrt-passwall2`。
5. **默认 IP**：修改 `package/base-files/files/bin/config_generate`，将 LAN IP 设为 `192.168.10.1`。
6. **镜像前缀**：修改 `include/image.mk`，为镜像文件名添加日期时间前缀（`YYYYMMDD-24.10-6.6-`）。

### 2.3 配置文件

`cudy-tr3000.config` 是由 `make menuconfig` 生成的完整 `.config` 文件。关键选择：

- `CONFIG_TARGET_mediatek=y` / `CONFIG_TARGET_mediatek_filogic=y`
- 设备：`CONFIG_TARGET_MULTI_PROFILE=y`，包含 `cudy_tr3000_v1`
- `CONFIG_PACKAGE_luci-app-passwall2=y`（仅 xray，不含 ss/v2ray 插件）
- `CONFIG_PACKAGE_luci-app-argon-config=y`
- `CONFIG_PACKAGE_luci-app-modem=y` + `luci-app-sms-tool=y` + `sms-tool=y` + `sendat=y`
- MTK 硬件加速：`CONFIG_MTK_HSDMA=y`、`CONFIG_PACKAGE_kmod-mtk_hnat=y`
- USB 支持：`kmod-usb2`、`kmod-usb3`、`kmod-usb-net`、`kmod-usb-serial-*`
- 5G 模组内核模块：RNDIS、USB serial、USB net

### 2.4 文件系统覆写

`files/` 目录镜像路由器的根文件系统。放在此目录下的文件在构建时会被复制到固件镜像中，
覆盖上游源码的默认文件。

目前仅包含 `files/etc/opkg/distfeeds.conf`，将 opkg 软件源指向
`mirrors.vsean.net/openwrt/releases/24.10-SNAPSHOT`，涵盖 `core`、`base`、`luci`、
`packages`、`routing` 和 `telephony` 六个 feed。

---

## 3. 关键约定（面向助手）

### 3.1 添加新包的位置

- **内核模块 / 硬件支持**：在 `cudy-tr3000.config` 中添加 `CONFIG_PACKAGE_*` 条目。
- **Luci / 需要 Makefile 的用户态包**：在 `cudy-tr3000.sh` 中添加 `git clone` / `svn export` 命令，
  然后在配置中添加对应的 `CONFIG_PACKAGE_*` 条目。
- **随镜像发布的配置文件**：将文件放入 `files/` 目录，路径与目标系统一致
  （例如 `files/etc/config/network`）。

### 3.2 镜像前缀规范

镜像文件名前缀通过 `cudy-tr3000.sh` 中的 `include/image.mk` 修改：

```
sed -i 's|IMG_PREFIX:=|IMG_PREFIX:=$(shell TZ="Asia/Shanghai" date +"%Y%m%d")-24.10-6.6-|' include/image.mk
```

版本字符串变更（如 ImmortalWrt 版本升级）必须同步更新此行。

### 3.3 上游更新流程

`update-checker.yml` 工作流的流程：

1. 获取上游仓库的最新 commit hash。
2. 与缓存键（`repo-hash_{workflow-filename}_{hash}`）进行比较。
3. 不匹配时：触发构建并写入新的缓存条目。
4. 同时检查上游的 `defconfig/mt7981-ax3000.config`，有变更则创建 Issue。

上游源码更新后，构建会自动拉取新代码 —— 除非上游新增了内核选项，否则无需手动更新 `.config`。

---

## 4. 助手操作指南

### 4.1 添加新的 Luci 应用

```bash
# 1. 在 cudy-tr3000.sh 中添加克隆命令（在 feeds install 之后）
git clone <仓库地址> package/<应用名>

# 2. 在 cudy-tr3000.config 中添加 CONFIG_PACKAGE_<应用名>=y

# 3. 如果应用有 Luci 翻译文件，验证 po 文件或补充翻译
```

### 4.2 升级上游版本 / 分支

1. 在 `.github/workflows/openwrt-builder.yml` 中更新 `REPO_BRANCH`。
2. 在 `cudy-tr3000.sh` 中更新 `sed` 命令中的 IMG_PREFIX 版本字符串。
3. 从新源码的 `make menuconfig` 重新生成 `cudy-tr3000.config`。
4. 更新 `files/etc/opkg/distfeeds.conf` 中的 feed URL 以匹配新版本。
5. 更新 `README.md` 中的版本信息。

### 4.3 向镜像中添加文件

将文件放入 `files/` 目录，路径与目标文件系统一致：

```
files/etc/config/network   → /etc/config/network（固件中）
files/usr/bin/my-script    → /usr/bin/my-script
```

### 4.4 修改构建步骤

- feeds update 之前的步骤 → `diy-part1.sh`
- feeds install 之后的步骤 → `cudy-tr3000.sh`（不是 `diy-part2.sh`）
- 系统级配置覆写 → `cudy-tr3000.config`

---

## 5. 范围内 vs. 范围外

| 范围内                                                | 范围外                                        |
|-------------------------------------------------------|-----------------------------------------------|
| 修改 `cudy-tr3000.sh`、`diy-part1.sh`、`diy-part2.sh` | 修改 ImmortalWrt 源码树（本仓库不包含）         |
| 更新 `cudy-tr3000.config`                             | 本地执行 `make menuconfig`（无源码检出）        |
| 添加 / 更新 `files/` 覆写                             | 调试 CI 中的内核编译错误                        |
| 编辑 GitHub Actions 工作流                            | 修改上游包的 Makefile                          |
| 更新 `README.md`、`cudy-tr3000.md`、`FM350.md`        | 修改工作流 YAML 之外的 CI 基础设施              |
| 修改 `AGENTS.md`                                      | 设备树或内核配置变更                           |

---

## 6. Git 约定

- **分支前缀**：助手创建的分支使用 `codex/` 前缀。
- **提交信息**：简洁祈使句，如有作用域则以作用域开头。
- **切勿**撤回用户已有的更改。
- `.idea/` 目录是 IDE 元数据（JetBrains）—— 除非明确要求，否则不动。

---

## 7. 相关文档

| 文件              | 内容                               |
|-------------------|------------------------------------|
| `README.md`       | 面向用户的项目概览与致谢            |
| `cudy-tr3000.md`  | Release 说明 / 包含的包列表          |
| `FM350-GL.md`        | Fibocom FM350-GL 5G 模组设置指南    |
