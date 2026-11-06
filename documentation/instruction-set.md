# Custom 32-bit Instruction Set Architecture (ISA)

## 1. Overview and Source Comparison

This document reconstructs the custom 32-bit Instruction Set Architecture (ISA) by cross-referencing the author's handwritten ISA specifications (Sources B and C) against the actual hardware implementation inside the `INST_DECODER` (`2480,1290`) and `main` subcircuits of [`../circuit/CPU_Harvard_Arc.circ`](../circuit/CPU_Harvard_Arc.circ) (Source A).

Every discrepancies or hardware wiring quirk discovered during netlist analysis is explicitly flagged so that software authors and hardware reviewers can distinguish intended ISA behavior from current circuit behavior.

---

## 2. Verified 32-bit Instruction Formats

In `INST_DECODER`, the 32-bit input pin `DATA_IN[31:0]` (`710,1800`) enters an 8-way Splitter (`760,1800`) that partitions the 32-bit word into eight 4-bit hexadecimal nibbles (Nibbles 0 through 7, from MSB to LSB):

| Bit Range | Width | Nibble Index (Splitter `760,1800`) | Primary Architectural Role | Decoder / Datapath Destination in `INST_DECODER` |
| :---: | :---: | :---: | :--- | :--- |
| **`[31:28]`** | 4 bits | Branch `0` | **Primary Opcode** | Selects 1-of-16 output on Primary Opcode Decoder `(1110,1770)` |
| **`[27:24]`** | 4 bits | Branch `1` | **Destination Register (`Rd`)** or **Extended Subopcode 1** | Selects `Rd` (`R0`–`R15`) on Decoder `(1110,2190)`; also drives Extended Opcode `0x0` Decoder `(1110,1150)` |
| **`[23:20]`** | 4 bits | Branch `2` | **Source Register (`Rs`)**, **Extended Subopcode 2**, or **24-bit Target `[23:20]`** | Selects `Rs` (`R0`–`R15`) on Decoder `(1120,2520)`, I/O Decoder `(1110,440)`, `CFG_REG` Decoder `(1110,720)`, and `RJMP_JMP_L_ADD[3:0]` |
| **`[19:16]`** | 4 bits | Branch `3` | **Operation Type / Subtype (`MULS`, `SRC`, `SLC`)**, **System Subopcode**, or **JMP High `[7:4]`** | Selects ALU subtype on Decoder `(1120,2770)`, System Op on Decoder `(1110,190)`, and `RJMP_JMP_L_ADD[7:4]` |
| **`[15:12]`** | 4 bits | Branch `4` | **Immediate / Address `[15:12]`** | Routed via Splitter `(890,1890)` to `DATA_OUT[15:12]` and `ADD_OUT[15:12]` |
| **`[11:8]`** | 4 bits | Branch `5` | **Immediate / Address `[11:8]`** | Routed via Splitter `(890,1890)` to `DATA_OUT[11:8]` and `ADD_OUT[11:8]` |
| **`[7:4]`** | 4 bits | Branch `6` | **Immediate / Address `[7:4]`** *(Note: bit 6 omitted in Splitter `760,1800`)* | Routed via Splitter `(890,1890)` to `DATA_OUT[7:4]` and `ADD_OUT[7:4]` |
| **`[3:0]`** | 4 bits | Branch `7` | **Immediate / Address `[3:0]`** | Routed via Splitter `(890,1890)` to `DATA_OUT[3:0]` and `ADD_OUT[3:0]` |

### Format Variants by Instruction Class

