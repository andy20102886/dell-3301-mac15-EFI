# GPU 驱动完整试错记录 — Dell Latitude 3301 黑苹果
# V1 → V7.8 全链路

> 📅 整理时间：2026-06-05  
> 🍎 系统：macOS 15.5 Sequoia (Darwin 24.5.0)  
> 🖥️ 硬件：Dell Latitude 3301 / i5-8265U Whiskey Lake / UHD 620 (Device ID: 0x5916)  
> ⚠️ 特殊情况：无机身内置屏幕，仅 HDMI 外接显示器  
> 🔧 SMBIOS：MacBookPro15,2 (Coffee Lake 2018)

---

## 一、核心技术背景

### macOS 15 显卡驱动格局

| 架构 | 代号 | macOS 13 | macOS 14 | macOS 15 |
|------|------|----------|----------|----------|
| Kaby Lake | KBL | ✅ | ✅ | ✅ 保留 |
| Coffee Lake | CFL | ✅ | ✅ | ❌ 移除 |
| Whiskey Lake-R | WHL-R (实质同 CFL) | ✅ | ✅ | ❌ 移除 |

> **根因**：Intel UHD 620 (0x5916) 是 Whiskey Lake-R，驱动框架等同 CFL。macOS 15 移除了 CFL 驱动，导致 7MB VRAM + 无硬件加速。

### 无内屏问题

Dell Latitude 3301 无机身内置显示器，HDMI 外接。所有带 LVDS/eDP 连接器的 framebuffer 人格（如 `0x3EA50009` CFL 移动版）都会在初始化时尝试检测不存在的内屏 → 驱动初始化失败 → 7MB VRAM。

### OCLP 无效的原因

OCLP 根据 SMBIOS 机型决定是否打补丁：
- `MacBookPro15,2`（CFL 2018）→ OCLP 认为"无需补丁"→ 按钮灰色
- `MacBookPro14,1`（KBL 2017）→ 不在 macOS 15 `PlatformSupport.plist` 里 → 启动禁止符号

---

## 二、完整版本历史

### V1 — 源文件基线
- **ig-platform-id**: `0x87C00000`
- **device-id**: `0x5916`
- **状态**: ❌ 安装通过（VESA 模式），但完整启动时 IOFB panic

---

### V2 — CFL 移动版基线
- **ig-platform-id**: `0x3EA50009`（CFL 移动版，有内屏）
- **device-id**: `0x5916`
- **Lilu**: 1.6.9 | **WEG**: 1.6.8
- **状态**: ⚠️ 进桌面，WiFi 可用，但 **VRAM 7MB**
- **原因**: macOS 15 已移除 CFL 驱动框架

---

### V3.1 — IRQ 尝试
- **改动**: 添加 IRQ 优先级相关 ACPI 补丁
- **状态**: ❌ 无效，7MB 不变

---

### V4 — KBL SMBIOS
- **改动**: SMBIOS → `MacBookPro14,1`（KBL 机型）
- **状态**: ❌ 启动禁止符号（board-id `Mac-B4831CEBD52A0C4C` 不在 macOS 15 支持列表）

---

### V5 — CFL device-id 伪装
- **ig-platform-id**: `0x3EA50009`
- **device-id**: `0x9B3E`（CFL UHD 630）
- **状态**: ❌ 进度条 1/3 后 HDMI 黑屏

---

### V6 — KBL 无内屏人格
- **ig-platform-id**: `0x3E9B0000`（KBL 桌面，3 DP/HDMI，无 eDP）
- **device-id**: `0x5916`
- **stolenmem**: 3MB → 19MB（KBL 驱动要求 ≥19MB）
- **状态**: ⚠️ VRAM 仍 7MB

---

### V7 — Lilu/WEG 升级 + kext 加载修复系列

所有 V7.x 的公共配置：
```
ig-platform-id:  0x3E9B0000
device-id:       0x5916
stolenmem:       19MB (0x30010000)
unifiedmem:      96MB
SecureBootModel: Disabled
```

