# Coral NPU 标量处理分析报告

## 1. 分析范围

本报告聚焦 Coral NPU 中的标量处理前端，核心目标是回答三个问题：

1. 标量核心由哪些模块组成
2. 标量核心如何完成取指、译码、调度、执行、访存和异常处理
3. 标量核心如何与 RVV、存储系统和 CSR/调试子系统协同

本次分析的核心源码目录为 `hdl/chisel/src/coralnpu/scalar/`，并结合上层装配模块 `Core.scala` 与 `SCore.scala` 进行理解。

## 2. 结论先行

Coral NPU 的标量处理部分不是一个孤立的“通用 CPU 前端”，而是整个 NPU 的控制核心，承担以下职责：

- 负责取指、分支修正、译码和多发射调度
- 通过记分板和互锁逻辑控制标量、浮点、向量之间的数据依赖
- 驱动 ALU、分支、乘除、LSU、CSR 等执行单元
- 把向量访存请求下沉到 LSU，与 RVV 后端进行数据交换
- 统一处理异常、调试、单步、中断和 flush/fence

可以把它理解为：`Scalar Frontend = 控制平面`，`RVV/ML Dataplane = 计算平面`。

## 3. 标量处理的顶层组成

在 `Core.scala` 中，`Core` 先实例化 `SCore`，然后在启用 RVV 时把 `RvvCore` 直接接到 `SCore` 的 `rvvcore` 接口上。这说明标量核心是整个 Coral NPU 的前端控制入口，RVV 是受其调度和协作的并行执行后端。

在 `SCore.scala` 中，可以看到标量核心顶层统一装配了以下模块：

- `Regfile`
- `Fetch` 或 `UncachedFetch`
- `Csr`
- `DispatchV2`
- `Lsu`
- `FaultManager`
- `RetirementBuffer`
- `Alu`
- `Bru`
- `Mlu`
- `Dvu`

也就是说，`SCore` 本身就是标量微架构的“总装层”。

## 4. 标量流水线主链路

### 4.1 取指阶段

`Fetch.scala` 展示了有缓存的取指路径，关键点如下：

- 支持 4 路前端取指
- 内部带一个 L0 指令缓存
- 在取指阶段就做了部分分支预译码
- 采用静态分支策略：后向分支倾向 taken，前向分支倾向 not-taken
- 接收来自 LSU 的 `iflush`，以完成 `fence.i` 或其他控制流修正后的前端清空

这意味着 Coral NPU 的标量前端不是简单的串行取指，而是面向多发射调度优化过的前端。

### 4.2 译码与调度阶段

`Decode.scala` 中的 `DecodedInstruction` 覆盖了：

- RV32I 基本整数指令
- RV32M 乘除扩展
- ZBB 位操作扩展
- CSR 指令
- 标量 load/store
- 浮点相关指令
- RVV 压缩指令
- 自定义控制指令，如 `mpause`、`flushat`、`flushall`

真正的控制核心是 `DispatchV2`。它并不是仅做 opcode 分发，而是做了完整的发射判定，包括：

- 标量寄存器记分板冲突检测
- 浮点寄存器记分板冲突检测
- branch 后续指令限制
- slot0-only 指令限制
- LSU 队列容量限制
- RVV 队列容量限制
- RVV 配置状态合法性检查
- `vstart` 约束检查
- CSR 必须等待 ROB 清空
- 单步模式与 `mpause` 下的全局停顿条件

所以 `DispatchV2` 是整个标量控制面的关键模块。

### 4.3 执行阶段

标量前端把指令分发到多个执行单元：

- `Alu.scala`：单周期整数算术、逻辑、移位以及 ZBB 位操作
- `Bru.scala`：跳转、条件分支、异常入口、`mret`、`wfi`
- `Mlu.scala`：乘法相关
- `Dvu.scala`：除法与取余
- `Lsu.scala`：所有标量/浮点/向量访存，以及 flush/fence
- `Csr.scala`：控制状态寄存器、异常、中断、调试、向量控制状态

执行结果最终回写到寄存器文件或经由 ROB/LSU/RVV 接口返回。

## 5. 记分板与寄存器文件机制

`Regfile.scala` 说明该标量寄存器堆是 32 项、支持多读多写的寄存器文件，并维护全局 scoreboard。

它的关键作用有三点：

1. 跟踪哪些标量寄存器已经被尚未完成的指令占用
2. 给 `DispatchV2` 提供 `regd` 与 `comb` 两类依赖视图
3. 提供带写前递的读口，减少短相关造成的额外停顿

在 `DispatchV2` 中：

