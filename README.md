# 联想拯救者 Y7000P IRX10（83NN）黑苹果 EFI — macOS 26 Tahoe

给 **Lenovo Legion Y7000P IRX10（83NN，i9-14900HX + RTX 5060 Laptop）** 用的 OpenCore EFI，目标是 macOS 26 Tahoe。
本仓库在我自己的机器上实测通过：**macOS 26.7.3（25G342）可安装、可引导，RTX 5060 在装有 NullMoth 驱动后可用（Metal 4）**。

> ⚠️ **这不是一个开箱即用的 EFI**。三码已清空、`SimpleHDA` 未包含，必须按下文「开用之前必须做的三件事」补完才能正常启动和出声。
> 换机型、换平台直接套用大概率起不来。

---

## 一、适用硬件

| 部件 | 型号 / ID |
|---|---|
| 机型 | Legion Y7000P IRX10（83NN） |
| CPU | Intel Core i9-14900HX（8P+16E，Raptor Lake-HX Refresh） |
| 独显 | NVIDIA RTX 5060 Laptop（`10DE:2D59`），8GB |
| 核显 | Intel UHD（`8086:A788`）— macOS 永远无驱动，忽略它 |
| 无线 | MediaTek MT7925（`14C3:7925`）+ 蓝牙 `0489:E111` |
| 有线 | Realtek RTL8111/8168（`10EC:8168`） |
| 声卡 | Realtek ALC257（layout-id 99） |
| 硬盘 | NVMe（已带 NVMeFix） |
| 目标机型 | `MacBookPro16,4`（SMBIOS 用 Custom 模式） |

**内核参数（boot-args）现值：**

```
-v debug=0x100 keepsyms=1 -vi2c-force-polling ctrsmt=full amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0 mtkfe=native mtknativeif=1 nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth
```

最后那串 `nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth` 是给 NullMoth NVIDIA 驱动用的，**第一次装系统时要去掉**（见下面的安装态配置）。

---

## 二、仓库内容

```
EFI/
├── BOOT/BOOTx64.efi              # OpenCore 引导壳
└── OC/
    ├── config.plist              # 日常态：三码已清空，等待你自己生成
    ├── OpenCore.efi              # OpenCore 1.0.8
    ├── ACPI/                     # SSDT-EC / GPI0 / MCHC / PLUG-ALT / RTCAWAC / SBUS / USB-Reset / USBX / XOSI
    ├── Drivers/                  # OpenHfsPlus / OpenRuntime / ResetNvramEntry
    ├── Kexts/                    # 28 个 kext（含 _licenses 许可证目录）
    ├── Resources/                # OpenCanopy 图标字体（PickerMode=Builtin，可不依赖）
    └── Tools/
config-install.plist              # 安装态：第一次装系统用这个
```

两个配置只差两处，但**决定了安装器会不会黑屏重启循环**：

| 项 | `config.plist`（日常态） | `config-install.plist`（安装态） |
|---|---|---|
| `Kernel > Block` → `com.apple.iokit.IONDRVSupport` | `Enabled = true`（屏蔽） | `Enabled = false`（不屏蔽） |
| `boot-args` | 带 `nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth` | 去掉这串 |

为什么：日常态的屏蔽是为了让 NullMoth 的 `NVRMFB` 独占 framebuffer（`boot framebuffer IONDRVFramebuffer steals index 0 from NVRMFB`）。但在**安装阶段还没有 NVRMFB**，屏蔽掉 `IONDRVSupport` 就等于把仅有的 framebuffer 提供者也关了 —— 安装器第二阶段起不来、因为 `-no_panic_dialog` 静默重启、OpenCore 5 秒后又回同一个 installer，就是你看到的「安装器一直在重复」。

---

## 三、开用之前必须做的三件事

### 1. 生成你自己的三码

仓库里的三码与 ROM 是清零状态（`000000000000` / `00000000000000000` / 全 0 UUID / ROM 八字节零）。不补上会因为机型信息全零而无法正常启动、也登录不了 Apple 服务。

- 工具：**OCAT**（OpenCore Auxiliary Tools）——Windows 版是单个文件夹，不用装 Python
  下载：`https://github.com/ic005k/OCAuxiliaryTools/releases`
- 步骤：
  1. 打开 `EFI/OC/config.plist`
  2. 左侧树点 **PlatformInfo**
  3. 机型下拉选 **`MacBookPro16,4`**
  4. 点 **Generate**，会一次性生成 `SystemSerialNumber` / `MLB` / `SystemUUID`
  5. `ROM` 填**你自己的网卡 MAC**（例如 `AA-BB-CC-DD-EE-FF`，OCAT 会自动转 base64）
  6. 保存