#### V7.0 — Lilu 1.7.2 + WEG 1.7.1 升级
- **改动**: Lilu 1.6.9 → 1.7.2，WEG 1.6.8 → 1.7.1（修复 Sequoia 不兼容）
- **状态**: ❌ 7MB，Lilu 仍未加载

#### V7.1 — AMFIPass + SIP 关闭
- **改动**: 新增 AMFIPass.kext 1.4.1；`amfi=0x80`；`csr-active-config=FF0F0000`；`SecureBootModel=Disabled`
- **状态**: ❌ 7MB，kextstat 无 Lilu/WEG

#### V7.2 — Beta 启动参数
- **改动**: boot-args 添加 `-lilubetaall -wegbeta`
- **状态**: ❌ 7MB

#### V7.3 — 清除错误连接器属性
- **改动**: 删除 framebuffer-con0/1/2/3-alldata 等 8 个错误连接器属性（这些属性是 CFL 人格的，与 KBL 0x3E9B0000 不匹配）
- **状态**: ❌ 7MB

#### V7.4 — `-liluforce` 强制加载
- **改动**: boot-args 添加 `-liluforce`
- **备份**: `config.plist.bak.v7.4.20260604234126`
- **状态**: ❌ 7MB，kextstat 仍无 Lilu/WEG

#### V7.5 — GitHub 官方 kext 重新下载
- **改动**: 从 GitHub acidanthera 官方下载全新 Lilu 1.7.2 + WEG 1.7.0，替换 OC 盘上可能损坏的旧版
- **旧版备份**: `Kexts/old_backup_20260604/`
- **状态**: ❌ 7MB

#### V7.6 — SIP 完全关闭（为 OCLP 准备）
- **改动**: `csr-active-config`: `FF0F0000` → `FF070000`（完全关闭所有 SIP 标志位）
- **原因**: OCLP payloads.dmg 挂载失败（hdiutil: 操作不被允许）→ SIP 文件系统保护位未关
- **状态**: SIP 完全关闭，但 OCLP GUI 仍灰色

#### V7.7 — OCLP 命令行强制补丁（失败）
- **操作**: `/Applications/OpenCore-Patcher.app/Contents/MacOS/OpenCore-Patcher --patch_sys_vol`
- **错误**: `No Root Patches required for your machine!`
- **根因**: SMBIOS = MacBookPro15,2（CFL），OCLP 认为无需补丁；而 KBL 机型 MacBookPro14,1 不在 macOS 15 支持列表
- **状态**: ❌ 7MB，根本原因确认：macOS 15 已完全封锁 KBL kext 加载路径

#### V7.8 — OpenCore 强制注入 KBL kext（最新）
- **改动**:
  1. 复制 `AppleIntelKBLGraphics.kext` + `AppleIntelKBLGraphicsFramebuffer.kext` 从 S/L/E → OC 盘 `Kexts/KBL_drivers/`
  2. 向 `Kernel/Add` 索引 [1][2] 插入两个 KBL kext 条目（强制 OC 在启动阶段注入）
- **备份**: `config.plist.bak.v7.8.20260605002303`
- **原理**: OpenCore 在启动阶段直接注入 kext 到 XNU，早于 `kmutil` 的排除列表机制
- **状态**: ⏳ 待重启验证

---

## 三、所有已尝试方案总结

