<!-- SPDX-License-Identifier: Apache-2.0 -->

# OK8MP Cortex-M7 openvela/NuttX BSP 移植指南

## 1. 文档范围

本文说明 OK8MP 开发板 Cortex-M7 核的 openvela/NuttX BSP 设计、配置、构建、
启动和实机验收方法。适用目标为 i.MX8M Plus（i.MX8MP）异构处理器，M7 固件由
U-Boot `bootaux` 启动，串口控制台使用 UART4。

本文同时作为第三届中国研究生操作系统开源创新大赛“基于 openvela 的新平台适配
与 AI Agent 智能应用开发”项目的路径 A 移植记录。文中只把已经在 OK8MP 实机上
验证的资源列为“已完成”；保留但未在目标板确认的参考代码默认关闭，不计入验收结果。

## 2. 路径 A 判定和新增范围

所选 NuttX 上游基线已经具备 i.MX8MP 架构层和其他厂商开发板支持，但不存在
`boards/arm/mx8mp/ok8mp-m7`。本项目复用了通用的 i.MX8MP CPU、异常入口和基础
外设接口，重新完成了 OK8MP 板级 BSP。路径 A 的目标是“上游未支持的开发板”，
不是宣称 i.MX8MP 芯片架构此前完全不存在。

可用以下命令核对上游基线和本项目的新增范围：

```sh
git ls-tree -r f0c94bb64acff1471d4ba5b23b1e0e5c8d57019c \
  -- boards/arm/mx8mp/ok8mp-m7
git show --stat 84f25cba13
```

`f0c94bb64acff1471d4ba5b23b1e0e5c8d57019c` 是 OK8MP 首次提交的直接父版本。
第一条命令在该基线上无输出，第二条命令可查看 OK8MP BSP 首次提交的 22 个板级
文件。新增工作包括：

- OK8MP 板级 Kconfig、四套 defconfig、Make/CMake 构建入口；
- UART4 控制台、LED、GPIO、定时器和板级 bring-up；
- DDR/TCM 两套链接布局和 U-Boot `bootaux` 启动流程；
- ENET1/FEC、YT8521 PHY、RGMII、RDC、OCRAM DMA 和 NuttX `net_driver_s`；
- M7 I2C3 时钟、门控、IOMUX 和 `/dev/i2c3`；
- NuttX 上游不存在的 OK8MP 语音识别/RGB 模块驱动
  `/dev/voicesensor0`；
- uORB 语音主题适配；
- openvela 官方 AI Agent、严格 TLS、板级 C Tool 和 Markdown Skill 的配置入口。

## 3. BSP 目录和代码规范

### 3.1 目录职责

```text
boards/arm/mx8mp/ok8mp-m7/
├── Kconfig                         板级可配置项
├── CMakeLists.txt                  CMake 构建入口
├── include/board.h                 公共板级定义和接口
├── scripts/
│   ├── Make.defs                   链接脚本选择
│   ├── ddr.ld                      完整系统 DDR/OCRAM 布局
│   └── itcm.ld                     小型诊断固件 TCM 布局
├── configs/
│   ├── nsh/defconfig               基础 NSH 与外设诊断
│   ├── nsh-net/defconfig           以太网/TCP-IP 联调
│   ├── nsh-agent/defconfig         比赛最终配置
│   └── rpmsg/defconfig             A/M 核通信参考配置
└── src/
    ├── ok8mp-m7.h                  板级私有引脚和原型
    ├── mx8mp_boot.c                早期初始化入口
    ├── mx8mp_bringup.c             设备注册和文件系统挂载
    ├── mx8mp_i2cdev.c              I2C 字符设备注册
    ├── ok8mp_voice_i2c.c           语音模块驱动
    ├── ok8mp_fec.c                 ENET1/FEC 网络驱动
    └── mx8mp_*                     LED、按键、SPI 等板级实现
```

公共接口放在 `include/board.h`；只供本板源码使用的寄存器、引脚和内部函数放在
`src/ok8mp-m7.h`。驱动不直接引用用户态应用头文件，应用通过 NuttX 设备节点、
`ioctl`、网络接口或 uORB 主题访问硬件。

### 3.2 命名和配置规则

- 板级配置统一以 `CONFIG_OK8MP_M7_` 为前缀；SoC 通用配置保留
  `CONFIG_MX8MP_` 前缀。
- 可选功能必须由 Kconfig 控制，不用源码中的临时常量代替配置项。
- `defconfig` 只保存可复现的最终选项，不保存 API Key、证书私钥、主机绝对路径
  和个人账号。
- 驱动向上返回 NuttX 约定的负 errno；日志同时给出总线、地址、寄存器和返回值。
- 寄存器轮询必须设置超时，禁止无限等待硬件状态。
- 新文件使用 Apache License 2.0 SPDX 标识；文件头路径必须与 OK8MP 目录一致。
- 参考板上未确认的 LED、按键、INA219、SPI 资源不得默认启用，也不得作为 OK8MP
  实机完成项。

### 3.3 Kconfig

`Kconfig` 提供以下板级开关：

