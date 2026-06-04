# Repo Intake: picorv32

- Analysis date: 2026-05-29
- Repository path: /home/hanszhang/code/picorv32

## 仓库概览

本仓库是 PicoRV32 RISC-V 软核的 RTL、仿真 testbench、测试固件、指令级测试、示例 SoC 与若干综合/形式验证脚本集合。若只有 `picorv32.v` 这一份 RTL，要做有意义的仿真测试，还需要 testbench、程序镜像生成链、指令测试源、仿真器/交叉编译工具链等配套文件。

## 仓库目录结构

```text
picorv32/
|-- picorv32.v              # PicoRV32 RTL，含 native/AXI/Wishbone wrapper 与 PCPI 乘除法模块
|-- testbench.v             # 标准 AXI4-Lite 仿真 testbench，默认读取 firmware/firmware.hex
|-- testbench_wb.v          # Wishbone 接口版本 testbench
|-- testbench_ez.v          # 极简 testbench，内嵌小程序，不依赖外部 hex
|-- testbench.cc            # Verilator C++ 驱动
|-- Makefile                # 编译固件、生成 hex、运行 Icarus/Verilator/Yosys 测试入口
|-- showtrace.py            # 将 trace 输出辅助反汇编/分析
|-- firmware/               # 标准测试固件源码、链接脚本、hex 生成脚本及生成物
|-- tests/                  # 来自 riscv-tests 风格的指令级汇编测试
|-- dhrystone/              # Dhrystone benchmark 固件和 testbench
|-- picosoc/                # PicoRV32 示例 SoC、SPI flash/UART/板级文件
|-- scripts/                # 综合、形式验证、torture/csmith 等扩展测试脚本
`-- ai_doc/                 # AI 生成的仓库分析文档
```

## 项目设计结构

RTL 顶层候选来自 `picorv32.v`：`picorv32` 是 native memory interface CPU；`picorv32_axi` 是 AXI4-Lite master 版本；`picorv32_wb` 是 Wishbone master 版本；另有 `picorv32_axi_adapter`、`picorv32_pcpi_mul`、`picorv32_pcpi_fast_mul`、`picorv32_pcpi_div`。

标准仿真路径使用 `testbench.v` 中的 `picorv32_wrapper` 实例化 `picorv32_axi`，再连接 `axi4_memory` 模型。内存模型用 `$readmemh` 读取 `firmware/firmware.hex`，把地址 `0x1000_0000` 当字符输出口，把地址 `0x2000_0000` 写入 `123456789` 当作测试通过标志。

## 技术栈

- RTL/验证语言：Verilog，少量 Verilator C++ harness。
- 仿真工具：Icarus Verilog (`iverilog`/`vvp`)；可选 Verilator。
- 固件工具链：RISC-V bare-metal GCC/binutils，默认 `TOOLCHAIN_PREFIX ?= /opt/riscv_ultimate/bin/riscv64-unknown-elf-`。
- 辅助脚本：Python 3，用于 bin-to-hex 和 trace 处理。
- 可选形式/综合：Yosys、yosys-smtbmc、Vivado、Quartus、IceStorm 等。

## 如何运行

README 和 Makefile 明确给出：

- `make test`：编译 `testbench.v + picorv32.v`，生成标准固件 hex 后运行。
- `make test_wb`：运行 Wishbone testbench。
- `make test_ez`：运行极简测试，不需要 RISC-V 交叉编译器。
- `make test_verilator`：使用 Verilator 编译并运行。
- `make check`：使用 Yosys SMTBMC 对 `picorv32.v` 做基础形式检查。

## 如何测试

只存在 RTL 时，最小冒烟测试可补充 `testbench_ez.v` 和 Icarus Verilog，运行 `make test_ez` 或等价命令 `iverilog -o testbench_ez.vvp testbench_ez.v picorv32.v && vvp -N testbench_ez.vvp`。它内嵌几条 RV32I 指令和小 RAM，能验证取指、读写存储、基本循环，但覆盖很浅。

完整标准测试需要 `testbench.v`、`Makefile`、`firmware/`、`tests/`、Python 3、RISC-V 交叉编译器和 Icarus Verilog。流程是把固件 C/汇编和 `tests/*.S` 编成 `firmware/firmware.elf`，再转成 `firmware/firmware.bin` 和 `firmware/firmware.hex`，最后由 testbench 加载并仿真，看到 `ALL TESTS PASSED.` 为通过。

## 主要入口

- RTL 入口：`picorv32.v` 中的 `picorv32`、`picorv32_axi`、`picorv32_wb`。
- 标准 testbench 入口：`testbench.v` 的 `testbench` / `picorv32_wrapper`。
- 极简 testbench 入口：`testbench_ez.v` 的 `testbench`。
- Wishbone testbench 入口：`testbench_wb.v`。
- 固件入口：`firmware/start.S`，链接脚本为 `firmware/sections.lds`。
- 构建入口：顶层 `Makefile`。

## 配置与环境变量

- `TOOLCHAIN_PREFIX`：RISC-V 交叉工具链前缀，可在 make 命令行覆盖。
- `COMPRESSED_ISA`：默认 `C`，Makefile 用它决定是否传入 `-DCOMPRESSED_ISA` 以及固件 `-march`。
- `IVERILOG`、`VVP`、`VERILATOR`、`PYTHON`：仿真和脚本工具命令，可覆盖。
- plusargs：`+vcd` 生成波形，`+trace` 生成执行 trace，`+axi_test` 打开随机 AXI 延迟/异步行为测试，`+firmware=<hex>` 指定 hex 文件。

## 开发工作流

常见流程：

1. 修改 `picorv32.v`。
2. 先跑 `make test_ez` 做最小冒烟。
3. 再跑 `make test` 覆盖标准 AXI testbench、固件、指令测试、IRQ、乘除法等。
4. 若改到 Wishbone 接口，跑 `make test_wb`。
5. 若改到综合敏感逻辑，可跑 `make check` 或 `scripts/` 下对应综合/形式验证流。

## 风险与疑问

- 标准测试依赖外部 RISC-V 工具链；若只有 RTL 和普通 Verilog 仿真器，只能跑 `testbench_ez.v` 这类浅层测试。
- 仓库里有一些生成物如 `.vvp`、`.o`、`.elf`、`.hex`、`.map`，不是源文件；重建时应以源码、Makefile 和工具链为准。
- `testbench.v` 的 RVFI 路径引用 `rvfimon.v`，当前文件列表未见该文件；普通 `make test` 不需要它，`test_rvf` 需要额外补齐。

## 建议下一步

1. 如果目标是移植到只有 RTL 的环境，先保留 `testbench_ez.v` 作为最小自检模板。
2. 若要接近原仓库覆盖率，补齐 `Makefile`、`testbench.v`、`firmware/`、`tests/` 和 RISC-V 交叉工具链。
3. 将生成物和源文件区分开：`firmware.hex` 可作为预编译输入保留，但可重复构建时更推荐保留源码和工具链。
4. 对 AXI/Wishbone/native 三种接口分别建立 testbench，不要只测其中一种 wrapper。
