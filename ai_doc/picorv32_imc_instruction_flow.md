# PicoRV32 中 IMC 指令执行流程与 RTL 对应关系

本文以 `picorv32.v` 中的 `picorv32` 核为对象，说明一个包含地址准备、`load`、乘法、`store` 的小程序在 CPU 内部执行时，大致经过哪些过程，以及这些过程分别对应 RTL 的哪一部分。

## 示例程序

假设 `ENABLE_MUL=1` 或 `ENABLE_FAST_MUL=1`，寄存器 `x5` 中已有乘数，内存中 `0x100` 地址处有一个 32 位数据：

```asm
addi x1, x0, 0x100    # 地址准备：x1 = 0x100
lw   x2, 0(x1)        # load：x2 = MEM[x1 + 0]
mul  x3, x2, x5       # 乘法：x3 = x2 * x5
sw   x3, 4(x1)        # store：MEM[x1 + 4] = x3
```

这里的 `mul` 属于 RISC-V M 扩展。PicoRV32 主译码表没有把 M 扩展乘法列为普通 ALU 指令，而是通过 PCPI 接口交给内部乘法协处理器处理。

## 公共取指、译码和写回路径

每条指令都会先经过相同的前端流程：

1. `cpu_state_fetch` 中发起取指：`mem_do_rinst <= !decoder_trigger && !do_waitirq`，并维护 `reg_pc`、`reg_next_pc`、`latched_rd` 等状态。对应 `picorv32.v:1491` 到 `picorv32.v:1575`。
2. 内存接口根据 `mem_do_rinst`、`mem_do_rdata`、`mem_do_wdata` 生成 `mem_la_read`、`mem_la_write`、`mem_la_addr`，并用 `mem_done` 表示访问完成。对应 `picorv32.v:350` 到 `picorv32.v:388`。
3. 指令数据在 `mem_xfer` 时进入 `mem_rdata_q` 和 `next_insn_opcode`。对应 `picorv32.v:430` 到 `picorv32.v:434`。
4. 取指完成后，主时序逻辑产生 `decoder_trigger <= mem_do_rinst && mem_done`。对应 `picorv32.v:1446`。
5. 第一级译码识别大类：`load`、`store`、`ALU immediate`、`ALU register` 等，并抽取 `rd/rs1/rs2`。对应 `picorv32.v:866` 到 `picorv32.v:884`。
6. 第二级译码在 `decoder_trigger` 时识别具体指令，如 `lw`、`sw`、`addi`、`add/sub` 等，并生成 `decoded_imm`。对应 `picorv32.v:1037` 到 `picorv32.v:1133`。
7. 通用寄存器写回只在 `cpu_state_fetch` 阶段发生：如果前一条指令设置了 `latched_store`，则把 `reg_out` 或 `alu_out_q` 写到 `latched_rd`。对应 `picorv32.v:1303` 到 `picorv32.v:1345`。

因此，这个核的典型节奏不是经典五级流水，而是以 `fetch -> ld_rs1/ld_rs2 -> exec/ldmem/stmem/shift -> fetch 写回` 为主的多周期状态机。

## 状态机总览

主状态定义在 `picorv32.v:1172` 到 `picorv32.v:1179`：

| 状态 | 作用 |
|---|---|
| `cpu_state_fetch` | 取指、处理前一条指令写回、更新 PC |
| `cpu_state_ld_rs1` | 读取源寄存器 1，按指令类型决定下一步 |
| `cpu_state_ld_rs2` | 单端口寄存器配置下读取源寄存器 2 |
| `cpu_state_exec` | 普通 ALU、分支、`jalr` 等执行 |
| `cpu_state_ldmem` | 计算 load 地址并发起数据读 |
| `cpu_state_stmem` | 计算 store 地址并发起数据写 |
| `cpu_state_shift` | 非 barrel shifter 配置下的移位执行 |
| `cpu_state_trap` | 异常或非法指令陷入 |

## 第 1 条：`addi x1, x0, 0x100`

这条指令用于地址准备，结果是 `x1 = 0 + 0x100`。