| 配置项 | 作用 | 默认值 |
| --- | --- | --- |
| `MX8MP_RUN_FROM_ITCM` | 选择 TCM 诊断链接脚本 | `n` |
| `OK8MP_M7_VOICE_I2C` | 注册 OK8MP 语音模块 | `n` |
| `OK8MP_M7_VOICE_I2C_BUS` | M7 侧 I2C 控制器编号 | `3` |
| `OK8MP_M7_VOICE_I2C_ADDR` | 语音模块 7 位地址 | `0x0f` |
| `OK8MP_M7_VOICE_I2C_REG` | `read()` 默认读取的识别结果寄存器 | `0x08` |
| `OK8MP_M7_VOICE_I2C_FREQ` | I2C 时钟 | `100000` Hz |
| `OK8MP_M7_FEC_PROBE` | ENET1/PHY/DMA 诊断接口 | `n` |
| `OK8MP_M7_FEC_NET` | 注册标准 NuttX `eth0` | `n` |
| `OK8MP_M7_HEARTBEAT` | M7/A53 共存诊断心跳 | `n` |

修改 Kconfig 后必须执行 `make olddefconfig` 或重新运行 `menuconfig`。不得只手工修改
生成的 `.config`，否则清理构建目录后无法复现。

### 3.4 defconfig 分工

| 配置 | 用途 | 地址/资源 | 是否用于最终验收 |
| --- | --- | --- | --- |
| `nsh` | UART、GPIO、I2C、语音驱动基础诊断 | DDR 基础配置 | 否 |
| `rpmsg` | RPMsg UART 和 A/M 核通信验证 | TCM/共享内存 | 否 |
| `nsh-net` | FEC、IPv4、DNS、TCP 联调 | DDR；旧联调网段 `192.168.50.0/24` | 否 |
| `nsh-agent` | 网络、TLS、uORB、AI Agent、Skill/Tool | DDR；`192.168.2.0/24` | 是 |

`nsh-agent` 是提交和演示使用的唯一完整配置。其主要网络参数为：

```text
M7 IPv4       192.168.2.2/24
默认网关      192.168.2.1
DNS           1.1.1.1
MAC           由 OK8MP FEC 驱动设置
```

`nsh-net` 保留早期网络调试参数，不能和最终验收日志混用。若修改网络拓扑，应更新
`nsh-agent/defconfig`，再重新生成并保存最小 defconfig。

## 4. 启动与内存设计

### 4.1 完整固件 DDR 布局

`scripts/ddr.ld` 用于 `nsh-net` 和 `nsh-agent`：

| 区域 | M7 地址 | 长度 | 用途 |
| --- | ---: | ---: | --- |
| `flash` | `0x80000000` | 1 MiB | 向量、代码、只读数据、加载映像 |
| `sram` | `0x80100000` | 1 MiB | `.data`、`.bss`、堆栈和运行数据 |
| `ocram` | `0x20200000` | 32 KiB | 非缓存 `.fec_dma` |

FEC 总线主设备看到的 OCRAM 别名为 `0x00900000`，M7 本地地址为
`0x20200000`，因此驱动使用：

```text
FEC_DMA_ADDR = M7_LOCAL_ADDR - 0x1f900000
```

描述符和收发缓冲区放入 `NOLOAD` 的 `.fec_dma` 段并按 64 字节对齐，链接脚本导出
`__fec_dma_start__` 和 `__fec_dma_end__`。该区由驱动初始化，不从固件映像复制。

DDR 区域必须与 A53 Linux 的内存规划相互排斥。当前比赛启动流程是在 A53 Linux
运行时把固件复制到 FAT 分区，随后回到 U-Boot 停止 Linux，再启动 M7；不允许 Linux
和 M7 同时使用上述 DDR 区域或 ENET1 控制器。

### 4.2 TCM 诊断布局

`scripts/itcm.ld` 用于小型启动/RPMsg 诊断：

| 区域 | 地址 | 长度 |
| --- | ---: | ---: |
| ITCM `flash` | `0x00000000` | 128 KiB |
| DTCM `sram` | `0x20000000` | 128 KiB |

完整 Agent、mbedTLS 和网络栈超过 TCM 容量，最终固件不得选择
`CONFIG_MX8MP_RUN_FROM_ITCM=y`。TCM 映像由 U-Boot 临时加载到 DDR 后复制到 M7
TCM 总线窗口，详见第 9.3 节。

### 4.3 启动顺序

板级初始化遵循以下顺序：

1. M7 复位，设置向量、栈、时钟和 UART4；
2. 初始化 GPIO 和系统定时器，进入 NuttX；
3. `board_late_initialize()` 调用 OK8MP bring-up；
4. 挂载 `procfs`，为 Agent 挂载 `/data` tmpfs；
5. 配置 I2C3 root clock、CCGR 和 IOMUX，注册 `/dev/i2c3`；
6. 探测语音模块并注册 `/dev/voicesensor0`；
7. 配置 RDC、ENET1、YT8521 和 OCRAM DMA，注册 `eth0`；
8. 运行 netinit 和 NSH；uORB、HTTPS 工具和 Agent 由应用命令启动。

FEC 必须先于 netinit 注册，否则静态 IP 已配置但系统中没有可承载它的网络设备。

## 5. 板级外设实现

### 5.1 UART、GPIO 和定时器

