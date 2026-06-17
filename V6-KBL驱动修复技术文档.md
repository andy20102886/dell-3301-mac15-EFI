# V6 KBL 驱动修复技术文档 — Dell Latitude 3301

> 📅 2026-06-04 | 🍎 macOS 15.5 Sequoia (Darwin 24)  
> 🖥️ Dell Latitude 3301 / i5-8265U Whiskey Lake / UHD 620  
> 🔑 无机身内屏，仅 HDMI 外接显示器  
> 🔧 SMBIOS: MacBookPro15,2 / board-id: Mac-827FB448E656EC26  

---

## 零、修改摘要

| # | 参数 | 位置 | 改前 | 改后 | 原因 |
|---|------|------|------|------|------|
| 1 | `framebuffer-stolenmem` | DeviceProperties → GPU | `00003000` (3MB) | `00003001` (19MB) | KBL 驱动最低要求 19MB |
| 2 | `framebuffer-con1-type` | DeviceProperties → GPU | 缺失 | `00080000` (HDMI) | 0x3E9B0000 人格 con1 默认为 DP，需改为 HDMI |

备份: `config_V6_backup.plist` 在同目录

---

## 一、V6 完整 GPU 配置

```
路径: PciRoot(0x0)/Pci(0x2,0x0)

  AAPL,ig-platform-id       = 00009b3e   (0x3E9B0000, KBL GT2, 3 DP, 无内屏)
  device-id                 = 16590000   (0x5916, KBL-R UHD 620 原生)
  model                     = Intel UHD Graphics 620

  framebuffer-patch-enable  = 01000000   (启用)
  framebuffer-stolenmem     = 00003001   (19MB) ← 本次修改
  framebuffer-unifiedmem    = 00000006   (96MB)

  framebuffer-con0-enable   = 01000000   (启用)
  framebuffer-con0-alldata  = 000008000200000098000000
  framebuffer-con1-enable   = 01000000   (启用)
  framebuffer-con1-alldata  = 010509000008000087010000
  framebuffer-con1-type     = 00080000   (HDMI) ← 本次新增
  framebuffer-con2-enable   = 01000000   (启用)
  framebuffer-con2-alldata  = 02040a000004000087010000

  igfxonln                  = 01000000
```

---

## 二、硬件环境

| 项目 | 值 |
|------|-----|
| CPU | Intel Core i5-8265U, Whiskey Lake, 4C/8T, 1.6-3.9GHz |
| GPU | Intel UHD 620, PCI device-id 0x5916, revision 0x02 |
| 显示 | 无内屏，仅 HDMI 1920x1080@60Hz |
| 内存 | 8GB LPDDR3 2133MHz |
| SMBIOS | MacBookPro15,2 (2018 MBP, Coffee Lake) |

---

## 三、macOS 15 GPU 驱动格局

### 3.1 系统中的驱动文件

```
/System/Library/Extensions/

KBL 驱动（完整栈 ✅）:
  AppleIntelKBLGraphics.kext                ← 主驱动
  AppleIntelKBLGraphicsFramebuffer.kext     ← 帧缓冲（支持 0x5916）
  AppleIntelKBLGraphicsGLDriver.bundle      ← OpenGL 加速
  AppleIntelKBLGraphicsMTLDriver.bundle     ← Metal 加速
  AppleIntelKBLGraphicsVADriver.bundle      ← 视频解码
  AppleIntelKBLGraphicsVAME.bundle          ← 视频编码

CFL 驱动（残废 ⚠️）:
  AppleIntelCFLGraphicsFramebuffer.kext     ← 只有帧缓冲
  AppleIntelCFLGraphicsVAME.bundle          ← 只有编码器
  ❌ 无主驱动、GLDriver、MTLDriver       ← 无 3D 加速
```

### 3.2 KBL 驱动 IOPCIMatch

`AppleIntelKBLGraphicsFramebuffer.kext/Info.plist`:
```
IOPCIMatch = 0x59128086 0x59168086 0x591B8086 0x591C8086 0x591E8086
             0x59268086 0x59278086 0x59238086 0x87C08086
```
✅ **0x5916 (UHD 620) 在支持列表中**

