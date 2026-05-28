# Coral NPU 宏观分析报告

## 1. 项目定位

Coral NPU 是 Google 开源的机器学习推理加速器 IP，面向超低功耗 SoC 场景。项目不是单一的 RTL 仓库，而是一个覆盖硬件设计、SoC 集成、FPGA 落地、软件运行时、仿真与测试验证的完整工程。

从顶层设计上看，Coral NPU 以 RISC-V 为控制基础，组合了三类处理能力：

- Scalar：负责控制流、调度、地址生成和系统管理
- Vector / SIMD：负责向量算子与并行数据处理
- Matrix / ML Dataplane：面向机器学习推理的数据通路

## 2. 顶层目录职责

### 2.1 核心目录

- `hdl/`
  - 项目最核心的硬件实现目录
  - `hdl/chisel/src/coralnpu/` 是 Coral NPU 主体微架构代码
  - `hdl/chisel/src/soc/` 是 SoC 级交叉总线、子系统与系统配置
  - `hdl/verilog/rvv/` 提供 RVV 向量后端及相关 Verilog 实现
- `fpga/`
  - 面向 FPGA 的综合与集成目录
  - 包含顶层封装、外设接口、约束文件和 bitstream 生成脚本
- `sw/`
  - 软件运行时、算子实现、模拟器绑定和工具代码
  - 既包含 host 侧工具，也包含贴近硬件的软件栈
- `tests/`
  - 验证工程目录
  - 包含 cocotb、ISA 行为验证、RVV 指令测试和系统级用例
- `doc/`
  - 项目文档目录
  - 包括总体说明、微架构说明、LSU/Dispatch 等专题文档
- `hw_sim/`
  - 硬件仿真包装与软件可调用模拟层
- `examples/`
  - 示例程序和最小工作样例
- `rules/`
  - Bazel 构建规则与代码生成规则

### 2.2 工程属性总结

- 构建系统以 Bazel 为主
- 硬件主体以 Chisel 生成 SystemVerilog
- 向量子系统部分同时包含独立 Verilog 实现
- 软件与验证环境围绕仿真器、FPGA 和 RISC-V 用例展开

## 3. 宏观架构分层

从系统视角，可以把 Coral NPU 拆成 5 个层次：

### 3.1 处理核心层

位于 `hdl/chisel/src/coralnpu/`。

- `Core.scala`
  - 核心处理器顶层
  - 将标量核心 `SCore` 作为前端控制核心，并在启用 RVV 时接入 `RvvCore`
- `scalar/`
  - 标量核心实现，负责取指、译码、调度、访存、异常和 CSR
- `rvv/`
  - 向量核心接口与封装
- `float/`
  - 浮点相关模块

这一层可以理解为 NPU 的“执行核心”，其中标量部分承担控制和编排职责，向量/矩阵部分承担高吞吐计算职责。

### 3.2 存储与总线层

这一层负责把核心与片上存储、外部总线和外设连接起来。

- `CoreTlul.scala`
  - 把核心桥接到 TileLink UL 接口
- `DBus2Axi.scala`、`IBus2Axi.scala`
  - 负责不同总线协议与主从接口适配
- `L1DCache.scala`、`L1ICache.scala`
  - 提供一级缓存能力
- `TCM.scala`、`SRAM.scala`
  - 提供紧耦合存储与 SRAM 支撑

这一层决定了 Coral NPU 如何读写指令、数据，以及如何接入更大的系统互连。

### 3.3 SoC 集成层

位于 `hdl/chisel/src/soc/`。

- `CoralNPUChiselSubsystem.scala`
  - 系统级子系统顶层
  - 负责外部 TileLink 端口、DDR AXI、时钟域和中断连线
- `CoralNPUXbar.scala`
  - 数据驱动的 TileLink 交叉总线生成器
- `SoCChiselConfig.scala`
  - 系统模块、地址空间、端口和连接关系的配置源

这一层是“把 Coral NPU 变成可集成 SoC IP”的关键部分。

### 3.4 FPGA 落地层

位于 `fpga/`。

- `fpga/rtl/coralnpu_soc.sv`
  - FPGA 侧顶层封装
- `fpga/ip/`
  - 外设与辅助 IP
- `fpga/sw/`
  - 面向板级验证和启动的配套软件

这一层把 Chisel 生成的核心真正接到 UART、I2C、SPI、DDR、显示等外围资源上。

### 3.5 软件与验证层

- `sw/opt/litert-micro/`
  - 面向模型推理的轻量算子实现
- `sw/coralnpu_sim/`
  - 模拟器软件绑定
- `tests/cocotb/`
  - 用例驱动的功能验证
- `examples/`
  - 面向开发者的最小示例

这一层决定了项目是否“可运行、可验证、可交付”。

## 4. 关键执行链路

从宏观视角看，Coral NPU 的执行链路如下：

1. 标量核心从指令存储取指
2. 标量前端完成 Decode/Dispatch
3. 标量单元决定指令流向
   - 普通标量指令发往 ALU/BRU/MLU/DVU/LSU
   - 向量相关指令发往 RVV 核心
   - 浮点相关指令发往浮点模块
4. LSU 负责把标量与向量访存请求连接到数据总线和外设总线
5. SoC 交叉总线与桥接模块把请求送到 TCM、SRAM、DDR 或外设
6. 结果回写寄存器、退休缓冲区和状态寄存器

所以从系统角色上看，Scalar Core 不是“附属部件”，而是整个 Coral NPU 的控制中枢。

## 5. 标量处理在整体架构中的位置

虽然本报告以宏观分析为主，但标量部分是理解整体结构的关键。

位于 `hdl/chisel/src/coralnpu/scalar/` 的 `SCore.scala` 展示了这一点：

- 统一集成 `Fetch`、`DispatchV2`、`Regfile`、`Lsu`、`Csr`
- 根据配置接入 `RvvCoreIO`
- 负责中断、异常、debug、flush 和 retirement buffer
- 通过 LSU 同时连接标量访存和向量访存协同接口

这意味着：

- Scalar 是前端控制器
- RVV 是后端并行执行器
- 存储系统为两者提供统一的数据通路

## 6. 关键源码入口建议

如果后续要继续深入分析，建议按下面顺序阅读：

1. `README.md`
   - 先了解项目定位、特性和构建方式
2. `doc/overview.md`
   - 了解项目的设计意图和模块划分
3. `doc/microarch/microarch.md`
   - 理解流水线、执行单元与时序关系
4. `hdl/chisel/src/coralnpu/Core.scala`
   - 理解标量核心与 RVV 的总装关系
5. `hdl/chisel/src/coralnpu/scalar/SCore.scala`
   - 理解标量控制核心
6. `hdl/chisel/src/soc/CoralNPUChiselSubsystem.scala`
   - 理解 SoC 集成、外设与 DDR 接入
7. `hdl/chisel/src/soc/CoralNPUXbar.scala`
   - 理解片上互连与地址解码

## 7. 当前结论

当前可以把 Coral NPU 概括为：

- 一个以 RISC-V 标量前端为控制核心的 AI 加速器
- 一个把标量、向量和 ML 数据通路耦合起来的异构处理系统
- 一个同时覆盖 IP 设计、SoC 集成、FPGA 验证与软件栈的完整工程

如果下一步继续做深度分析，最值得优先展开的两个方向是：

- 标量前端如何通过 `DispatchV2` 驱动 RVV 与 LSU
- SoC 交叉总线如何把核心访问映射到 TCM、DDR 和外设