| 版本 | ig-platform-id | device-id | Lilu | WEG | 关键变更 | 结果 |
|------|---------------|-----------|------|-----|---------|------|
| V1 | 0x87C00000 | 0x5916 | 1.6.9 | 1.6.8 | 源文件基线 | ❌ IOFB panic |
| V2 | 0x3EA50009 CFL | 0x5916 | 1.6.9 | 1.6.8 | CFL 移动 | ⚠️ 7MB 基线 |
| V3.1 | 同上 | 同上 | 同上 | 同上 | IRQ 补丁 | ❌ 无效 |
| V4 | — | — | — | — | KBL SMBIOS(MBP14,1) | ❌ 禁止符号 |
| V5 | 0x3EA50009 | 0x9B3E | 1.6.9 | 1.6.8 | CFL UHD630 伪装 | ❌ 黑屏 |
| V6 | 0x3E9B0000 KBL | 0x5916 | 1.6.9 | 1.6.8 | KBL 无内屏 | ⚠️ 7MB |
| V7 | 同上 | 同上 | **1.7.2** | **1.7.1** | kext 升级 | ⚠️ 7MB |
| V7.1 | 同上 | 同上 | 1.7.2 | 1.7.1 | +AMFIPass+SIP 关 | ⚠️ 7MB |
| V7.2 | 同上 | 同上 | 1.7.2 | 1.7.1 | +beta 参数 | ⚠️ 7MB |
| V7.3 | 同上 | 同上 | 1.7.2 | 1.7.1 | 清除错误连接器 | ❌ 7MB |
| V7.4 | 同上 | 同上 | 1.7.2 | 1.7.1 | +-liluforce | ❌ 7MB |
| V7.5 | 同上 | 同上 | 1.7.2 新下载 | 1.7.0 新下载 | GitHub 官方 kext | ❌ 7MB |
| V7.6 | 同上 | 同上 | 1.7.2 | 1.7.0 | csr=FF070000 | ⚠️ OCLP 仍灰色 |
| V7.7 | 同上 | 同上 | 1.7.2 | 1.7.0 | OCLP 命令行 | ❌ 7MB |
| **V7.8** | **同上** | **同上** | **1.7.2** | **1.7.0** | **OC 强制注入 KBL kext** | **⏳ 待测** |

---

## 四、当前系统诊断数据（2026-06-05）

```
VRAM:       7MB（无加速）
kextstat:   无 Lilu/WEG/AMFIPass 条目
ioreg:      AppleIntelKBLGraphics = 0（KBL framebuffer 未初始化）
SIP 状态:   System Integrity Protection status: unknown (Custom Configuration)
            → 文件系统保护：disabled
boot-args:  debug=0x100 keepsyms=1 amfi=0x80 -lilubetaall -wegbeta -liluforce
csr:        FF070000
SMBIOS:     MacBookPro15,2
```

---

## 五、如果 V7.8 仍然失败，下一步方向

### 方向 A：降级到 macOS 14 Sonoma（推荐）
macOS 14 原生支持 CFL/WHL-R 驱动（`AppleIntelCFLGraphics`），UHD 620 可正常工作，无需任何 kext 注入。

### 方向 B：VoodooI2C GPIO 中断补丁
通过 SSDT 修补 I2C GPIO 中断，让 VoodooI2C 更稳定初始化（此方法已部分尝试）。

### 方向 C：修改 OCLP 机型检测（高风险）
修改 OCLP 内部机型白名单，让其识别 MacBookPro15,2 为需要 KBL 补丁的机型。此方法需要重新编译 OCLP，技术难度高。

---

## 六、回退方法

### 回退到任意备份版本

```bash
# 列出所有备份
ls -la /Volumes/OC/EFI/OC/config.plist.bak.*

# 回退到指定版本
cp /Volumes/OC/EFI/OC/config.plist.bak.v7.4.20260604234126 /Volumes/OC/EFI/OC/config.plist
```

### 紧急亮屏（黑屏无法进入系统）

OC Picker → 空格键 → 在 boot-args 输入行添加 `-igfxvesa` → 回车启动

---

## 七、备份文件清单

| 文件 | 时间点 |
|------|--------|
| `config.plist.bak.v7.4.20260604234126` | V7.4 修改前（加 -liluforce 前）|
| `config.plist.bak.v7.8.20260605002303` | V7.8 修改前（KBL kext 注入前）|
| `Kexts/old_backup_20260604/` | V7.5 旧版 Lilu/WEG 备份 |
| `Kexts/KBL_drivers/` | V7.8 注入的 KBL kext |

---

> 📄 相关文档：
> - `/Volumes/OC/V7-完整修复记录.md` — V7 系列修复技术细节
> - `/Volumes/OC/V8.0-触摸板键盘修复.md` — 输入设备修复记录
> - `/Volumes/OC/GPU驱动修复试错总结.md` — 早期 V1-V6 总结（原版）