| 过程 | RTL 对应 |
|---|---|
| 取指 | `cpu_state_fetch` 发起 `mem_do_rinst`，内存接口用 `next_pc` 形成 `mem_la_addr`。见 `picorv32.v:1491` 到 `picorv32.v:1493`、`picorv32.v:379` 到 `picorv32.v:382`。 |
| 译码大类 | opcode `0010011` 被识别为 `is_alu_reg_imm`。见 `picorv32.v:877`。 |
| 译码具体指令 | `funct3=000` 时设置 `instr_addi`。见 `picorv32.v:1057`。 |
| 立即数生成 | I-type 立即数来自 `mem_rdata_q[31:20]`，赋给 `decoded_imm`。见 `picorv32.v:1125` 到 `picorv32.v:1126`。 |
| 读操作数 | `cpu_state_ld_rs1` 中读取 `cpuregs_rs1`，`reg_op2 <= decoded_imm`。见 `picorv32.v:1712` 到 `picorv32.v:1722`。 |
| ALU 执行 | `alu_add_sub` 做 `reg_op1 + reg_op2`，`alu_out` 选择加法结果。见 `picorv32.v:1239` 到 `picorv32.v:1245`、`picorv32.v:1267` 到 `picorv32.v:1271`。 |
| 写回 | `cpu_state_exec` 设置 `latched_store` 和 `latched_stalu`，回到 `fetch` 后写入 `x1`。见 `picorv32.v:1821` 到 `picorv32.v:1825`、`picorv32.v:1313` 到 `picorv32.v:1323`。 |

## 第 2 条：`lw x2, 0(x1)`

这条指令做地址计算 `x1 + 0`，从数据内存读出 32 位值并写入 `x2`。

| 过程 | RTL 对应 |
|---|---|
| 译码大类 | opcode `0000011` 被识别为 load 大类 `is_lb_lh_lw_lbu_lhu`。见 `picorv32.v:875`。 |
| 译码具体指令 | `funct3=010` 时设置 `instr_lw`。见 `picorv32.v:1047` 到 `picorv32.v:1050`。 |
| 立即数生成 | load 使用 I-type 立即数 `mem_rdata_q[31:20]`。见 `picorv32.v:1125` 到 `picorv32.v:1126`。 |
| 读基址寄存器 | `cpu_state_ld_rs1` 中 `reg_op1 <= cpuregs_rs1`，之后进入 `cpu_state_ldmem`。见 `picorv32.v:1696` 到 `picorv32.v:1702`。 |
| 地址计算 | `cpu_state_ldmem` 中执行 `reg_op1 <= reg_op1 + decoded_imm`，随后 `set_mem_do_rdata = 1`。见 `picorv32.v:1880` 到 `picorv32.v:1898`。 |
| 数据读请求 | 内存接口在 `mem_do_rdata` 下把 `mem_la_addr` 设为 `{reg_op1[31:2], 2'b00}`。见 `picorv32.v:379` 到 `picorv32.v:382`。 |
| 字宽和数据选择 | `lw` 设置 `mem_wordsize <= 0`，读回数据走 `mem_rdata_word = mem_rdata`。见 `picorv32.v:1884` 到 `picorv32.v:1888`、`picorv32.v:401` 到 `picorv32.v:407`。 |
| 写回 | 数据返回后 `reg_out <= mem_rdata_word`，设置 `latched_store` 后在下一次 `fetch` 写入 `x2`。见 `picorv32.v:1900` 到 `picorv32.v:1909`、`picorv32.v:1313` 到 `picorv32.v:1323`。 |

## 第 3 条：`mul x3, x2, x5`

`mul` 的 opcode 是 `0110011`，`funct7=0000001`，属于 M 扩展。PicoRV32 的普通 ALU 只覆盖基础整数 ALU 指令，M 扩展通过 PCPI 处理。

| 过程 | RTL 对应 |
|---|---|
| 主核译码 | 第一级只把 opcode `0110011` 识别为 `is_alu_reg_reg`，但第二级基础 R-type 只识别 `funct7=0000000/0100000` 的 `add/sub/...`，不会把 `mul` 置为普通 `instr_*`。见 `picorv32.v:878`、`picorv32.v:1068` 到 `picorv32.v:1077`。 |
| 进入 PCPI 路径 | 未被基础指令集合识别时，`instr_trap` 在启用 `WITH_PCPI` 时代表“交给 PCPI 尝试处理”。见 `picorv32.v:679` 到 `picorv32.v:685`。 |
| 读操作数并拉起 PCPI | `cpu_state_ld_rs1` 读取 `x2`，双端口配置下同时读取 `x5`，设置 `pcpi_valid <= 1`；单端口配置会先进入 `cpu_state_ld_rs2`，再设置 `pcpi_valid`。见 `picorv32.v:1585` 到 `picorv32.v:1616`、`picorv32.v:1759` 到 `picorv32.v:1775`。 |
| 内部乘法器实例化 | `ENABLE_MUL` 实例化 `picorv32_pcpi_mul`，`ENABLE_FAST_MUL` 实例化 `picorv32_pcpi_fast_mul`。见 `picorv32.v:272` 到 `picorv32.v:303`。 |
| PCPI 仲裁 | 内部 PCPI 汇总 `pcpi_mul_wait/ready/rd` 到 `pcpi_int_wait/ready/rd`。见 `picorv32.v:325` 到 `picorv32.v:344`。 |
| 慢速乘法识别 | `picorv32_pcpi_mul` 在 `pcpi_valid` 且 `funct7=0000001` 时按 `funct3` 识别 `mul/mulh/mulhsu/mulhu`。见 `picorv32.v:2221` 到 `picorv32.v:2238`。 |
| 慢速乘法执行 | 慢速乘法器用移位累加/进位保存方式迭代计算，完成后置 `pcpi_wr`、`pcpi_ready`，`pcpi_rd` 给出低 32 位或高 32 位。见 `picorv32.v:2248` 到 `picorv32.v:2314`。 |
| 快速乘法执行 | 快速乘法器直接用 `*` 生成 64 位乘积，通过 `active` 管线给出 `pcpi_ready` 和 `pcpi_rd`。见 `picorv32.v:2345` 到 `picorv32.v:2411`。 |
| 写回 | 主核看到 `pcpi_int_ready` 后把 `pcpi_int_rd` 放入 `reg_out`，`latched_store <= pcpi_int_wr`，回到 `fetch` 后写入 `x3`。见 `picorv32.v:1598` 到 `picorv32.v:1603` 或 `picorv32.v:1770` 到 `picorv32.v:1775`，以及 `picorv32.v:1313` 到 `picorv32.v:1323`。 |