- 控制台：UART4，115200 bit/s，8N1；
- 启动后出现 `NuttShell (NSH)` 并可连续输入命令，才算控制台完成；
- LED1/LED2 使用板级确认的 GPIO5_IO08/GPIO5_IO09；其他测试映射按载板版本
  复核后才能启用；
- `ostest`、`sleep` 和周期任务用于验证 SysTick、调度和定时器；
- 长时间测试必须同时观察 NSH 响应、网络和后台 uORB 任务，不能只看 LED 心跳。

### 5.2 I2C3 和语音模块驱动

语音模块连接到 M7 I2C3，7 位地址为 `0x0f`，总线速率 100 kHz。驱动注册
`/dev/voicesensor0`，为字符设备提供 `read`、`write` 和 `ioctl`。已使用的寄存器如下：

| 地址 | 功能 |
| ---: | --- |
| `0x01` | 添加识别词条 |
| `0x02` | 识别模式 |
| `0x03` | RGB 控制 |
| `0x04` | 灵敏度 |
| `0x05` | 清除词条 |
| `0x06` | 启动识别 |
| `0x07` | 提示音 |
| `0x08` | 最新识别结果 |
| `0x09` | 蜂鸣器 |
| `0x0a` | 词条数量 |
| `0x0b` | 固件版本 |
| `0x0c` | busy 状态 |

驱动初始化时先读取版本寄存器。字符设备、uORB 和 Agent 的分层关系为：

```text
I2C3 -> /dev/voicesensor0 -> voice_uorb -> sensor_voice_command
                       \-> Agent C Tools -> RGB/蜂鸣器/状态
```

直接验收命令：

```text
nsh> ls /dev
nsh> i2c get -b 3 -a 0x0f -r 0x0b
nsh> i2c get -b 3 -a 0x0f -r 0x0c
nsh> voice
nsh> voice rgb 0 0 255
nsh> voice_uorb start
```

执行直接 I2C 命令前先停止 `voice_uorb`，避免两个客户端同时访问模块：

```text
nsh> voice_uorb stop
```

### 5.3 ENET1/FEC 网络驱动

OK8MP 有线端口由 ENET1/FEC（基址 `0x30be0000`）和 YT8521 PHY 组成。PHY 地址
为 1，接口模式为 RGMII-ID，复位信号为 GPIO5_IO04。

实现分为四层：

1. 时钟、IOMUX、PHY 复位和 MDIO；
2. YT8521 自协商和链路状态；
3. RDC 权限、OCRAM 描述符、收发 DMA 和中断；
4. `struct net_driver_s`、ARP、IPv4、ICMP、UDP、TCP 和 DNS。

PHY carrier 只证明物理链路，不代表网络栈完成。FEC 只有在 `ifconfig` 中出现
`eth0`，且能依次通过网关、外网 IP、DNS、TLS 和 HTTP 验收后，才能标记为完成。

RDC 配置必须同时覆盖 M7 域、FEC TX/RX master、ENET1 peripheral 和 OCRAM。
DMA 段必须是非缓存或执行完整 cache maintenance；当前实现采用专用非缓存 OCRAM。

## 6. openvela 上层适配边界

路径 A 的 BSP 完成板级启动和驱动；比赛要求的 openvela 独有组件由最终
`nsh-agent` 配置接入：

- uORB：`voice_uorb` 发布 `sensor_voice_command`，验证驱动数据能进入消息总线；
- 官方 AI Agent：使用 openvela `packages/ai_agent`，通过 HTTPS 调用 LLM；
- Skill：语音灯光控制和网络守护 Markdown Skill；
- C Tool：`voice_status`、`set_voice_rgb`、`set_voice_buzzer`、
  `network_health_check`；
- TLS：mbedTLS 强制 X.509 链和主机名校验。

BSP 文档不把应用层代码混入板级目录。Agent 的具体 Tool/Skill 源码留在 openvela
`packages/ai_agent`，BSP 只提供稳定的设备、网络和配置契约。

## 7. 开发环境

### 7.1 依赖

主机需要：

- Arm GNU Toolchain（`arm-none-eabi-gcc`、`objcopy`、`size`）；
- GNU Make、CMake、Ninja（按 openvela 工程使用）；
- NuttX 配置工具（`kconfig-conf`/`kconfig-mconf` 或项目指定的 Kconfig 工具）；
- macOS 上可用的 `flock` 实现；
- `git`、`shasum`、`scp` 和串口终端。

检查：

```sh
arm-none-eabi-gcc --version
command -v kconfig-conf
command -v flock
```

在 macOS 上应把项目提供的兼容工具目录放在 PATH 最前面，并使用绝对路径确认：

```sh
export PATH="<workspace>/.build-tools:$PATH"
command -v flock
```

### 7.2 源码关系

下面用占位路径表示工作区，不在文档中写个人目录：

```text
<workspace>/nuttx
<workspace>/apps
<workspace>/packages/ai_agent
```

应用和 package 应由 openvela 构建系统接入 NuttX。禁止把另一个构建目录中的 `.o`、
`.a` 或 `.built` 复制进当前目录，这会使源码和最终映像失去对应关系。

## 8. 配置与构建

### 8.1 从干净目录配置

在 NuttX 根目录执行：

```sh
cd <workspace>/nuttx
./tools/configure.sh -l ok8mp-m7:nsh-agent
make olddefconfig
```

