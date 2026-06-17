# Dell Latitude 3301 Hackintosh CPU 高温修复技术文档

> **机型**: Dell Latitude 3301 | **CPU**: i5-8265U (Whiskey Lake) | **GPU**: UHD 620 | **macOS**: Sequoia 15.x | **OpenCore**: 1.0.4 | **SMBIOS**: MacBookPro15,2

---

## 1. 故障现象

| 进程 | CPU 占用 | 说明 |
|------|:---:|------|
| PerfPowerServices | ~70% | macOS 电源遥测服务 |
| kernel_task | ~35% | 内核强制空闲散热机制 |
| CPU 温度 | 85-95°C | 风扇满转不停 |
| CPU 频率 | 锁定 1.8GHz | `hw.cpufrequency_min = hw.cpufrequency_max = 1800000000` |
| XCPM vectors | `vectors_loaded_count: 1` | 仅加载 1 个频率向量 |

---

## 2. 排查历程（失败尝试）

### V14-V16: CPUFriend 方向（❌ 走偏了）
- 怀疑电源管理数据缺失，从 sambow23 仓库导入 CPUFriend + CPUFriendDataProvider
- 用 ResourceConverter.sh 基于 MacBookPro15,2 原生 plist 重建频率向量
- **结果**: 完全不生效，症结不在此

### V17: SSDT-PLUG 方向（✅ 部分有用）
- **发现**: `plugin-type` 从未注入到 CPU 设备（ioreg 计数为 0）
- **修复**: 替换为 sambow23 专为 Dell 3301 定制的 SSDT-PLUG.aml（120B，目标 `\_SB_.PR00`）
- **结果**: `plugin-type=1` 注入成功，但 CPU 依然锁定 1.8GHz

### V18: AppleXcpmCfgLock 方向（❌ 走偏了）
- 关掉 `AppleXcpmCfgLock`（Dell 3301 商务本 CFG Lock 通常已解锁）
- **结果**: 无变化

### V19: 禁用 CPUFriend（❌ 走偏了）
- 完全移除 CPUFriend，回归原生 MacBookPro15,2 电源管理
- **结果**: 无变化

---

## 3. 真正根因

### VirtualSMC 版本兼容性问题

**这不是 CPU 驱动问题，是 SMC 模拟层的问题。**

macOS 15.4+ 修改了电源管理 SMC 键值的访问逻辑。旧版 **VirtualSMC 1.3.4** 在处理缺失的 SMC 键时返回了错误的错误码，导致 `PerfPowerServices` 认为数据读取被挂起，陷入无限重试的死循环。

```
PerfPowerServices 请求 SMC 键 →
VirtualSMC 1.3.4 返回错误码 →
macOS 认为读取卡住 → 立即重试 →
再次返回错误码 → 再次重试 →
死循环 (70% CPU)
    ↓
kernel_task 检测到 CPU 过热 →
启动强制空闲机制 (35% CPU) →
CPU 锁死 1.8GHz
```