`AppleIntelKBLGraphics.kext/Info.plist`:
```
IOPCIMatch = 0x59128086 0x59168086 ... (KBL IDs)
            + 0x3E9B8086 0x3EA58086 0x3EA68086 0x3E918086 0x3E928086 ... (CFL IDs)
```
✅ **KBL 和 CFL device-id 都被主驱动支持**

### 3.3 结论

```
macOS 15: KBL 驱动栈完整存在，CFL 驱动栈已残废
→ KBL 伪装 CFL (0x3EA50009) = 有帧缓冲无加速 = 7MB ❌
→ KBL 原生驱动 (0x3E9B0000 + 0x5916) = 完整加速栈 ✅
```

---

## 四、0x3E9B0000 帧缓冲人格详解

### 4.1 人格特征

| 属性 | 值 |
|------|-----|
| platform-id | 0x3E9B0000 |
| 代号 | KBL GT2 (桌面) |
| 端口数 | 3 |
| 端口 0 (con0) | DP (独立显卡)，总线 0x00，索引 0x01 |
| 端口 1 (con1) | DP (数字输出)，总线 0x05，索引 0x02 |
| 端口 2 (con2) | DP (数字输出)，总线 0x04，索引 0x03 |
| 内屏 (eDP/LVDS) | **无** ← 关键！ |
| 最大 DVMT | 基于盗存内存 (BIOS 设置) |

### 4.2 选择原因

与 0x3EA50009 (CFL, 有 eDP) 及其他 KBL mobile 人格对比:

| 人格 | 内屏 | 数字输出 | macOS 15 驱动 | 适合无内屏? |
|------|------|----------|---------------|-------------|
| 0x3EA50009 | eDP | 2 | CFL (残废) | ❌ |
| 0x59160000 | eDP | 2 | KBL | ❌ 初始化失败 (等内屏) |
| 0x59120000 | eDP | 2 | KBL | ❌ |
| **0x3E9B0000** | **无** | **3 DP** | **KBL** | **✅** |
| 0x3E920000 | 无 | 3 DP | KBL | ✅ 备选 |

0x3E9B0000 是 macOS 上唯一原生不含内屏连接器的 KBL 帧缓冲人格。

### 4.3 为什么需要 con1-type=HDMI

0x3E9B0000 的 con1 默认类型是 DP。Dell Latitude 3301 的物理 HDMI 端口映射到 con1。如果 WEG 不能正确将其标记为 HDMI，帧缓冲向 HDMI 发送 DP 信号 → 显示器无信号或颜色异常。

设置 `framebuffer-con1-type=00080000` 告诉 WhateverGreen 将 con1 端口类型覆盖为 HDMI。

---

## 五、为什么 stolenmem < 19MB 会失败

### 5.1 技术原理

Intel KBL 帧缓冲驱动在初始化时：

1. 读取 `stolenmem` (DVMT 预分配内存) 值
2. 计算 ≥1920×1080×4 (帧缓冲表面) + 光标 + 命令缓冲所需的盗存内存
3. 如果盗存内存 < 所需 → 拒绝初始化 → IONDRVFramebuffer (VESA) 回退

### 5.2 最小计算

```
单帧缓冲面: 1920 × 1080 × 4 bytes = 8.3 MB
光标面:                   256 × 256 × 4 = 0.25 MB
压缩/命令缓冲/覆盖:        ~10 MB
─────────────────────────────────────────
最低要求:                              ≈ 19 MB
```

### 5.3 各值对应关系

| stolenmem 值 (hex→dec) | DVMT 大小 | WEG 是否接受 | 1080p |
|------------------------|-----------|-------------|-------|
| `00003000` = 3145728 | 3 MB | ✅ 初始化 | ❌ KBL 框架拒绝 |
| `00003001` = 19922944 | 19 MB | ✅ | ✅ |
| `0000A001` = 65536 KB | 64 MB | ✅ | ✅ WEG 一般推荐 |

---

## 六、系统诊断方法

### 6.1 验证加速是否生效

```bash
# VRAM（正常=1536 MB）
system_profiler SPDisplaysDataType | grep VRAM

# KBL 驱动加载
kextstat | grep -i "KBL\|AppleIntelFramebuffer"

# Metal 支持
system_profiler SPDisplaysDataType | grep Metal

# 帧缓冲实例数 (正常≥1)
ioreg -l -w0 | grep -c "AppleIntelKBLGraphicsFramebuffer"
```

