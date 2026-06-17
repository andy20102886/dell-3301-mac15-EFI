本人小白以上全部文件全部为ai生成，目前亮度不可调，hdmi不能用，其他正常，如果不清楚就把全部文件喂给ai，他给你解决！！！！
# OpenCore 1.0.4 — Dell Latitude 3301 macOS Sequoia 15

> **适用机型**: Dell Latitude 3301 (i5-8265U, Intel UHD 620, macOS 15 Sequoia)  
> **最后更新**: 2026-06-17  
> **状态**: 日常可用 ✅

---

## 硬件规格

| 组件 | 型号 |
|------|------|
| **机型** | Dell Latitude 3301 |
| **CPU** | Intel Core i5-8265U @ 1.60GHz (Whiskey Lake, 4C8T) |
| **iGPU** | Intel UHD Graphics 620 (Device ID: 0x3EA0，伪装为 0x5916) |
| **内存** | 8GB LPDDR3 2133MHz |
| **存储** | NVMe / SATA SSD (M.2 2280) |
| **屏幕** | 13.3" 1366×768 eDP (BOE07EA) |
| **音频** | Realtek ALC3253 |
| **WiFi** | Intel Wi-Fi 6 AX201 (CNVi, 集成在 PCH) |
| **触摸板** | I2C HID (TPD0, GPIO 中断) |
| **键盘** | PS/2 |
| **BIOS** | Dell BIOS（CFG Lock 已硬件解锁） |
| **SMBIOS** | MacBookPro15,2 |

---

## 功能状态

| 功能 | 状态 | 备注 |
|------|------|------|
| ✅ GPU 加速 | 正常 | 1536MB VRAM，Metal，H264/HEVC 硬解 |
| ✅ 内屏显示 | 正常 | 1366×768 @ 60Hz，背光可调 |
| ❌  HDMI 输出 | 不能使用正常 | 外接显示器 |
| ✅ WiFi | 正常 | itlwm 2.3.0 + HeliPort，需手动启动 HeliPort.app |
| ✅ 音频 | 正常 | AppleALC，内置扬声器和耳机 |
| ✅ 触摸板 | 正常 | VoodooI2C 多指手势 |
| ✅ 键盘 | 正常 | VoodooPS2Controller，含亮度快捷键 |
| ✅ USB | 正常 | 全部 USB 端口，已定制 USB 映射 |
| ✅ 电池 | 正常 | SMCBatteryManager，充放电状态正确 |
| ✅ CPU 电源管理 | 正常 | VirtualSMC 1.3.7，频率 800MHz~3.9GHz 动态调节 |
| ✅ 睡眠/唤醒 | 正常 | |
| ✅ 传感器 | 正常 | CPU 温度、风扇转速（SMCDellSensors） |
| ✅ 读卡器 | 正常 | GenericCardReaderFriend |
| ⚠️ 蓝牙 | 未验证 | AX201 BT 可能需要单独配置 |
| ❌ AirDrop / Handoff | 不可用 | 需换 BCM94360NG 等原生 WiFi 卡 |
| ❌ 指纹识别 | 不可用 | macOS 不支持非 Apple T2 指纹 |

---

## 主要组件版本

| 组件 | 版本 |
|------|------|
| OpenCore | 1.0.4 |
| Lilu | 1.6.9 |
| WhateverGreen | 1.6.8 |
| VirtualSMC | **1.3.7**（⚠️ 必须 ≥ 1.3.6，低版本导致高温） |
| AppleALC | 1.9.2 |
| VoodooI2C | 2.9.1 |
| VoodooPS2Controller | 2.3.7 |
| itlwm | 2.3.0 |
| macOS | Sequoia 15.x |

---

## 安装指南

### 前提条件

1. **制作 macOS 安装盘**

   在已有 macOS 的机器或 macOS Recovery 中执行：

   ```bash
   sudo /Applications/Install\ macOS\ Sequoia.app/Contents/Resources/createinstallmedia \
     --volume /Volumes/USB --nointeraction
   ```

   或者使用 [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher) 的联机恢复工具。