确认关键项：

```sh
grep -E 'CONFIG_ARCH_(ARMV7M|CORTEXM7)|CONFIG_ARCH_CHIP_MX8MP' .config
grep -E 'CONFIG_OK8MP_M7_(FEC_NET|VOICE_I2C)|CONFIG_UORB|CONFIG_AI_AGENT' .config
grep -E 'CONFIG_NETINIT_(IPADDR|DRIPADDR|DNSIPADDR)' .config
```

若 `make olddefconfig` 报 `kconfig-conf: command not found`，应补齐 Kconfig 工具后重新
执行；不能跳过配置同步并直接认定 defconfig 生效。

### 8.2 完整构建

建议首次和发布构建使用单线程，便于发现归档依赖问题：

```sh
make -j1
arm-none-eabi-size nuttx
shasum -a 256 nuttx.bin
```

正常产物包括：

```text
nuttx       ELF，可用于符号和 map 检查
nuttx.bin   U-Boot 加载的裸二进制
nuttx.map   链接映射
System.map  符号地址
```

如果修改了 `packages/ai_agent`，但顶层 make 没有重新生成 `libapps.a`，执行针对性的
依赖重建。只删除可再生标记，不删除源码：

```sh
cd <workspace>/nuttx
rm -f ../packages/ai_agent/.built
make -C ../apps TOPDIR="$PWD" APPDIR="$PWD/../apps" -j1
make -j1
```

发布前验证新增代码确实进入映像：

```sh
arm-none-eabi-nm nuttx | grep -E 'ok8mp_fec|ok8mp_voice|tool_ok8mp'
strings nuttx | grep -E 'voice_status|set_voice_rgb|network_health_check'
```

`nuttx.bin` 的 hash 只能证明文件传输一致，不能单独证明最新源文件已经参与链接。

### 8.3 构建记录

每次发布保存以下信息：

```sh
git rev-parse HEAD
git status --short
arm-none-eabi-gcc --version
arm-none-eabi-size nuttx
shasum -a 256 nuttx.bin
```

2026-07-20 的上一版实机功能验收映像记录为：

```text
SHA-256  55b0997d70427379a7379e01e9dd9c30170a05ae65c42f59dd7cc1ac24ff942c
size     538192 bytes
text     534592 bytes
data     3600 bytes
bss      78568 bytes
```

完成 BSP 路径、include guard 和 defconfig 规范清理后的首个候选映像为：

```text
SHA-256  b209748bc535bddc12f67beb019a4c70e3a4f1a47d85dc7e6ddf5d929360b156
size     538200 bytes
text     534600 bytes
data     3600 bytes
bss      78568 bytes
```

该映像完成了传输校验，但在 2026-07-20 访问 DeepSeek 时发现服务端已使用新的
`TrustAsia DV TLS RSA CA 2025` 中间证书，严格校验返回
`MBEDTLS_X509_BADCERT_NOT_TRUSTED (0x8)`。在 HTTPS 工具和 Agent 的信任库中补入
该中间证书并保持 `VERIFY_REQUIRED` 后，新的编译候选映像为：

```text
SHA-256  656cee10cfda4d79393d3e66b55662dfe81dee023ca43036b87d08fc17499afa
size     542208 bytes
text     538608 bytes
data     3600 bytes
bss      78568 bytes
```

该映像已完成传输和启动，但板端 DNS 将 `api.deepseek.com` 解析到 Amazon CloudFront
地址 `3.173.21.63`。实机收到的证书链为 `*.deepseek.com`、`Amazon RSA 2048`
和 `Amazon Root CA 1`，与开发主机此前解析到的 TrustAsia 节点不同。缺少 Amazon
根证书时，链顶层返回 `MBEDTLS_X509_BADCERT_NOT_TRUSTED (0x8)`；这说明只按一次
DNS 解析结果配置最小 CA 集不能覆盖服务端的多 CDN 调度。

在 `https_client` 和 Agent 两套信任库中继续保留 TrustAsia 中间证书，并补入
Amazon Trust Services 发布的 `Amazon Root CA 1`，保持 `VERIFY_REQUIRED`、SNI、
主机名和证书链校验不变。完成 apps 与外置 Agent 的同轮全量重建后，当前待上板映像为：

```text
SHA-256  02ffb2a7ad8ee94da7542b1ca41b369f566e047fe90051331de3078667f73a2d
size     544128 bytes
text     540524 bytes
data     3600 bytes
bss      78572 bytes
```

最终 ELF 中可检出两份 TrustAsia 中间证书和两份 Amazon 根证书，分别供
`https_client` 与 Agent TLS 使用。该 hash 尚需在实板完成 HTTPS 和 Agent 复测。
后续重建 hash 变化属于正常现象，应以同一轮构建、传输和实机日志为一组证据。

## 9. 下载与启动

### 9.1 复制到板载存储

先启动 A53 Linux，把 `nuttx.bin` 写入 FAT 启动分区：

```sh
cd <workspace>/nuttx
shasum -a 256 nuttx.bin
scp -O -oHostKeyAlgorithms=+ssh-rsa \
  nuttx.bin root@192.168.2.2:/run/media/mmcblk2p1/nuttx.bin
```

