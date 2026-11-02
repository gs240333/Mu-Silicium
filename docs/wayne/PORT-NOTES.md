# uefi-wayne：从 Mu-Silicium 派生 waynePkg 的改动清单

上游依据：`Project-Silicium/Mu-Silicium` 已含 `Platforms/Xiaomi/lavenderPkg`（Redmi Note 7，同为 SDM660），
以及 `Resources/Configs/lavender.toml`（FD base `0x9FC00000` / size `0x300000`、gzip、append_dtb、header v1）。
wayne 与 lavender 同 SoC、同代 ABL，所以固件这一层是"改板子"，不是"写平台"。

## 0. 先做一次"读"而不是"写"

```sh
git clone --recurse-submodules https://github.com/Project-Silicium/Mu-Silicium
cd Mu-Silicium
ls Platforms/Xiaomi/lavenderPkg
sed -n '1,80p' Platforms/Xiaomi/lavenderPkg/lavender.dsc
grep -rn "Fb\|Display\|SimpleFb" Platforms/Xiaomi/lavenderPkg/*.dsc Platforms/Xiaomi/lavenderPkg/*.fdf
```
要回答三个问题，答案决定后面怎么改：
1. lavender 的显示路径是走自带的 panel 初始化，还是走 `SimpleFbDxe`？
2. `Binaries/` 里哪些 DXE 是 SDM660 通用（时钟/PMIC/TLMM/USB/eMMC），哪些是 lavender 专属？
3. `lavender.fdf` 里 FD 的布局与大小是否真的落在 `0x9FC00000 + 0x300000`。

## 1. 建包

```sh
cp -r Platforms/Xiaomi/lavenderPkg Platforms/Xiaomi/waynePkg
cd Platforms/Xiaomi/waynePkg
mv lavender.dsc wayne.dsc
mv lavender.fdf wayne.fdf
grep -rl lavender . | xargs sed -i 's/lavender/wayne/g'
```
同时检查 `Platforms/Xiaomi/*.dsc` 的引用列表、`.github/workflows` 里是否只编自己。

## 2. 必须改的板级差异

| 项 | lavender | wayne 要做的事 |
|---|---|---|
| SMBIOS / 型号串 | Redmi Note 7 | 改成 Mi 6X（`DEVICE_MODEL = "wayne"`），只用于固件自述，不影响 XNU 身份 |
| 面板 | lavender 自己的 resolution | 1080x2160；**优先保留 ABL 点亮的 splash 缓冲**，别自己跑 panel init（见第 3 条） |
| 触摸 | lavender 的触摸 | 第一阶段不需要，UEFI 只靠音量/电源键选菜单 |
| 分区表 | lavender | 无关；固件只读 `EFI\UniXNU\unixnu.toml` 所在卷 |
| FD base/size | `0x9FC00000 / 0x300000` | 先照抄；若 ABL 报 `Failed to load/authenticate boot image`，说明 FD 越界或 image_size 语义不对 |
| boot header | v1 | **实测** stock boot.img 后决定，见 `Resources/Configs/wayne.toml` 注释 |

## 3. 显示：让 ABL 的 splash 直接变成 GOP

原理：ABL 已经把面板点亮并把画面留在 `cont_splash_mem`，所以 UEFI 阶段只要把这个物理缓冲包装成
`EFI_GRAPHICS_OUTPUT_PROTOCOL`，UniXNU 菜单和 DarwinBoot 的 framebuffer 交接就都有了，**完全不需要写 DSI 面板驱动**。

做法（按第 0 步读出来的结果二选一）：
- 若 lavender 已经用 `SimpleFbDxe` + splash：只需改分辨率/stride 常量；
- 若 lavender 会跑自己的 panel init：在 `wayne.dsc`/`wayne.fdf` 里把那条路径换成 `SimpleFbDxe`，
  并把 splash 的物理地址与大小写进平台的 PCD/宏。

验证信号：UniXNU 菜单能显示、DarwinBoot 日志里有
`DARWINBOOT: video base=... width=1080 height=2160 depth=32`。depth 必须是 32，且像素序为 B/R/G/A（BGRA）。

## 4. 把 UniXNU 全家桶塞进 FD（上游硬要求）

`wayne.dsc` 的 `[Components]` 与 `wayne.fdf` 的 `INF` 都要有：

```
UniXNUPkg/Application/UniXNU/UniXNU.inf
UniXNUPkg/Driver/UniXNUDxe/UniXNUDxe.inf
DarwinBootPkg/Driver/DarwinBoot/DarwinBoot.inf
UniXNUPkg/Driver/UniXNURamDiskDxe/UniXNURamDiskDxe.inf
MdeModulePkg/Universal/Disk/DiskIoDxe/DiskIoDxe.inf
FatPkg/EnhancedFatDxe/Fat.inf
MdeModulePkg/Universal/Disk/PartitionDxe/PartitionDxe.inf   # 需要读分区表时必须
```
`[LibraryClasses]` 要补：
```
HobLib|MdePkg/Library/DxeHobLib/DxeHobLib.inf
FdtLib|MdePkg/Library/BaseFdtLib/BaseFdtLib.inf
```
漏了 DarwinBoot 的症状很典型：菜单里能看到 xnu 条目，点下去什么都不发生（因为没有 loader 接管）。
漏了 `UniXNURamDiskDxe`/`DiskIoDxe`/`Fat` 的症状是：菜单起不来，因为读不到 `\EFI\UniXNU\unixnu.toml`。

## 5. 厂商 DXE 从 Mi 6X 自己的固件包里取

不要从别的设备拿二进制。`board.toml` 的 `[firmware.image]` 指向 Mi 6X 官方 fastboot 包，
由 `scripts/download-*.py` / `extract-*.py` 解出（sheng 的 `preflight` 就是这套写法），
再导入 `waynePkg/Binaries/`。每加一个 blob，都在 `proprietary-files.txt` 里记 hash 与用途。

## 6. 构建与验收

```sh
# 在 UniXNU 主 checkout 里
./unixnu build firmware --uefi ../uefi-wayne
ls -l ../uefi-wayne/Build/waynePkg/*/FV/SILICIUM_UEFI.fd     # 期望 0x300000 = 3145728 B
./unixnu build bootimg
```
真机：
```powershell
cd C:\platform-tools
.\fastboot.exe boot D:\path\to\boot.img
```
- 验收 A：屏幕上出现 UEFI 标识 + UniXNU 菜单。
- 验收 B：`fastboot devices` 出现固件上报的名字（如果 UniXNU 已经按 BootOrder 自动启动，则会变空，属正常）。
- 回 Android：**长按电源 10~15 秒断电**。UEFI 的 reset 是热复位，ABL 还会从 DDR 里的镜像启动。

## 7. 纪律

- bring-up 阶段只 `fastboot boot`，不 `fastboot flash boot`。SDM660 的 ABL 对未签名镜像很敏感，
  写坏之后要靠断电 + 刷回 stock 才能恢复。
- 每次只改一个变量（分辨率 / header_version / FD base 三样永远别一起动）。
- 固件体积有上限：RAM 启动的 boot.img 越小越稳；payload 放到分区里（`[payload] volume` 的 FAT 卷），
  不要塞进 boot.img。