> 三码务必自己生成，不要和别人共用同一套；撞号会导致 iCloud / App Store 异常。如果登录时提示「无法验证设备」，重新生成一套即可。

### 2. 补 SimpleHDA（内建音频）

`SimpleHDA.kext` 因版权原因不随本仓库分发，需要自己从上游获取：

```bash
# macOS
./tools/get-SimpleHDA.sh
# Windows PowerShell
./tools/get-SimpleHDA.ps1
```

拿到后：

1. 放到 `EFI/OC/Kexts/SimpleHDA.kext`
2. 打开 config，`Kernel > Add` 里 `SimpleHDA.kext` 那条把 `Enabled` 改成 `true`
   （本仓库默认是 `false`，不改这条内建扬声器没有声音）

脚本地址：`https://github.com/arcticovo/legion-y7000p-irx10-tahoe-efi/tree/main/tools`

### 3. BIOS 设置

- **显卡模式切成「独显直连 / Discrete Graphics（dGPU Only）」** —— 内屏方案依赖它
- **Secure Boot / 安全启动 → 关闭**
- 这台机器内屏、以及绝大多数外接接口都挂在 RTX 5060 上；macOS 侧的驱动是装完系统才装的，所以**安装阶段必须接外接显示器**

---

## 四、装机流程（逐步）

### 步骤 0：制作安装 U 盘

1. 一个 ≥16GB U 盘，`磁盘工具`抹成 **APFS（GUID 分区图）**，命名随意
2. 下载 macOS Tahoe 安装器，在终端执行（`/Volumes/你的U盘` 换成实际名称）：

```bash
sudo /Applications/Install\ macOS\ Tahoe.app/Contents/Resources/createinstallmedia \
  --volume /Volumes/你的U盘 --nointeraction
```

3. 把本仓库的 `EFI/` 整个拷到该 U 盘的 **EFI 分区**（用 `磁盘工具`挂载 EFI 后拷贝，或用 OCAT 的挂载功能）
4. **用安装态配置**：把 `config-install.plist` 复制进 `EFI/OC/` 并改名为 `config.plist`
   —— 这一步做错，安装阶段大概率黑屏 + 重启循环

### 步骤 1：启动进安装器

1. 接好外接显示器，U 盘插上，重启
2. 开机连按 **F12**，选 `UEFI: <你的U盘>`（若列表里没有，进 F2 的 BIOS，Boot 里把 U 盘置顶）
3. OpenCore 菜单里选 **`macOS Base System`**
4. 会先滚一段白字（`-v` 是开着的），然后出现 Apple 标志与进度条

> 判定标准只有一个：**第二阶段有没有出现进度条**。全程黑屏先等满 5 分钟，不要急着断电。

### 步骤 2：抹盘 + 安装

1. 进安装器，选语言
2. 顶部菜单 **实用工具 → 磁盘工具**
3. 左上角 **显示 → 显示所有设备**
4. 选中目标盘的 **APFS 容器/卷**（本机是内置 NVMe 上的 270GB 分区），**只抹它**，双系统里的 Windows 分区不要动
5. 抹掉：名称 `macOS`、格式 `APFS`、方案 `GUID 分区图`
6. 关掉磁盘工具 → 返回安装器 → **安装 macOS Tahoe** → 选刚抹的磁盘
7. 等待（约 20–40 分钟，中间会自动重启）

### 步骤 3：第一次重启 —— 最容易踩的坑

安装完第一阶段会自动重启，回到 OpenCore 菜单时**会多出一项 `macOS Installer`**：

- ✅ **选 `macOS Installer`**（它来自内置盘的 `macOS Install Data`，是第二阶段）
- ❌ 不要又选 U 盘上的 `macOS Base System`

选错就会从头再装一遍，并且因为第二阶段反复失败而死循环 —— 这就是「安装器一直重复」的直接原因。

第二阶段可能还会再重启一次，之后选 **`macOS Tahoe`**（内置盘）进入系统，走完设置助手。

### 步骤 4：装 NVIDIA 驱动

这一步不做，内屏点不亮、也没有硬件加速。

- 用 **NullMoth 2.8.1**（1401 图形工具）安装：`NVRM` / `NVRMFB` / `NVAccel` / `NVRMAGDC` / `NMXAudio`
- 安装器会往 `/Library/Extensions` 装 kext，并自动在 config 里补上 `nvfb` 那串 boot-args、把 `IONDRVSupport` 屏蔽
- 重启后系统信息里应该能看到 `NVIDIA GeForce RTX 5060 Laptop GPU` 且 `Metal` 有值

### 步骤 5：换回日常态配置

```text
把 config-install.plist（或从本仓库 EFI/OC/config.plist）放回 EFI/OC/config.plist
```

