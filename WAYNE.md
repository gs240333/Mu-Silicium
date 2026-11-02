# waynePkg — Xiaomi Mi 6X (SDM660) UEFI firmware

Derived from `Platforms/Xiaomi/lavenderPkg` (Redmi Note 7, same SDM660) following
UniXNU's documented firmware layout, with the boot-chain integration copied from
`UniXNU/uefi-sheng` (`shengPkg/sheng.dsc`).

## What was changed and why

| Change | Reason |
| --- | --- |
| `PLATFORM_NAME/GUID`, `OUTPUT_DIRECTORY = Build/waynePkg`, file names | new platform identity |
| `USE_CUSTOM_DISPLAY_DRIVER = 0` | `Include/APRIORI.inc` then packs `SiliciumPkg/Drivers/SimpleFbDxe` instead of lavender's `DisplayDxe` + `DisplayReEnablerDxe`. SimpleFbDxe takes the frame buffer ABL already initialized — no panel driver, correct for wayne's 1080×2160 |
| `PcdFrameBufferHeight 2340 → 2160` | wayne panel |
| SMBIOS model/board strings | firmware self-description only; XNU identity comes from the payload ADT |
| `[Components]` + `[LibraryClasses]`: `UniXNU`, `UniXNUDxe`, `UniXNURamDiskDxe`, `DarwinBoot`, `UniXNULib`, `FdtLib` | required integration (`docs/porting/uefi-firmware`) |
| `Include/DXE.inc`: the four INF lines | that file is `!include`d by the FDF into `FV.FvMain`; a DSC-only component is built but never packed, and the symptom is a menu entry that no loader answers |

## Deliberately still pointing at lavender

| Reference | Why |
| --- | --- |
| `Binaries/lavender/QcomPkg/...` | the vendor DXE submodule has no `wayne/` yet. Same SoC, so this is the correct first attempt; board-specific pieces to re-import from the Mi 6X package are the four `RawFiles/` (`SecParti.cfg`, `tracer_event.cfg`, `QcomChargerCfg.cfg`, `BATTERY.PROVISION`) and any board cfg under `Binaries/` |
| `lavender/AcpiTables.inf` | the ACPI tables live in the `Silicium-ACPI` submodule, which has no `wayne/` directory. Same SoC tables are used as-is |

## Must be measured before trusting

`Library/MemoryMapLib/MemoryMapLib.c` still carries lavender's DDR layout. Every
board-describing region (Kernel size, DXE Heap, Display Reserved/splash, FV Region,
ABOOT FV, UEFI FD, SEC Heap, MMU PageTables, UEFI Stack, DBI Dump, Log Buffer,
Info Blk) has to come from the stock wayne boot image. A wrong region does not fail
the build — it hangs or corrupts memory at run time.

`Library/ConfigurationMapLib/ConfigurationMapLib.c` carries values such as
`NumActiveCores`, `Sdc1GpioConfigOn`, `TzAppsRegnAddr/Size`; check them against
wayne's stock DTB before relying on them.

## Build

The repository builds the platform with its own driver script:

```sh
./setup_env.sh -p apt
./build_uefi.py -d wayne -r RELEASE -c
# -> Mu-wayne.img
# -> Build/waynePkg/RELEASE_CLANGPDB/FV/SILICIUM_UEFI.fd   (0x300000 expected)
```

`.github/workflows/wayne.yml` does exactly that on `ubuntu-24.04`, and clones
`UniXNU` (for `UniXNUPkg`) and `DarwinBoot` (pinned) into `Platforms/Xiaomi/` so the
DSC references resolve. `unixnu build firmware --uefi ../uefi-wayne` is the
equivalent path from the UniXNU side.

## Boot chain

```
boot.img (BootShim + this FD + DTB) -> waynePkg -> UniXNU.efi (menu)
  -> DarwinBoot -> XNU
```

`Resources/Configs/wayne.toml` supplies `[uefi_fd] base/size` (FD placement),
`[boot_shim]`, and the boot-image header parameters. `base/size` are copied from
lavender because it is the same SoC — verify against wayne's stock boot image.
