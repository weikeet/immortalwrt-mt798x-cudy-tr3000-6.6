
## 模组信息

网络-移动通信模组-模组信息

### 基本信息

制造商 Fibocom Wireless Inc.
固件版本 81600.0000.00.29.23.08
数据接口 USB
模式 RNDIS
AT串口 /dev/ttyUSB1
移动网络 eth2
温度 36°C

### SIM卡信息

SIM卡信息
运营商 CHN-UNICOM
SIM卡卡槽 1
SIM卡号码 8615519037701
国际移动设备识别码 860839050566264
国际移动用户识别码 460019031421798
集成电路卡识别码 8986 0124 8016 1513 1714

### 网络信息

网络类型 NR

### 小区信息

NR5G-SA 模式
移动国家代码 (MCC) 460
移动网络代码 (MNC) 11
小区ID (Cell ID) 06036B001
物理小区ID (Physical Cell ID) 300
跟踪区编码 (TAC) 067014
绝对射频信道号 (ARFCN) 627264
频段 (Band) N78

### 指令查询模组信息

```shell
# 模组信息 ATI
2026-06-04 13:01:33 ATI

Manufacturer: Fibocom Wireless Inc.
Model: FM350-GL
Revision: 81600.0000.00.29.23.08
SVN: 09
```

## 首次使用

确认模组已经正确安装并且加载成功，查看 `模组信息` 中的 `AT串口` 和 `移动网络` 是否正确显示。

然后按照一下操作添加配置：

网络-移动通信模组-拨号总览-添加配置
通用配置-启用（暂时不勾选）
通用配置-移动网络（默认识别的就行）
高级配置-拨号工具-选择 调制解调器管理工具
高级配置-网络类型-默认 IPv4/IPv6
高级配置-网络桥接-可以 勾选
高级配置-接入点-根据 运营商选择
高级配置-认证类型-默认 无

保存并应用后，进入模组调试界面，执行 `查询 sim 卡状态` 指令，如果 ready，执行 `手动拨号` 指令，如果成功，会有 `+CGEV: ME PDN ACT 3` 的返回。

回到网络-移动通信模组-拨号总览，启用配置，等待连接成功。

## 常用指令执行

```shell
# [查询 sim 卡状态]
AT+CPIN?
2026-06-13 16:19:10 
+CPIN: READY

# [手动设置接入点] 移动 CBNET, 电信 CMNET, 联通 UNINET, 通常不需要设置，拨号配置里面设置过接入点了
AT+CGDCONT=3,"IPV4V6","UNINET"
2026-06-13 16:20:46 

# [手动拨号]
AT+CGACT=1,3
2026-06-13 16:21:33 
+CGEV: ME PDN ACT 3
```

## Q & A

1. SIM 卡一直显示未插入
   - 重新插拔卡
   - 重启模组

2. 模组拨号一直重试失败
   - 确认模组启动成功并且加载成功
   - 网络-移动通信模组-拨号总览-取消勾选启用配置
   - 执行 查询 sim 卡状态 指令，如果 ready，执行 手动拨号 指令