日常态 = `IONDRVSupport` 屏蔽 + `nvfb` 参数。装完驱动后必须用这一个，否则内屏和加速都出不来。

### 步骤 6：把 EFI 写进内置硬盘（很重要）

只在 U 盘上有 EFI，拔了 U 盘就开不了机。装进内置盘 ESP：

```bash
diskutil list                        # 找到内置盘的 EFI 分区，例如 disk0s1
sudo diskutil mount disk0s1
cp -R /Volumes/你的U盘/EFI /Volumes/EFI/
sudo bless --mount /Volumes/EFI --setBoot \
  --file /Volumes/EFI/EFI/OC/OpenCore.efi --shortform
```

然后进 BIOS，把内置硬盘的 EFI 启动项（`EFI\OC\OpenCore.efi`）置顶。
验证方法：**关机、拔掉 U 盘、开机** —— 能进 OpenCore 菜单并进入 macOS 就算成功。

---

## 五、常见问题排查

| 现象 | 原因 / 处理 |
|---|---|
| 装到一半黑屏、自动重启、反复回到 installer | 用了日常态配置：`IONDRVSupport` 被屏蔽 + 没有 NVRMFB → 没有 framebuffer。换 `config-install.plist` |
| 第一阶段正常，第二阶段起不来 | 同上；另外确认目标盘的 `macOS Install Data` 是干净的（失败过一次就整个抹掉重来） |
| 装完进不去、卡在 Apple 标志 | 三码是否已生成；`SecureBootModel` 是否 `Disabled`；BIOS 是否独显直连 |
| 内屏黑、只有外接屏亮 | NVIDIA 驱动没装，或装完没换回日常态配置 |
| 没声音 | `SimpleHDA.kext` 没补 / `Kernel > Add` 里那条 `Enabled` 还是 `false` |
| 触控板不动 | 已知问题，见下 |
| 想看引导日志 | U 盘根目录会生成 `opencore-*.txt`；关键行：`#[EB|LOG:EXITBS:START]`（内核已交接）、`OC: Prelinked blocker ... IONDRVSupport`（framebuffer 被挡）、`#[EB|SB:P] 0x1`（安全启动开着） |

改了 config 之后建议用官方校验器过一遍：

```bash
# OpenCore 1.0.8 的 ocvalidate
./Utilities/ocvalidate/ocvalidate EFI/OC/config.plist
# 应该输出：No issues found.
```

---

## 六、已知限制（照上游 README 的诚实声明，别白折腾）

- **触控板不可用**：`VoodooI2C` 会加载，但枚举不到 `AppleMultitouchDevice`
- **睡眠 / 唤醒不工作**
- **改分辨率可能导致画面冻结**
- **AirDrop / 接力 / 通用控制 / iPhone 镜像 / Mac 上的 AirPlay 接收器全部不可用**
  原因：iPhone 镜像硬性要求 Apple 芯片或 T2 安全芯片（本机没有 T2）；且第三方无线驱动不创建 AWDL 接口（`awdl0` 不存在）
- **`amfi=0x80`** 会让部分 TCC 权限弹窗不出现（例如屏幕录制、输入监控的授权窗口）

---

## 七、许可与致谢

本仓库的 EFI 基于以下上游，版权与许可归原作者：

- **EFI 与配置**：[`arcticovo/legion-y7000p-irx10-tahoe-efi`](https://github.com/arcticovo/legion-y7000p-irx10-tahoe-efi)（本仓库的上游来源）
- **OpenCore / macserial / ocvalidate**：[acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)（BSD-3-Clause）
- **Lilu / WhateverGreen / VirtualSMC / AppleALC**：acidanthera
- **VoodooI2C / VoodooPS2Controller**：VoodooI2C 社区
- **MediaTek 无线三件套**（`MtkWlan` / `MtkWlanNative` / `MtkBt` / `LegionBtFix`）：许可证随包放在 `EFI/OC/Kexts/_licenses/`
- **NullMoth NVIDIA 驱动**：不在本仓库内，需自行获取并自行承担风险
- **SimpleHDA**：按上游声明不随仓库分发，请用上游脚本自行获取

---

## 免责声明

黑苹果属于非官方支持的用法。刷 BIOS、改分区、装第三方 kext 都有风险，可能损坏系统或丢失数据。
本仓库按现状提供，不提供任何担保；请确认自己知道每一步在做什么之后再动手，重要数据先备份。

---

> 实测环境补充：本机为内屏 2560×1600@240Hz + 外接 2560×1440@60Hz 双屏在线；有线网卡在 macOS 下为 `en0`；
> 电池、SMC 传感器、内建音频、Wi-Fi、蓝牙均正常。`config.plist` 与 `config-install.plist` 均已通过 OpenCore 1.0.8 官方 `ocvalidate` 校验。