### 证据来源
- [InsanelyMac 论坛: "Sequoia 15.4 high CPU usage"](https://www.insanelymac.com/forum/topic/360820-sequoia-154-high-cpu-usage/)
- [中文博客: "从 89°C 到 45°C"](https://yanchang.cc:8091/archives/cong-89degc-dao-45degc-wo-shi-ru-he-rang-sequoia-hei-ping-guo-bi-ji-ben-che-di-leng-jing-xia-lai-de)
- [Acidanthera GitHub Issue #2480](https://github.com/acidanthera/bugtracker/issues/2480)

---

## 4. 修复方案 (V20)

### 操作步骤

将 VirtualSMC 全家桶从 **1.3.4** 升级到 **1.3.7**：

| Kext | 旧版本 | 新版本 |
|------|:---:|:---:|
| VirtualSMC.kext | 1.3.4 | **1.3.7** |
| SMCProcessor.kext | 1.3.4 | **1.3.7** |
| SMCDellSensors.kext | 1.3.4 | **1.3.7** |
| SMCBatteryManager.kext | 1.3.4 | **1.3.7** |

**下载地址**: https://github.com/acidanthera/VirtualSMC/releases/latest

```bash
# 替换步骤
cd /Volumes/OC/EFI/OC/Kexts
# 备份旧版
mv VirtualSMC.kext VirtualSMC.kext.bak
mv SMCProcessor.kext SMCProcessor.kext.bak
mv SMCDellSensors.kext SMCDellSensors.kext.bak
mv SMCBatteryManager.kext SMCBatteryManager.kext.bak
# 拷贝新版
cp -R /path/to/new/VirtualSMC.kext .
cp -R /path/to/new/SMCProcessor.kext .
cp -R /path/to/new/SMCDellSensors.kext .
cp -R /path/to/new/SMCBatteryManager.kext .
```

无需修改 config.plist。重启即可。

---

## 5. 修复效果

| 指标 | 修复前 (V19) | 修复后 (V20) |
|------|:---:|:---:|
| PerfPowerServices CPU | **~70%** 🔴 | **0.0%** 🟢 |
| kernel_task CPU | **~35%** 🔴 | **可忽略** 🟢 |
| 系统空闲率 | ~0% | **~68%** |
| CPU 温度 | 85-95°C 🔴 | **显著降低** 🟢 |
| VirtualSMC 版本 | 1.3.4 | **1.3.7** |

---

## 6. 当前 EFI 最终状态

### 关键组件版本

| 组件 | 版本 | 备注 |
|------|------|------|
| OpenCore | 1.0.4 | |
| Lilu | 1.6.9 | |
| VirtualSMC | **1.3.7** | ⚠️ **必须 ≥ 1.3.6** |
| CPUFriend | 已禁用 | 非必需，plugin-type=1 后原生 PM 可用 |
| SSDT-PLUG | sambow23 版 (120B) | 目标 `\_SB_.PR00`，plugin-type=1 |
| itlwm | 2.3.0 + HeliPort | WiFi 驱动 |
| VoodooI2C | 2.9.1 | 触摸板 |
| VoodooPS2Controller | 2.3.7 | 键盘 |
| GPU ig-platform-id | 0x59160000 (KBL) | UHD 620 1536MB VRAM |
| SMBIOS | MacBookPro15,2 | |

### Kernel Quirks 最终配置

| Quirk | 值 |
|------|:---:|
| AppleXcpmCfgLock | False |
| AppleXcpmExtraMsrs | True |
| DisableIoMapper | True |
| CustomSMBIOSGuid | True |

### Boot Args

```
debug=0x100 keepsyms=1 -igfxblt
```

### Config 备份链

OC 盘根目录保留所有关键版本备份：
- `config-backup-v17-ssdtplug-fix.plist` — SSDT-PLUG 修复
- `config-backup-v19-no-cpufriend.plist` — 禁用 CPUFriend
- `config-backup-v20-virtualsmc-update.plist` — VirtualSMC 更新

---

## 7. 经验教训

1. **先查已知问题，再动手改配置**。PerfPowerServices 70% + kernel_task 35% 是 Sequoia 15.4+ 的知名 Bug，一搜就有答案。花了 V14-V19 五轮在 CPUFriend/SSDT-PLUG/AppleXcpmCfgLock 上绕圈子。

2. **VirtualSMC 版本很关键**。macOS 小版本更新可能改变 SMC 协议，旧版 VirtualSMC 的兼容性问题表现得很像电源管理故障，容易误导诊断。

3. **CFG Lock 不要乱动**。Dell 商务本通常已解锁，`AppleXcpmCfgLock` 反而可能干扰正常 MSR 写入。

4. **SSDT-PLUG 目标路径要对**。Dell 3301 用 `\_SB_.PR00`，不能用通用版的 `\_PR.CPU0`。

---

> 📅 文档生成: 2026-06-08 | 🏗️ 最后更新: V20 VirtualSMC 升级