在 A53 Linux 核对：

```sh
sha256sum /run/media/mmcblk2p1/nuttx.bin
stat -c '%s bytes' /run/media/mmcblk2p1/nuttx.bin
sync
```

Mac 和 A53 Linux 的 SHA-256、字节数必须完全一致。

如果同一 IP 更换过系统，SSH 会提示 host key 变化。先确认当前连接确实是目标开发板，
核对指纹，再清除旧条目：

```sh
ssh-keygen -R 192.168.2.2
scp -O -oHostKeyAlgorithms=+ssh-rsa \
  -oStrictHostKeyChecking=accept-new \
  nuttx.bin root@192.168.2.2:/run/media/mmcblk2p1/nuttx.bin
```

不要用 `StrictHostKeyChecking=no` 绕过未知设备的身份检查。

### 9.2 DDR 完整映像启动

停止 Linux，回到 U-Boot：

```text
forlinx=> mmc dev 2
forlinx=> fatload mmc 2:1 0x80000000 nuttx.bin
forlinx=> printenv filesize
forlinx=> md.l 0x80000000 2
forlinx=> bootaux 0x80000000
```

向量表前两个字应类似：

```text
0x801xxxxx   初始栈地址
0x800xxxxx   复位入口，最低位为 Thumb 位
```

当前板载 U-Boot 没有 `hash` 命令，因此完整性校验放在 Mac 和 A53 Linux 两端完成。
`fatload` 必须指定 `nuttx.bin`；省略文件名会得到 `No boot file defined`。

### 9.3 TCM 诊断映像启动

仅在 TCM 小型配置下使用：

```text
forlinx=> fatload mmc 2:1 0x48000000 nuttx.bin
forlinx=> cp.b 0x48000000 0x7e0000 ${filesize}
forlinx=> bootaux 0x7e0000
```

不要把完整 `nsh-agent` 映像按 TCM 方式启动，也不要把 TCM 和 DDR 启动地址混用。

## 10. 网络和 HTTPS/TLS 验收

### 10.1 macOS 互联网共享

当前拓扑为：

```text
Internet <- Wi-Fi(en0) <- macOS NAT/bridge100 <- AX88179A(en5)
         <- OK8MP ENET1/FEC(M7 eth0)
```

在“系统设置 -> 通用 -> 共享 -> 互联网共享”中：

- 共享连接来源：Wi-Fi；
- 共享给：AX88179A 对应的网络服务；
- 启用互联网共享。

macOS 界面显示的是网络服务名，不一定显示 `en5`。用以下命令确认服务和硬件端口：

```sh
networksetup -listallhardwareports
networksetup -listallnetworkservices
ifconfig en5
ifconfig bridge100
sysctl net.inet.ip.forwarding
```

正常情况下 `bridge100` 为 `192.168.2.1`，转发值为 1。共享开启后物理 `en5` 没有
IPv4 地址并不一定是故障，地址可能配置在 `bridge100`。

### 10.2 分层验收命令

M7 侧执行：

```text
nsh> ifconfig
nsh> route ipv4
nsh> ping -c 3 192.168.2.1
nsh> ping -c 3 1.1.1.1
nsh> nslookup example.com
```

若板上没有可用 RTC/NTP 样本，先设置实际 UTC 时间。月份必须使用英文缩写：

```text
nsh> date -u -s "Jul 20 02:00:00 2026"
nsh> date -u
```

上例仅说明格式，验收时必须填写当时真实 UTC，不能照抄旧时间。

最后执行：

```text
nsh> https_client probe example.com 443
nsh> https_client get example.com /
```

验收层次如下：

| 证据 | 能证明的内容 | 不能证明的内容 |
| --- | --- | --- |
| `ping 192.168.2.1` | 本地链路和网关 | NAT、DNS、TLS |
| `ping 1.1.1.1` | 外网 IPv4/NAT | DNS、TLS |
| `nslookup` | DNS | TCP、TLS |
| `https_client probe` | DNS 和 TCP 443 | TLS 握手、证书 |
| `TLS established` | TLS 会话 | HTTP 服务结果 |
| `HTTP/1.1 200 OK` | 完整 HTTPS 请求 | 其他主机也一定可用 |

本项目实机已获得 `TLSv1.2`、严格 X.509 校验、`HTTP/1.1 200 OK` 和 868 字节
响应。正式答辩保留从 `ifconfig` 到 HTTP 200 的连续串口日志。

## 11. uORB 和官方 AI Agent 验收

### 11.1 uORB

```text
nsh> voice_uorb start
nsh> uorb_listener sensor_voice_command 1
```

说出模块已配置的命令词，监听端应收到 `sensor_voice_command`。验证完成后可停止：

```text
nsh> voice_uorb stop
```

### 11.2 Agent 基础对话

```text
nsh> ai_agent
vela> set_llm deepseek <DEEPSEEK_API_KEY>
vela> ask Reply exactly OK8MP_AGENT_OK
```

必须看到 LLM HTTPS 握手完成、HTTP 200 和最终 `OK8MP_AGENT_OK`。`ask` 是异步命令，
发出后等待当前请求的最终结果，不要连续输入多条 `ask` 干扰日志。