[f350重启路由后，无法自动勾选拨号 · Issue #19 · Siriling/5G-Modem-Support](https://github.com/Siriling/5G-Modem-Support/issues/19)

[需要编译哪些插件到固件中呢？ · Issue #50 · Siriling/5G-Modem-Support](https://github.com/Siriling/5G-Modem-Support/issues/50)

- 对于FM350都不需要，quectel_cm_5G 是移远或高通模组使用的。其他一些是以前的旧插件。
- kmod-qmi_wwan_f 需要打进去吗？ai 说这个是广和通专属驱动
  - 不需要，广和通高通芯片的模组才需要。
- gobinet 已经被 qmi_wwan 取代了

[广和通 FM350-GL 模块使用记录/测评 - Kiritake Kumi ‘s Blog](https://blog.yazawaniko.com/index.php/archives/294/)

```shell
# 5G 模块组件
CONFIG_PACKAGE_sms-tool=y
# 错误的: PCIE 版本的模组才需要
CONFIG_PACKAGE_kmod-pcie_mhi=y
# 错误的: gobinet 已经被 qmi_wwan 取代了，并且只有高通模组需要
CONFIG_PACKAGE_kmod-gobinet=y
# 错误的: quectel_cm_5G 是移远或高通模组使用的
CONFIG_PACKAGE_quectel-CM-5G=y
# 必须的: 5G模组信息插件+AT工具
CONFIG_PACKAGE_luci-app-modem=y
# 没啥用: 看着没吊用？
CONFIG_PACKAGE_luci-i18n-modem-zh-cn=y
# 有啥用: 不知道和 sms-tool 有什么区别？
CONFIG_PACKAGE_luci-app-sms-tool=y
```

## 相关软件包

```shell
# rndis
kmod-usb-net-rndis
# USB
kmod-usb2
kmod-usb3
kmod-usb-net（USB 转以太网）
usb-modeswitch
# 串口
kmod-usb-serial
kmod-usb-serial-option
kmod-usb-serial-wwan
# 命令行工具
usbutils（USB工具包）
```

### Actions-RAX3000M config

- https://github.com/Siriling/Actions-RAX3000M/blob/main/MT7981-eMMC-USB-hanwckf/.mtwifi-cfg.config

```shell
# 5G模组短信插件
# CONFIG_PACKAGE_luci-app-sms-tool=y
# 底层短信工具 专门用于通过 3G/4G/5G 模块接收、发送和管理短信（SMS）的命令行程序
CONFIG_PACKAGE_sms-tool=y
# 5G模组管理插件+AT工具
CONFIG_PACKAGE_luci-app-modem=y
# 高通相关 5G 模组使用,
# 移远通信 (Quectel) 专属驱动
CONFIG_PACKAGE_kmod-qmi_wwan_q=y
# 广和通 (Fibocom) 专属驱动
# CONFIG_PACKAGE_kmod-qmi_wwan_f=y
# 美格智能 (MeigSmart) 专属驱动
# CONFIG_PACKAGE_kmod-qmi_wwan_m=y
# CONFIG_PACKAGE_kmod-qmi_wwan_t=y

# 底层命令工具: 一个用于向调制解调器（Modem）发送 AT 指令并读取返回结果的轻量级命令行工具。
sendat

# 串口调试工具
CONFIG_PACKAGE_minicom=y
```

### istoreos-actions/scripts/diy-part2

[istoreos-actions/scripts/diy-part2.sh at main · Siriling/istoreos-actions](https://github.com/Siriling/istoreos-actions/blob/main/scripts/diy-part2.sh)

- 适用于移远 [istoreos-actions/tools/5G模组拨号脚本 at main · Siriling/istoreos-actions](https://github.com/Siriling/istoreos-actions/tree/main/tools/5G%E6%A8%A1%E7%BB%84%E6%8B%A8%E5%8F%B7%E8%84%9A%E6%9C%AC)

```shell
mkdir Modem-Support
pushd Modem-Support
git clone --depth=1 https://github.com/Siriling/5G-Modem-Support .
popd

# 5G通信模组拨号工具
mkdir quectel_QMI_WWAN
# mkdir fibocom_QMI_WWAN
# mkdir meig_QMI_WWAN
# mkdir tw_QMI_WWAN
mkdir quectel_cm_5G
mkdir quectel_MHI
# mkdir luci-app-hypermodem
cp -rf ../../Modem-Support/quectel_QMI_WWAN/* quectel_QMI_WWAN
# cp -rf ../../Modem-Support/fibocom_QMI_WWAN/* fibocom_QMI_WWAN
# cp -rf ../../Modem-Support/meig_QMI_WWAN/* meig_QMI_WWAN
# cp -rf ../../Modem-Support/tw_QMI_WWAN/* tw_QMI_WWAN
cp -rf ../../Modem-Support/quectel_cm_5G/* quectel_cm_5G
cp -rf ../../Modem-Support/quectel_MHI/* quectel_MHI
# cp -rf ../../Modem-Support/luci-app-hypermodem/* luci-app-hypermodem

# 5G模组短信插件
# cp -rf temp/luci-app-sms-tool/* luci-app-sms-tool
mkdir sms-tool
mkdir luci-app-sms-tool
cp -rf ../../Modem-Support/sms-tool/* sms-tool
cp -rf ../../Modem-Support/luci-app-sms-tool/* luci-app-sms-tool
cp -rf ../../MyConfig/configs/istoreos/general/applications/luci-app-sms-tool/* luci-app-sms-tool

# 5G模组信息插件
# svn export https://github.com/qiuweichao/luci-app-modem-info/trunk/luci-app-3ginfo-lite
# svn export https://github.com/owner888/luci-app-3ginfo-zh_cn/trunk/3ginfo
# svn export https://github.com/owner888/luci-app-3ginfo-zh_cn/trunk/luci-app-3ginfo

# 5G模组IPv6
mkdir ndisc
cp -rf ../../Modem-Support/ndisc/* ndisc

# 5G模组信息插件+AT工具
mkdir luci-app-modem
cp -rf ../../Modem-Support/luci-app-modem/* luci-app-modem
rm -rf ../../Modem-Support/luci-app-modem/po/zh_Hans #解决汉化问题
popd

# 5G模组拨号脚本
# mkdir -p package/base-files/files/root/5GModem
# cp -rf $GITHUB_WORKSPACE/tools/5G模组拨号脚本/5GModem/* package/base-files/files/root/5GModem
# echo -e "#* * * * * bash /root/5GModem/5g_crontab.sh" >> package/istoreos-files/files/etc/crontabs/root
```
