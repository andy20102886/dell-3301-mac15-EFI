# GPU 驱动修复试错总结 — Dell Latitude 3301 黑苹果

> 📅 截至 2026-06-04 | 🍎 macOS 15.5 Sequoia
> 🖥️ Dell Latitude 3301 / i5-8265U Whiskey Lake / UHD 620 (0x5916)
> 🔑 特殊：无内置屏幕，仅 HDMI 外接显示器
> 🔧 SMBIOS: MacBookPro15,2 (CFL)

---

## 一、当前 V6 配置（待测试）

| 参数 | 值 | 说明 |
|------|-----|------|
| ig-platform-id | **0x3E9B0000** (AACbPg==) | 🔑 KBL，原生无 eDP/LVDS，仅 3 个数字输出 |
| device-id | 0x5916 (FlkAAA==) | 硬件原生 ID（KBL-R HD 620） |
| con0/1/2-enable | 全部 1 | 保留源文件 |
| stolenmem | 3MB | |
| unifiedmem | 96MB | |
| igfxonln | 1 | DeviceProperties |
| boot-args | debug=0x100 keepsyms=1 | |

**本次只改了一个值：**
```
ig-platform-id: 0x3EA50009 (CFL, 有内屏) → 0x3E9B0000 (KBL, 无内屏)
```

## 为什么选 0x3E9B0000
- Kaby Lake 桌面/无内屏 framebuffer 人格
- 原生 3 个 DP/HDMI 数字输出，无 eDP/LVDS
- 驱动初始化时不会去找不存在的内屏
- macOS 15 保留了 KBL 驱动（AppleIntelKBLGraphicsFramebuffer）

---

## 二、全部试错记录

### 方案 1：源文件 ❌
ig-platform-id: 0x87C00000 + device-id: 0x5916
→ 安装通过（VESA），完整启动 IOFB panic

### 方案 2：Mac13 配置 ❌
ig-platform-id: 0x59160003
→ IOFB panic

### 方案 3：CFL 伪装 → 7MB VRAM ⚠️ ← V2 基线
ig-platform-id: 0x3EA50009 + device-id: 0x5916
→ 进桌面、WiFi 可用，但 VRAM 7MB
→ 原因: macOS 15 移除了 CFL 驱动

### 方案 4：OCLP ❌
→ OCLP "No patches required"（SMBIOS MacBookPro15,2 是 CFL 机型）
→ 换 MacBookPro14,2 → macOS 15 禁行标志

### 方案 5-7：CFL device-id 伪装 ⚠️
ig-platform-id: 0x3EA50009 + device-id: 0x9B3E (CFL UHD 630)
→ 进度条 1/3 后 HDMI 黑屏
→ 原因: macOS 15 无 CFL 驱动

---

## 三、核心结论

### macOS 15 GPU 驱动格局
| 驱动 | macOS 12 | macOS 13 | macOS 15 |
|------|----------|----------|----------|
| KBL | ✅ | ❌ | ✅ |
| CFL | ✅ | ✅ | ❌ |

### 7MB 根因
帧缓冲驱动初始化必须先找到主显示器（con0=eDP/LVDS 内屏）。找不到→失败→VESA→7MB。
本机无内屏，所有带内屏连接器的 ig-platform-id 都失败。

### 唯一方向
用 **无内屏连接器的 framebuffer 人格** + 当前 macOS 版本仍存在的驱动

| macOS | 存在驱动 | 应用 ig-platform-id |
|-------|---------|-------------------|
| macOS 12 | KBL+CFL | 0x3E9B0000 (KBL 无内屏) |
| macOS 13 | 仅 CFL | CFL 无内屏等效（未知） |
| **macOS 15** | **仅 KBL** | **0x3E9B0000** ← 当前 V6 |

---

## 四、测试验证命令

```bash
# VRAM（正常 1536 MB）
system_profiler SPDisplaysDataType | grep VRAM
# KBL 驱动
kextstat | grep -i "AppleIntelKBL\|AppleIntelFramebuffer"
# Metal
system_profiler SPDisplaysDataType | grep Metal
```

## 五、V6 失败后的方向

1. WhateverGreen 版本 → macOS 15 需 WEG 2.0+
2. device-id 伪装到纯 KBL ID (0x5912/0x591B)
3. busid 补丁匹配硬件
4. SMBIOS 兼容性（CFL SMBIOS + KBL framebuffer）
5. -igfxsklaskbl 等 WEG boot-args
6. 降级 macOS 12（最后一版双驱动）

## 六、版本历史

| 版本 | ig-platform-id | device-id | 状态 |
|------|---------------|-----------|------|
| V1 | 0x3EA50009 | 0x5916 | 进桌面，无WiFi |
| V2 | 0x3EA50009 | 0x5916 | ✅ 基线，WiFi，7MB |
| V3.1 | 0x3EA50009 | 0x5916 | IRQ尝试(无效) |
| V4 | 0x59160003 | 0x5916 | ❌ 禁行 |
| V5 | 0x3EA50009 | 0x9B3E | ❌/黑屏 |
| **V6** | **0x3E9B0000** | **0x5916** | ⏳ 待测 |
