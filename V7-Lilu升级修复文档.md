# V7 Lilu/WhateverGreen 升级修复 — Dell Latitude 3301

> 📅 2026-06-04 | 🍎 macOS 15.5 Sequoia (Darwin 24.5.0)  
> 🔄 修复类型: Kext 版本升级（不改 config.plist）

---

## 零、修改摘要

| # | Kext | 旧版 | 新版 | 来源 |
|---|------|------|------|------|
| 1 | Lilu.kext | **1.6.9** | **1.7.2** | EFI-DELL3301-FIXED-v2 (Acidanthera 官方) |
| 2 | WhateverGreen.kext | **1.6.8** | **1.7.1** | 同上 |
| 3 | AMFIPass.kext | **缺失** | **1.4.1** | EFI-V6-kextup |

备份: `Lilu_1.6.9_old.kext` / `WhateverGreen_1.6.8_old.kext` 在 `/Volumes/OC/EFI/OC/Kexts/`

---

## 一、根因分析

### 1.1 发现过程

```
V6 重启 → 仍 7MB VRAM
    ↓
检查 AppleIntelKBLGraphicsFramebuffer: 0 实例
    ↓
检查 IOReg: Lilu + WhateverGreen 均为 !registered, !matched
    ↓
Darwin 版本: 24.5.0 (macOS 15.5 Sequoia)
Lilu 版本: 1.6.9 (2024年Q1，基于 Darwin 23 编译)
    ↓
根因: Lilu 1.6.9 不支持 Darwin 24 内核符号表
```

### 1.2 技术原理

Lilu 是 Acidanthera 生态的**插件加载器**。它的工作原理:

1. 加载时扫描 Darwin 内核的符号表中关键函数地址
2. 将这些符号暴露给依赖 kext (WhateverGreen, AppleALC, VirtualSMC 等)
3. 通过这些符号，依赖 kext 可以 hook 内核函数、修改 IOKit 匹配、注入设备属性

Darwin 24 (macOS 15) 对内核符号表做了重新排列。Lilu 1.6.9 的符号查找表基于 Darwin 23，在 24 上:
- 关键符号偏移量错误 → 找不到函数
- IOKit 用户端注册失败 → `!registered, !matched`
- 依赖 kext 无法通信 → WhateverGreen 孤岛化

### 1.3 影响链路

```
Lilu !registered
  ↓
WhateverGreen 无法获取 Lilu API
  ↓
GPU 帧缓冲匹配过程中，WEG 无法:
  - 修改 AAPL,ig-platform-id 匹配
  - 应用 framebuffer-con*-alldata 补丁
  - 修正端口类型 (con1-type=HDMI)
  - 调整盗存内存报告
  ↓
KBL framebuffer 匹配失败
  ↓
IONDRVFramebuffer (VESA) 回退 → 7 MB VRAM
```

### 1.4 为什么其他 kext 正常

| Kext | 依赖 Lilu? | 状态 |
|------|-----------|------|
| itlwm.kext (WiFi) | ❌ | ✅ 正常 |
| SMCBatteryManager.kext | ❌ (直接对 VirtualSMC) | ✅ 正常 |
| VoodooPS2Controller.kext | ❌ | ✅ 正常 |
| VirtualSMC.kext | ✅ | ⚠️ 可能受限但足够核心功能 |
| WhateverGreen.kext | ✅ | ❌ 完全受阻 |
| AppleALC.kext | ✅ | ❌ 声卡也无法初始化 |

非 Lilu kext 直接通过 IOKit 匹配工作，不需要 Lilu 的符号表桥接。

---

## 二、kext 版本变更详情

### 2.1 Lilu: 1.6.9 → 1.7.2

| 属性 | 1.6.9 | 1.7.2 |
|------|-------|-------|
| CFBundleVersion | 1.6.9 | 1.7.2 |
| OSBundleCompatibleVersion | 1.2.0 | 1.2.0 |
| DTPlatformVersion (编译目标) | 13.3 (Ventura) | 14.x (Sonoma+) |
| Darwin 24 支持 | ❌ | ✅ |