```text
1. Register-Register ALU & Subtype Format (ADD, SUB, MUL, MULS, CMP, OR, AND, XOR, SR, SRC, SL, SLC, MOV):
+---------------+---------------+---------------+---------------+-------------------------------+
| 31         28 | 27         24 | 23         20 | 19         16 | 15                          0 |
+---------------+---------------+---------------+---------------+-------------------------------+
|  Primary Op   |      Rd       |      Rs       |  Type/Subtype |        Unused (0x0000)        |
|   (4 bits)    |   (4 bits)    |   (4 bits)    |   (4 bits)    |           (16 bits)           |
+---------------+---------------+---------------+---------------+-------------------------------+
  * Type [19:16] = 0x0 for standard ops; 0xC for SRC; 0xE for SLC; 0xF for MULS.

2. Immediate & Memory Format (LDI, LDS, STS):
+---------------+---------------+---------------+---------------+-------------------------------+
| 31         28 | 27         24 | 23         20 | 19         16 | 15                          0 |
+---------------+---------------+---------------+---------------+-------------------------------+
|  Primary Op   |  Rd (LDI/LDS) |  Rs (for STS) |    Unused     |   16-bit Immediate / Address  |
|  (0xB..0xD)   |  or Unused    |  or Unused    |    (0x0)      |     (DATA_OUT / ADD_OUT)      |
+---------------+---------------+---------------+---------------+-------------------------------+

3. Unconditional Jump Format (JMP, Primary Op = 0xF):
+---------------+---------------+---------------+---------------+-------------------------------+
| 31         28 | 27         24 | 23         20 | 19         16 | 15                          0 |
+---------------+---------------+---------------+---------------+-------------------------------+
|     1111      |    Unused     |  JMP_H[3:0]   |  JMP_H[7:4]   |     16-bit Target Address     |
|    (0xF)      |    (0x0)      |   (4 bits)    |   (4 bits)    |           (ADD_OUT)           |
+---------------+---------------+---------------+---------------+-------------------------------+

4. Call / Conditional Branch / Return Format (CALL = 0xE, or Primary Op = 0x0 with SubOp1 = 0x1..0xD):
+---------------+---------------+---------------+---------------+-------------------------------+
| 31         28 | 27         24 | 23         20 | 19         16 | 15                          0 |
+---------------+---------------+---------------+---------------+-------------------------------+
|  0xE or 0x0   |  0x0 or Cond  |  Target[23:20]|    Unused     |     Target Address [15:0]     |
|   (4 bits)    |   (4 bits)    |   (4 bits)    |    (0x0)      |           (16 bits)           |
+---------------+---------------+---------------+---------------+-------------------------------+

5. System / Port / RJMP Extended Format (Primary Op = 0x0, SubOp1 [27:24] = 0xE):
+---------------+---------------+---------------+---------------+-------------------------------+
| 31         28 | 27         24 | 23         20 | 19         16 | 15                          0 |
+---------------+---------------+---------------+---------------+-------------------------------+
|     0000      |     1110      |  SubOp2 (0..2)|  SubOp3 (0..4)|     16-bit Port Imm Data      |
|    (0x0)      |    (0xE)      |   (4 bits)    |   (4 bits)    |           (16 bits)           |
+---------------+---------------+---------------+---------------+-------------------------------+
```

---

## 3. Primary Opcode Inventory (`[31:28] = 0x0` to `0xF`)

The table below compares the handwritten ISA specification against the verified decoding logic in `INST_DECODER`.

