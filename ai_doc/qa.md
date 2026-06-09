# Q&A 记录

> 本文档记录用户与 Claude Code 在 picorv32 项目中的问答交互，按时间倒序排列。

---

## 2026-06-09

### Q13: C.SLLI 译码为何需要对 `decoded_rs2` 赋值？

**问：** [picorv32.v:1006-1010](picorv32.v#L1006-L1010) 中 C.SLLI 指令的 case 分支，为什么要对 `decoded_rs2` 进行赋值？C.SLLI 的移位量是立即数而非寄存器，为何使用 `decoded_rs2`？

**答：**

**核心原因：`decoded_rs2` 不是"第二源寄存器编号"，而是"指令字中位于 rs2 字段位置的比特值"。对移位立即数指令，该字段承载的是移位量 shamt，而非寄存器索引。**

---

#### RISC-V 指令编码中的字段复用

在 RISC-V I-type 编码格式中，移位立即数指令（SLLI/SRLI/SRAI）的移位量 (shamt) 恰好位于 `[24:20]` 位——即 rs2 字段的位置：

```
I-type 32-bit 编码（移位指令展开后）:
  [31:25]=funct7  [24:20]=shamt  [19:15]=rs1  [14:12]=funct3  [11:7]=rd  [6:0]=opcode
                       ↑
                  shamt 与 R-type 中 rs2 是同一字段位置
```

因此 PicoRV32 中 `decoded_rs2` 是**语义多态**的——根据指令类型承载不同的含义：

| 指令类别 | `decoded_rs2` 实际含义 | 下游使用 |
|---|---|---|
| R-type (ADD, SUB, SLT...) | 第二源寄存器编号 | `cpuregs[decoded_rs2]` ([line 1371](picorv32.v#L1371)) |
| **Shift imm (SLLI, SRLI, SRAI)** | **移位量 (shamt)** | `reg_sh <= decoded_rs2` ([line 1727](picorv32.v#L1727)) |
| Store (SB, SH, SW) | 存储数据源寄存器 (rs2) | `cpuregs[decoded_rs2]` |
| Branch | 第二比较操作数 (rs2) | `cpuregs[decoded_rs2]` |

---

#### C.SLLI 的赋值逻辑

```verilog
// picorv32.v:1005-1011
3'b000: begin // C.SLLI
    if (!mem_rdata_latched[12]) begin
        is_alu_reg_imm <= 1;
        decoded_rd  <= mem_rdata_latched[11:7];
        decoded_rs1 <= mem_rdata_latched[11:7];
        decoded_rs2 <= {mem_rdata_latched[12], mem_rdata_latched[6:2]};
    end
end
```

C.SLLI 编码：`[15:13]=000`, `[12]=nzuimm[5]`, `[11:7]=rd/rs1`, `[6:2]=nzuimm[4:0]`

其中 `{mem_rdata_latched[12], mem_rdata_latched[6:2]}` 是 6-bit 拼接，赋给 5-bit 的 `decoded_rs2[4:0]`。Verilog 语义下高位被截断（bit 12 被丢弃）。由于 line 1006 的 `if (!mem_rdata_latched[12])` 已守卫 bit12=0，截断无害。

> **设计意图**：6-bit 拼接反映了 RV64C 编码约定（6-bit shift amount），picorv32 (RV32) 通过截断自然适配到 5-bit。`!mem_rdata_latched[12]` 守卫确保 RV32 合法性（shift amount ≤ 31）。

---

#### 下游使用路径

`decoded_rs2` 在 C.SLLI → SLLI 执行时有两条消耗路径，均将其作为 shift amount 使用：

**路径 A — 无 barrel shifter**（[picorv32.v:1727](picorv32.v#L1727)）：
```verilog
reg_sh <= decoded_rs2;          // 移位量存入迭代移位寄存器
cpu_state <= cpu_state_shift;   // 进入多周期移位状态机
```

**路径 B — 有 barrel shifter**（[picorv32.v:1735](picorv32.v#L1735)）：
```verilog
reg_op2 <= ... decoded_rs2;     // 移位量直连 ALU 第二操作数
mem_do_rinst <= mem_do_prefetch; // 单周期完成
```

---

#### 完整数据流

```
C.SLLI (16-bit compressed)
  │
  ├─ C-ISA decode (line 1003-1011): is_alu_reg_imm=1, decoded_rs2=shamt
  ├─ C-ISA expansion (line 447-549): mem_rdata_q → 32-bit SLLI encoding
  ├─ Fine decode (line 1082): instr_slli=1
  │
  └─ ld_rs1: cpuregs[decoded_rs1] → reg_op1
       ├─ !BARREL_SHIFTER: reg_sh <= decoded_rs2 → cpu_state_shift (line 1727)
       └─  BARREL_SHIFTER: reg_op2 <= decoded_rs2 → cpu_state_exec (line 1735)
```

**结论**：`decoded_rs2` 是"rs2 字段位置的值"，不是"第二寄存器编号"。C.SLLI 对其赋值是提取该字段位置上的移位立即数，用于驱动下游移位器。这是 RISC-V 编码级别的固有设计——移位立即数复用 I-type 中的 rs2 字段位置。

**涉及文件：**
- [picorv32.v:1005-1011](picorv32.v#L1005-L1011) — C.SLLI decode 与 `decoded_rs2` 赋值
- [picorv32.v:1722-1728](picorv32.v#L1722-L1728) — `decoded_rs2` 作为 `reg_sh` 移位量的路径 A
- [picorv32.v:1730-1735](picorv32.v#L1730-L1735) — `decoded_rs2` 作为 `reg_op2` 移位量的路径 B
- [picorv32.v:1082](picorv32.v#L1082) — `instr_slli` 的最终判定

---

### Q12: `dbg_*` / `q_*` / `cached_*` 信号及 `new_ascii_instr` 的作用与用途

**问：** [picorv32.v:705](picorv32.v#L705) 行开始定义的 `dbg_*` 信号和 `new_ascii_instr`，以及 [picorv32.v:774](picorv32.v#L774) 行开始定义的 `q_*`、`cached_*` 信号的作用分别是什么，用于 RTL 级验证还是芯片实际运行时 debug？

**答：**

所有上述信号均**仅用于 RTL 级验证**（仿真 trace + 形式化验证），不用于芯片运行时 debug（无 JTAG/APB 等硅片 debug 接口），普通综合时被 `ifdef` 裁剪或优化掉。

---

#### 信号分组与数据流

```
           instr_xxx ──────► new_ascii_instr (combinational)
                              │
  decoded_imm, next_insn_opcode, decoded_rd/rs1/rs2
                              │
                              ▼
     ┌─────────── dbg_* 信号组 (FORMAL_KEEP reg) ───────────┐
     │  dbg_ascii_instr / dbg_insn_opcode / dbg_insn_imm    │
     │  dbg_insn_rs1/2 / dbg_insn_addr                      │
     │  dbg_rs1val / dbg_rs2val / dbg_rs1/2val_valid        │
     │  dbg_ascii_state / dbg_next                          │
     └──────────────────────┬───────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
     ┌──────────┐   ┌──────────────┐  ┌─────────────────┐
     │  q_* reg │   │ cached_* reg │  │ 输出接口         │
     │1-cycle   │   │C-ISA repeat  │  │ $display(DEBUG/  │
     │pipeline  │   │exec cache    │  │  DEBUGASM)       │
     │delay     │   │              │  │ rvfi_* (RISCV_   │
     └──────────┘   └──────────────┘  │  FORMAL)         │
                                      └─────────────────┘
```

---

#### 1. `new_ascii_instr`（[picorv32.v:704](picorv32.v#L704)）

纯组合逻辑，将 `instr_lui`、`instr_add` 等解码标志映射为对应指令的 ASCII 字符串（如 `"lui"`、`"add"`）。仅在仿真中有意义——将指令译码为人类可读助记符，方便 `$display` 输出。

#### 2. `dbg_*` 信号组（[picorv32.v:705-713](picorv32.v#L705-L713), [1198](picorv32.v#L1198), [181](picorv32.v#L181)）

声明为 `FORMAL_KEEP reg`。`FORMAL_KEEP`（[picorv32.v:39](picorv32.v#L39)）仅在 `FORMAL` 或 `DEBUGNETS` 宏定义时才展开为 `(* keep *)`，阻止综合工具优化。普通综合时不定义这些宏，信号被裁掉。

**功能**：聚合当前正在执行的指令的所有关键信息——PC、操作码、源寄存器、立即数、操作数值、当前 CPU 状态名。

**数据来源**：在组合逻辑块（[picorv32.v:823-849](picorv32.v#L823-L849)）中动态选择：
- **默认**：从 `q_*` 流水寄存器取旧值（保持稳定）
- **新指令发射**（`dbg_next && decoder_pseudo_trigger_q`）：从 `cached_*` 回放缓存的指令信息
- **新指令发射**（`dbg_next && !decoder_pseudo_trigger_q`）：从 `new_ascii_instr`、`next_insn_opcode`、`decoded_imm` 等获取最新译码结果

#### 3. `q_*` 信号（[picorv32.v:774-780](picorv32.v#L774-L780)）+ `dbg_next`

一级流水线延迟寄存器（[picorv32.v:792-799](picorv32.v#L792-L799)）：每个时钟周期将 `dbg_*` 的值锁存一拍，形成"先取旧值，有条件时更新"的流水模式。确保 `dbg_*` 在无新指令的周期内保持稳定输出。

#### 4. `cached_*` 信号（[picorv32.v:785-790](picorv32.v#L785-L790)）

当 `decoder_trigger_q` 置位时（[picorv32.v:806-816](picorv32.v#L806-L816)），将译码结果（助记符、立即数、操作码、寄存器编号）缓存。用途：C-ISA 指令重复执行时（如移位指令的 `cpu_state_shift`），`decoder_pseudo_trigger_q` 置位，`dbg_*` 组合逻辑从 `cached_*` 回放，无需重新译码。

#### 两个消费者

| 消费者 | 宏 | 用途 |
|---|---|---|
| `$display` trace | `DEBUG` / `DEBUGASM` | 仿真中每条指令发射时打印 PC + 操作码 + 助记符 |
| `rvfi_*` 输出端口 | `RISCV_FORMAL` | 驱动 RISC-V Formal Interface，供 `riscv-formal` 等工具做等价性/属性验证 |

RVFI 数据映射（[picorv32.v:1996-2009](picorv32.v#L1996-L2009)）：`rvfi_insn <= dbg_insn_opcode`、`rvfi_pc_rdata <= dbg_insn_addr`、`rvfi_rs1_addr <= ... dbg_insn_rs1` 等。

#### 结论

这些信号是**纯 RTL 级验证基础设施**。`FORMAL_KEEP` 和所有消费者均受 `ifdef` 保护，不在硅片中存在。它们服务于两个验证层次——仿真可见性（`$display` trace）和形式化验证（RVFI 接口）。

**涉及文件：**
- [picorv32.v:39-45](picorv32.v#L39-L45) — `FORMAL_KEEP` 宏定义
- [picorv32.v:704-713](picorv32.v#L704-L713) — `new_ascii_instr` 与 `dbg_*` 声明
- [picorv32.v:715-772](picorv32.v#L715-L772) — `new_ascii_instr` 组合赋值
- [picorv32.v:774-790](picorv32.v#L774-L790) — `q_*`、`cached_*` 声明
- [picorv32.v:792-816](picorv32.v#L792-L816) — `q_*`/`cached_*` 时序逻辑
- [picorv32.v:823-849](picorv32.v#L823-L849) — `dbg_*` 组合赋值（三级 MUX）
- [picorv32.v:853-869](picorv32.v#L853-L869) — `$display` 消费者
- [picorv32.v:1198-1209](picorv32.v#L1198-L1209) — `dbg_ascii_state`
- [picorv32.v:1991-2009](picorv32.v#L1991-L2009) — RVFI 消费者

---

### Q11: `mem_xfer` 状态下 line 620 `else` 分支的访存场景分析

**问：** [picorv32.v:620](picorv32.v#L620) 的 `else` 分支对应访存时的哪种情况？即上一条指令为 16-bit 还是 32-bit，对应哪种访存类型，当前 PC 要取的指令对应哪种访存类型？

**答：**

第 620 行 `else` 分支是第 615 行 `if (COMPRESSED_ISA && mem_la_read)` 的补集。在后综合逻辑中，state 1（读等待）下 `mem_xfer` 有效时进入。

#### `mem_la_read` 在 state 1 中的行为

`mem_la_read`（[picorv32.v:388-389](picorv32.v#L388-L389)）的两个 OR 项中，Term 1 要求 `!mem_state`（state 0），在 state 1 时恒为假。所以 state 1 下 `mem_la_read=1` 的唯一可能是 Term 2：

```
COMPRESSED_ISA && mem_xfer && mem_la_firstword && !mem_la_secondword && &mem_rdata_latched[1:0]
```

即：C-ISA 模式下，PC 指向上半字（`PC[1]=1`，首次取指），且取到的数据 `bits[1:0]==2'b11`（上半字是 32-bit 指令的上半部分，需再读一次）。

**结论：else 分支 = 所有"不需要再读 secondword"的情况。**

---

#### 四大场景详细分析

##### 场景 A：非 C-ISA 模式（`COMPRESSED_ISA=0`）

| 项目 | 内容 |
|---|---|
| 上一条指令 | 32-bit |
| 本周期访问类型 | `mem_do_rinst` / `mem_do_rdata` / `mem_do_prefetch` |
| 当前取指类型 | 32-bit，字对齐 |
| 623-629 行 | 不执行（COMPRESSED_ISA 为假） |
| 状态转移 | `mem_state <= mem_do_rinst \|\| mem_do_rdata ? 0 : 3` |

##### 场景 B：C-ISA，PC[1:0]=00（字对齐），取到 16-bit 压缩指令（`mem_rdata[1:0]!=11`）

```
总线读 word_N: [15:0] = 当前指令（16-bit 压缩, bits[1:0]!=11）
               [31:16] = 同字上半字 = 顺序执行的下一条指令
```

| 项目 | 内容 |
|---|---|
| 上一条指令 | 16-bit 压缩（PC+2 跳至当前地址），或分支/跳转至此 |
| 本周期访问类型 | 需求取指（rinst）或预取完成（prefetch → state 3） |
| 当前取指类型 | **16-bit 压缩指令**，位于 word_N 下半字 |
| 预取缓存 | `prefetched_high_word <= 1`，`mem_16bit_buffer <= mem_rdata[31:16]` |
| 作用 | 顺序执行 PC+2 进入 word_N 上半字时，复用缓存无需总线读 |

##### 场景 C：C-ISA，PC[1:0]=00（字对齐），取到 32-bit 指令（`mem_rdata[1:0]==11`）

| 项目 | 内容 |
|---|---|
| 上一条指令 | 取决于执行流 |
| 本周期访问类型 | 需求取指或预取 |
| 当前取指类型 | **32-bit 指令**，完整包含在 word_N 中 |
| 预取缓存 | `prefetched_high_word <= 0`（清空） |
| 原因 | 32-bit 指令占满整个 word，PC+4 指向 word_{N+1}，同字上半字不构成独立指令 |

##### 场景 D：C-ISA，PC[1:0]=10（上半字），取到 16-bit 压缩指令

**触发条件**：分支/跳转/中断使 PC 以 `10` 结尾，且上半字 `bits[17:16]!=11`（16-bit 压缩指令）。

```
mem_la_firstword=1, mem_la_read=0（因 &mem_rdata_latched[1:0] 为假）
→ 进入 else 分支
```

| 项目 | 内容 |
|---|---|
| 上一条指令 | 分支/跳转指令，已改变 PC |
| 本周期访问类型 | 需求取指（rinst，分支目标） |
| 当前取指类型 | **16-bit 压缩指令**，位于 word_N 上半字 |
| 623-629 行注意点 | `mem_rdata[1:0]` 对应的是**下半字**的 bits（已过时的数据）。此处的缓存更新对顺序执行无直接帮助（PC+2 跳到 word_{N+1}，不依赖本字的缓存） |
| 此行与 line 615 的分水岭 | 同一地址（PC[1]=1）、同一 `mem_la_firstword=1`，区别见下图 |

```
PC[1:0]=10 的上半字取指, mem_la_firstword=1
         │
         ├─ mem_rdata[17:16] == 2'b11 → 32-bit 指令上半部分 → line 615（需 secondword）
         │
         └─ mem_rdata[17:16] != 2'b11 → 16-bit 压缩指令     → else 分支（已完成）
```

##### 场景 E：C-ISA secondword 完成（两次读取的收尾）

第一次 `mem_xfer` 走了 line 615 分支（置 `mem_la_secondword<=1`），第二次读完成后 `mem_la_secondword=1` 使 `mem_la_read` Term 2 条件 `!mem_la_secondword` 为假 → 进入 else 分支。

```
第一次 mem_xfer: line 615-619, mem_la_secondword <= 1, mem_valid 保持 1
                 发起第二次总线读（word_{N+1}）
第二次 mem_xfer: mem_la_read Term 2 因 !mem_la_secondword=0 为假
                 → 进入 else 分支
```

| 项目 | 内容 |
|---|---|
| 上一条"指令" | 被拼接为跨字 32-bit 指令 |
| 本周期访问类型 | secondword 完成 |
| 当前取指 | 32-bit 指令已完整（由两次读组装） |
| 623-629 行 | `mem_la_secondword` 在行 624 被读取时**仍为 1**（非阻塞赋值未生效），条件为真 → 缓存 `mem_rdata[31:16]`，`prefetched_high_word<=1` |

---

#### else 分支逻辑拆解

```verilog
// 行 620-632
end else begin
    mem_valid <= 0;                              // ① 关闭总线请求
    mem_la_secondword <= 0;                      // ② 重置 secondword 状态
    if (COMPRESSED_ISA && !mem_do_rdata) begin   // ③ C-ISA 指令取指的预取缓存更新
        if (~&mem_rdata[1:0] || mem_la_secondword)
            {mem_16bit_buffer, prefetched_high_word} <= {mem_rdata[31:16], 1'b1};
        else
            prefetched_high_word <= 0;
    end
    mem_state <= mem_do_rinst || mem_do_rdata ? 0 : 3;  // ④ 下一状态
end
```

| 步骤 | 作用 |
|---|---|
| ① `mem_valid <= 0` | 撤销总线请求，标志本次传输结束 |
| ② `mem_la_secondword <= 0` | 重置跨字取指状态，为下次做准备 |
| ③ 缓存更新 | 当本次是 C-ISA 指令取指（非 load）时，判断是否需要缓存上半字供下一条指令预取 |
| ④ 状态转移 | 需求取指/load 完成 → state 0 idle；纯预取完成 → state 3 prefetch done |

---

#### 与 Q10 的对应关系

| Q10 分析的 `if` 分支 | 本问分析的 `else` 分支 |
|---|---|
| `COMPRESSED_ISA && mem_la_read` 为真 | 上述条件为假 |
| 当前取指被识别为 32-bit 指令的上半部分 | 当前取指已是一次完整的读取 |
| 需再发起一次总线读（secondword） | 无需再读，直接完成 |
| 访问类型仅为 C-ISA 上半字 `bits[1:0]==11` | 覆盖所有其他场景（非 C-ISA、字对齐、16-bit 压缩指令、secondword 收尾） |

**涉及文件：**
- [picorv32.v:613-632](picorv32.v#L613-L632) — state 1 `mem_xfer` 完整 if-else 分支
- [picorv32.v:388-389](picorv32.v#L388-L389) — `mem_la_read` 定义
- [picorv32.v:367](picorv32.v#L367) — `mem_la_firstword` 定义
- [picorv32.v:394-397](picorv32.v#L394-L397) — `mem_rdata_latched` 四级选择

---

### Q10: `mem_xfer` 状态下 line 615-619 分支中 `!mem_la_use_prefetched_high_word` 的场景分析

**问：** 在 state 1 的 `mem_xfer` 分支中（[picorv32.v:613-619](picorv32.v#L613-L619)），`if (COMPRESSED_ISA && mem_la_read)` 进入需要 secondword 的路径。其中 `!mem_la_use_prefetched_high_word`（line 618）的判断对应什么场景？为什么此时可能有 `mem_la_use_prefetched_high_word` 为真的情况？上一条/下一条指令分别可能是什么？

**答：**

该分支的进入条件（由 `mem_la_read` 第二条件推导）：C-ISA 模式下，`mem_xfer` 刚完成，`mem_la_firstword=1`（PC 指向上半字），且刚取到的半字 `bits[1:0] == 2'b11`。即：**首次取上半字发现它是 32-bit 指令的上半部分，需要 secondword 拼接**。

在此分支内，line 618 `if (!mem_la_use_prefetched_high_word)` 区分了两种子场景：

---

#### 子场景 A：`mem_la_use_prefetched_high_word = 0` → 执行 line 619

**数据来源**：真实的总线读取（mem_rdata_latched Case 3），`mem_rdata[31:16]` 是本周期刚通过 `mem_valid && mem_ready` 握手拿到的新鲜数据。**必须保存到 `mem_16bit_buffer`**，供第二次读完成后拼接。

**触发条件**：跳转/分支到 halfword 对齐的地址，且该地址恰好落在 32-bit 指令的中间（`PC[1]=1`，对应半字 `bits[1:0]==11`）。`clear_prefetched_high_word` 已将旧缓存清零，无预取可用。

```
上一条: 分支/跳转/中断 → PC 被改变，clear_prefetched_high_word 清零缓存
当前PC: word 的上半字（PC[1]=1），无预取 → Case 3 读总线
         ↓ mem_rdata[31:16] bits[1:0]==11 → 进入 line 615
         ↓ !mem_la_use_prefetched_high_word → 执行 line 619 保存数据
下一条: 被硬件解释为跨字的 32-bit 指令（需 secondword 拼接）
```

#### 子场景 B：`mem_la_use_prefetched_high_word = 1` → 跳过 line 619

**数据来源**：预取缓存（mem_rdata_latched Case 1），`mem_16bit_buffer` 已持有正确的上半字。`mem_rdata` 此时是**过期/无效数据**（`mem_valid=0`，未发起总线读），绝不能覆盖 buffer。

**触发条件**：上一条指令是 word 下半字的 16-bit 压缩指令，预取时将同字的上半字缓存到 `mem_16bit_buffer`（`prefetched_high_word=1`）。PC+2 后进入上半字，复用缓存，但该半字恰好 `bits[1:0]==11`。

```
上一条: 16-bit 压缩指令，位于 word_N 的下半字（PC[1:0]=00）
         ↓ 取指时读到整个 word_N
         ↓ mem_rdata[15:0] bits[1:0]≠11 → 压缩指令
         ↓ mem_rdata[31:16] → 预取到 mem_16bit_buffer, prefetched_high_word=1
         ↓ 执行完, PC += 2
当前PC: word_N 的上半字, mem_la_use_prefetched_high_word=1 → Case 1
         ↓ 检查 mem_16bit_buffer bits[1:0] == 11
         ↓ 进入 line 615, 但跳过 line 619（buffer 已有数据）
下一条: 被硬件解释为跨字 32-bit 指令
```

在合法 RISC-V 代码中此序列极少出现——压缩指令流中上半字一般也是压缩指令（`bits[1:0]≠11`）。`bits[1:0]==11` 在顺序压缩指令流中不合法，但硬件仍按编码规则处理，最终大概率落入 trap。

---

**总结**：line 618 保护的是 `mem_16bit_buffer` 的数据完整性。当数据来自预取缓存时已在 buffer 中，不应被可能过期的 `mem_rdata[31:16]` 覆盖。

**涉及文件：**
- [picorv32.v:613-632](picorv32.v#L613-L632) — state 1 `mem_xfer` 分支完整逻辑
- [picorv32.v:388](picorv32.v#L388) — `mem_la_read` 第二条件（触发 secondword 的关键）
- [picorv32.v:394-396](picorv32.v#L394-L396) — `mem_rdata_latched` 四级选择（Case 1 vs Case 3）

---

### 问 2（追问）：mem_la_secondword 取的是 32-bit 指令的低半还是高半？子场景 B 中 mem_valid 的真实时序？以及为什么 mem_la_use_prefetched_high_word 会 =1？

**此追问纠正了此前回答中的两处错误：mem_valid 时序和指令半字方向。**

#### 答 2.1：mem_la_secondword 取的是 32-bit 指令的 LOWER 16 bits

依据代码 [picorv32.v:395](picorv32.v#L395)：

```verilog
COMPRESSED_ISA && mem_la_secondword ?
    {mem_rdata_latched_noshuffle[15:0], mem_16bit_buffer}
```

`mem_16bit_buffer` 存的是第一次读从 PC 处获取的上半字（UPPER 16 bits，从 `mem_rdata[31:16]` 来），第二次读取回后拼接时被放在高 16 位。第二次读的 `[15:0]` 被放在低 16 位。所以：

> **第一次取的是 UPPER 16 bits（PC 所在的半字），第二次（secondword）取的是 LOWER 16 bits。**

这和"地址高低"不同——UPPER 16 bits 地址是 word_N+2（PC），LOWER 16 bits 地址是 word_{N+1}+0（更高的字对齐地址的下半部分）。

#### 答 2.2：子场景 B 的 mem_valid 真实时序

此前错误地描述为"mem_valid=0, 不发总线读"。实际时序如下：

```
Cycle 1: mem_state=0, mem_la_use_prefetched_high_word=1
         mem_valid <= 0     (line 595)  ← 第一次读跳过，复用缓存
         mem_state <= 1     (line 598)
         mem_xfer = 1                   ← prefetch 复用条件成立

Cycle 2: mem_state=1, mem_xfer=1
         mem_la_read 第二条件成立 → 进入 line 615
         mem_valid <= 1      (line 616) ← 第二次读发起！
         mem_la_secondword <= 1
```

**第二次读确实通过 `mem_valid=1` 发起了真实总线读。** 子场景 A 和 B 在第二次读的行为上完全一致——都通过 `mem_valid=1` 读 word_{N+1}。区别仅在于第一次读是否经过总线。

#### 答 2.3：mem_la_use_prefetched_high_word=1 的含义（再次确认）

`mem_la_use_prefetched_high_word=1` 仅表示**第一次读**（PC 所在的上半字）来自预取缓存。它与第二次读的目标地址无关——`mem_16bit_buffer` 存的是当前 PC 处的上半个 halfword（来自上一次取指的预取），不是第二次读的目标数据。

---

### Q9: `mem_do_prefetch` / `mem_do_rinst` / `mem_do_rdata` / `mem_do_wdata` 的含义及触发条件

**问：** `mem_do_prefetch`、`mem_do_rinst`、`mem_do_rdata`、`mem_do_wdata` 的含义分别是什么？这几个信号由哪些条件确定，对应什么情况？

**答：**

这四个信号是 PicoRV32 内存接口的**请求类型寄存器**，声明于 [picorv32.v:357-360](picorv32.v#L357-L360)。生命周期：在各 CPU 状态中被置 1，在 `mem_done` 或 `!resetn` 时统一清零（[picorv32.v:1962-1966](picorv32.v#L1962-L1966)）。

**互斥性说明**：断言（[picorv32.v:561-571](picorv32.v#L561-L571)）检查的是写 vs 读（`wdata` vs 其他）和指令 vs 数据（`rdata` vs `prefetch`/`rinst`）的互斥。但 `mem_do_prefetch` 与 `mem_do_rinst` **可以同时为 1**——见文末追问。

---

#### 1. `mem_do_prefetch` — 预取（投机取指）

**含义**：在当前指令译码阶段，**提前发起下一条指令的取指**，不等流水线真正空闲。预取回的数据暂存在 `mem_rdata_q` 中，不立即译码。

**唯一置 1 处**（[picorv32.v:1586](picorv32.v#L1586)）：

```verilog
// cpu_state_fetch, decoder_trigger 分支（非 JAL 指令）
mem_do_rinst <= 0;
mem_do_prefetch <= !instr_jalr && !instr_retirq;
```

**触发条件**：当前指令译码完成（`decoder_trigger`），且不是 `JAL`/`JALR`/`RETIRQ`。`JALR` 和 `RETIRQ` 的目标地址取决于寄存器值，不可预测，投机预取无意义。

**核心作用**：实现单周期指令吞吐的关键机制——ALU 类指令在 `cpu_state_exec` 执行的同时，下一条指令的预取已与内存接口握手。当流水线回到 fetch 时，数据早已锁存好。

---

#### 2. `mem_do_rinst` — 指令读取（需求取指）

**含义**：流水线**现在就需要**一条指令。与 `mem_do_prefetch` 的区别：`mem_do_rinst` 取回的指令会被立即送入译码。

**置 1 的多种场景：**

**场景 A — fetch 状态入口（无预取可复用）：**

| 位置 | 条件 | 说明 |
|---|---|---|
| [1505](picorv32.v#L1505) | `!decoder_trigger && !do_waitirq` | 默认：没有待译码指令，也未等待中断 |
| [1522](picorv32.v#L1522) | `irq_state[0]` | 中断发生，跳转到 trap handler |
| [1566](picorv32.v#L1566) | `instr_waitirq && irq_pending` | WFI 被中断唤醒 |
| [1581](picorv32.v#L1581) | `instr_jal` | JAL 需要改变 PC，直接发起取指 |
| [1970](picorv32.v#L1970) | `!resetn` 后 | 复位后第一条取指 |

**场景 B — 流水线各级回到 fetch → 预取升级为需求取指：**

最常见模式（[1660](picorv32.v#L1660), [1734](picorv32.v#L1734), [1762](picorv32.v#L1762), [1812](picorv32.v#L1812), [1821](picorv32.v#L1821), [1846](picorv32.v#L1846)）：

```verilog
mem_do_rinst <= mem_do_prefetch; // 预取已发起→直接复用；否则发起新取指
```

**场景 C — Load/Store 进入访存时并行取指：**

| 位置 | 状态转换 | 说明 |
|---|---|---|
| [1715](picorv32.v#L1715) | `ld_rs1` → `ldmem` | Load 访存与下一条取指并行 |
| [1752](picorv32.v#L1752) | `ld_rs2` → `stmem` | Store 访存与下一条取指并行 |
| [1784](picorv32.v#L1784) | PCPI 立即完成 | 乘法器单周期完成，同时发起取指 |

**场景 D — 分支跳转目标取指：**

| 位置 | 条件 | 说明 |
|---|---|---|
| [1832](picorv32.v#L1832) | 分支成立 | `set_mem_do_rinst = 1`，清 `decoder_trigger`，发跳转目标取指 |

---

#### 3. `mem_do_rdata` — Load 数据读取

**含义**：执行 `LB`/`LH`/`LW`/`LBU`/`LHU` 时从内存读取数据。

**置 1 路径**（[picorv32.v:1911](picorv32.v#L1911) → [picorv32.v:1971-1972](picorv32.v#L1971-L1972)）：

```verilog
// cpu_state_ldmem
if (!mem_do_prefetch || mem_done) begin  // 等待 mem 空闲
    if (!mem_do_rdata) begin              // 还未发起 load
        // 设定 mem_wordsize: sb→2, sh→1, sw→0
        reg_op1 <= reg_op1 + decoded_imm;
        set_mem_do_rdata = 1;
    end
end
```

**门控条件** `!mem_do_prefetch || mem_done`：Load 必须等待预取完成（`mem_done`），因为内存接口不能同时进行预取和 load。但如果当前没有预取在进行（`!mem_do_prefetch`），可直接发起。

---

#### 4. `mem_do_wdata` — Store 数据写入

**含义**：执行 `SB`/`SH`/`SW` 时向内存写入数据。

**置 1 路径**（[picorv32.v:1883](picorv32.v#L1883) → [picorv32.v:1973-1974](picorv32.v#L1973-L1974)）：

```verilog
// cpu_state_stmem
if (!mem_do_prefetch || mem_done) begin  // 等待 mem 空闲
    if (!mem_do_wdata) begin              // 还未发起 store
        // 设定 mem_wordsize: sb→2, sh→1, sw→0
        reg_op1 <= reg_op1 + decoded_imm;
        set_mem_do_wdata = 1;
    end
end
```

逻辑与 `mem_do_rdata` 完全对称，同样受 `!mem_do_prefetch || mem_done` 门控。

---

#### 整体关系图

```
cpu_state_fetch
  ├─ decoder_trigger?
  │    ├─ instr_jal  → mem_do_rinst = 1
  │    └─ others     → mem_do_prefetch = 1 (if !jalr && !retirq)
  └─ !decoder_trigger → mem_do_rinst = 1 (初始/中断)

流水线各级 (ld_rs1, ld_rs2, exec, shift...)
  ├─ 回到 fetch: mem_do_rinst <= mem_do_prefetch (预取→需求)
  └─ Load/Store:   mem_do_rinst <= 1 (访存与取指并行)

cpu_state_stmem → set_mem_do_wdata = 1 } 均受
cpu_state_ldmem → set_mem_do_rdata = 1  } !mem_do_prefetch||mem_done 门控

mem_done 或 !resetn → 全部清零
```

**核心设计思想**：`mem_do_prefetch` ↔ `mem_do_rinst` 之间是"**低保真预测 → 按需升级**"关系。预取在上一指令译码时就投机发出；流水线回来时若预取仍在进行，直接升级为需求取指；若已完成，数据已在 `mem_rdata_q` 中零延迟消费。Load/Store 则在这套取指流水线之外，需显式等待内存空闲。

---

### 问 2（追问）：`mem_do_prefetch` 和 `mem_do_rinst` 是互斥的吗？为什么断言没有检查这一点？

**答：**

**不是互斥的，两者可以同时为 1。** 断言（[picorv32.v:559-575](picorv32.v#L559-L575)）没有声明 `prefetch` 与 `rinst` 互斥，是**刻意为之**，而非遗漏。

#### 可以同时为 1 的证据

在代码中 `mem_do_rinst <= mem_do_prefetch` 这种赋值出现了 **6 次**（[1660](picorv32.v#L1660), [1734](picorv32.v#L1734), [1762](picorv32.v#L1762), [1812](picorv32.v#L1812), [1821](picorv32.v#L1821), [1846](picorv32.v#L1846)）。以 ALU 指令的快速路径为例（[picorv32.v:1659-1660](picorv32.v#L1659-L1660)）：

```verilog
// LUI/AUIPC/JAL 译码完成，即将进入 exec，且 TWO_CYCLE_ALU=0
mem_do_rinst <= mem_do_prefetch;
```

若上一周期 `mem_do_prefetch` 被置 1（[1586](picorv32.v#L1586)）且预取尚未完成（`mem_done` 未触发），则执行此行后下一周期 `mem_do_rinst = 1, mem_do_prefetch = 1`，两者同时为 1。

#### 断言检查了哪些互斥对

| 断言行 | 检查内容 | 原因 |
|:---:|---|---|
| 561-562 | `prefetch\|rinst\|rdata` → `!wdata` | **读写互斥**：不能同时对总线读写 |
| 564-565 | `prefetch\|rinst` → `!rdata` | **指令/数据互斥**：取指和 load 用同一套状态机，不能并存 |
| 567-568 | `rdata` → `!prefetch && !rinst` | 同上（对称） |
| 570-571 | `wdata` → `!`(所有读信号) | 同上（对称） |

**唯独缺少**：`prefetch → !rinst` 和 `rinst → !prefetch`。

#### 为什么允许同时为 1

`mem_do_prefetch` 和 `mem_do_rinst` 代表的是**同一笔内存读事务的不同"认领状态"**：

```
时序:
  cycle N:   mem_do_prefetch = 1, mem_do_rinst = 0   ← 投机预取已向总线发出
  cycle N+1: mem_do_prefetch = 1, mem_do_rinst = 1   ← 流水线确认需要，两者共存
  cycle N+2: mem_done → 两者同时清零                    ← 事务完成，数据被消费
```

从内存接口的视角看，它们对应的是同一笔读请求——预取阶段已经发出了 `mem_valid` 和 `mem_la_read`，`mem_do_rinst` 置 1 时**不发起新的总线请求**，只是把飞行中的事务从"投机"升级为"需求"：

| 信号 | 决定的事 | 是否发起总线请求 |
|---|---|---|
| `mem_do_prefetch` | 提前发起一次读 | ✅ 是（state 0 → 1，`mem_valid=1`） |
| `mem_do_rinst`（prefetch 已在进行） | 认领飞行中的读结果 | ❌ 否（复用现有请求） |
| `mem_do_rinst`（无 prefetch） | 发起一次需求读 | ✅ 是（state 0 → 1，`mem_valid=1`） |

验证：`mem_busy`（[382](picorv32.v#L382)）是简单 OR ——两者都为 1 时仍然正确；`mem_done`（[384](picorv32.v#L384)）检查的是 `mem_do_rinst` 而非 `mem_do_prefetch` —— 因为 rinst 才是"我要消费这个数据"的信号。

> **一句话：`mem_do_prefetch` 和 `mem_do_rinst` 是同一笔读事务的投机态和确认态，允许叠加。断言只检查真正不能共享硬件资源的维度——读写互斥和指令/数据互斥。**

**涉及文件：**
- [picorv32.v:357-360](picorv32.v#L357-L360) — 四个信号声明
- [picorv32.v:561-571](picorv32.v#L561-L571) — 互斥断言
- [picorv32.v:1505-1589](picorv32.v#L1505-L1589) — `cpu_state_fetch` 中的置位逻辑
- [picorv32.v:1660](picorv32.v#L1660) — `mem_do_rinst <= mem_do_prefetch` 关键赋值
- [picorv32.v:1870-1915](picorv32.v#L1870-L1915) — `stmem` / `ldmem` 访存状态
- [picorv32.v:1962-1974](picorv32.v#L1962-L1974) — 清零与 `set_mem_do_*` 触发

---

## 2026-06-08

### Q8: `mem_rdata_latched` shuffle 的四种情况分别对应什么访存场景？

**问 1：** [picorv32.v:394-396](picorv32.v#L394-L396) 中 `mem_rdata_latched` 赋值的几种情况分别对应哪些场景？比如当前访存地址是 `00` 结尾还是 `10` 结尾，取到的前 16 bit 是单独的指令还是作为 32 bit 指令的一半等。

**问 2（追问）：** 在什么情况下会出现需要取一个 `PC[1:0]=10` 的地址，但是上半个 halfword 又没有预取（即情况 3 `mem_la_firstword`）？

---

#### 答 1：四级优先级的 ternary 链

```verilog
assign mem_rdata_latched =
    COMPRESSED_ISA && mem_la_use_prefetched_high_word ? {16'bx, mem_16bit_buffer} :          // 情况1
    COMPRESSED_ISA && mem_la_secondword ? {mem_rdata_latched_noshuffle[15:0], mem_16bit_buffer} :  // 情况2
    COMPRESSED_ISA && mem_la_firstword ? {16'bx, mem_rdata_latched_noshuffle[31:16]} :        // 情况3
    mem_rdata_latched_noshuffle;                                                               // 情况4
```

**关键控制信号：**

| 信号 | 定义（行号） | 含义 |
|---|---|---|
| `mem_la_firstword` | [366](picorv32.v#L366) | `next_pc[1] && !mem_la_secondword` — PC 指向字的**上半字**（地址以 `10` 结尾） |
| `mem_la_secondword` | [364/614](picorv32.v#L614) | 寄存器，`mem_la_firstword` 之后需要再取一个半字 |
| `mem_la_use_prefetched_high_word` | [378](picorv32.v#L378) | 可复用之前预取缓存的**上半字**，无需再读内存 |
| `mem_16bit_buffer` | [372/616](picorv32.v#L616) | 16-bit 缓冲器，暂存第一次读出的 `mem_rdata[31:16]` |
| `mem_rdata_latched_noshuffle` | [391](picorv32.v#L391) | 传输完成时取 `mem_rdata`，否则保持 `mem_rdata_q` |

**四种情况详解：**

**情况 1** — `mem_la_use_prefetched_high_word`（预取缓存命中）

- 数据：`{16'bx, mem_16bit_buffer}` — 上半字来自缓存，下半字无关
- 场景：上一条 16-bit 压缩指令位于字的下半字，预取时将同字的上半字（`mem_rdata[31:16]`）缓存进 `mem_16bit_buffer`。PC+2 后进入该上半字，**无需发起新的内存读**，`mem_xfer` 通过 [378](picorv32.v#L378) 直接为真

**情况 2** — `mem_la_secondword`（第二半字拼接）

- 数据：`{mem_rdata_latched_noshuffle[15:0], mem_16bit_buffer}` — 低 16 位来自第二次读，高 16 位来自第一次暂存，做半字 swap
- 场景：情况 3 拿到的上半字 `bits[1:0] == 2'b11`，说明该半字是 32-bit 指令的上半部分，需再读一次拼接成完整 32-bit 指令

**情况 3** — `mem_la_firstword`（首次取上半字，无预取）

- 数据：`{16'bx, mem_rdata_latched_noshuffle[31:16]}` — 抽取内存字的高 16 位
- 子场景 A（常见）：`bits[1:0] ≠ 11` → 16-bit 压缩指令，`mem_done` 立即为真
- 子场景 B：`bits[1:0] == 11` → 转入情况 2

**情况 4** — 默认（字对齐标准读取）

- 数据：`mem_rdata_latched_noshuffle` — 原始 32-bit 数据，不做洗牌
- 场景：PC 字对齐（`PC[1:0]=00`），或非 C-ISA 模式。一次读获取完整指令
- 若 `mem_rdata[1:0] ≠ 11`：16-bit 压缩指令在下半字，上半字被预取缓存（[621-623](picorv32.v#L621-L623)）
- 若 `mem_rdata[1:0] == 11`：完整 32-bit 指令

**PC 对齐与 case 映射：**

```
PC[1:0] = 00 (字对齐)                PC[1:0] = 10 (上半字)
─────────────────────                ─────────────────────
                                     无预取缓存 → 情况3: {16'bx, mem_rdata[31:16]}
一次32bit读→ 情况4: mem_rdata              │
    │                                   ├─ bits[1:0]≠11 → 16-bit压缩, 完成
    ├─ bits[1:0]≠11 → 16-bit压缩       │
    │   (上半字→ 预取缓存)               └─ bits[1:0]=11 → 转入情况2
    └─ bits[1:0]=11 → 32-bit指令              │
                                              ▼
有预取缓存 → 情况1: {16'bx, mem_16bit_buffer}   第二次读 → 情况2: {新[15:0], 旧[31:16]}
```

---

#### 答 2：情况 3 出现的两大场景

**预取缓存的建立与销毁：**

- **建立**（[621-623](picorv32.v#L621-L623)）：只有当前取出的下半字是 16-bit 压缩指令（`bits[1:0] ≠ 11`）时，才将同字的上半字缓存
- **销毁**（[1306-1311](picorv32.v#L1306-L1311)）：`latched_branch || irq_state || !resetn` 时 `clear_prefetched_high_word` 拉高，清零缓存

**场景一：分支/跳转到半字对齐的目标地址（最主要）**

```asm
    C.ADDI x10, 5        # PC=0x100 (字对齐, 下半字)
    C.J    target        # PC=0x102 (上半字, 预取命中 → 情况1)
target:
    C.LI  x11, 3         # PC=0x206 (上半字, 但地址完全不同!)
```

执行 `C.J target` 时 `latched_branch=1`，预取缓存被清零。新 PC = `0x206`，`PC[1]=1` 但 `prefetched_high_word=0`，**必须走情况 3**。

**场景二：中断/复位**

`irq_state` 或 `!resetn` 同样清零预取缓存。若 trap handler 入口地址的 bit 1 为 1，则走情况 3。

**为什么顺序执行不会出现情况 3：**

| 当前指令 | PC 变化 | 预取缓存？ | 下一条走哪个 case |
|---|---|---|---|
| 16-bit 压缩 @ 下半字 | PC+2 → 上半字 | ✅ `prefetched_high_word=1` | Case 1（预取命中） |
| 16-bit 压缩 @ 上半字 | PC+2 → 下一字的下半字 | ❌ | Case 4（字对齐） |
| 32-bit @ 字对齐 | PC+4 → 下一字的下半字 | ❌（`bits[1:0]=11` 时显式清缓存 [625](picorv32.v#L625)） | Case 4（字对齐） |

顺序执行中，上半字要么有预取缓存（Case 1），要么 PC 直接跳到下一字的对齐位置（Case 4）。**情况 3 本质上处理的是控制流变更后，PC 恰好落在某个字上半字的场景。**

**涉及文件：**
- [picorv32.v:394-396](picorv32.v#L394-L396) — `mem_rdata_latched` 四级 ternary 赋值
- [picorv32.v:366](picorv32.v#L366) — `mem_la_firstword` 定义
- [picorv32.v:378](picorv32.v#L378) — `mem_la_use_prefetched_high_word` 定义
- [picorv32.v:611-629](picorv32.v#L611-L629) — `mem_xfer` 后的状态转移
- [picorv32.v:1306-1312](picorv32.v#L1306-L1312) — `clear_prefetched_high_word` 逻辑

---

### Q7: RV32 指令集中由 opcode 最低两位决定的四个象限分别表示什么含义？

**问 1：** RV32 指令集中由 opcode 最低两位决定的四个象限分别表示什么含义？

**问 2（追问）：** C.SLLI 为什么是栈指针/寄存器类操作？（此前回答将 C.SLLI 归入"栈指针/寄存器操作"不准确）

---

#### 答 1：四个象限的含义

RISC-V 变长指令编码的核心规则——**只需检查指令的最低两位即可确定指令长度**：

| `instr[1:0]` | 象限 | 含义 |
|:---:|:---:|---|
| **00** | Quadrant 0 | **16 位压缩指令** (RVC)，`funct3[15:13]` 进一步细分 |
| **01** | Quadrant 1 | **16 位压缩指令** (RVC) |
| **10** | Quadrant 2 | **16 位压缩指令** (RVC) |
| **11** | Quadrant 3 | **≥32 位标准指令**（RV32 中即 32 位指令） |

**核心规则：**
- `instr[1:0] ≠ 11`（`00`/`01`/`10`）→ **16 位压缩指令**，按象限和 `funct3` 进一步解码
- `instr[1:0] = 11` → **至少 32 位**的指令；在 RV64/RV128 中还需检查 `bits[4:2]` 判断是否 >32 位

**PicoRV32 中的实现**（[picorv32.v:904-913](picorv32.v#L904-L913)）：

```verilog
if (COMPRESSED_ISA && mem_rdata_latched[1:0] != 2'b11) begin
    compressed_instr <= 1;
    ...
    case (mem_rdata_latched[1:0])
        2'b00: begin // Quadrant 0 → C.ADDI4SPN, C.LW, C.SW 等
        2'b01: begin // Quadrant 1 → C.ADDI, C.JAL, C.LI, C.LUI 等
        2'b10: begin // Quadrant 2 → C.SLLI, C.LWSP, C.JR, C.ADD 等
    endcase
end
```

#### 答 2：C.SLLI 的分类纠正

C.SLLI 归入 Quadrant 2 并**不是**因为它属于"栈指针/寄存器操作"——这个说法是错的。C.SLLI 是**移位指令**，与栈指针无关。

**象限是按编码格式（encoding format）划分的，不是按操作语义划分的。** 同一象限内的指令可能在语义上完全不相关。以 Quadrant 2（`bits[1:0]=10`）为例：

| `funct3[15:13]` | 指令 | 编码格式 | 语义类别 |
|:---:|---|---|---|
| `000` | **C.SLLI** | CI（立即数） | **移位** |
| `010` | C.LWSP | CI | 整数加载 (SP 相对) |
| `100` | C.JR / C.MV / C.ADD | CR（寄存器间） | 寄存器操作 |
| `110` | C.SWSP | CSS | 整数存储 (SP 相对) |

Quadrant 2 实际混搭了三种格式：CI（加载/移位立即数）、CR（纯寄存器操作）、CSS（SP 相对存储）。C.SLLI 之所以在 Q2，单纯因为它的 CI 格式编码恰好落在了 `bits[1:0]=10` 这片编码空间，与功能无关。

**正确的象限划分逻辑：**

| 象限 | bits[1:0] | 寄存器寻址方式 |
|:---:|:---:|---|
| Q0 | `00` | 使用 **3-bit 压缩寄存器**（映射到 x8-x15），主要是访存指令 |
| Q1 | `01` | 使用 **5-bit 完整寄存器**，主要是立即数算术和控制转移 |
| Q2 | `10` | 混合 CI/CR/CSS 三种格式，移位+SP加载+SP存储+寄存器操作 |
| Q3 | `11` | 32 位标准指令（非压缩） |

**象限的本质是编码空间的边界划分，不是语义分组。**

**涉及文件：**
- [picorv32.v:903-913](picorv32.v#L903-L913) — 压缩指令 detection 与 quadrant 分发
- [picorv32.v:447-549](picorv32.v#L447-L549) — C-ISA 解压缩块（按象限 case 展开）

---

## 2026-06-07

### Q6: picorv32 是否实现了 mulh/mul 微架构融合？编译器如何优化？

**问 1：** 在 picorv32 和 PCPI 的 RTL 中，是否实现了将两条相邻的源寄存器相同的 `mulh`/`mulhsu`/`mulhu` 指令和 `mul` 指令进行微架构融合，使得实际只需要做一次乘法来实现？

**问 2（追问）：** 编译器怎么在软件层面优化两条相邻的 `mulh` + `mul`？

---

#### 答 1：硬件层面 — 没有实现融合

经过对全部相关 RTL 代码的审查（主核心 `picorv32.v`、`picorv32_pcpi_mul`、`picorv32_pcpi_fast_mul`），**未发现任何指令融合逻辑**。

**审查证据：**

**1. 主核心（picorv32.v）— 无融合检测**

PCPI 乘法指令的执行路径（[picorv32.v:1598-1626](picorv32.v#L1598-L1626)、[picorv32.v:1781-1798](picorv32.v#L1781-L1798)）：

- 遇到 `MUL` 类指令 → 进入 `instr_trap` 状态
- 读取 `reg_op1 ← cpuregs_rs1`，`reg_op2 ← cpuregs_rs2`
- 置 `pcpi_valid = 1`，将 `pcpi_insn` 发给乘法器
- 等待 `pcpi_int_ready`，捕获 `pcpi_int_rd` 写回
- **无任何**：跟踪上一条指令类型、比较相邻指令源寄存器、跳过第二次乘法请求的逻辑

搜索 `fuse`、`fusion`、`reuse`、`redundant` 等关键字 → 零匹配。

**2. 两个乘法器模块 — 均无融合**

| 模块 | 位置 | 行为 |
|---|---|---|
| `picorv32_pcpi_mul`（顺序） | [picorv32.v:2210-2329](picorv32.v#L2210-L2329) | 每指令一次完整迭代，无历史记录，无缓存 |
| `picorv32_pcpi_fast_mul`（快速） | [picorv32.v:2331-2426](picorv32.v#L2331-L2426) | 单周期 `*` 运算，无历史记录，无缓存 |

**为什么不需要硬件融合：**

1. picorv32 是**面积优先**的微型核心，硬件融合需要比较器、指令历史寄存器、额外 MUX，与设计哲学矛盾
2. `fast_mul` 已做到 1 周期/乘法，两条只需 2 周期，融合收益仅 1 周期
3. 编译器侧已有部分优化能力（见下文）

**如需实现融合的改动要点：**
- 主核心增加"上一条 PCPI 指令"寄存器（指令码 + rs1 + rs2）
- 第二条 MUL 指令时比较 rs1/rs2，相同时跳过 `pcpi_valid`，复用 64 位乘积
- PCPI 接口需暴露完整 64 位乘积（当前 `pcpi_rd` 仅 32 位）

---

#### 答 2：编译器层面 — 四种优化手段

根本限制：**RISC-V RV32M 标准扩展没有"一次乘法同时返回高、低 32 位"的指令**。编译器无法在 ISA 层面真正融合两条指令，但可从以下角度减少开销：

**① 死代码消除（DCE）——最直接有效**

```asm
# 如果程序只用乘积的低 32 位，编译器自动消除 mulh
# 优化前（-O0）
mulh  t0, a0, a1    # 高32位 → def-use 链断裂，可删除
mul   a0, a0, a1

# 优化后（-O2）
mul   a0, a0, a1    # 只保留 mul
```

**② 值域分析 → 用移位替代 `mulh`**

当编译器通过 VRP（Value Range Propagation）证明操作数位宽足够窄、乘积不溢出 32 位时，`mulh` 退化为符号/零扩展，用移位即可：

```asm
# int16_t * int16_t → 乘积在 [-2^30, 2^30)，必不溢出
mul   a0, a0, a1      # 低 32 位
srai  a1, a0, 31      # 高 32 位 = 全是符号位，无需真正的 mulh
```

**③ 指令调度 — 隐藏延迟**

对顺序乘法器（mulh ~64 周期），在两条乘法间插入独立指令：

```asm
mulh  a1, a0, a1    # 发出，需 ~64 周期
lw    t0, 0(sp)     # 独立 load，不依赖乘法结果
addi  sp, sp, 16    # 独立 ALU
mul   a0, a0, a1    # 借此隐藏部分等待周期
```

**④ 自定义扩展 — 唯一真正"一次乘法"的方案**

修改硬件让 PCPI 保留完整 64 位乘积并通过 CSR 暴露，配合内联汇编：

```c
static inline uint64_t mul64(int32_t a, int32_t b) {
    uint32_t lo, hi;
    asm volatile (
        "mul   %0, %2, %3\n\t"
        "csrr  %1, 0xFC0"    // 读自定义 CSR（上次乘法的高 32 位）
        : "=r"(lo), "=r"(hi) : "r"(a), "r"(b));
    return ((uint64_t)hi << 32) | lo;
}
```

**优化效果对比：**

| 优化方法 | 效果 | 依赖条件 |
|---|---|---|
| DCE（死代码消除） | 消除无用指令 | 编译器 def-use 分析 |
| 值域分析 | 用移位替代 `mulh` | 编译器能证明操作数位宽足够窄 |
| 指令调度 | 隐藏延迟 | 指令间存在独立性 |
| 自定义扩展 | 真正减少为 1 次乘法 | **需修改硬件** |

其中 DCE 和值域分析是编译器自动完成的，对用户透明。要实现真正的"融合"（一次乘法得两半），必须修改硬件。

**涉及文件：**
- [picorv32.v:1598-1626](picorv32.v#L1598-L1626) — PCPI 指令发射路径
- [picorv32.v:1781-1798](picorv32.v#L1781-L1798) — `cpu_state_ld_rs2` PCPI 路径
- [picorv32.v:2210-2329](picorv32.v#L2210-L2329) — `picorv32_pcpi_mul` 顺序乘法器
- [picorv32.v:2331-2426](picorv32.v#L2331-L2426) — `picorv32_pcpi_fast_mul` 快速乘法器

---

### Q5: `mem_state` 状态编码及每种状态的访存含义

**问：** `mem_state` 包含哪几种状态，分别表示什么访存含义？

**答：**

`mem_state` 是 picorv32 内存接口的**状态机寄存器**，2 位宽，共 4 种状态，控制所有对外内存访问（取指、预取、load、store）的总线握手流程。

**状态编码**（[picorv32.v:351-352](picorv32.v#L351-L352)）：

| 编码 | 名称 | 访存含义 |
|---|---|---|
| `2'b00` | **idle** | 空闲，无进行中的内存事务 |
| `2'b01` | **read** | 读请求已发出，等待总线 `mem_ready` 响应 |
| `2'b10` | **write** | 写请求已发出，等待总线 `mem_ready` 响应 |
| `2'b11` | **prefetch done** | 预取读已完成，数据已锁存，等待指令流水线消费 |

---

### 状态转换详解（[picorv32.v:594-648](picorv32.v#L594-L648)）

#### State 0 — idle（空闲）

当前无总线事务。根据 CPU 流水线需求，发起不同类型的请求：

| 触发条件 | 动作 | 目标状态 |
|---|---|---|
| `mem_do_prefetch \|\| mem_do_rinst \|\| mem_do_rdata` | 发起读请求：置 `mem_valid=1`，`mem_instr=1`（取指）或 `0`（load），`mem_wstrb=0` | **State 1** read |
| `mem_do_wdata` | 发起写请求：置 `mem_valid=1`，`mem_instr=0` | **State 2** write |

> 注：C-ISA 预取复用（`mem_la_use_prefetched_high_word`）时，即使发起读请求也不置 `mem_valid`（无需实际访问总线），但仍进入 state 1。

---

#### State 1 — read（读等待）

读请求（取指/预取/load）已发送到总线，等待 `mem_ready`。

**当 `mem_xfer` 完成时**，分三种转移路径：

| 条件 | 动作 | 目标状态 | 场景说明 |
|---|---|---|---|
| `COMPRESSED_ISA && mem_la_read`（需再读第二字） | 保持 `mem_valid=1`，置 `mem_la_secondword=1`，保存高半字到 `mem_16bit_buffer` | **State 1** | C-ISA 指令跨字边界，需要发起第二次读以获取指令后半部分 |
| `!mem_do_rdata && !mem_do_rinst`（即 `mem_do_prefetch`） | 撤销 `mem_valid`，保存半字缓存信息 | **State 3** prefetch done | 预取完成，数据已锁存，等待 `mem_do_rinst` 消费 |
| `mem_do_rinst \|\| mem_do_rdata` | 撤销 `mem_valid` | **State 0** idle | 普通取指或 load 完成，数据已就绪 |

---

#### State 2 — write（写等待）

写请求（store）已发送到总线，等待 `mem_ready`。

| 条件 | 动作 | 目标状态 |
|---|---|---|
| `mem_xfer` 完成 | 撤销 `mem_valid` | **State 0** idle |

写操作只有一种终点——传输完成即回到 idle。

---

#### State 3 — prefetch done（预取完成）

**这是一个特殊状态：总线事务已经完成，但还没人消费数据。**

预取（prefetch）在读回数据后不是直接进入译码，而是先停留在 state 3，把数据"暂存"在 `mem_rdata_q` 中。当下游流水线准备好取指时，发出 `mem_do_rinst`。

| 条件 | 动作 | 目标状态 | 说明 |
|---|---|---|---|
| `mem_do_rinst` 到来 | — | **State 0** idle | 指令流水线接收预取数据，无需再次访问总线 |

> **状态 3 不需要 `mem_xfer`**，因为它代表"数据已就绪"而非"等待传输"。这也是 `mem_done` 中 Term 2（`&mem_state && mem_do_rinst`）的设计基础——见 [Q3](#q3-mem_done-信号的含义及置位条件分析)。

---

### 状态机图示

```
                    ┌──────────────────────────────────────────────┐
                    │ C-ISA 跨字边界，需再读第二字                     │
                    │ (保持 state 1)                                │
                    ▼                                               │
    ┌────┐  读请求  ┌──────┐  mem_xfer && !data && !rinst   ┌──────────┐
    │    │─────────▶│      │───────────────────────────────▶│          │
    │  0 │          │  1   │                                │    3     │
    │idle│◀─────────│ read │                                │ prefetch  │
    │    │ rinst/   │      │                                │   done   │
    └────┘  rdata   └──────┘                                └────┬─────┘
      ▲     完成        ▲                                        │
      │                 │               mem_do_rinst             │
      │   写请求  ┌──────┐               (数据被消费)              │
      │──────────▶│      │◀───────────────────────────────────────┘
      │           │  2   │
      │◀──────────│ write│
         mem_xfer └──────┘
            完成
```

**涉及文件：**
- [picorv32.v:351-352](picorv32.v#L351-L352) — `mem_state` 声明及编码注释
- [picorv32.v:594-648](picorv32.v#L594-L648) — 状态机完整实现
- [picorv32.v:384](picorv32.v#L384) — `mem_done` 中对 `&mem_state` 的使用

---

### Q4: `mem_rdata_q` 与 `next_insn_opcode` 的区别及各自用途

**问：** [picorv32.v:443-444](picorv32.v#L443-L444) 中，`mem_rdata_q` 和 `next_insn_opcode` 的区别是什么？分别用于哪些地方？

**答：**

**赋值（完全相同）：**

```verilog
// picorv32.v:443-444
mem_rdata_q      <= COMPRESSED_ISA ? mem_rdata_latched : mem_rdata;
next_insn_opcode <= COMPRESSED_ISA ? mem_rdata_latched : mem_rdata;
```

两者在 `mem_xfer` 时被赋予**完全相同的值**——来自内存总线的原始取指数据。但之后它们的命运截然不同：

---

### 核心区别：一个被"加工"，一个保持"原样"

| 维度 | `mem_rdata_q` | `next_insn_opcode` |
|---|---|---|
| **是否被 C-ISA 解压缩修改** | ✅ 是 — 在 [picorv32.v:447-549](picorv32.v#L447-L549) 被大量覆写 | ❌ 否 — 始终保持总线原始值 |
| **本质** | **解压缩后的 32 位等效指令**（C-ISA 16→32 位展开后的结果） | **总线上读回的原始比特**（对 C-ISA 就是 16 位压缩指令本身） |
| **用途定位** | 供 CPU 执行流水线使用 | 供缓存和调试使用 |

---

### `mem_rdata_q` — 被 C-ISA 解压缩逻辑"加工"

在 `mem_xfer` 拿到原始数据后，如果 `COMPRESSED_ISA && mem_done && (mem_do_prefetch || mem_do_rinst)`，C-ISA 解压缩块（[picorv32.v:447-549](picorv32.v#L447-L549)）会根据压缩指令的 quadrant 和 funct3 字段，将 `mem_rdata_q` 的部分比特**覆写**为对应的 32 位等效指令格式。例如：

- **C.ADDI4SPN** → 覆写 `mem_rdata_q[14:12]` 和 `mem_rdata_q[31:20]`（[picorv32.v:452-453](picorv32.v#L452-L453)）
- **C.LW** → 覆写 `mem_rdata_q[31:20]` 和 `mem_rdata_q[14:12]`（[picorv32.v:456-457](picorv32.v#L456-L457)）
- **C.JAL** → 重组 `mem_rdata_q` 的多个位段（[picorv32.v:506-513](picorv32.v#L506-L513)）

**用途：驱动整个指令执行流水线**

| 使用位置 | 用途 |
|---|---|
| [picorv32.v:1050](picorv32.v#L1050) | `pcpi_insn <= mem_rdata_q` — 送给 PCPI 协处理器 |
| [picorv32.v:1052-1089](picorv32.v#L1052-L1089) | `instr_xxx <= ... mem_rdata_q[14:12] ...` — 解码 funct3/ funct7 确定具体指令 |
| [picorv32.v:1091-1105](picorv32.v#L1091-L1105) | `instr_rdcycle / instr_ecall_ebreak / instr_fence / ...` — CSR/特权指令解码 |
| [picorv32.v:1136-1142](picorv32.v#L1136-L1142) | `decoded_imm <= mem_rdata_q[...]` — 提取立即数字段 |

**设计意图**：通过将 C-ISA 16 位指令展开为标准 32 位格式存入 `mem_rdata_q`，后续的译码逻辑可以**统一按 32 位指令处理**，无需区分压缩/非压缩指令。

---

### `next_insn_opcode` — 保持总线原始值不变

**永远不会被 C-ISA 解压缩逻辑修改**，始终保存从内存读回的原始比特。

**用途：指令缓存和调试输出**

| 使用位置 | 用途 |
|---|---|
| [picorv32.v:807-810](picorv32.v#L807-L810) | `cached_insn_opcode <= next_insn_opcode` — 缓存原始指令码，用于指令重复检测（同一 PC 重复执行时直接复用缓存结果） |
| [picorv32.v:839-842](picorv32.v#L839-L842) | `dbg_insn_opcode = next_insn_opcode` — 调试接口输出原始指令码 |

**为什么缓存/调试需要原始值？**
- **缓存匹配**：缓存命中检测需要比对的是"总线上读到的原始指令"，而非解压缩后的等效 32 位指令。两条不同的 C-ISA 指令可能解压缩后恰好相同（罕见但理论上可能），只有比对原始值才能准确判断是否命中。
- **调试透明度**：调试器（如 VCD trace、GTKWave 波形）通常期望看到真实的指令编码，而非内部解压缩后的表示。

---

### 数据流对比

```
mem_rdata (总线)
      │
      ▼ mem_xfer 时采样
┌─────────────────────────────────────┐
│  相同的初始值                        │
├─────────────────┬───────────────────┤
│ mem_rdata_q     │ next_insn_opcode  │
│      ↓          │      ↓            │
│ C-ISA 解压缩    │ (保持不变)         │
│ (覆写部分位段)   │      ↓            │
│      ↓          │ cached_insn_opcode │
│ instr_xxx 解码  │ dbg_insn_opcode   │
│ decoded_imm     │                   │
│ pcpi_insn       │                   │
│                 │                   │
│ → 执行流水线    │ → 缓存 + 调试      │
└─────────────────┴───────────────────┘
```

**涉及文件：**
- [picorv32.v:179](picorv32.v#L179) — `next_insn_opcode` 寄存器声明
- [picorv32.v:355](picorv32.v#L355) — `mem_rdata_q` 寄存器声明
- [picorv32.v:443-444](picorv32.v#L443-L444) — 两者的赋值（完全相同）
- [picorv32.v:447-549](picorv32.v#L447-L549) — C-ISA 解压缩块（仅修改 `mem_rdata_q`）
- [picorv32.v:807-810](picorv32.v#L807-L810) — `next_insn_opcode` 用于缓存
- [picorv32.v:839-842](picorv32.v#L839-L842) — `next_insn_opcode` 用于调试输出
- [picorv32.v:1050-1142](picorv32.v#L1050-L1142) — `mem_rdata_q` 驱动指令解码

---

### Q3: `mem_done` 信号的含义及置位条件分析

**问：** [picorv32.v:384](picorv32.v#L384) 中 `mem_done` 信号的含义是什么，置位的条件分别对应什么情况？

**答：**

**`mem_done` 的含义：**

`mem_done` 表示**当前内存操作已完全结束，结果数据有效且可供流水线后续阶段使用**。它是连接内存接口状态机与 CPU 流水线（译码、执行、写回）的关键握手信号。

其使用场景覆盖：指令译码触发（[picorv32.v:878](picorv32.v#L878)、[picorv32.v:1458](picorv32.v#L1458)）、分支跳转（[picorv32.v:1827](picorv32.v#L1827)）、store/load 状态推进（[picorv32.v:1869](picorv32.v#L1869)、[picorv32.v:1894](picorv32.v#L1894)）、C-ISA 指令解压缩（[picorv32.v:447](picorv32.v#L447)）、以及各类状态复位（[picorv32.v:1961](picorv32.v#L1961)）。

**条件拆解：**

```verilog
wire mem_done = resetn
    && ((mem_xfer && |mem_state && (mem_do_rinst || mem_do_rdata || mem_do_wdata)) || (&mem_state && mem_do_rinst))
    && (!mem_la_firstword || (~&mem_rdata_latched[1:0] && mem_xfer));
```

整个表达式由三部分组成（AND 关系），缺一不可：

---

### 第一部分：`resetn` — 全局复位门控

复位有效时 `mem_done` 强制为 0，所有内存操作被抑制。

---

### 第二部分（核心）：两种完成场景（OR 关系）

```
( mem_xfer && |mem_state && (mem_do_rinst || mem_do_rdata || mem_do_wdata) )
||
( &mem_state && mem_do_rinst )
```

#### 场景 A：总线传输完成（Term 1）

> `mem_xfer && |mem_state && (mem_do_rinst || mem_do_rdata || mem_do_wdata)`

| 子条件 | 含义 |
|---|---|
| `mem_xfer` | 当前周期有一次数据传输完成（valid/ready 握手成功，或 C-ISA 预取复用） |
| `\|mem_state` | `mem_state != 0`，即不在 idle 状态（处于 state 1/2/3） |
| `mem_do_rinst` | 指令读取（取指） |
| `mem_do_rdata` | 数据读取（load） |
| `mem_do_wdata` | 数据写入（store） |

此条件覆盖三种子情况：

| 子情况 | mem_state | 触发条件 | 说明 |
|---|---|---|---|
| **取指完成** | 1 → 0 | `mem_xfer && mem_do_rinst` | 指令从总线读回，数据已锁存到 `mem_rdata_q`，可送入译码阶段 |
| **Load 完成** | 1 → 0 | `mem_xfer && mem_do_rdata` | load 数据从总线读回，`mem_rdata_word` 有效，可写回寄存器 |
| **Store 完成** | 2 → 0 | `mem_xfer && mem_do_wdata` | 数据已写入总线，store 操作结束 |

#### 场景 B：预取数据就绪（Term 2）

> `&mem_state && mem_do_rinst`

| 子条件 | 含义 |
|---|---|
| `&mem_state` | `mem_state == 2'b11`，即 state 3（prefetch done） |
| `mem_do_rinst` | 指令读取请求已发起 |

当 CPU 处于 state 3 时，表示之前发起的预取（prefetch）已经完成，读回的数据已锁存在 `mem_rdata_q` 中。此时如果流水线发起 `mem_do_rinst` 来消费这条预取的指令，数据**立即可用**，无需等待总线握手，`mem_done` 直接置位。

> **为什么 Term 2 不需要 `mem_xfer`？** 因为预取阶段（state 1 → 3 的转换）已经通过 `mem_xfer` 将数据锁存到了 `mem_rdata_q`。State 3 本身就是一个"数据已就绪，等待消费"的状态，`mem_do_rinst` 的到达即意味着完成。

---

### 第三部分：C-ISA 指令跨字对齐约束

```
!mem_la_firstword || (~&mem_rdata_latched[1:0] && mem_xfer)
```

这是针对压缩指令集（C-ISA）的附加条件。C-ISA 指令为 16 位，可能跨越 32 位字的边界。

| 条件 | 情况 | mem_done |
|---|---|---|
| `!mem_la_firstword` | 非 C-ISA 普通取指，或已是第二次取指（`mem_la_secondword`） | ✅ 通过 |
| `mem_la_firstword && mem_rdata_latched[1:0] != 2'b11 && mem_xfer` | C-ISA 首次取指，指令未跨字边界（低 2 位不全为 1） | ✅ 通过 |
| `mem_la_firstword && mem_rdata_latched[1:0] == 2'b11` | C-ISA 首次取指，指令跨字边界 — **需要再取一个字** | ❌ 阻塞 |

`~&mem_rdata_latched[1:0]` 等价于 `mem_rdata_latched[1:0] != 2'b11`。当 `mem_rdata_latched[1:0] == 2'b11` 时，说明 PC 指向了 32 位对齐字中最高 16 位的位置（quadrant 3），该 16 位指令的高半部分跨越到了下一个 32 位字，核心需要发起 `mem_la_secondword` 取回后半部分。此时 `mem_done` 被抑制，状态机继续等待第二次传输。

---

### 设计意图总结

`mem_done` 的精髓在于区分了**两种"完成"的语义**：

1. **总线传输完成**（Term 1）— 数据刚在总线上完成握手，新鲜到达
2. **逻辑就绪**（Term 2）— 数据早已到达（预取），流水线现在才来取

同时通过第三部分确保在 C-ISA 跨字场景下不会过早宣布完成，保证取指的正确性。

**涉及文件：**
- [picorv32.v:384-385](picorv32.v#L384-L385) — `mem_done` 定义
- [picorv32.v:380](picorv32.v#L380) — `mem_xfer` 定义
- [picorv32.v:349-360](picorv32.v#L349-L360) — `mem_state` 编码及相关寄存器
- [picorv32.v:363-369](picorv32.v#L363-L369) — `mem_la_firstword` / `mem_la_secondword` C-ISA 相关逻辑
- [picorv32.v:600-648](picorv32.v#L600-L648) — 内存状态机转换

---

### Q2: `mem_xfer` 的含义及 `mem_rdata_latched_noshuffle` 的数据选择逻辑

**问：** `mem_xfer` 表示的是访存的哪种情况，为什么在 [picorv32.v:392](picorv32.v#L392) 中 `mem_xfer` 有效就直接使用 `mem_rdata`，无效的话就用 `mem_rdata_q`？

**答：**

**`mem_xfer` 的含义**（[picorv32.v:380](picorv32.v#L380)）：

```verilog
assign mem_xfer = (mem_valid && mem_ready) || (mem_la_use_prefetched_high_word && mem_do_rinst);
```

`mem_xfer` 表示**当前周期有一次内存数据传输完成**，涵盖两种情况：

| 情况 | 条件 | 说明 |
|---|---|---|
| 正常握手完成 | `mem_valid && mem_ready` | 标准的 valid/ready 握手，CPU 发起请求且总线响应，数据在当前周期有效 |
| C-ISA 预取复用 | `mem_la_use_prefetched_high_word && mem_do_rinst` | 压缩指令取指时，所需的高半字已在之前的预取中获取，无需再次访问总线即可"完成"传输 |

**为什么 `mem_xfer` 有效用 `mem_rdata`，无效用 `mem_rdata_q`**（[picorv32.v:392](picorv32.v#L392)）：

```verilog
assign mem_rdata_latched_noshuffle = (mem_xfer || LATCHED_MEM_RDATA) ? mem_rdata : mem_rdata_q;
```

核心原因在于 **数据的有效期不同**：

- **`mem_rdata`**（内存总线输入）：仅在 `mem_xfer=1` 的周期内有效。这是组合逻辑信号，来自外部内存总线，只在握手完成的那一拍携带有效数据，下一拍可能变化。

- **`mem_rdata_q`**（锁存寄存器，[picorv32.v:355](picorv32.v#L355)）：在每次 `mem_xfer=1` 时由 `mem_rdata` 采样得到（[picorv32.v:443](picorv32.v#L443)），在下一次 `mem_xfer` 之前保持稳定。

- **`LATCHED_MEM_RDATA`**（参数，[picorv32.v:67](picorv32.v#L67)）：当设为 `1` 时，表示外部内存接口本身就会锁存读数据（类似 AXI 的 read data channel 会保持有效直到被接受），此时即使没有新的 transfer，`mem_rdata` 也是有效的，可以直接使用。

因此这个 MUX 的逻辑是：

> **如果当前周期正好有数据传输完成（或外部接口保证数据持续有效），就用总线上的实时数据 `mem_rdata`；否则，使用上一次传输时锁存到 `mem_rdata_q` 的历史数据。**

这是处理器设计中典型的**数据有效窗口管理**模式——总线数据只在握手瞬间有效，而 CPU 流水线可能需要在后续多个周期中使用该数据（例如取指后的译码阶段、load 指令的结果写回等），因此必须有一个寄存器将瞬时的总线数据"保持住"，供后续周期使用。

**涉及信号汇总：**

| 信号 | 方向 | 说明 |
|---|---|---|
| `mem_rdata` | input (总线) | 内存读数据总线，仅 `mem_xfer` 周期有效 |
| `mem_rdata_q` | reg (内部) | 锁存的读数据，在 `mem_xfer` 时更新，跨周期保持 |
| `mem_xfer` | wire | 传输完成标志，=1 表示当前周期有数据从总线到达 |
| `LATCHED_MEM_RDATA` | parameter | 外部接口数据持续有效标志 |
| `mem_valid` | output | CPU 向总线发起请求 |
| `mem_ready` | input | 总线响应请求 |

**涉及文件：**
- [picorv32.v:380](picorv32.v#L380) — `mem_xfer` 定义
- [picorv32.v:392](picorv32.v#L392) — `mem_rdata_latched_noshuffle` 的 MUX 逻辑
- [picorv32.v:441-444](picorv32.v#L441-L444) — `mem_rdata_q` 的更新逻辑

---

### Q1: 建立 Q&A 文档机制与项目配置

**问：** 在本目录和对话下，将后面问的问题和回答进行总结后写入 `ai_doc` 目录下的 qa markdown 文档中。文档书写语言规则：除了引用原文、专业术语、特殊缩写以及英语或其他语言比中文表达更简洁清楚的情况，或者明确要求使用其他语言的情况之外，都使用中文书写。将这条规则写入本项目下的 agent 项目配置中。

**答：** 已完成以下操作：

1. **创建 `ai_doc/qa.md`**（即本文件）：用于记录后续所有 Q&A 交互的总结文档，按时间倒序组织。
2. **创建 `CLAUDE.md`**：作为项目级 agent 配置文件，写入以下持久化规则：
   - 所有 `ai_doc/` 目录下的文档默认使用中文书写（专业术语、特殊缩写、引用原文等除外）
   - 每次问答结束后，自动将问答总结追加到 `ai_doc/qa.md` 中

**涉及文件：**
- [ai_doc/qa.md](ai_doc/qa.md) — Q&A 记录文档
- [CLAUDE.md](CLAUDE.md) — 项目 agent 配置