### 2.2 WhateverGreen: 1.6.8 → 1.7.1

| 属性 | 1.6.8 | 1.7.1 |
|------|-------|-------|
| CFBundleVersion | 1.6.8 | 1.7.1 |
| Lilu 依赖 | ≥1.2.0 | ≥1.2.0 |
| kpi 依赖 | 10.0.0 | 10.0.0 |
| macOS 15 适配 | ❌ | ✅ |

---

## 三、完整 V7 配置状态

### 3.1 GPU 注入 (PciRoot(0x0)/Pci(0x2,0x0))

```
  AAPL,ig-platform-id       = 00009b3e   (0x3E9B0000, KBL GT2, 无内屏)
  device-id                 = 16590000   (0x5916, 原生 KBL-R UHD 620)
  model                     = Intel UHD Graphics 620

  framebuffer-patch-enable  = 01000000   (启用)
  framebuffer-stolenmem     = 00003001   (19MB)
  framebuffer-unifiedmem    = 00000006   (96MB)
  igfxonln                  = 01000000

  framebuffer-con0-enable   = 01000000
  framebuffer-con0-alldata  = 000008000200000098000000
  framebuffer-con1-enable   = 01000000
  framebuffer-con1-alldata  = 010509000008000087010000
  framebuffer-con1-type     = 00080000   (HDMI) ← V6 新增
  framebuffer-con2-enable   = 01000000
  framebuffer-con2-alldata  = 02040a000004000087010000
```

### 3.2 系统环境

| 参数 | 值 |
|------|-----|
| SMBIOS | MacBookPro15,2 |
| board-id | Mac-827FB448E656EC26 |
| Darwin | 24.5.0 |
| SIP | 状态未改（无关 kext 替换） |
| SecureBootModel | Default（无关） |

### 3.3 Kext 栈

```
/Volumes/OC/EFI/OC/Kexts/
├── Lilu.kext                   ← 1.7.2 (新)
├── WhateverGreen.kext          ← 1.7.1 (新)
├── VirtualSMC.kext
├── SMCBatteryManager.kext
├── SMCProcessor.kext
├── SMCDellSensors.kext
├── AppleALC.kext
├── itlwm.kext
├── VoodooI2C.kext
├── VoodooI2CHID.kext
├── VoodooPS2Controller.kext
├── GenericCardReaderFriend.kext
├── Lilu_1.6.9_old.kext         ← 备份
└── WhateverGreen_1.6.8_old.kext ← 备份
```

---

## 四、预期效果

重启后 GPU 初始化链路:

```
① Lilu 1.7.2 在 Darwin 24 上正确注册 ✅
② WhateverGreen 1.7.1 连接 Lilu API ✅
③ WEG 读取 DeviceProperties 注入值 ✅
④ WEG 修正 KBL framebuffer 端口匹配 ✅
⑤ AppleIntelKBLGraphicsFramebuffer 匹配成功 ✅
⑥ 帧缓冲初始化（19 MB stolemmen, con1=HDMI）✅
⑦ Metal/GL 驱动加载完成 ✅
⑧ VRAM: 1536 MB ✅
```

验证命令:
```bash
system_profiler SPDisplaysDataType | grep VRAM
# 预期: VRAM (Total): 1536 MB

kextstat | grep KBL
# 预期: AppleIntelKBLGraphicsFramebuffer 等

ioreg -l -w0 | grep -c "AppleIntelKBLGraphicsFramebuffer"
# 预期: ≥1

system_profiler SPDisplaysDataType | grep Metal
# 预期: Metal Support: Metal 3
```

---

## 五、故障排除

### 如果重启后仍 7MB