`/data` 当前挂载为 tmpfs，`set_llm` 写入的配置只在本次 M7 启动期间有效；M7 复位
后需要重新设置。正式持久化应使用独立非易失文件系统或由 A53/RPMsg 提供配置服务。
不得把真实 API Key 固化到 defconfig、源码、固件、Git、文档或演示视频中。

### 11.3 多工具硬件闭环

```text
vela> ask 检查语音模块状态，然后把指示灯设置为蓝色。必须先调用 voice_status，再调用 set_voice_rgb。
```

合格日志至少包含：

```text
Tool call: voice_status
Tool voice_status result: ... ok=true ...
Tool call: set_voice_rgb
Tool set_voice_rgb result: ... ok=true ...
END status=ok
```

同时观察实物 RGB 指示灯变蓝。网络守护场景还应调用
`network_health_check`，根据 DNS、TCP、TLS、X.509 和 HTTP 结果把灯设为绿色或红色，
形成不少于两个 Tool 的多步推理闭环。

## 12. 已知问题和排查记录

本节把移植过程中出现过的问题整理为可复现的排查规则。失败过程不作为功能完成证据，
但用于说明硬件边界和后续维护注意事项。

### 12.1 参考板资源残留

**现象：** 早期代码头仍出现 `verdin-mx8mp`，LED、按键或 INA219 定义与 OK8MP
实物不一致。

**原因：** 新 BSP 以同 SoC 的板级实现作为接口参考，但板级资源没有逐项替换。

**处理：** 文件路径、include guard 和板级说明统一改为 `ok8mp-m7`；未核对的参考
外设默认关闭；只把数据手册、原理图、Linux DT 和实测同时吻合的引脚写入最终配置。

**判据：**

```sh
rg -n 'verdin|Verdin' boards/arm/mx8mp/ok8mp-m7 --glob '!README.md'
```

应无参考板残留。仅在移植历史文档中说明来源时例外。

### 12.2 M7 DDR 与 A53 Linux 冲突

**现象：** M7 固件单独运行正常，启动 Linux 后 M7 停止、Linux 卡住或内存内容被
覆盖。

**原因：** Linux DT 未为 M7 保留 DDR，两个核访问同一物理区域；或两个核同时接管
ENET1。

**处理：** TCM 映像只用于共存诊断；最终比赛流程先用 Linux 复制文件，再停止 Linux，
由 U-Boot 单独启动 DDR M7 固件。需要长期 A/M 共存时，必须修改 DT reserved-memory、
RDC 和外设所有权，不能只改变链接地址。

**判据：** U-Boot 启动 M7 后 NSH、网络和长时间任务稳定，A53 不再占用同一 DDR 和
ENET1。

### 12.3 PHY 有 carrier 但没有网络

**现象：** PHY link 灯和 MDIO 正常，但 `ifconfig` 没有 `eth0` 或不能 ping。

**原因：** 只完成了 PHY 层，没有完成 FEC DMA 和 NuttX `net_driver_s`。

**处理：** 依次验证时钟/IOMUX/MDIO、PHY、自协商、RDC、DMA、netdev、ARP/IPv4。
FEC 软件复位后重新设置 MDIO 时钟分频寄存器；YT8521 以实板读到的 PHY ID 和 Clause
22 通信为准，不套用其他 SDK 的固定 ID。

**判据：** `eth0` 注册成功，网关 ping、外网 IP、DNS、TLS 和 HTTP 分层通过。

### 12.4 FEC DMA 无响应或触发 AXI 错误

**现象：** 描述符地址已写入，但 TDAR/RDAR 不推进，或进入 ENET error/AXI 中断。

**原因：** 描述符位于缓存 DDR、地址别名错误、对齐错误，或 RDC 未放行 FEC master
访问 OCRAM。

**处理：** 使用 `.fec_dma` 非缓存 OCRAM 段、64 字节对齐；把 M7 地址转换为 FEC
系统总线别名；检查 M7、FEC TX/RX、ENET1 和 OCRAM 的 RDC 配置。

**判据：** TX 描述符 READY 能被硬件清除，RX 计数增长，无 AXI error。

### 12.5 手工构包触发 Cortex-M7 UNALIGNED fault

**现象：** 发送 ARP/ICMP 测试包时进入 UsageFault，CFSR 显示 UNALIGNED。

**原因：** 对 Ethernet/IP packed header 中未对齐字段执行 16/32 位直接访问。

**处理：** 按字节写入网络头，或使用 `memcpy` 和显式字节序转换；不要把未对齐
`uint8_t *` 强制转换为 `uint32_t *`。

**判据：** 连续发送 ARP/ICMP 不进入 fault，校验和正确，能收到回复。

### 12.6 Linux I2C 编号和 M7 I2C 编号混淆

**现象：** Linux 上可访问的 `/dev/i2c-X` 号直接填入 M7 配置后无 ACK。

**原因：** Linux 适配器编号是枚举结果，不等同于 i.MX8MP M7 控制器编号。

**处理：** 以原理图 net name、IOMUX 和控制器基址为准。当前接线对应 M7 I2C3；在
M7 侧显式配置 root clock、CCGR、SCL/SDA IOMUX 和 pad，不能依赖 U-Boot/Linux
遗留寄存器状态。