| Primary Opcode `[31:28]` | Subtype `[19:16]` | Mnemonic | Handwritten Specification | Verified Circuit Implementation (`CPU_Harvard_Arc.circ`) | Verification Status |
| :---: | :---: | :--- | :--- | :--- | :--- |
| `0001` (`0x1`) | `0000` (`0x0`) | **`ADD`** | Add `Rd` and `Rs`; result in `Rd` | Asserts `ADD` -> `ALU.SUM`. Computes `Rd + Rs -> Rd`. Latches flags. | **Verified in Circuit** |
| `0010` (`0x2`) | `0000` (`0x0`) | **`SUB`** | Subtract `Rd` and `Rs`; result in `Rd` | Asserts `SUB` -> `ALU.SUB`. Computes `Rd - Rs -> Rd` (`Rd + ~Rs + 1`). Latches flags. | **Verified in Circuit** |
| `0011` (`0x3`) | `0000` (`0x0`) | **`MUL`** | Multiply register operands; result associated with `R0` and `R1` | Asserts `MUL`. Multiplies `Rd * Rs` (unsigned 16x16->32). Low 16b -> `Rd` (or `R0` if `Rd=0`); High 16b (`MUL_OUT_H`) -> forced into `R1` (`4710,1760`) and `R0` (`4500,1760` tri-state conflict). | **Verified (with R0/R1 hardware note)** |
| `0011` (`0x3`) | `1111` (`0xF`) | **`MULS`** | Special multiplication variant | Asserts `MUL` (`[31:28]=0x3`) and `MULS` (`[19:16]=0xF`). In `ALU`, `MULS` drives `SUM` (`ADD`) instead of `MUL`, while `MUL_MULS` still forces `MUL_OUT_H` into `R1`/`R0`. | **Discrepancy Documented** |
| `0100` (`0x4`) | `0000` (`0x0`) | **`CMP`** | Compare operands | Asserts `CMP`. Computes `Rd - Rs` in subtractor; sets `CMP_F = 1` if `Rd == Rs`. Also writes `Rd - Rs` back to `Rd` because `CMP` is wired into the `ALU_EN` write-back OR gate. | **Verified (non-destructive intent vs. write-back quirk)** |
| `0101` (`0x5`) | `0000` (`0x0`) | **`OR`** | Bitwise OR; result in `Rd` | Asserts `O_R` -> `ALU.O_R`. Computes `Rd \| Rs -> Rd`. | **Verified in Circuit** |
| `0110` (`0x6`) | `0000` (`0x0`) | **`AND`** | Bitwise AND; result in `Rd` | Asserts `A_ND` -> `ALU.A_ND`. Computes `Rd & Rs -> Rd`. | **Verified in Circuit** |
| `0111` (`0x7`) | `0000` (`0x0`) | **`XOR`** | Bitwise XOR; result in `Rd` | Asserts `X_OR` -> `ALU.X_OR`. Computes `Rd ^ Rs -> Rd`. | **Verified in Circuit** |
| `1000` (`0x8`) | `0000` (`0x0`) | **`SR`** | Shift right | Asserts `SR` -> `ALU.R_SFT`. Shifts `Rd` right by 1 bit (`{CARRY_F, Rd[15:1]}`). `Rd[0]` becomes new `CARRY_F` if `SRC` is active. | **Verified in Circuit** |
| `1000` (`0x8`) | `1100` (`0xC`) | **`SRC`** | Shift right with carry-related behavior | Asserts `SR` (`[31:28]=0x8`) + `SRC` (`[19:16]=0xC`). Asserts `SFT_W_CRY`, enabling `Rd[0]` to update `CARRY_F`. | **Verified in Circuit** |
| `1001` (`0x9`) | `0000` (`0x0`) | **`SL`** | Shift left | Asserts `SL` -> `ALU.L_SFT`. Shifts `Rd` left by 1 bit (`{Rd[14:0], CARRY_F}`). *(Note: In `main`, `SL` and `SR` are swapped at Splitter `2980,1650`, and `SL` is missing from Stage 3 `RD` decoder OR gate).* | **Discrepancy Documented** |
| `1001` (`0x9`) | `1110` (`0xE`) | **`SLC`** | Shift left with carry-related behavior | Asserts `SL` (`[31:28]=0x9`) + `SLC` (`[19:16]=0xE`). In `main`, `SLC` is ORed into `ALU.SUB` rather than `SFT_W_CRY`. | **Discrepancy Documented** |
| `1010` (`0xA`) | `0000` (`0x0`) | **`MOV`** | Copy `Rs` to `Rd` | Asserts `MOV`. Enables source register `Rs` onto the 16-bit write-back bus and latches into `Rd`. Does not alter ALU flags. | **Verified in Circuit** |
| `1011` (`0xB`) | `0000` (`0x0`) | **`LDS`** | Load from memory location into `Rd` | Asserts `LDS`. Routes 16-bit `ADD_OUT` (`[15:0]`) to `DATA_RAM` address; reads 16-bit word from `DATA_RAM` into `Rd`. | **Verified in Circuit** |
| `1100` (`0xC`) | `0000` (`0x0`) | **`STS`** | Store register data to memory location | Asserts `STS`. Routes `Rs` (`[23:20]`) onto write-back bus and into `DATA_RAM` data input at address `ADD_OUT` (`[15:0]`). | **Verified in Circuit** |
| `1101` (`0xD`) | `0000` (`0x0`) | **`LDI`** | Load immediate value into `Rd` | Asserts `LDI`. Routes 16-bit immediate `DATA_OUT` (`[15:0]`) onto write-back bus and latches into `Rd`. | **Verified in Circuit** |
| `1110` (`0xE`) | `0000` (`0x0`) | **`CALL`** | *(Handwritten notes list `1110` as `RJMP`; circuit implements `1110` as `CALL`)* | Decoder `(1110,1770)` output `14` (`0xE`) is labeled **`CALL`** (`1890,1890`). Loads 24-bit target into PC and flushes IF1/IF2. | **Discrepancy Documented (`CALL` vs. `RJMP`)** |
| `1111` (`0xF`) | `xxxx` | **`JMP`** | Jump | Decoder `(1110,1770)` output `15` (`0xF`) is labeled **`JMP`** (`1890,1910`). Loads 24-bit target (`JMP_H` + `ADD_OUT`) into PC and flushes IF1/IF2. | **Verified in Circuit** |

---

## 4. Extended Opcode `0000` (`[31:28] = 0x0`) Inventory

When the primary opcode `[31:28]` is `0000`, Decoder `(1110,1770)` output `0` enables the secondary Decoder `(1110,1150)` which decodes bits `[27:24]`. These instructions are present in `INST_DECODER` and wired into `main`:

| `[31:28]` | `[27:24]` | `[23:20]` | `[19:16]` | Decoder Pin Label | Condition Checked in `INST_DECODER` | Architectural Operation in Circuit |
| :---: | :---: | :---: | :---: | :--- | :--- | :--- |
| `0000` | `0000` (`0x0`) | `0000` | `0000` | **`NOP`** | Unconditional | No operation (`0x00000000`); no registers, memory, or flags modified. |
| `0000` | `0001` (`0x1`) | Target Hi | `0000` | **`CLC`** | `FLAGS_IN[7] == 1` | Conditional Call (wired to bit 7 of `FLAGS_IN`; intended Carry). |
| `0000` | `0010` (`0x2`) | Target Hi | `0000` | **`CLNC`** | `FLAGS_IN[7] == 0` | Conditional Call on Not Carry. |
| `0000` | `0011` (`0x3`) | Target Hi | `0000` | **`CLZ`** | `FLAGS_IN[3] == 1` | Conditional Call (wired to bit 3 of `FLAGS_IN`; intended Zero). |
| `0000` | `0100` (`0x4`) | Target Hi | `0000` | **`CNZ`** | `FLAGS_IN[3] == 0` | Conditional Call on Not Zero. |
| `0000` | `0101` (`0x5`) | Target Hi | `0000` | **`BRE`** | `FLAGS_IN[3] == 1` | Conditional Branch if Equal. |
| `0000` | `0110` (`0x6`) | Target Hi | `0000` | **`BRNE`** | `FLAGS_IN[3] == 0` | Conditional Branch if Not Equal. |
| `0000` | `0111` (`0x7`) | Target Hi | `0000` | **`BRC`** | `FLAGS_IN[7] == 1` | Conditional Branch if Carry. |
| `0000` | `1000` (`0x8`) | Target Hi | `0000` | **`BRNC`** | `FLAGS_IN[7] == 0` | Conditional Branch if Not Carry. |
| `0000` | `1001` (`0x9`) | Target Hi | `0000` | **`RET`** | Unconditional | Return (loads 24-bit `CALL_RET_ADD` into PC and flushes pipeline). |
| `0000` | `1010` (`0xA`) | Target Hi | `0000` | **`RTC`** | `FLAGS_IN[7] == 1` | Conditional Return if Carry. |
| `0000` | `1011` (`0xB`) | Target Hi | `0000` | **`RTNC`** | `FLAGS_IN[7] == 0` | Conditional Return if Not Carry. |
| `0000` | `1100` (`0xC`) | Target Hi | `0000` | **`RTZ`** | `FLAGS_IN[3] == 1` | Conditional Return if Zero. |
| `0000` | `1101` (`0xD`) | Target Hi | `0000` | **`RTNZ`** | `FLAGS_IN[3] == 0` | Conditional Return if Not Zero. |
| `0000` | `1110` (`0xE`) | `0..2` | `0000` | **`PORT_A_IN_OUT`** | SubOp2 in `0..2` | Output pin on `INST_DECODER` (not connected inside `main` Stage 3). |
| `0000` | `1110` (`0xE`) | `0..2` | `0001` | **`PORT_B_IN_OUT`** | SubOp2 in `0..2` | Output pin on `INST_DECODER` (not connected inside `main` Stage 3). |
| `0000` | `1110` (`0xE`) | `0..2` | `0010` | **`HLT`** | SubOp2 in `0..2` | Halts processor by asserting `CTR_STP1` (`1290,1160`), switching `CLK` MUX to `0`. |
| `0000` | `1110` (`0xE`) | `0..2` | `0011` | **`RST`** | SubOp2 in `0..2` | Software Reset: asserts `RST_MAIN` and `DFR_RST`, clearing PC and all registers. |
| `0000` | `1110` (`0xE`) | `0..2` | `0100` | **`RJMP`** | SubOp2 in `0..2` | Relative/Page Jump: replaces low 8 bits of PC with `JMP_H` and flushes pipeline. |
| `0000` | `1111` (`0xF`) | `0..7` | `xxxx` | **`CFG`** | Unconditional | Writes 16-bit immediate `DATA_OUT` (or `Rd` if `CFG_RD=1`) into special register `CFG_REG[Rs]`. |

---

## 5. Complete Instruction Reference Table

