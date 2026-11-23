# NCore8 Instruction Set Architecture (ISA)

**Status:** ISA encoding frozen; peripheral register details and several implementation conventions remain to be specified.

## Architecture

- Parameterized `NUM_CORES` (1–256), independent 8-bit cores.
- Four 8-bit general-purpose registers per core: `R0`–`R3`.
- 8-bit PC and 8-bit fixed-length instructions; 256-byte instruction address space.
- Instruction format: `[7:4]` opcode, `[3:0]` payload.
- No arithmetic flags; arithmetic wraps modulo 256.
- `SLT` compares signed two's-complement values.
- Shared 8-bit data address space with arbitrated memory operations; separate instruction memory.
- Memory-mapped peripherals and per-core interrupt state.
- Core IDs are 0 through `NUM_CORES-1`.

## Main opcode table

| Opcode | Mnemonic | Semantics | Payload |
|---|---|---|---|
| `0` | ADD | `rd = rd + rs` | `rd[3:2], rs[1:0]` |
| `1` | SUB | `rd = rd - rs` | same |
| `2` | OR | `rd = rd \| rs` | same |
| `3` | XOR | `rd = rd ^ rs` | same |
| `4` | AND | `rd = rd & rs` | same |
| `5` | SLT | `rd = ($signed(rd) < $signed(rs)) ? 1 : 0` | same |
| `6` | LOADI | `R0 = zero_extend(imm4)` | `imm4` |
| `7` | LUI | `R0[7:4] = imm4` (low nibble unchanged) | `imm4` |
| `8` | SHLI | `rd = rd << (imm2 + 1)` | `rd[3:2], imm2[1:0]` |
| `9` | SHRI | Right shift R1/R2 by 1–4, logical or arithmetic | see below |
| `A` | LOAD | `rd = data_mem[rs]` | `rd[3:2], rs[1:0]` |
| `B` | STORE | `data_mem[rs] = rd` | `rd[3:2], rs[1:0]` |
| `C` | BEZ | If `rs == 0`, `PC = PC + 2 + imm2`; else `PC = PC + 1` | `rs[3:2], imm2[1:0]` |
| `D` | MOV | `rd = rs` | `rd[3:2], rs[1:0]` |
| `E` | ADDI | `rd += step` | `rd[3:2], step_code[1:0]` |
| `F` | SYSTEM | See below | `subopcode[3:0]` |

All PC arithmetic wraps modulo 256. For non-control-flow instructions, the PC advances by one.

### Immediate and shift encoding

- `ADDI` step codes: `00 = +1`, `01 = +2`, `10 = -1`, `11 = -2`. No zero step; `OR r,r` is available as a NOP.
- `SHLI`: all four registers supported; shift count is `imm2 + 1` (1–4).
- `SHRI`: payload bit 3 selects `R1` (0) or `R2` (1); bits `[2:1]` encode shift count minus 1; bit 0 selects logical (0) or arithmetic (1) right shift. Arithmetic right shift sign-extends bit 7.
- `BEZ`: skips 1–4 instructions when taken (relative to the following instruction).

## Reclaimed self-register encodings

For `rd == rs`, these originally redundant instructions are reassigned. For each group, register index `r = 0,1,2,3` corresponds to payloads `0,5,A,F`.

| Bytes | New mnemonic | Semantics |
|---|---|---|
| `00,05,0A,0F` | NOT r | `r = ~r` |
| `10,15,1A,1F` | NEG r | `r = -r` modulo 256 |
| `40,45,4A,4F` | ROL r | `r = {r[6:0],r[7]}` |
| `50,55,5A,5F` | ROR r | `r = {r[0],r[7:1]}` |
| `D0,D5,DA,DF` | SWAPN r | `r = {r[3:0],r[7:4]}` |

`XOR r,r` still clears a register; `OR r,r` is a NOP. `ADD r,r` is replaceable with `SHLI r,1` in this flagless architecture.

## SYSTEM opcode (`F`)

| Byte | Mnemonic | Semantics |
|---|---|---|
| `F0`–`F3` | LINK rd | `rd = PC + 1` |
| `F4`–`F7` | JMP rs | `PC = rs` |
| `F8` | HALT | Stop this core until reset |
| `F9` | CID | `R3 = core_id` |
| `FA` | SLEEP | Pause this core; resume at next instruction after wake |
| `FB` | WAKE | Wake sleeping core with ID in full 8-bit `R0` |
| `FC` | XCHG | Atomically exchange `R0` with `data_mem[R3]` |
| `FD` | RETI | Return from interrupt using saved return PC |
| `FE` | SENDIRQ | Set pending interrupt on core selected by `R0` |
| `FF` | FADD | Atomically add `R1` to `data_mem[R3]`, return old memory value in `R1` |

- All cores start running after reset.
- HALT is terminal until reset; WAKE only resumes a sleeping core.
- WAKE/SENDIRQ targeting a nonexistent core should be treated as a no-op (recommended implementation convention).
- XCHG and FADD are indivisible memory-controller transactions.
- FADD: `old = mem[R3]; mem[R3] = (old + R1) & 0xFF; R1 = old`.
- No nested interrupts in the initial design. A pending interrupt is serviced at an instruction boundary when enabled; SENDIRQ remains pending if disabled.

## Interrupt architecture

Interrupt configuration and status are exposed through **per-core banked MMIO**, meaning each core accesses its own interrupt controller registers at the same addresses. Hardware interrupt entry saves the next-to-execute PC, disables further interrupt acceptance, and transfers control to the configured vector. `RETI` restores the saved PC and re-enables interrupts. Handlers must preserve general-purpose registers they modify. External sources may include GPIO, timer, UART, and inter-core SENDIRQ.

Conceptual interrupt registers: `IRQ_ENABLE`, `IRQ_PENDING`, `IRQ_VECTOR`, `IRQ_RETURN`, `IRQ_ACK`, plus optional active/source/mask fields. Their detailed register-level behavior is not yet frozen.

## Memory map

The 8-bit **data** address space is distinct from the 256-byte instruction memory.

| Address range | Allocation | Size |
|---|---|---|
| `00`–`DF` | Shared RAM | 224 bytes |
| `E0`–`E7` | Per-core interrupt controller (banked) | 8 bytes |
| `E8`–`EB` | GPIO | 4 bytes |
| `EC`–`EF` | SPI | 4 bytes |
| `F0`–`F3` | Timer/PWM | 4 bytes |
| `F4`–`F7` | I²C | 4 bytes |
| `F8`–`FB` | UART | 4 bytes |
| `FC`–`FF` | Multicore control | 4 bytes |

These regions are **address allocations**, not claims that the corresponding peripherals are implemented. Protocol support requires appropriate controllers, timing, electrical interfaces, and firmware. Detailed peripheral register maps remain to be designed.

## Verification targets

- Exhaustive unary tests: 256 input values per register per unary operation.
- Exhaustive binary ALU tests: 256 × 256 = 65,536 operand pairs per operation, with all register combinations where appropriate.
- Directed tests for branch targets and wraparound, register dependencies, memory addressing, interrupts, sleep/wake/halt, and invalid core IDs.
- N-core concurrency tests for arbitration, XCHG, FADD, and inter-core interrupt delivery.
- Independent architectural reference model and self-checking SystemVerilog scoreboard.

## Remaining implementation-level choices

ISA opcodes and major semantics are fixed. Before implementing peripherals, specify precise MMIO register addresses and bitfields, interrupt pending/acknowledgment rules, reset values, instruction fetch contention, and how external events wake sleeping cores. A single shared memory port should define ordering for both normal and atomic accesses.