### 6.2 如果仍然 7MB VRAM

按优先级排查:

| 优先级 | 检查项 | 命令/操作 |
|--------|--------|-----------|
| 1 | stolenmem 是否 ≥19MB | 检查 config.plist → `framebuffer-stolenmem` |
| 2 | con1-type = HDMI | 检查 config.plist → `framebuffer-con1-type` |
| 3 | 物理 HDMI 端口是否映射到 con0/con2 | 依次尝试只启用单端口 |
| 4 | WEG/Lilu 版本 | 检查 kext → Info.plist (需 ≥1.6.9/1.7.0) |
| 5 | SMBIOS 拒绝 | 尝试 iMac18,3 (KBL) 或使用 -no_compat_check |
| 6 | DVMT 冲突 | `framebuffer-stolenmem` 增到 64MB (`0000A001`) |

---

## 七、备选方案 (如果 V6 仍失败)

### 方案 A: 换 con0 为主输出

某些机型 HDMI 映射到 con0 而非 con1:

```
framebuffer-con0-type = 00080000 (HDMI)
framebuffer-con0-enable = 01000000
framebuffer-con1-enable = 00000000
framebuffer-con2-enable = 00000000
```

### 方案 B: 换 0x3E920000 人格

```
AAPL,ig-platform-id = 0000923E     (0x3E920000, KBL GT2, 3 DP, 无内屏)
其他属性不变
```

### 方案 C: HECI/I2C patch (如果 60Hz 卡死)

部分 KBL 帧缓冲在初始化 I2C-over-AUX 时崩溃:
```
boot-args += igfxfcms=1 igfxagdc=0
```

### 方案 D: WEG 升级

当前 WEG: 1.6.8 / Lilu: 1.6.9  
macOS 15.5 推荐: WEG ≥ 1.6.9 / Lilu ≥ 1.7.0

---

## 八、故障排除速查

| 症状 | 可能原因 | 修复 |
|------|----------|------|
| 黑屏 (启动后) | con1 端口类型错误 | 试 con0/con1/con2 分别开 |
| 7MB VRAM | 框架未初始化 | 检查 stolenmem ≥19MB |
| 花屏/颜色错 | busid 映射错误 | 修改 conX-alldata busid |
| 卡顿 | Metal 未启用 | 检查 `kextstat \| grep MTL` |
| 禁行 (🚫) | SMBIOS 不兼容 | 回退 MacBookPro15,2 |
| Kernel Panic | DVMT/盗存内存冲突 | stolenmem 调到 64MB |

---

## 九、版本历史

| 版本 | ig-platform-id | device-id | stolenmem | 状态 |
|------|---------------|-----------|-----------|------|
| V1 | 0x3EA50009 CFL | 0x5916 KBL | — | 桌面, 无 WiFi |
| V2 | 0x3EA50009 CFL | 0x5916 KBL | — | ✅ 基线, 7MB |
| V3.1 | 0x3EA50009 CFL | 0x5916 KBL | — | IRQ 尝试 |
| V4 | 0x59160003 KBL | 0x5916 KBL | — | ❌ 禁行 |
| V5 | 0x3EA50009 CFL | 0x9B3E CFL | — | ❌/黑屏 |
| **V6** | **0x3E9B0000 KBL** | **0x5916 KBL** | **3→19MB** | **⏳ 待测** |

---

## 十、验证命令集

```bash
# 一键诊断
echo "VRAM:" && system_profiler SPDisplaysDataType | grep "VRAM" && \
echo "Metal:" && system_profiler SPDisplaysDataType | grep "Metal" && \
echo "KBL FB:" && kextstat | grep "KBL" && \
echo "stolenmem:" && ioreg -l -w0 | grep "framebuffer-stolenmem"
```

---

> 📄 相关文档:  
> `/Volumes/OC/EFI修改日志.md` — 前 6 轮 EFI 修改日志  
> `/Volumes/OC/GPU驱动修复试错总结.md` — 用户 V6 总结  
> `/Volumes/OC/config_V6_backup.plist` — V6 修改前备份