| Mnemonic | Primary Opcode `[31:28]` | Subtype | Instruction Format | Operands | Architectural Operation | Destination | Flags Affected | Memory Effects | Control Flow | Verified Encoding Example (Hex) |
| :--- | :---: | :---: | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :---: |
| **`ADD`** | `0001` | `0000` | Reg-Reg | `Rd, Rs` | `Rd <- Rd + Rs` | `Rd` (`R0`–`R15`) | `CARRY_F`, `NEG_F`, `V_F`, `ZERO_F`, `H_F` (`CMP_F=0`) | None | Sequential (`PC+1`) | `0x10100000` (`ADD R0, R1`) |
| **`SUB`** | `0010` | `0000` | Reg-Reg | `Rd, Rs` | `Rd <- Rd - Rs` (`Rd + ~Rs + 1`) | `Rd` (`R0`–`R15`) | `NEG_F`, `V_F`, `ZERO_F`, `H_F` (`CARRY_F=0`, `CMP_F=0`) | None | Sequential (`PC+1`) | `0x22300000` (`SUB R2, R3`) |
| **`MUL`** | `0011` | `0000` | Reg-Reg | `Rd, Rs` | `Product[31:0] <- Rd * Rs` (unsigned) | Low 16b -> `Rd`; High 16b -> `R1` (and `R0`) | `NEG_F`, `ZERO_F` (`CARRY_F=0`) | None | Sequential (`PC+1`) | `0x30100000` (`MUL R0, R1`) |
| **`MULS`** | `0011` | `1111` | Reg-Reg | `Rd, Rs` | Intended signed multiply; in circuit `MULS` triggers `SUM` + `MUL_OUT_H` write | `Rd`, `R1`, `R0` | Same as `ADD` | None | Sequential (`PC+1`) | `0x301F0000` (`MULS R0, R1`) |
| **`CMP`** | `0100` | `0000` | Reg-Reg | `Rd, Rs` | Compares `Rd` and `Rs` (`Rd - Rs`); sets `CMP_F=1` if equal (also writes `Rd - Rs` to `Rd` in current circuit) | `FLAGS` (and `Rd` in circuit) | `CMP_F`, `NEG_F`, `ZERO_F` | None | Sequential (`PC+1`) | `0x40100000` (`CMP R0, R1`) |
| **`OR`** | `0101` | `0000` | Reg-Reg | `Rd, Rs` | `Rd <- Rd \| Rs` | `Rd` (`R0`–`R15`) | `NEG_F`, `ZERO_F` | None | Sequential (`PC+1`) | `0x54500000` (`OR R4, R5`) |
| **`AND`** | `0110` | `0000` | Reg-Reg | `Rd, Rs` | `Rd <- Rd & Rs` | `Rd` (`R0`–`R15`) | `NEG_F`, `ZERO_F` | None | Sequential (`PC+1`) | `0x64500000` (`AND R4, R5`) |
| **`XOR`** | `0111` | `0000` | Reg-Reg | `Rd, Rs` | `Rd <- Rd ^ Rs` | `Rd` (`R0`–`R15`) | `NEG_F`, `ZERO_F` | None | Sequential (`PC+1`) | `0x74500000` (`XOR R4, R5`) |
| **`SR`** | `1000` | `0000` | Reg-Shift | `Rd` | Shift right by 1 bit (`{CARRY_F, Rd[15:1]}`) *(see splitter swap note)* | `Rd` (`R0`–`R15`) | `NEG_F`, `ZERO_F` | None | Sequential (`PC+1`) | `0x82000000` (`SR R2`) |
| **`SRC`** | `1000` | `1100` | Reg-Shift | `Rd` | Shift right by 1 bit with carry update (`CARRY_F <- Rd[0]`) | `Rd` (`R0`–`R15`) | `CARRY_F`, `NEG_F`, `ZERO_F` | None | Sequential (`PC+1`) | `0x820C0000` (`SRC R2`) |
| **`SL`** | `1001` | `0000` | Reg-Shift | `Rd` | Shift left by 1 bit (`{Rd[14:0], CARRY_F}`) *(see splitter swap & RD note)* | `Rd` (`R0`–`R15`) | `NEG_F`, `ZERO_F` | None | Sequential (`PC+1`) | `0x92000000` (`SL R2`) |
| **`SLC`** | `1001` | `1110` | Reg-Shift | `Rd` | Shift left by 1 bit with carry update (`CARRY_F <- Rd[15]`) | `Rd` (`R0`–`R15`) | `CARRY_F`, `NEG_F`, `ZERO_F` | None | Sequential (`PC+1`) | `0x920E0000` (`SLC R2`) |
| **`MOV`** | `1010` | `0000` | Reg-Reg | `Rd, Rs` | `Rd <- Rs` | `Rd` (`R0`–`R15`) | None (`FLAG_WR=0`) | None | Sequential (`PC+1`) | `0xA2000000` (`MOV R2, R0`) |
| **`LDS`** | `1011` | `0000` | Memory | `Rd, Addr16` | `Rd <- DATA_RAM[Addr16]` | `Rd` (`R0`–`R15`) | None (`FLAG_WR=0`) | Reads `DATA_RAM[Addr16]` | Sequential (`PC+1`) | `0xB2000001` (`LDS R2, 0x0001`) |
| **`STS`** | `1100` | `0000` | Memory | `Rs, Addr16` | `DATA_RAM[Addr16] <- Rs` | `DATA_RAM[Addr16]` | None (`FLAG_WR=0`) | Writes `DATA_RAM[Addr16]` | Sequential (`PC+1`) | `0xC0000001` (`STS R0, 0x0001`) |
| **`LDI`** | `1101` | `0000` | Immediate | `Rd, Imm16` | `Rd <- Imm16` | `Rd` (`R0`–`R15`) | None (`FLAG_WR=0`) | None | Sequential (`PC+1`) | `0xD0000001` (`LDI R0, 0x0001`) |
| **`CALL`** | `1110` | `0000` | Control | `Addr24` | `PC <- CALL_RET_ADD` (flushes IF1/IF2) | `PC` | None | None | Unconditional Call | `0xE0000001` (`CALL 0x000001`) |
| **`JMP`** | `1111` | `xxxx` | Control | `Addr24` | `PC <- {bit_rev8(JMP_H), bit_rev16(ADD_OUT)}` (flushes IF1/IF2) | `PC` | None | None | Unconditional Jump | `0xF0000001` (`JMP 0x000001`) |
| **`RJMP`** | `0000` | `E04` | Control | `Offset8` | `PC <- {PC[23:8], JMP_H[7:0]}` (in `INST_DECODER`; `RJMP` tunnel in `main` is stubbed) | `PC` | None | None | Relative/Page Jump | `0x0E040000` (`RJMP`) |
| **`NOP`** | `0000` | `000` | System | None | No operation | None | None | None | Sequential (`PC+1`) | `0x00000000` (`NOP`) |
| **`HLT`** | `0000` | `E02` | System | None | Stops processor clock (`CTR_STP1 = 1`) | Clock MUX | None | None | Halts execution | `0x0E020000` (`HLT`) |
| **`RST`** | `0000` | `E03` | System | None | Resets PC, pipeline, and all registers (`RST_MAIN = 1`) | All Regs & PC | Clears `FLAGS` | None | Resets to `0x000000` | `0x0E030000` (`RST`) |