- `readAfterWrite` 用来拦截 RAW 冒险
- `writeAfterWrite` 用来拦截 WAW 冒险
- 浮点路径还维护了独立的浮点记分板

所以 Coral NPU 的标量前端虽然是 in-order dispatch，但并不是“无脑顺序发射”，而是通过记分板精细控制每个 lane 的发射合法性。

## 6. 分支、异常与控制流修正

`Bru.scala` 是控制流修正的核心。

它不仅处理：

- `jal`
- `jalr`
- `beq/bne/blt/bge/...`

还处理：

- `ecall`
- `ebreak`
- `mret`
- `wfi`
- fault 重定向
- interrupt 触发后的跳转

其核心逻辑是：

- 在执行阶段比较真实分支结果
- 若与前端静态预测不一致，则通过 `taken.valid` 触发前端修正
- 在 `pipeline0` 中结合 CSR 输出的 `mtvec/mepc/interrupt` 完成 trap/return 路径切换

因此，取指阶段的轻量预测和执行阶段的 BRU 修正共同构成了 Coral NPU 的控制流闭环。

## 7. LSU 是标量与向量访存的交汇点

`Lsu.scala` 是本次分析最重要的模块之一，因为它不是单纯的标量 load/store 单元，而是：

- 标量 load/store 的执行单元
- 浮点访存执行单元
- RVV 访存的落地点
- `fence.i` / `flush` 的发起者

### 7.1 标量访存路径

标量 LSU 的基本流程是：

1. `DispatchV2` 把 load/store 指令转换为 `LsuCmd`
2. LSU 再把它展开为内部 `LsuUOp`
3. 根据地址映射选择访问：
   - `ibus`：指令存储区域
   - `dbus`：本地数据存储区域
   - `ebus`：外设或外部存储
4. 完成读写后：
   - 标量 load 回写 `io.rd`
   - 浮点 load 回写 `io.rd_flt`
   - store 通过 `storeComplete` 上报退休逻辑

### 7.2 向量访存路径

Coral NPU 的一个非常关键的设计点是：向量访存并不完全由 RVV 自己闭环处理，而是由标量 LSU 与 RVV 协同完成。

具体表现为：

- `Decode.scala` 把 RVV load/store 指令映射成 `VLOAD_*` 和 `VSTORE_*` 等 LSU 操作
- `Lsu.scala` 接收 `rvv2lsu` 输入，里面包含索引向量、向量寄存器数据和 mask
- LSU 内部通过 `vectorLoop` 跟踪 subvector、segment、lmul 等展开状态
- 访存完成后，再通过 `lsu2rvv` 把数据或 store 完成信号回传给 RVV

这表明设计上把“地址生成与总线访问”集中在标量 LSU，把“向量寄存器与执行语义”保留在 RVV 核心，形成了清晰的职责分工。

### 7.3 Flush 与 Fence

LSU 还承担了流水线 flush 发起者的角色：

- `fence.i`
- `flushat`
- `flushall`

它们在 LSU 中被编码为特殊命令，并通过 `io.flush` 通知前端/缓存路径完成清空或同步。

因此在 Coral NPU 中，LSU 既是数据访问单元，也是全局一致性控制点之一。

## 8. CSR、异常、中断与调试

`Csr.scala` 体现了 Coral NPU 标量控制面的“系统软件接口”。

### 8.1 CSR 职责

CSR 模块维护和转发：

- 标准机器态 CSR
- 调试 CSR
- 向量 CSR，如 `vstart`、`vl`、`vtype`、`vxrm`、`vxsat`
- 浮点相关 CSR，如 `frm` 和 `fflags`
- 周期与退休计数器

### 8.2 中断与 Trap

CSR 会综合：

- 外部中断 `irq`
- 定时器中断 `timer_irq`
- 软件中断 `software_irq`

然后根据 `mie` 与 `mstatus_mie` 生成 `interrupt_pending`，再把：

- `interrupt`
- `interrupt_cause`
- `mtvec`
- `mepc`

前递给 BRU，用于控制 trap 入口与 `mret` 返回。

### 8.3 Debug 与单步

CSR 同时还负责：

- 进入/退出 debug mode
- 维护 `dcsr` 与 `dpc`
- 支持单步执行
- 支持 trigger match

也就是说，调试控制并不是外挂逻辑，而是深度嵌入在标量前端控制流中的。

## 9. 标量与 RVV 的交互方式

这是 Coral NPU 项目中最重要的架构关系之一。

### 9.1 调度关系

在 `Core.scala` 中，`RvvCore` 挂接在 `SCore` 之下；在 `Decode.scala` 中，RVV 指令由 `DispatchV2` 判定是否可发射。

这说明：