| 排查项 | 操作 |
|--------|------|
| Lilu 是否 registered | `ioreg -p IOService \| grep Lilu` → 应显示 registered |
| WEG 是否 registered | `ioreg -p IOService \| grep WhateverGreen` → 应显示 registered |
| 尝试 con0 主输出 | `framebuffer-con0-type=00080000` + con1/2 disabled |
| 尝试 0x3E920000 | 备选平台 ID `0000923E` |
| stolenmem 调至 64MB | `framebuffer-stolenmem=0000A001` |

### 如果重启后内核崩溃

1. OC 启动界面按空格
2. 选 macOS → 在 boot-args 加 `-liluoff`
3. 这会禁用 Lilu（也会禁用 WEG），但至少能进系统
4. 回退旧版 kext: 删除新版目录，将 `_old` 目录改名回来

---

## 六、回退方案

```bash
# 恢复到 Lilu 1.6.9 + WEG 1.6.8
cd /Volumes/OC/EFI/OC/Kexts/
rm -rf Lilu.kext
rm -rf WhateverGreen.kext
mv Lilu_1.6.9_old.kext    Lilu.kext
mv WhateverGreen_1.6.8_old.kext WhateverGreen.kext
```

---

## 七、版本历史

| 版本 | 主要变更 | GPU 配置 | Lilu/WEG | 状态 |
|------|---------|---------|----------|------|
| V1 | 基线 | 0x87C00000 | 1.6.9/1.6.8 | 安装通过 |
| V2 | WiFi | 0x3EA50009 CFL | 1.6.9/1.6.8 | 7MB 基线 |
| V3.1 | IRQ 尝试 | 同上 | 1.6.9/1.6.8 | 无效 |
| V4 | KBL SMBIOS | 0x59160003 | 1.6.9/1.6.8 | ❌ 禁行 |
| V5 | CFL device-id | 0x3EA50009+0x9B3E | 1.6.9/1.6.8 | ❌ 黑屏 |
| V6 | KBL 无内屏 | 0x3E9B0000+0x5916 | 1.6.9/1.6.8 | ⚠️ 7MB (Lilu 不兼容) |
| V7 | Lilu/WEG 升级 | 0x3E9B0000+0x5916 | 1.7.2/1.7.1 | ⚠️ Lilu 仍 !registered |
| **V7.1** | **+AMFIPass** | **0x3E9B0000+0x5916** | **1.7.2/1.7.1+1.4.1** | **⏳ 待测** |

---

> 📄 相关文档:  
> `/Volumes/OC/V6-KBL驱动修复技术文档.md` — V6 GPU 配置详解  
> `/Volumes/OC/GPU驱动修复试错总结.md` — V1-V6 用户试错总结  
> `/Volumes/OC/EFI修改日志.md` — 前 6 轮 EFI 修改日志  
> `/Volumes/OC/config_V6_backup.plist` — V6 修改前备份


---

## V7.1 增补: 添加 AMFIPass.kext

### 问题
V7 重启后 Lilu 仍显示 `!registered, !matched` — 升级到 1.7.2 不够。

### 根因
macOS 12+ 的 AMFI (Apple Mobile File Integrity) 阻止 Lilu hook 内核函数。
**没有 AMFIPass → Lilu 无法注册 → WEG 无法工作 → GPU 无加速。**

### 修复
| 操作 | 内容 |
|------|------|
| 文件 | 复制 `AMFIPass.kext` 1.4.1 (来源: EFI-V6-kextup) |
| config | Kernel → Add 位置 [0] (AMFIPass 必须比 Lilu 先加载) |
| 备份 | `config_V7_backup_YYYYMMDD_HHMMSS.plist` |

### Kernel Add 加载顺序
```
[0] AMFIPass.kext      ← 旁路 AMFI
[1] Lilu.kext          ← 插件加载器
[2] VirtualSMC.kext
[3] WhateverGreen.kext  ← GPU 补丁
[4] SMCBatteryManager.kext
...
```