---

## 6. Important Implementation Details & Hardware Discrepancies

In accordance with the project's strict source-handling requirements, the following six implementation nuances discovered in `CPU_Harvard_Arc.circ` must be accounted for when writing machine code or analyzing circuit behavior:

1. **Splitter `(760,1800)` Bit 6 Omission in `INST_DECODER`**:
   - In `INST_DECODER`, the 32-bit input Splitter `(760,1800)` assigns bits `4`, `5`, and `7` to branch `6` (`bit4="6"`, `bit5="6"`, `bit7="6"`), but `bit6` is unassigned (`none`).
   - **Impact**: Bit 6 of any 16-bit immediate (`DATA_OUT`) or 16-bit memory address (`ADD_OUT`) is floating (`U`) after leaving `INST_DECODER`. In `main`, Pull Resistors `(3910,1440)` on `DATA_OUT` and `(6420,1310)` on the `DATA_RAM` address bus pull floating lines to `0`, meaning bit 6 (`0x0040`) of immediate constants and memory addresses always resolves to `0` unless `bit6="6"` is restored on Splitter `(760,1800)`.

2. **`STS` Operand Field Position (`[23:20]` vs. `[27:24]`)**:
   - In `INST_DECODER`, the `STS` signal (`0xC`) is wired into the 9-input OR gate `(1450,1830)` that enables the **source register decoder `RS`** (`1120,2520`, decoding bits `[23:20]`), and is **not** wired into the 14-input OR gate `(1570,1830)` that enables the `RD` decoder (`1110,2190`, decoding bits `[27:24]`).
   - Furthermore, in `main` Stage 3 (`3780,2070`), `STS` gates the `RS` bus into the Stage 4 register `(3980,1970)` that enables the register output buffers onto the `DATA_RAM` write bus.
   - **Impact**: To store register `Rn` into memory, the register index `n` must be placed in bits **`[23:20]`** (`Rs` nibble), not `[27:24]`. Note that in the sample `PRG_ROM` program (`c0100001`), the author placed `0` in `[27:24]` and `1` in `[23:20]`, which stores **`R1`** into `DATA_RAM[0x0001]`.