## 第 4 条：`sw x3, 4(x1)`

这条指令做地址计算 `x1 + 4`，把 `x3` 的值写到数据内存。

| 过程 | RTL 对应 |
|---|---|
| 译码大类 | opcode `0100011` 被识别为 store 大类 `is_sb_sh_sw`。见 `picorv32.v:876`。 |
| 译码具体指令 | `funct3=010` 时设置 `instr_sw`。见 `picorv32.v:1053` 到 `picorv32.v:1055`。 |
| 立即数生成 | S-type 立即数来自 `{mem_rdata_q[31:25], mem_rdata_q[11:7]}`。见 `picorv32.v:1129` 到 `picorv32.v:1130`。 |
| 读源寄存器 | `cpu_state_ld_rs1` 读取基址 `x1` 到 `reg_op1`，双端口配置下同时读取待写数据 `x3` 到 `reg_op2`；单端口配置下经 `cpu_state_ld_rs2` 再读取 `x3`。见 `picorv32.v:1724` 到 `picorv32.v:1754`、`picorv32.v:1759` 到 `picorv32.v:1789`。 |
| 地址计算 | `cpu_state_stmem` 中执行 `reg_op1 <= reg_op1 + decoded_imm`，随后 `set_mem_do_wdata = 1`。见 `picorv32.v:1854` 到 `picorv32.v:1871`。 |
| 写请求生成 | 内存接口在 `mem_do_wdata` 下置 `mem_la_write`，地址来自 `reg_op1`，写数据和字节选通信号由 `mem_wordsize` 选择。见 `picorv32.v:379` 到 `picorv32.v:382`、`picorv32.v:401` 到 `picorv32.v:427`。 |
| `sw` 数据和掩码 | `mem_wordsize=0` 时 `mem_la_wdata = reg_op2`，`mem_la_wstrb = 4'b1111`。见 `picorv32.v:401` 到 `picorv32.v:407`。 |
| 完成 | 写事务 `mem_done` 后回到 `fetch`，并用 `decoder_pseudo_trigger` 接续前端状态。见 `picorv32.v:1872` 到 `picorv32.v:1876`。 |

## 四条指令的状态流

```text
addi: fetch -> ld_rs1 -> exec  -> fetch(写 x1)
lw:   fetch -> ld_rs1 -> ldmem -> fetch(写 x2)
mul:  fetch -> ld_rs1[/ld_rs2] -> PCPI等待 -> fetch(写 x3)
sw:   fetch -> ld_rs1[/ld_rs2] -> stmem -> fetch(完成内存写)
```

其中 `[/ld_rs2]` 表示只有在 `ENABLE_REGS_DUALPORT=0` 的单端口寄存器文件配置下才需要单独的 `ld_rs2` 状态；双端口配置可以在 `ld_rs1` 分支里同时拿到 `rs1` 和 `rs2`。

## 关键结论

- 地址计算没有独立地址生成单元，而是在 `ldmem/stmem` 中用 `reg_op1 + decoded_imm` 更新 `reg_op1`，之后内存接口用更新后的 `reg_op1` 形成对齐后的 `mem_la_addr`。
- 普通整数加法、逻辑、比较等由主 ALU 完成，核心逻辑在 `picorv32.v:1221` 到 `picorv32.v:1284`。
- `load/store` 通过 `mem_do_rdata/mem_do_wdata` 与统一内存接口交互，字宽、符号扩展、写掩码在内存辅助逻辑和 `ldmem/stmem` 状态中完成。
- `mul` 不走主 ALU，而是借助 PCPI 内部乘法器；主核负责发起 `pcpi_valid`、等待 `pcpi_ready`、接收 `pcpi_rd` 并写回目的寄存器。
