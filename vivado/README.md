# Vivado 板级集成与行为仿真

工作目录：`/srv/seagate2g/home/userr/physics_project/project/vivado`。

## 已得到的结果

Vivado 2024.2 / XSim 整板行为仿真通过：**`BOARD_TEST_PASS checks=845`**。
仿真时间 **136.524 µs**；检查计数包括 AXI 事务响应检查及功能断言，不表示
845 个独立测试场景。详细日志：`logs/simulate.log`。

| 验证项 | 结果 |
| --- | --- |
| PS GP1 → LQG AXI，独立的 offset 0/4/8 | 通过 |
| 三组配置各 81 words 写入/读回、busy/load/apply | 通过 |
| LED XOR mask、零配置、正负输出 offset | 通过 |
| 非零 L/K 矩阵与两档 ADC 输入的精确数值比较 | 通过 |
| DAC DDR 两个输出相位与原 BD 通道映射 | 通过 |
| PS GP0 → descriptor BRAM、DMA 状态、throttle 配置寄存器 | 通过 |
| PS warm reset 清除配置与 AXI 寄存器 | 通过 |

注意：这些结果不是时序收敛、DMA 连续传输或实板验证结果。
已逐文件比较 `ip_repo/lqg_axi_test/hdl` 与上一级已测试 IP 的 HDL，完全一致；
板级 XDC 与原参考文件也完全一致。修改集中在本地板级 wrapper/控制器、BD 和元数据。

## 打开工程

在服务器的 Vivado 2024.2 GUI 中选择 **Open Project**，打开：

`/srv/seagate2g/home/userr/physics_project/project/vivado/lqg_fp7/lqg_fp7.xpr`

在 Sources 中展开 `design_0_wrapper → design_0`，双击 `design_0.bd` 查看 BD。
Design Sources 顶层是 `design_0_wrapper`；Simulation Sources 顶层是
`tb_board`。运行行为仿真后点击 Run All，检查 `BOARD_TEST_PASS`。

已有波形位于 `artifacts/board_test.wdb`，布局位于 `artifacts/board_test.wcfg`。
在 Vivado Tcl Console 中查看已保存结果，无需重新跑仿真：

```tcl
open_wave_database /srv/seagate2g/home/userr/physics_project/project/vivado/artifacts/board_test.wdb
open_wave_config /srv/seagate2g/home/userr/physics_project/project/vivado/artifacts/board_test.wcfg
```

**本次不运行 synthesis、implementation 或 write_bitstream。**
`Generate Output Products` 中出现的 “Synthesis target” 仅表示生成供综合使用的
HDL 文件，不代表运行了综合。最终运行状态另存于 `artifacts/final_audit.txt`。

## 复现基准与组成

使用原仓库 README 指定的默认 `lqg_fp7`（第一代 Red Pitaya）设计，器件为
`xc7z020clg400-1`。这不是对实际待使用开发板型号、速度等级及 PCB 版本的确认。
没有采用标注不再作为权威入口的 rpgen2 初始化 Tcl。

| 组成 | 当前实验中的位置 / 作用 |
| --- | --- |
| 已测试的 LQG IP | `ip_repo/lqg_axi_test`，来自上一级已修正、验证的 IP；HDL 不改 |
| LQG IP 实例 | `lqg_fp7/lqg_fp7.srcs/sources_1/ip/lqg_axi_test_0` |
| DMA throttle IP | `ip_repo/Thrtl_8ch_cfg`，独立的上游参考 IP |
| BD | `src/bd/design_0.bd`，复制参考 BD 后由 2024.2 重新生成输出 |
| 板级 RTL / XDC | `src/rtl`、`src/constr/red_pitaya.xdc` |
| 自检 testbench | `sim/tb_board.sv`，完整板级 wrapper + AMD PS7 VIP |
| 操作脚本 | `scripts/build.tcl`、`repair.tcl`、`simulate.tcl` |
| 错误与限制 | 根目录 `ERRORS.txt`，保留失败原因及修复记录 |

数据路径：ADC 引脚 → ADC 格式转换/跨时钟域 → LQG → DAC 格式转换/DDR 输出。
PS 通过 AXI 写入配置 BRAM，再触发 load/apply；LQG 输出和状态也连接到 DMA 路径。
原设计把第二个 LQG 传感器输入接为常量 0；没有擅自改成双 ADC 通道运行。
DAC CH1 对应 `uk1`，CH2 对应 `uk0`，保留原设计通道映射。
这里的 CH1/CH2 指 BD 的端口名称：内部 concat 低半字是 CH2/uk0，
所以处理时钟上升沿输出 uk0、下降沿输出 uk1；实板模拟通道名称需按原理图核对。

时钟保留原设计：ADC 输入 125 MHz，LQG 处理时钟 15.625 MHz（64 ns），
PS AXI 125 MHz，DAC MMCM/输出时钟 250 MHz。后者的实际电气时序和器件允许范围
需要在确认真实板卡后核对，不能把行为仿真通过当作 DAC 板级时序正确。

## PS 地址映射