3. **`JMP` and `CALL` Bit-Reversal Splitters in `main` (`2930,2680` and `3090,2680`)**:
   - In `main` at `(2930,2680)`, `(2970,2680)`, `(2970,2760)`, and `(3090,2680)`, the 8-bit `JMP_H`, 16-bit `ADD_OUT`, and 24-bit `CALL_RET_ADD` buses pass through custom Splitters where `bit0` is mapped to `MSB` and `bitN` is mapped to `0`.
   - **Impact**: For `JMP` (`0xF0000001` in `PRG_ROM`), `ADD_OUT = 0x0001` and `JMP_H = 0x00` are bit-reversed within their respective 16-bit and 8-bit fields and then reversed across the 24-bit splitter `(2930,2680)`, so that `ADD_OUT` bit 0 (`1`) lands on **bit 0 of the 24-bit PC (`0x000001`)**! Thus, the double reversal preserves `ADD_OUT` in `PC[15:0]` and `JMP_H` in `PC[23:16]` in standard bit order (`0xF0000001` jumps to `0x000001`).

4. **Opcode `1110` (`0xE`): `CALL` in Circuit vs. `RJMP` in Handwritten Notes**:
   - The handwritten ISA table lists `1110` (`0xE`) as `RJMP` and `1111` (`0xF`) as `JMP`.
   - In `INST_DECODER`, primary opcode `1110` (`0xE`) is wired to pin **`CALL`** (`1890,1890`), while **`RJMP`** (`1890,980`) is decoded under extended opcode `0000 -> 1110 -> [0..2] -> 0100` (`0x0E04xxxx`). Moreover, in `main`, the `RJMP` output pin of `INST_DECODER` (`2480,1490`) is routed into the 30-bit pipeline register `(2630,1630)` as bit 0 of `CALL_RET_BRN`, whereas the `RJMP` tunnel (`3050,2370`) has only 1 instance and is not connected to a driver. Consequently, relative jumps executed via `0x0E04xxxx` trigger the `CALL_RET_BRN` 24-bit branch path rather than the 8-bit `RJMP` buffer `(1240,1480)`.

5. **`SR` / `SL` Splitter Mapping & `SL` Destination Enable**:
   - In `main`, Splitter `(2780,1780)` bundles the 9 ALU operation signals (`ADD`, `SUB`, `MUL`, `CMP`, `O_R`, `A_ND`, `X_OR`, `SR`, `SL` on fanout pins `0..8`) into the 32-bit ID/EX register `(2630,1750)`, and Splitter `(2980,1650)` unpacks them in Stage 3. In `(2980,1650)`, `bit7` (`SR`) is routed to fanout `1` (`ALU.L_SFT`) and `bit8` (`SL`) is routed to fanout `2` (`ALU.R_SFT`). Additionally, `SL` (`0x9`) is omitted from the Stage 3 `RD` write-back OR gate `(3190,1740)`.

6. **`CMP` Register Write-Back**:
   - In `main` Stage 3, `CMP` (`3000,1600`) is connected to one of the 9 inputs of OR gate `(3190,1740)`. That OR gate drives `ALU_EN`, `FLAG_WR`, and the Stage 4 register write-back enable (`bit 6` of the 7-bit control register `3980,1750`). As a result, `CMP Rd, Rs` updates `CMP_F` in the status register **and** writes the subtraction result `Rd - Rs` (via `ALU` MUX input 6) into `Rd`.

---

## 7. Worked Instruction Encoding Examples

All six instructions pre-loaded in `PRG_ROM` (`1730,1300`) of `CPU_Harvard_Arc.circ`—plus representative memory and branch instructions—are decoded step-by-step below.

### Example 1: Load Immediate into `R0` (`LDI R0, 0x0001` — ROM Address `0x000000`)
- **Assembly Instruction**: `LDI R0, 0x0001`
- **Instruction Format**: Immediate Format (`[31:28]=Op`, `[27:24]=Rd`, `[23:16]=0x00`, `[15:0]=Imm16`)
- **Binary Fields**:
  - Primary Opcode `[31:28]`: `1101` (`0xD` = `LDI`)
  - Destination `Rd` `[27:24]`: `0000` (`0x0` = `R0`)
  - Unused `[23:16]`: `0000 0000` (`0x00`)
  - Immediate `[15:0]`: `0000 0000 0000 0001` (`0x0001`)
- **Complete 32-bit Instruction**: `1101 0000 0000 0000 0000 0000 0000 0001`
- **Hexadecimal Representation**: `0xD0000001`
- **Expected Processor State Changes**: `R0 <- 0x0001`; Status flags unchanged; `PC` advances to `0x000001`.