- RVV 不是独立的前端
- RVV 的入口在标量调度器
- 标量前端决定 RVV 指令什么时候可以合法进入后端

### 9.2 数据与寄存器关系

标量前端需要感知 RVV 对寄存器状态的影响：

- RVV 指令可能写回标量寄存器 `rd`
- RVV 指令可能写回浮点寄存器 `frd`
- RVV 指令可能写回向量寄存器，需要 ROB 跟踪

因此 `DispatchV2` 在记分板层面就已经把 RVV 作为一等公民处理，而不是单纯的外部协处理器。

### 9.3 向量访存关系

在 `SCore.scala` 中，LSU 与 RVV 的连接方式很明确：

- `lsu.io.rvvState` 从 RVV 获取当前配置状态
- `lsu.io.lsu2rvv` 把 LSU 结果送回 RVV
- `rvv2lsu` 把 RVV 的索引/数据/mask 输入送入 LSU

所以 RVV 与 LSU 的耦合并不是旁路式的，而是通过 `SCore` 明确汇接的系统级协作接口。

## 10. 标量与存储系统的交互方式

从系统角度看，标量前端通过 LSU 把访问映射到三类通道：

- `ibus`：主要服务指令侧
- `dbus`：主要服务本地数据侧
- `ebus`：主要服务外设和外部内存

这种结构有两个直接意义：

1. 标量前端既服务控制流，也承担地址生成和数据搬运调度
2. 外设、DDR、TCM 等不同存储目标在 LSU 层被统一抽象

因此若要继续分析 Coral NPU 的系统行为，LSU 和 SoC Crossbar 是最关键的连接点。

## 11. 关键实现定位

下面列出本次分析最关键的代码定位，格式为 `起始行:结束行:文件路径`：

- `62:66:hdl/chisel/src/coralnpu/Core.scala`
- `55:64:hdl/chisel/src/coralnpu/scalar/SCore.scala`
- `117:137:hdl/chisel/src/coralnpu/scalar/SCore.scala`
- `237:244:hdl/chisel/src/coralnpu/scalar/SCore.scala`
- `268:302:hdl/chisel/src/coralnpu/scalar/SCore.scala`
- `16:18:hdl/chisel/src/coralnpu/scalar/Fetch.scala`
- `105:110:hdl/chisel/src/coralnpu/scalar/Fetch.scala`
- `146:162:hdl/chisel/src/coralnpu/scalar/Fetch.scala`
- `184:199:hdl/chisel/src/coralnpu/scalar/Fetch.scala`
- `330:388:hdl/chisel/src/coralnpu/scalar/Decode.scala`
- `395:425:hdl/chisel/src/coralnpu/scalar/Decode.scala`
- `447:521:hdl/chisel/src/coralnpu/scalar/Decode.scala`
- `635:676:hdl/chisel/src/coralnpu/scalar/Decode.scala`
- `703:808:hdl/chisel/src/coralnpu/scalar/Decode.scala`
- `91:113:hdl/chisel/src/coralnpu/scalar/Regfile.scala`
- `163:205:hdl/chisel/src/coralnpu/scalar/Regfile.scala`
- `119:145:hdl/chisel/src/coralnpu/scalar/Bru.scala`
- `190:218:hdl/chisel/src/coralnpu/scalar/Bru.scala`
- `120:150:hdl/chisel/src/coralnpu/scalar/Alu.scala`
- `824:855:hdl/chisel/src/coralnpu/scalar/Lsu.scala`
- `888:967:hdl/chisel/src/coralnpu/scalar/Lsu.scala`
- `1015:1046:hdl/chisel/src/coralnpu/scalar/Lsu.scala`
- `384:405:hdl/chisel/src/coralnpu/scalar/Csr.scala`
- `493:500:hdl/chisel/src/coralnpu/scalar/Csr.scala`
- `527:558:hdl/chisel/src/coralnpu/scalar/Csr.scala`
- `584:615:hdl/chisel/src/coralnpu/scalar/Csr.scala`

## 12. 最终判断

Coral NPU 的标量处理部分有三个鲜明特点：

1. 它是一个偏控制型、强协同的标量前端，而非单独追求高 IPC 的通用 CPU
2. 它把 RVV、浮点、LSU、CSR、调试与异常统一纳入同一套调度和互锁框架
3. 它通过 LSU 统一承接标量和向量访存，因此在系统层面具备很强的“中枢”属性

如果继续深入，最推荐的后续专题是：

- `DispatchV2` 的 lane 级互锁与发射策略
- `Lsu.scala` 中 `vectorLoop` 的展开机制
- `Csr.scala` 与 `Bru.scala` 如何协同实现 trap/debug/interrupt 路径
