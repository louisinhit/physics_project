
# LQG 工程复现记录：截至 IP 包

日期：2026-09-21。运行位置：当前 Linux 服务器。

## 结论

已经生成并交付 `acin:vmc:lqg_axi_test:1.0` IP。交付副本修复了一处工具生成的 AXI 组合逻辑敏感列表，随后通过普通 Verilog 编译、顶层 elaboration 和 IP 打包完整性检查。

**这不是原工程的一键无错误复现。** `vmcExport` 会先生成 HDL/IP，然后在通用 `XilinxCompiler` 阶段抛出 `basic_string::_M_construct null not valid`。这个异常仍未解决；详细尝试、堆栈和剩余警告见 [ISSUES.txt](ISSUES.txt)。未执行板级综合、实现、bitstream 生成或上板测试。

## 按顺序的实际工作流

| 阶段 | 入口／操作 | 实际结果 |
|---|---|---|
| 1. 隔离与复制 | 复制原 firmware/matlab；不复制原有 netlist | 仅 project 内可写，工具与原源码只读 |
| 2. 参数初始化 | `test_kalman_scaling_init.m` | 完成；矩阵尺寸与系数有限性检查通过 |
| 3. 原模型仿真 | `lqg_with_axi_test.slx` | 0–0.4 ms，8 ns 基础步长；50,001 个时间点，无仿真错误 |
| 4. Model Composer 导出 | `vmcExport`，DUT=`fpga_lqg_axi` | 实际生成 HDL、component.xml、ZIP，但随后 API 抛出上述异常 |
| 5. 独立 RTL 检查 | xvlog | 发现原始生成 AXI RTL 的数组敏感列表不被接受；SystemVerilog 模式也失败 |
| 6. 交付副本修正 | 一处 `always @(...)` 改为 `always @*` | 不改 RHS、赋值语句、算法、位宽、接口或时钟触发逻辑 |
| 7. 再验证／重新打包 | xvlog、xelab、ipx::check_integrity | 均通过；ZIP 内容与交付目录逐项一致 |

使用 MATLAB R2024a Update 9、Vitis Model Composer 2024.2、Vivado 2024.2。没有安装、修改任何服务器软件，也没有请求 UniKey 凭据。Model Composer 自身会调用 Vivado 后端完成 IP 打包；独立验证只使用 RTL 编译／展开和 IP 打包操作，未运行综合或实现。

## 看哪些文件

| 路径（相对本目录） | 用途 |
|---|---|
| `ip_repo/lqg_axi_test/` | **修正后交付 IP 目录**；包含 component.xml、hdl、constrs、drivers、xgui |
| `artifacts/lqg_axi_test_patched.zip` | **修正后交付 ZIP**，与上述目录的描述文件及全部 20 个声明文件一致 |
| `work/matlab/netlist_lqg_axitest/` | 最后一次 Model Composer 的原始输出；没有应用 AXI 补丁，不要与交付副本混淆 |
| `artifacts/raw_generator_ip.zip` | 原始生成 ZIP，保留作对照，不是修正后的交付包 |
| `artifacts/axi_sensitivity.patch` | 唯一 RTL 修改的精确差异 |
| `artifacts/init_workspace.mat` | 初始化工作区，包括 p、k、clqr、lqg 等 |
| `artifacts/init_figure_1.fig` | 初始化脚本产生的图形，并非闭环仿真波形 |
| `artifacts/simulation_output.mat` | 原模型的 SimulationOutput：时间与运行元数据；原模型没有配置波形导出 |
| `artifacts/ip_packaging_check.json` | 交付目录的文件引用、SHA-256、ZIP 一致性检查 |
| `artifacts/model_parameter_changes.txt` | 模型块及 InstanceData 参数比较：只有 Hub 的 LogFileName 改变 |
| `artifacts/source_copy_differences.txt` | 源码副本比较：只有模型和运行生成的 kalman_model.mat 不同；原 `.m` 和手写 `.v` 未改 |
| `logs/` | 每次实际运行、异常、编译与打包日志 |
| `ISSUES.txt` | 问题总表及各问题最终状态 |

关键成功日志：`logs/01_init.log`、`logs/simulate.log`、`logs/rtl_compile.log`、`logs/rtl_elaborate.log`、`logs/repackage.log`。
最后一次通用导出异常：`logs/export_selected.log`。首次内部异常堆栈：`logs/export_cpp_backtrace_retry.log`。

## 以后如何重跑

在**当前服务器**的本目录下，使用隔离启动脚本，不要直接用服务器默认 MATLAB 启动命令代替，以免偏好设置、缓存或临时文件写到本目录之外：

```bash
bash scripts/run.sh init
bash scripts/run.sh simulate
bash scripts/run.sh export_selected
```

第三步目前仍会非零退出，不能忽略其日志并宣称成功。它生成的是 `work/matlab/netlist_lqg_axitest/` 中的原始包，不会自动更新 `ip_repo/` 中的修正版。重新生成后如要重新交付，需要把新 IP 复制成交付副本、复核并应用 `artifacts/axi_sensitivity.patch`，然后执行：

```bash
bash scripts/run.sh check_rtl
bash scripts/run.sh repackage
/usr/bin/python3 -B scripts/verify_ip.py
```

这些命令会更新本目录下的工作文件、日志、验证缓存或交付 ZIP；需要保留某次结果时先在本目录下另存。`export_hdl`、`export_sysgen` 等是失败诊断尝试，不是可用的替代入口。

## 尚未证明／后续需要处理

- 通用 `vmcExport` 的 C++ 异常仍需进一步排查；不能从当前证据直接归因于 MATLAB Update 9 或算法错误。
- IP 的两个时钟接口缺少 `FREQ_HZ` 元数据，`proc_sys_clk` 没有关联总线接口，且缺少 Product Guide；完整性检查通过不代表这些信息无需处理。
- 模型使用器件速度等级 `-2`，原板级工程使用 `-1`；板级集成前需要核实。
- 当前验证没有检查冷却效果、估计误差、定点数值等价性、AXI 事务正确性、CDC、时序收敛或实际硬件。
- 所有 `.m` 与手写 RTL 保持原样；AXI 补丁只在交付副本中，未修改安装目录中的代码生成模板。下一次原始导出还会包含同一敏感列表。

`runtime/` 约 2.1 GB，主要是隔离用户目录、工具缓存和自动创建的运行组件；其中包含本机许可证副本，不要把整个 project 直接公开上传。诊断原始日志也不宜未经检查就公开。交付文件与重要记录不依赖外部临时目录。