### Example 2: Load Immediate into `R1` (`LDI R1, 0x0001` — ROM Address `0x000001`)
- **Assembly Instruction**: `LDI R1, 0x0001`
- **Instruction Format**: Immediate Format
- **Binary Fields**: `1101` (`LDI`) | `0001` (`R1`) | `0000 0000` | `0000 0000 0000 0001` (`0x0001`)
- **Complete 32-bit Instruction**: `1101 0001 0000 0000 0000 0000 0000 0001`
- **Hexadecimal Representation**: `0xD1000001`
- **Expected Processor State Changes**: `R1 <- 0x0001`; Status flags unchanged; `PC` advances to `0x000002`.

### Example 3: Add Two Registers (`ADD R0, R1` — ROM Address `0x000002`)
- **Assembly Instruction**: `ADD R0, R1`
- **Instruction Format**: Register-Register ALU Format (`[31:28]=Op`, `[27:24]=Rd`, `[23:20]=Rs`, `[19:16]=Type`, `[15:0]=0x0000`)
- **Binary Fields**:
  - Primary Opcode `[31:28]`: `0001` (`0x1` = `ADD`)
  - Destination `Rd` `[27:24]`: `0000` (`0x0` = `R0`)
  - Source `Rs` `[23:20]`: `0001` (`0x1` = `R1`)
  - Subtype `[19:16]`: `0000` (`0x0`)
  - Unused `[15:0]`: `0000 0000 0000 0000` (`0x0000`)
- **Complete 32-bit Instruction**: `0001 0000 0001 0000 0000 0000 0000 0000`
- **Hexadecimal Representation**: `0x10100000`
- **Expected Processor State Changes**: `R0 <- R0 + R1` (`0x0001 + 0x0001 = 0x0002`); `FLAGS` updated (`CARRY_F=0, NEG_F=0, V_F=0, ZERO_F=0, H_F=0`); `PC` advances to `0x000003`.

### Example 4: Store Register to Data RAM (`STS R1, 0x0001` — ROM Address `0x000003`)
- **Assembly Instruction**: `STS R1, 0x0001`
- **Instruction Format**: Memory Store Format (`[31:28]=0xC`, `[27:24]=0x0`, `[23:20]=Rs`, `[19:16]=0x0`, `[15:0]=Addr16`)
- **Binary Fields**:
  - Primary Opcode `[31:28]`: `1100` (`0xC` = `STS`)
  - Unused `[27:24]`: `0000` (`0x0`)
  - Source Register `Rs` `[23:20]`: `0001` (`0x1` = `R1`) *(or `0000` to store `R0`: `0xC0000001`)*
  - Unused `[19:16]`: `0000` (`0x0`)
  - Memory Address `[15:0]`: `0000 0000 0000 0001` (`0x0001`)
- **Complete 32-bit Instruction**: `1100 0000 0001 0000 0000 0000 0000 0001`
- **Hexadecimal Representation**: `0xC0100001`
- **Expected Processor State Changes**: `DATA_RAM[0x0001] <- R1`; Registers and flags unchanged; `PC` advances to `0x000004`.

### Example 5: Load Register from Data RAM (`LDS R2, 0x0000` — ROM Address `0x000004`)
- **Assembly Instruction**: `LDS R2, 0x0000`
- **Instruction Format**: Memory Load Format (`[31:28]=0xB`, `[27:24]=Rd`, `[23:16]=0x00`, `[15:0]=Addr16`)
- **Binary Fields**: `1011` (`LDS`) | `0010` (`R2`) | `0000 0000` | `0000 0000 0000 0000` (`0x0000`)
- **Complete 32-bit Instruction**: `1011 0010 0000 0000 0000 0000 0000 0000`
- **Hexadecimal Representation**: `0xB2000000`
- **Expected Processor State Changes**: `R2 <- DATA_RAM[0x0000]`; Status flags unchanged; `PC` advances to `0x000005`.

### Example 6: Unconditional Jump (`JMP 0x000001` — ROM Address `0x000005`)
- **Assembly Instruction**: `JMP 0x000001`
- **Instruction Format**: Jump Format (`[31:28]=0xF`, `[27:24]=0x0`, `[23:16]=JMP_H`, `[15:0]=ADD_OUT`)
- **Binary Fields**: `1111` (`JMP`) | `0000` | `0000 0000` (`0x00`) | `0000 0000 0000 0001` (`0x0001`)
- **Complete 32-bit Instruction**: `1111 0000 0000 0000 0000 0000 0000 0001`
- **Hexadecimal Representation**: `0xF0000001`
- **Expected Processor State Changes**: `PC <- 0x000001`; IF1 (`1560,1280`) and IF2 (`2070,1280`) pipeline registers flushed via `DFR_RST1` and `DFR_RST2`.