| 地址 | 大小 | 作用 |
| --- | --- | --- |
| `0x40000000` | 8 KiB | DMA descriptor / shared BRAM |
| `0x40002000` | 4 KiB | DMA throttle 配置 |
| `0x40400000` | 64 KiB | AXI DMA 寄存器 |
| `0x80000000` | 8 KiB | LQG 配置 BRAM |
| `0x80002000` | 4 KiB | 配置控制器：写 1 加载、读 bit0 忙状态、空闲后写 2 应用 |
| `0x80003000` | 4 KiB | LQG IP：offset 0 LED XOR mask，offset 4 状态复位，offset 8 LED 状态 |

配置有效长度 2576 bit，通过 81 个 32-bit little-endian AXI 写事务装入。
BRAM wrapper 实例使用 `BRAM_READ_LATENCY=1`，保持参考 BD 的参数。
LED 输出逻辑是配置 LED 值与 AXI mask 的 **XOR**，不是 AND。
上游板级 wrapper 的旧 AXI 地址宽度为 3 bit，与当前已测试 IP 的 4 bit 不匹配；
已将本目录 wrapper 的 AWADDR/ARADDR 改成 4 bit，恢复 offset 8 独立访问。
该修正不修改 IP 本身的接口或算法，详见 `ERRORS.txt`。

## 操作过程

1. 新建本目录；复制上一级已测试 LQG IP、参考 RTL、XDC、BD、DMA throttle IP。
   不引用原作者机器上的绝对源码路径，不复用其 DCP 或 bitstream。
2. 在仅本目录可写的 `bwrap` 沙箱中调用 Vivado 2024.2。
   临时目录、Vivado 用户缓存与日志均重定向至本目录；未设置或改写服务器 HOME。
3. `build.tcl` 建立 `.xpr`，注册两个本地 IP 包，建立 `lqg_axi_test_0` 实例，
   导入 BD、板级文件和约束，校验并生成 BD/IP 输出文件以及顶层 wrapper。
4. 首次仿真修正 testbench 的 PS inout 连接方式；第二次仿真重现配置控制器的
   状态读回错误。历史日志保存在 `logs/simulate_01_*`、`logs/simulate_02_*`。
5. 只在本目录的板级副本中修复配置读状态机复位，增加配置寄存器的同步释放复位，
   给 apply 脉冲 CDC 接上两端复位，修正 wrapper 的 4-bit AXI 地址匹配和
   BRAM/时钟接口元数据；为悬空扩展输出及闲置 SPI0 SS_I 明确接上常量。
   通过 `repair.tcl` 刷新 module references 和 BD。
6. `simulate.tcl` 使用 XSim 编译、展开并运行 `tb_board`，不执行任何综合命令。
7. 最终审计：项目无缺失文件，IP 无锁定；`synth_1` 和 `impl_1` 均为
   `Not started / 0%`。详情在 `artifacts/final_audit.txt`，警告和限制保留在 `ERRORS.txt`。

关键产物还包括 `artifacts/design_0_recreate.tcl`（BD Tcl）、
`artifacts/recreate_project.tcl`（项目重建脚本）、`artifacts/address_map.txt`、
`artifacts/ip_status.txt` 和 `artifacts/final_audit.txt`。
`runtime` 是隔离运行环境，包含本机工具许可副本和缓存；对外分享工程时不要打包它。

维护命令（在**这台 Linux 服务器**执行；工程已经存在，不要重新跑 `build`）：

```bash
cd /srv/seagate2g/home/userr/physics_project/project/vivado
bash scripts/run_isolated.sh /bin/bash "$PWD/scripts/vivado_batch.sh" simulate
```

本次使用的 RTL module-reference 刷新命令参数按照
[AMD UG835](https://docs.amd.com/r/2024.1-English/ug835-vivado-tcl-commands/update_module_reference)
使用 IP 实例对象；失败的 Tcl 调用亦记录在 `ERRORS.txt`，未隐藏历史失败。

## 验证范围与限制

testbench 不 force 内部功能信号，而由 PS7 VIP 发起真实 BD 内的 AXI 事务；
外部 ADC 时钟和采样值由 testbench 提供，内部信号只用于断言和观察。
测试覆盖配置 BRAM 全部 81 words 的写回读、load/busy/done/apply、LED mask、
有符号输出 offset、非零矩阵运算、ADC 输入变化、DAC DDR 输出通道和 warm reset。
精确数值例使用 `F=Gamma=0, L[0,0]=1, K[0,0]=1/2`、单位校准系数，
因此 `xhat0=y0, u0=y0/2`，不涉及饱和或舍入边界。

这不是完整闭环物理模型验证：没有复核任意 LQG 系数下的稳定性、全部精度/溢出边界、
端到端 DMA 持续流量或吞吐、Linux 驱动、真实 DDR、电气时序及实板 ADC/DAC。
FIFO/BRAM 厂商行为模型也不能证明 CDC、亚稳态或读写碰撞的硬件行为。
后续综合/实现需要你检查 timing、CDC、DRC、资源与 pinout；本次没有进行 timing closure。
