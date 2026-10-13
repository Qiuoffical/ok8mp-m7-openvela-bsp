# Forlinx OK8MP Cortex-M7 openvela BSP

Public source and supporting material for the proposed OK8MP Cortex-M7 board contribution to openvela.

Community discussion: [open-vela/nuttx#403](https://github.com/open-vela/nuttx/issues/403).

## Hardware

- Development board: Forlinx OK8MP.
- Processor: NXP i.MX 8M Plus, Cortex-M7 core.
- Console: UART4, 115200 baud, 8N1.
- Boot: U-Boot loads the M7 firmware from eMMC and starts it with `bootaux`.

## Repository contents

The following paths preserve the layout of the downstream project:

| Path | Contents |
| --- | --- |
| [src/source/nuttx/boards/arm/mx8mp/ok8mp-m7/](src/source/nuttx/boards/arm/mx8mp/ok8mp-m7/) | Board code, configurations, linker scripts and BSP README |
| [src/source/nuttx/arch/arm/src/mx8mp/](src/source/nuttx/arch/arm/src/mx8mp/) | i.MX8MP architecture source used by the downstream port |
| [docs/OK8MP_M7_openvela移植指南.md](docs/OK8MP_M7_openvela移植指南.md) | Downstream porting and boot documentation |
| [tests/xts/cases/](tests/xts/cases/) | Archived hardware test records |

This is a source snapshot for community review. It contains the four selected parts of the downstream project, rather than the complete openvela build workspace. Application code, packages and firmware referenced by the archived documentation are outside this snapshot. The architecture directory includes existing NuttX/openvela code as well as downstream changes; it is not presented as entirely new work.

## Existing implementation and validation

The downstream port includes board selection, Kconfig, Make/CMake integration, startup and bring-up hooks, DDR linker configuration, UART4 console support, and additional I2C3, ENET1/FEC and FlexCAN1 work.

Existing physical-board runs cover U-Boot loading, the UART4 NSH prompt, basic shell interaction and memory reporting. Additional downstream integration records cover peripheral and diagnostic functions. Test records retain their original configurations and test versions; they do not certify a new build made from this snapshot.

The first proposed upstream PR will focus on a minimal DDR-based NSH BSP. Its configuration will be separated from project-specific applications and verified against the target openvela `dev` revision before submission.

### Recorded manual boot procedure

```text
mmc dev 2
fatload mmc 2:1 0x80000000 nuttx.bin
md.l 0x80000000 2
bootaux 0x80000000
```

## License and attribution

Original source headers and the NuttX [LICENSE](LICENSE) and [NOTICE](NOTICE) files are preserved. Consult those files and individual source headers for the applicable licenses and attribution. Upstream operating-system and architecture code retain their original authorship.

## 中文说明

本仓库用于公开展示飞凌 OK8MP Cortex-M7 的 openvela 板级适配源码及已有实板验证材料，供社区讨论和审阅。目录与原项目保持一致。后续将从这些材料中整理基础 BSP 补丁，并提交至 openvela 社区。