2. **把本 EFI 复制到 U 盘的 EFI 分区**

   Windows 下用 [DiskGenius](https://www.diskgenius.cn/) 或 [EFI Agent](https://github.com/headkaze/EFI-Agent) 挂载 EFI 分区，把 `EFI` 文件夹整个复制进去。

3. **生成三码**（⚠️ 必须做，否则 iCloud / App Store 无法使用）

   使用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 针对 `MacBookPro15,2` 生成一套新的序列号：

   ```
   SystemProductName:   MacBookPro15,2
   SystemSerialNumber:  生成你自己的
   MLB:                 生成你自己的
   SystemUUID:          生成你自己的
   ```

   在 `config.plist → PlatformInfo → Generic` 中填入你生成的值，替换掉默认的占位符。

   > ⚠️ 本 EFI 的序列号已设为 `W00000000001`（明显无效值），请务必替换，否则无法登录 Apple ID。

### BIOS 设置

> 按 `F2` 进入 BIOS

**必须禁用：**

| BIOS 选项 | 设置 |
|-----------|------|
| Secure Boot | **Disabled** |
| Absolute / Computrace | **Disabled**（如有） |
| Intel Platform Trust Technology (PTT) | **Disabled** |
| VT for Direct I/O (VT-d) | **Disabled** |
| Fast Boot | **Disabled** |

**必须启用：**

| BIOS 选项 | 设置 |
|-----------|------|
| UEFI Boot | **Enabled** |
| SATA Mode | **AHCI** |
| USB Boot | **Enabled** |

**Dell 3301 特别说明**：

- CFG Lock 已默认解锁（商务机固件特性），`config.plist` 中 `AppleXcpmCfgLock` 设为 `false`
- 如遇启动异常，进 BIOS 清除 NVRAM（`F12 → Reset NVRAM`）

### 安装步骤

1. 将 U 盘插入 Dell 3301，开机按 `F12` 进入 Boot Menu
2. 选择 U 盘的 UEFI 启动项
3. 在 OpenCore 选择器中选择 `Install macOS Sequoia`
4. 按照 macOS 安装向导完成安装（重启 2-3 次属正常）
5. 安装完成后，进入已安装的 macOS，将 EFI 复制到系统盘的 EFI 分区

### WiFi 配置（必须手动操作）

macOS 15 Sequoia 不再支持 `AirportItlwm`，使用 `itlwm.kext + HeliPort` 方案：

1. 重启进入 macOS 后，WiFi 不会自动出现在菜单栏
2. 下载并安装 [HeliPort](https://github.com/OpenIntelWireless/HeliPort/releases)
3. 启动 HeliPort.app，菜单栏出现 WiFi 图标
4. 扫描并连接 WiFi
5. 建议设置开机自启：`系统设置 → 通用 → 登录项 → 添加 HeliPort`

---

## GPU 驱动说明

### 背景

Dell Latitude 3301 搭载 **Intel UHD 620 (Whiskey Lake)**，其真实 Device ID 是 `0x3EA0`（CFL 架构）。

macOS 15 Sequoia 删除了 CFL 图形驱动（`AppleIntelCFLGraphics`），但保留了 KBL 驱动。本 EFI 采用以下方案：

| 配置 | 值 | 说明 |
|------|-----|------|
| `ig-platform-id` | `0x87C00000` | KBL 平台，带 LVDS 内屏 + HDMI 输出 |
| `device-id` | `0x5916` | 伪装为 KBL-R UHD 620，触发 KBL 驱动加载 |
| `framebuffer-stolenmem` | 19MB | KBL 帧缓冲最低需求 |
| `boot-args` | `-igfxblt` | WhateverGreen 背光寄存器修复（CFL macOS 13.4+ 必须） |

### 关于亮度调节

目前所有方案都不能亮度调节

## CPU 电源管理注意事项

### ⚠️ VirtualSMC 版本要求

**必须使用 VirtualSMC ≥ 1.3.6**（本 EFI 已包含 1.3.7）。

macOS 15.4+ 修改了 SMC 键值访问逻辑，旧版 VirtualSMC 1.3.4 会触发 `PerfPowerServices` 死循环，表现为：

- `PerfPowerServices` CPU 占用 ~70%
- CPU 温度 85-95°C，风扇满转不停
- `kernel_task` CPU 占用 ~35%
- CPU 频率锁死在 1.8GHz

升级到 VirtualSMC 1.3.7 后完全消除，CPU 空闲温度恢复正常。

---

## 文件结构

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/
    │   ├── SSDT-XOSI.aml          # _OSI → XOSI 基础补丁
    │   ├── SSDT-3-xh_OEMBD.aml   # USB 端口定制映射
    │   ├── SSDT-EC-USBX.aml      # USB 供电
    │   ├── SSDT-GPI0.aml          # I2C 触摸板 GPIO 引脚
    │   ├── SSDT-GPRW.aml          # 睡眠修复
    │   ├── SSDT-HPET-DISABLE.aml  # 禁用 HPET
    │   ├── SSDT-PLUG.aml          # XCPM 电源管理 (目标 \_SB_.PR00)
    │   ├── SSDT-PNLF.aml          # 背光控制
    │   ├── SSDT-PS2K.aml          # 键盘映射
    │   ├── SSDT-SBUS-MCHC.aml    # SMBus
    │   ├── SSDT-TPD0.aml          # I2C 触摸板设备定义
    │   └── SSDT-TPD0-GPIO.aml    # 触摸板 GPIO 中断补丁
    ├── Drivers/
    │   ├── HfsPlus.efi            # HFS+ 文件系统支持
    │   ├── OpenRuntime.efi        # 必须
    │   ├── OpenCanopy.efi         # 图形化 OpenCore 界面
    │   ├── FirmwareSettingsEntry.efi
    │   └── ResetNvramEntry.efi    # 清除 NVRAM 菜单项
    ├── Kexts/
    │   ├── Lilu.kext              # 基础补丁引擎 (1.6.9)
    │   ├── VirtualSMC.kext        # SMC 模拟 (1.3.7)
    │   ├── WhateverGreen.kext     # GPU 补丁 (1.6.8)
    │   ├── AppleALC.kext          # 音频 (1.9.2)
    │   ├── itlwm.kext             # Intel WiFi (2.3.0)
    │   ├── SMCBatteryManager.kext # 电池传感器
    │   ├── SMCDellSensors.kext    # Dell 传感器 (风扇/温度)
    │   ├── SMCProcessor.kext      # CPU 温度
    │   ├── CPUFriend.kext         # CPU 频率管理
    │   ├── CPUFriendDataProvider.kext  # i5-8265U 频率策略数据
    │   ├── BrightnessKeys.kext    # 亮度快捷键
    │   ├── GenericCardReaderFriend.kext # 读卡器
    │   ├── VoodooI2C.kext         # I2C 触摸板主驱动 (2.9.1)
    │   ├── VoodooI2CHID.kext      # I2C HID 设备
    │   ├── VoodooPS2Controller.kext # 键盘 (2.3.7)
    │   └── (VoodooInput 等内置插件)
    ├── config.plist               # OpenCore 配置文件
    └── OpenCore.efi
```

---

## 关键配置参数速查

```xml
<!-- GPU -->
<key>AAPL,ig-platform-id</key><data>AADAhw==</data>  <!-- 0x87C00000 KBL -->
<key>device-id</key><data>FlkAAA==</data>              <!-- 0x5916 UHD 620 -->

<!-- SMBIOS -->
<key>SystemProductName</key><string>MacBookPro15,2</string>

<!-- Boot Args -->
<key>boot-args</key><string>debug=0x100 keepsyms=1 -igfxblt</string>

<!-- SIP (当前启用，可按需关闭) -->
<key>csr-active-config</key><data>AAAAAA==</data>  <!-- 0x00000000 SIP 全开 -->
```

---

## 常见问题

### Q: 安装后 WiFi 不可用？

A: 需要手动安装并启动 [HeliPort.app](https://github.com/OpenIntelWireless/HeliPort/releases)。macOS 15 不支持 AirportItlwm，必须使用 itlwm + HeliPort 方案。

### Q: 进入系统后 CPU 发热严重，风扇一直转？

A: 确认 VirtualSMC 版本 ≥ 1.3.6。这是 macOS 15.4+ 的已知问题，旧版 VirtualSMC 与 SMC 协议不兼容。本 EFI 已包含 1.3.7。

### Q: 亮度滑块存在但无法调节？

A: 确认 boot-args 包含 `-igfxblt`（不是 `-igfxblr`）。CFL 平台在 macOS 13.4+ 必须使用 `-igfxblt`。

### Q: 开机卡在苹果图标无进度条？

可能原因：
1. ACPI 补丁配置错误（尤其是 `_STA → XSTA` 之类的全局重命名）
2. Kext 加载顺序错误（VoodooI2C 插件必须在主 kext 之前）
3. Secure Boot 未关闭

快速恢复：开机按 `F12 → Reset NVRAM`，或在 OpenCore 启动选项中按 `空格` 选择 Reset NVRAM 条目。

### Q: 触摸板不工作？

A: 检查 `config.plist → ACPI → Patch` 中是否存在 `_STA → XSTA` 的全局重命名补丁并将其禁用（此补丁是致命的）。验证 VoodooI2C 插件加载顺序：`VoodooInput → VoodooI2CServices → VoodooGPIO → VoodooI2C(主) → VoodooI2CHID`。

### Q: iCloud / App Store 登录失败？

A: 必须替换序列号。使用 GenSMBIOS 生成新的三码，填入 `config.plist → PlatformInfo → Generic`。

### Q: 安装时提示磁盘不兼容 / 找不到安装盘？

A: 确认 BIOS 中 SATA 模式设为 AHCI，而非 RAID。

---

## 恢复方法

### 方法一：清除 NVRAM（最快）

开机时在 OpenCore 选择界面按 `空格`，选择 `Reset NVRAM` 条目。

### 方法二：从 Windows 还原 config.plist

1. 重启进 Windows
2. 打开 D 盘（ESP 分区），找到各版本备份文件（`config-backup-vXX.plist`）
3. 将对应的备份文件重命名为 `EFI/OC/config.plist` 覆盖即可

### 方法三：从 macOS Recovery 还原

```bash
# 挂载 EFI 分区（假设盘符为 disk0s1）
diskutil mount disk0s1
# 还原
cp /Volumes/OC/config-backup-v9.5.plist /Volumes/OC/EFI/OC/config.plist
```

---

## 修复历程（版本记录）

本 EFI 经过大量迭代，以下是关键里程碑：

| 版本 | 问题 | 结果 |
|------|------|------|
| V1 | 基础 HDMI 显示 | ✅ |
| V2 | WiFi (itlwm + HeliPort) | ✅ |
| V8.1 | CPU 电源管理 (CPUFriend) | ✅ |
| V9.0 | GPU 加速 (KBL ig-platform-id) | ✅ 1536MB VRAM |
| V9.3 | 触摸板 (_STA→XSTA 全局重命名) | ❌ 致命，卡苹果 |
| V9.4 | 修复 V9.3 崩溃，禁用错误重命名 | ✅ |
| V9.5 | 触摸板 + 键盘完整方案 | ✅ |
| V9.9~V9.11 | 背光寄存器修复 (-igfxblr/-igfxblt) | ✅ |
| V20 | VirtualSMC 1.3.7，修复高温问题 | ✅ |

**完整修复记录**见仓库 `docs/` 目录或 OC 盘根目录下的 `修复文档索引.md`。

---

## 重要说明 / 免责声明

- 此 EFI 仅在 **Dell Latitude 3301 i5-8265U** 上验证。其他子型号（i3/i7、不同屏幕）可能需要调整。
- **使用前务必替换序列号**，使用他人序列号可能导致账号封禁。
- 本项目仅供学习和研究目的。在受支持的 Mac 硬件上使用 macOS 才是苹果官方支持的方式。
- 安装过程中可能损坏数据，请提前备份重要文件。

---

## 致谢

- [Acidanthera](https://github.com/acidanthera) — OpenCore、Lilu、WhateverGreen、VirtualSMC、AppleALC 等核心组件
- [Dortania](https://dortania.github.io/) — OpenCore 安装指南
- [sambow23](https://github.com/sambow23/Dell-Latitude-3301-macOS) — Dell Latitude 3301 参考 EFI 和 SSDT-PLUG/PNLF
- [alexandred](https://github.com/alexandred/VoodooI2C) — VoodooI2C 触摸板驱动
- [OpenIntelWireless](https://github.com/OpenIntelWireless) — itlwm 和 HeliPort
- [CorpNewt](https://github.com/corpnewt) — GenSMBIOS、ProperTree 等工具
- [dreamwhite/dell-inspiron-5370-hackintosh](https://github.com/dreamwhite/dell-inspiron-5370-hackintosh) — 5370 参考配置
- 所有为黑苹果社区贡献过的开发者和爱好者们

---

## 相关资源

- [OpenCore Install Guide (Dortania)](https://dortania.github.io/OpenCore-Install-Guide/)
- [WhateverGreen FAQ.IntelHD](https://github.com/acidanthera/WhateverGreen/blob/master/Manual/FAQ.IntelHD.en.md)
- [Getting Started With ACPI](https://dortania.github.io/Getting-Started-With-ACPI/)
- [HeliPort Releases](https://github.com/OpenIntelWireless/HeliPort/releases)
- [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)