**判据：** 冷启动后 `i2c get -b 3 -a 0x0f -r 0x0b` 稳定返回版本值。

### 12.7 `ioctl failed: 13` 不是文件权限

**现象：** `voice` 和 Agent `voice_status` 返回 errno 13。

**原因：** i.MX8MP I2C lower-half 把 I2SR 的 IAL（仲裁丢失）映射为 `-EACCES`；
错误来自总线事务，不是 `/dev/voicesensor0` 的 Unix 权限。本次持续失败的最终原因是
语音模块线序接错。

**处理顺序：**

1. `voice_uorb stop`，避免后台占用；
2. 断电核对 VCC、GND、SCL、SDA、共地和模块电压；
3. 确认上拉和线长；
4. 用 `i2c get` 直接读 `0x0b`、`0x0c`、`0x08`、`0x0a`；
5. 再运行 `voice`、`voice rgb`；
6. 最后启动 uORB 和 Agent。

**判据：** 四个寄存器可重复读取，`voice` 无 errno，Agent 两个 Tool 都返回
`ok=true`，实物灯颜色正确。

### 12.8 只能 ping Mac，不能访问外网

**现象：** `ping 192.168.2.1` 成功，`ping 1.1.1.1` 失败。

**原因：** macOS 只配置了本地以太网地址，未真正启用互联网共享/NAT；常见状态是
`bridge100` 不存在或 `net.inet.ip.forwarding=0`。

**处理：** 从 Wi-Fi 共享到 AX88179A 网络服务，确认 `bridge100=192.168.2.1` 和
转发为 1。不要因为设置界面只显示 `en4` 就猜测接口，先用
`networksetup -listallhardwareports` 对照硬件端口和 Device。

**判据：** 依次通过网关、`1.1.1.1` 和 DNS。

### 12.9 DNS 查询触发随机源日志

**现象：** 早期 `nslookup` 或 TLS 过程中出现 CAAM Job Ring 访问失败，随后
`ctr_drbg_seed failed`。

**原因：** CAAM 全局 RNG 状态寄存器可读，但 Job Ring 的安全域所有权不属于 M7。
修改 RDC、CSU 或 MID 并不能保证硬件所有权切换成功。

**处理：** 最终配置不依赖不可用的 CAAM Job Ring，使用 NuttX random pool 和中断
熵；TLS 在熵不足时失败关闭，不使用固定种子或 xorshift 伪随机替代。CAAM RNG 不是
比赛第一阶段的独立验收项，不再把时间投入到越权访问 Job Ring。

**判据：** 冷启动后通过正常中断和网络活动积累熵，TLS DRBG 初始化成功；熵不足时
明确失败，不降级证书校验。

### 12.10 TLS 报 `system time is not valid` 或 `-0x2700`

**现象：** TCP 443 已连接，但 HTTPS 被拒绝；系统时间显示 1950，或 mbedTLS 返回
`X509 -0x2700`。

**原因：** 证书有效期检查需要正确 UTC；`-0x2700` 还可能表示缺少服务端证书链的
信任根，不是普通网络断开。

**处理：** 优先使用 NTP；NTP 样本仍为 0 时，用英文月份设置实际 UTC。启用严格
`VERIFY_REQUIRED`、SNI/hostname 和 verify result 检查，并提供目标 LLM 服务链所需的
最小 CA 集。当前 CA 集覆盖 ISRG Root X1、DigiCert Global Root G2、GTS Root R4、
SSL.com TLS ECC Root CA 2022、Amazon Root CA 1，以及 DeepSeek 的 TrustAsia DV TLS
RSA CA 2025 中间证书。DeepSeek 可能按 DNS/CDN 调度返回 TrustAsia 或 Amazon 链，
排查时必须同时记录板端解析地址、服务端完整证书链和本地 CA 列表，不能只比较域名。

**判据：** 日志显示 TLS 版本、cipher suite、证书校验成功，随后收到 HTTP 状态码。

### 12.11 `https_client probe` 成功但 HTTPS 未完成

**现象：** probe 显示 `TCP connected`，但 `get` 握手失败。

**原因：** probe 只检查 DNS 和 TCP 连接，不执行完整 TLS/X.509/HTTP。

**处理：** 必须继续执行 `https_client get`。无法打开 `/tmp/llm_api.log` 只是可选日志
路径不存在，不等同于 HTTPS 失败。

**判据：** 以 `TLS established`、严格 verify 结果和 `HTTP/1.1 200 OK` 为准。

### 12.12 LLM 已返回但 Agent 仍在 120 秒后超时

**现象：** 日志已经显示 `Response: 14 bytes`，watchdog 仍判定约 122 秒超时。

**原因：** 旧 HTTP reader 在 chunked keep-alive 响应的零长度 chunk 后继续等连接
关闭；watchdog 使用 realtime 时又会受手工校时影响。

**处理：** 正确解析 zero chunk 和 trailer；M7 上使用 `Connection: close` 或设置读空闲
超时；watchdog 改用 `CLOCK_MONOTONIC`。用户发出 `ask` 后等待最终响应再输入下一条。

**判据：** `Response` 后立即进入 `END status=ok`，返回
`OK8MP_AGENT_OK`，不再等待 120 秒。

### 12.13 macOS 缺少 `flock` 导致旧库参与链接

