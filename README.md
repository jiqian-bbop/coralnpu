# Coral NPU
Coral NPU（谷歌开源神经网络处理单元）

Coral NPU is a hardware accelerator for ML inferencing. Coral NPU is an Open Source IP designed by Google Research and is freely available for integration into ultra-low-power System-on-Chips (SoCs) targeting wearable devices such as hearables, augmented reality (AR) glasses and smart watches.
Coral NPU 是一个用于机器学习推理的硬件加速器。Coral NPU 是由 Google Research 设计的开源 IP，可免费集成到面向助听设备、增强现实（AR）眼镜和智能手表等可穿戴设备的超低功耗片上系统（SoC）中。

Coral NPU is a neural processing unit (NPU), also known as an AI accelerator or deep-learning processor. Coral NPU is based on the 32-bit RISC-V Instruction Set Architecture (ISA).
Coral NPU 是一种神经网络处理单元（NPU），也称为 AI 加速器或深度学习处理器。Coral NPU 基于 32 位 RISC-V 指令集架构（ISA）。

Coral NPU includes three distinct processor components that work together: matrix, vector (SIMD), and scalar.
Coral NPU 包含三个相互协作的不同处理器组件：矩阵、向量（SIMD）和标量。

![Coral NPU Archicture](doc/images/arch_data_flow.png)
上图：Coral NPU 架构图。

[Coral NPU Architecture Datasheet](https://developers.google.com/coral/guides/hardware/datasheet)
[Coral NPU 架构数据手册](https://developers.google.com/coral/guides/hardware/datasheet)

## Coral NPU Features
## Coral NPU 特性

Coral NPU offers the following top-level feature set:
Coral NPU 提供以下顶层特性：

* RV32IMF_Zve32x RISC-V instruction set (specifically `rv32imf_zve32x_zicsr_zifencei_zbb`)
* RV32IMF_Zve32x RISC-V 指令集（具体为 `rv32imf_zve32x_zicsr_zifencei_zbb`）
* 32-bit address space for applications and operating system kernels
* 为应用程序和操作系统内核提供 32 位地址空间
* Four-stage processor, in-order dispatch, out-of-order retire
* 四级流水处理器，按序派发，乱序退休
* Four-way scalar, two-way vector dispatch
* 四路标量派发、两路向量派发
* 128-bit SIMD, 256-bit (future) pipeline
* 128 位 SIMD、256 位（未来版本）流水线
* 8 KB ITCM memory (tightly-coupled memory for instructions)
* 8 KB ITCM 存储器（用于指令的紧耦合存储器）
* 32 KB DTCM memory (tightly-coupled memory for data)
* 32 KB DTCM 存储器（用于数据的紧耦合存储器）
* Both memories are single-cycle-latency SRAM, more efficient than cache memory
* 这两种存储器都是单周期延迟的 SRAM，比缓存存储器更高效
* AXI4 bus interfaces, functioning as both manager and subordinate, to interact with external memory and allow external CPUs to configure Coral NPU
* AXI4 总线接口，既可作为主设备也可作为从设备，用于与外部存储器交互，并允许外部 CPU 配置 Coral NPU

## System Requirements
## 系统要求

* Bazel 7.4.1
* Bazel 7.4.1
* Python 3.9-3.12 (3.13 support is in progress)
* Python 3.9-3.12（正在推进对 3.13 的支持）
* [SRecord](https://srecord.sourceforge.net/)
* [SRecord](https://srecord.sourceforge.net/)

## Quick Start
## 快速开始

```bash
# Ensure that test suite passes
# 确保测试套件能够通过
bazel run //tests/cocotb:core_mini_axi_sim_cocotb

# Build a binary
# 构建一个二进制程序
bazel build //examples:coralnpu_v2_hello_world_add_floats

# Build the Simulator (non-RVV for shorter build time):
# 构建模拟器（不启用 RVV，以缩短构建时间）：
bazel build //tests/verilator_sim:core_mini_axi_sim

# Run the binary on the simulator:
# 在模拟器上运行该二进制程序：
bazel-bin/tests/verilator_sim/core_mini_axi_sim --binary bazel-out/k8-fastbuild-ST-dd8dc713f32d/bin/examples/coralnpu_v2_hello_world_add_floats.elf
```

![](doc/images/Coral_Logo_200px-2x.png)
上图：Coral 标志。