**现象：** 源文件看似重新编译，`libapps.a` 或 `.built` 时间戳未更新，最终固件仍输出
旧字符串。

**原因：** 应用构建规则依赖 `flock`；macOS 默认缺失时，归档/标记步骤失败，顶层仍
可能链接已有库。

**处理：** 把经过验证的兼容 `flock` 放入 `.build-tools` 并置于 PATH 前端；使用
`-j1` 观察首个错误；删除准确的可再生 `.built`，先构建 apps，再顶层链接。

**判据：** `nm`、`strings` 和 map 中出现新符号/新字符串，映像 hash 和尺寸更新，
上板行为与源码一致。

### 12.14 SSH host key 变化

**现象：** `REMOTE HOST IDENTIFICATION HAS CHANGED`，SCP 拒绝连接。

**原因：** `192.168.2.2` 在 A53 Linux、重装系统或其他设备之间复用，已保存的 key
类型/指纹与当前设备不同。

**处理：** 先确认实物和串口，核对新指纹，执行 `ssh-keygen -R 192.168.2.2`，再使用
`accept-new` 建立新记录。

**判据：** SCP 成功，Mac 与 A53 Linux hash 一致。

### 12.15 U-Boot 命令和地址使用错误

**现象：** `fatload` 报 `No boot file defined`、`hash` 不存在，或 `bootaux` 无输出。

**原因：** 漏写文件名；当前 U-Boot 未编译 hash 命令；DDR/TCM 启动地址混用。

**处理：** DDR 固件严格使用第 9.2 节命令，加载后查看 `filesize` 和向量前两字；hash
在 Mac/A53 Linux 校验。TCM 只用于对应的小型镜像。

**判据：** 向量栈位于运行 RAM，复位向量位于代码区且 Thumb 位为 1，随后出现 NSH。

### 12.16 API Key 看似保存但重启后消失

**现象：** 同一次启动中 Agent 重进后可自动载入 backend，M7 复位后又显示
`No available backend`。

**原因：** `/data` 是 tmpfs；“API key saved”只表示写入本次启动的配置文件。

**处理：** 演示启动后在不录制密钥的终端输入一次，或适配独立非易失文件系统/A53
配置服务。禁止把 key 编译进镜像。

**判据：** 同一次启动内 Agent 重启可恢复配置；若实现非易失存储，断电恢复测试和
密钥权限检查同时通过。

## 13. 准入和答辩验收清单

### 13.1 BSP

- [ ] 上游基线无 `boards/arm/mx8mp/ok8mp-m7`；新增范围可由 Git 提交证明；
- [ ] Kconfig、四套 defconfig、Make/CMake 和两套链接脚本齐全；
- [ ] 文件头、路径、include guard 无参考板残留；
- [ ] UART4、NSH、GPIO、定时器和长时间运行通过；
- [ ] `/dev/i2c3`、`/dev/voicesensor0` 注册，冷启动可重复读写；
- [ ] `eth0`、静态地址、路由和 DNS 与最终拓扑一致；
- [ ] 新增语音模块驱动源码、接口和测试方法完整；
- [ ] 所有新增源码使用 Apache License 2.0。

### 13.2 网络和 openvela 组件

- [ ] 网关 ping；
- [ ] 外网 IPv4；
- [ ] DNS；
- [ ] TLSv1.2；
- [ ] 严格 X.509 和 hostname 校验；
- [ ] HTTP 200；
- [ ] uORB 发布/监听；
- [ ] 官方 Agent 真实 LLM 对话
- [ ] 至少两个 Tool 的多步硬件闭环；
- [ ] 串口日志、视频和实物状态相互对应。

### 13.3 发布前检查

```sh
git status --short
git diff --check
rg -n 'sk-[A-Za-z0-9_-]{12,}|api[_-]?key\s*[:=]\s*[^<]' \
  boards/arm/mx8mp/ok8mp-m7 Documentation/platforms/arm/mx8mp/boards/ok8mp-m7
rg -n 'verdin|Verdin' boards/arm/mx8mp/ok8mp-m7 --glob '!README.md'
```

秘密扫描不得命中真实 key。参考板扫描在 OK8MP BSP 源码中应无输出。提交中不得包含
`.o`、`.a`、`nuttx`、`nuttx.bin`、个人串口日志和主机绝对路径。

## 14. 提交材料建议

评审材料按同一版本号组织：

1. NuttX/openvela/Agent/Apps 各仓库 commit ID；
2. `nsh-agent/defconfig` 和 BSP 源码；
3. 构建工具版本、ELF size、`nuttx.bin` SHA-256；
4. 上游基线不存在 OK8MP BSP 的 Git 证据；
5. U-Boot 启动、NSH、GPIO/定时器、语音驱动日志；
6. 网关、外网、DNS、TLS/X.509、HTTP 200 连续日志；
7. uORB 和 Agent 多 Tool 闭环日志；
8. 长时间稳定性、异常输入和冷启动复测结果；
9. Apache License 2.0、移植文档和已知限制。

代码、二进制、日志和视频必须来自同一轮构建。答辩中若使用不同 hash 的固件，应在
测试记录中说明变更内容，避免把早期失败日志与最终通过日志拼接为同一结果。
