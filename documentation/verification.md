# Systematic Processor Verification Plan

## 1. Verification Scope and Methodology

This document establishes a reproducible, systematic verification plan for the **Custom 16-bit Harvard Architecture CPU** in Logisim Evolution v4.1.0. In accordance with academic integrity requirements:
- **No test results are fabricated**. Tests that have been structurally verified via netlist trace are distinguished from interactive simulation runs, which are marked **`Not yet tested`** pending manual execution in the Logisim Evolution GUI.
- **Role of the `test` Subcircuit**: Inspection of `CPU_Harvard_Arc.circ` confirms that the `test` subcircuit (`26` components, `75` wires) is a standalone component scratchpad containing an isolated Priority Encoder `(420,290)`, an 8-bit Counter `(530,600)` driving a RAM `(1110,630)` and four Hex Digit Displays `(1680..1800,330)`, and a 7-bit ASCII TTY `(1840,700)`. It does **not** instantiate `main`, `ALU`, or `INST_DECODER`. Therefore, all processor verification must be performed inside the **`main`** circuit (or via unit-test vectors applied directly to `ALU` and `INST_DECODER`).

---

## 2. Step-by-Step Procedure for Running Tests in Logisim Evolution

1. **Launch Logisim Evolution**: Open `circuit/CPU_Harvard_Arc.circ` in Logisim Evolution v4.1.0 (or compatible v3.8+). If prompted for `ARRAY_MUL.circ`, click Cancel or remove the unused `<lib desc="file#ARRAY_MUL.circ" name="13"/>` line in a working copy, as no component in the file references `lib="13"`.
2. **Select the `main` Circuit**: Double-click `main` in the left Explorer pane so the magnifying glass icon appears on `main`.
3. **Reset the Processor**:
   - To perform a clean hardware reset before stepping, use **Simulate -> Reset Simulation (`Ctrl+R`)**, which initializes `Counter (1280,1310)`, all pipeline registers (`1560,1280`, `2070,1280`, `x=2630`, `x=3980`), `R0`–`R15`, `CFG_REG[0..7]`, `FLAGS`, and `DATA_RAM` to `0`.
4. **Load Test Instructions into `PRG_ROM`**:
   - Right-click the `PRG_ROM` component at `(1730,1300)` and select **Edit Contents...**.
   - Enter the 32-bit hexadecimal machine code words starting at address `000000`.
   - **Important Pipeline Hazard Rule**: Because the processor has a 4-stage pipeline register hierarchy (`IF1`, `IF2`, `ID`, `EX`, `WB`) without hardware data forwarding, insert **two `NOP` instructions (`0x00000000`)** between any instruction that writes a register (`Rd`) or flag (`FLAGS`) and a subsequent instruction that reads that same register or flag!
5. **Step the Clock (`CLK`) Cycle-by-Cycle**:
   - Use **Simulate -> Manual Tick Full Cycle (`Ctrl+T` twice)** or click the `Clock` component at `(1200,1060)` with the Poke tool (`Hand` icon) to advance one rising clock edge at a time.
   - Each single instruction takes **5 rising clock edges** from initial PC fetch (`IF1`) to register/memory write-back (`WB`).
6. **Inspect Internal State**:
   - **Registers `R0`–`R15`**: Observe the 16-bit hex value displayed inside each Register component at `y=1760` (`R0`–`R7`, `x=4500..5970`) and `y=2310` (`R8`–`R15`, `x=4500..5970`), or probe the four 64-bit tunnels `REG_0_1_2_3`, `REG_4_5_6_7`, `REG_8_9_10_11`, and `REG_12_13_14_15` (`6270,2400..2490`).
   - **Status Flags**: Observe `Tunnel: FLAGS` (`3730,1310`) or inspect the output pins `CARRY_F`..`INT_F` on the `ALU` block at `(3530,1260)`.
   - **Data Memory**: Right-click `DATA_RAM` at `(6610,1300)` and select **Edit Contents...** to verify word writes (`STS`) and reads (`LDS`).

---

## 3. Pre-Loaded ROM Baseline Verification (`PRG_ROM` Default Image)

The table below traces the exact cycle-by-cycle behavior of the 6-instruction program currently saved inside `PRG_ROM` (`d0000001 d1000001 10100000 c0100001 b2000000 f0000001`).

| Test ID | ROM Address & Hex Instruction | Initial State | Expected Architectural Behavior | Circuit-Specific Predicted Behavior (Netlist Trace) | Observed Simulation Result | Status |
| :---: | :--- | :--- | :--- | :--- | :--- | :---: |
| **BASE-01** | `0x000000`: `0xD0000001` (`LDI R0, 0x0001`) | All regs `0x0000`, `PC=0` | `R0 <- 0x0001` at end of Cycle 5 | `R0 (4500,1760) <- 0x0001` at end of Cycle 5 via Buffer `(4350,1880)` and `(4480,1790)` | *Not yet tested* | **Not yet tested** |
| **BASE-02** | `0x000001`: `0xD1000001` (`LDI R1, 0x0001`) | `R0=0x0001` (at Cycle 5) | `R1 <- 0x0001` at end of Cycle 6 | `R1 (4710,1760) <- 0x0001` at end of Cycle 6 | *Not yet tested* | **Not yet tested** |
| **BASE-03** | `0x000002`: `0x10100000` (`ADD R0, R1`) | First pass (Cycle 6 EX): `R0=1`, `R1=0` (RAW hazard); Second pass: `R0=1, R1=1` | `R0 <- R0 + R1` | Pass 1: `R0 <- 0x0001 + 0x0000 = 0x0001` (because `LDI R1, 1` is still in `WB` during Cycle 6). Pass 2 (after `JMP 1`): `R0 <- 0x0001 + 0x0001 = 0x0002` | *Not yet tested* | **Not yet tested** |
| **BASE-04** | `0x000003`: `0xC0100001` (`STS R1, 0x0001`) | `R1=0x0001`, `DATA_RAM[1]=0x0000` | Store `R1` (`Rs=1`) to `DATA_RAM[0x0001]` | `DATA_RAM[0x0001] <- 0x0001` at end of Cycle 8 | *Not yet tested* | **Not yet tested** |
| **BASE-05** | `0x000004`: `0xB2000000` (`LDS R2, 0x0000`) | `DATA_RAM[0x0000] = 0x0000` | `R2 <- DATA_RAM[0x0000]` | `R2 (4930,1760) <- 0x0000` at end of Cycle 9 | *Not yet tested* | **Not yet tested** |
| **BASE-06** | `0x000005`: `0xF0000001` (`JMP 0x000001`) | `PC` advancing past `0x000005` | `PC <- 0x000001`, flush `IF1` & `IF2` | In Stage 3 (Cycle 8/9), `JMP` latches `0x000001` into `(2630,2640)`, asserts `CTR_STP1`, flushes `(1560,1280)` & `(2070,1280)`, and loads `0x000001` | *Not yet tested* | **Not yet tested** |

---

## 4. Comprehensive Instruction & Edge-Case Verification Matrix

For each test below, assume two `NOP` (`0x00000000`) instructions follow any setup `LDI` instructions so that source registers have settled before the instruction under test enters Stage 4 (`EX`).

### 4.1 Arithmetic Operations (`ADD`, `SUB`, `MUL`, `MULS`)

| Test ID | Category / Case | Initial State | Instruction & Hex Encoding | Expected Architectural Behavior | Circuit-Specific Notes / Edge Cases | Observed Result | Status |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :---: |
| **ARITH-01** | `ADD` Normal | `R2=0x0015`, `R3=0x002A` | `ADD R2, R3` (`0x12300000`) | `R2 <- 0x003F`; `CARRY_F=0, NEG_F=0, V_F=0, ZERO_F=0, H_F=0` | `H_F` is carry out of bit 7 (`C7=0`). | *Not yet tested* | **Not yet tested** |
| **ARITH-02** | `ADD` Lower-Byte Carry (`H_F`) | `R2=0x00FF`, `R3=0x0001` | `ADD R2, R3` (`0x12300000`) | `R2 <- 0x0100`; `H_F=1` (`C7=1`), `CARRY_F=0, ZERO_F=0` | Verifies `H_F` connection to Adder 7 carry-out (`12380,7980`). | *Not yet tested* | **Not yet tested** |
| **ARITH-03** | `ADD` Signed Overflow (`V_F`) & Negative (`NEG_F`) | `R2=0x7FFF`, `R3=0x0001` | `ADD R2, R3` (`0x12300000`) | `R2 <- 0x8000`; `NEG_F=1`, `V_F=1` (`C14=1`), `CARRY_F=0` | Positive + Positive -> Negative (`0x8000`). | *Not yet tested* | **Not yet tested** |
| **ARITH-04** | `ADD` Unsigned Carry (`CARRY_F`) & Zero (`ZERO_F`) | `R2=0xFFFF`, `R3=0x0001` | `ADD R2, R3` (`0x12300000`) | `R2 <- 0x0000`; `CARRY_F=1`, `ZERO_F=1`, `NEG_F=0`, `H_F=1` | Verifies 16-bit wrap-around to zero and `CARRY_F` buffer `(12850,6890)`. | *Not yet tested* | **Not yet tested** |
| **ARITH-05** | `SUB` Normal (`Rd > Rs`) | `R2=0x0030`, `R3=0x0010` | `SUB R2, R3` (`0x22300000`) | Intended: `R2 <- 0x0020` | **Circuit Check**: In `ALU`, `SUB` (`11670,7080`) is not wired to Priority Encoder `(11760,6980)` input 7; verify if `ALU_OUT` is `X` unless `SUM` is co-asserted. | *Not yet tested* | **Not yet tested** |
| **ARITH-06** | `SUB` Zero Result | `R2=0x1234`, `R3=0x1234` | `SUB R2, R3` (`0x22300000`) | Intended: `R2 <- 0x0000`, `ZERO_F=1` | Same Priority Encoder input 7 check as `ARITH-05`. | *Not yet tested* | **Not yet tested** |
| **ARITH-07** | `MUL` 16-bit Product (`MUL_OUT_H = 0`) | `R2=0x0012`, `R3=0x0003` | `MUL R2, R3` (`0x32300000`) | `R2 <- 0x0036` (low 16b); `R0 <- 0x0000` (high 16b); `R1 <- 0x0036` | Verifies 16x16 array multiplier low word (`MUX[0]`) and `MUL_MULS` routing to `R0`/`R1`. | *Not yet tested* | **Not yet tested** |
| **ARITH-08** | `MUL` 32-bit Product (`MUL_OUT_H != 0`) | `R2=0x1000`, `R3=0x0020` | `MUL R2, R3` (`0x32300000`) | `32-bit Product = 0x00200000`: `R2 <- 0x0000` (`ZERO_F=1`), `MUL_OUT_H = 0x0020 -> R0` | Confirms high 16-bit product (`MUL_OUT_H`) is latched in `(3980,1520)` and written into `R0` (`4500,1760`). | *Not yet tested* | **Not yet tested** |
| **ARITH-09** | `MULS` Subtype (`0x3..F`) | `R2=0x8002`, `R3=0x0003` | `MULS R2, R3` (`0x323F0000`) | Intended: Signed multiply; Circuit: asserts `MUL` + `MULS` (which drives `SUM` in `main`) | Verify interaction between `MUL` (Priority 0) and `SUM` (Priority 7) in `ALU` Priority Encoder `(11760,6980)`. | *Not yet tested* | **Not yet tested** |

### 4.2 Comparison & Logical Operations (`CMP`, `AND`, `OR`, `XOR`)

| Test ID | Category / Case | Initial State | Instruction & Hex Encoding | Expected Architectural Behavior | Circuit-Specific Notes / Edge Cases | Observed Result | Status |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :---: |
| **LOGIC-01** | `CMP` Equal (`Rd == Rs`) | `R4=0x00A5`, `R5=0x00A5` | `CMP R4, R5` (`0x44500000`) | `CMP_F <- 1`, `ZERO_F <- 1`, `NEG_F <- 0` | Also writes `0x0000` (`R4 - R5`) into `R4` because `CMP` is wired to `ALU_EN` / WB OR gate `(3190,1740)`. | *Not yet tested* | **Not yet tested** |
| **LOGIC-02** | `CMP` Not Equal (`Rd != Rs`) | `R4=0x00A5`, `R5=0x0005` | `CMP R4, R5` (`0x44500000`) | `CMP_F <- 0`, `ZERO_F <- 0`, `NEG_F <- 0` | Writes `0x00A0` (`R4 - R5`) into `R4`. | *Not yet tested* | **Not yet tested** |
| **LOGIC-03** | `OR` Bitwise OR | `R4=0x0F0F`, `R5=0xF000` | `OR R4, R5` (`0x54500000`) | `R4 <- 0xFF0F`; `NEG_F=1, ZERO_F=0, CMP_F=0` | Uses 16-bit OR gate `(11930,7050)` (`MUX[4]`). | *Not yet tested* | **Not yet tested** |
| **LOGIC-04** | `AND` Bitwise AND | `R4=0x0F0F`, `R5=0x00FF` | `AND R4, R5` (`0x64500000`) | `R4 <- 0x000F`; `NEG_F=0, ZERO_F=0, CMP_F=0` | Uses 16-bit AND gate `(11930,6980)` (`MUX[3]`). | *Not yet tested* | **Not yet tested** |
| **LOGIC-05** | `XOR` Bitwise XOR (Self-Clear) | `R4=0xABCD`, `R5=0xABCD` | `XOR R4, R5` (`0x74500000`) | `R4 <- 0x0000`; `ZERO_F=1, NEG_F=0, CMP_F=0` | Uses 16-bit XOR gate `(11940,7130)` (`MUX[5]`). | *Not yet tested* | **Not yet tested** |

### 4.3 Shift Operations (`SR`, `SRC`, `SL`, `SLC`)

| Test ID | Category / Case | Initial State | Instruction & Hex Encoding | Expected Architectural Behavior | Circuit-Specific Notes / Edge Cases | Observed Result | Status |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :---: |
| **SHIFT-01** | `SR` (`0x8`) | `R2=0x0003`, `CARRY_F=0` | `SR R2` (`0x82000000`) | Intended: Shift right `R2` by 1 | In `main`, Splitter `(2980,1650)` routes `SR` to `ALU.L_SFT` (`MUX[2]`), performing `{R2[14:0], CARRY_F} = 0x0006`. | *Not yet tested* | **Not yet tested** |
| **SHIFT-02** | `SRC` (`0x8..C`) | `R2=0x8001`, `CARRY_F=0` | `SRC R2` (`0x820C0000`) | Shift with carry update (`SFT_W_CRY=1`) | Updates `CARRY_F` via Buffer `(12810,6770)` with `R2[15]` (since `L_SFT=1` selects `IN[1]` on MUX `12770,6770`). | *Not yet tested* | **Not yet tested** |
| **SHIFT-03** | `SL` (`0x9`) | `R2=0x0004`, `CARRY_F=0` | `SL R2` (`0x92000000`) | Intended: Shift left `R2` by 1 | In `main`, `SL` routes to `ALU.R_SFT` (`0x0002`), and `SL` is omitted from Stage 3 `RD` OR gate `(3190,1740)` so `R2` write-enable is not asserted. | *Not yet tested* | **Not yet tested** |
| **SHIFT-04** | `SLC` (`0x9..E`) | `R2=0x0005`, `CARRY_F=0` | `SLC R2` (`0x920E0000`) | Intended: Shift left with carry | In `main`, `SLC` is wired into `ALU.SUB` rather than `SFT_W_CRY`. | *Not yet tested* | **Not yet tested** |

### 4.4 Register, Immediate, and Memory Operations (`MOV`, `LDI`, `LDS`, `STS`, `CFG`)

| Test ID | Category / Case | Initial State | Instruction & Hex Encoding | Expected Architectural Behavior | Circuit-Specific Notes / Edge Cases | Observed Result | Status |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :---: |
| **MEM-01** | `LDI` Standard Immediate | `R6=0x0000` | `LDI R6, 0x1234` (`0xD6001234`) | `R6 <- 0x1234` | Note: Bit 6 (`0x0040`) of `0x1234` is `0`, so Splitter `(760,1800)` omission of `bit6` does not affect `0x1234`. | *Not yet tested* | **Not yet tested** |
| **MEM-02** | `LDI` Bit-6 Omission Check | `R6=0x0000` | `LDI R6, 0x007F` (`0xD600007F`) | Intended: `0x007F`; Circuit: `0x003F` | Verifies that unassigned `bit6` in Splitter `(760,1800)` is pulled to `0` by Pull Resistor `(3910,1440)`. | *Not yet tested* | **Not yet tested** |
| **MEM-03** | `MOV` Register Copy | `R6=0x1234`, `R7=0x0000` | `MOV R7, R6` (`0xA7600000`) | `R7 <- 0x1234`, `R6` unchanged | Uses `RS` buffer `(5860,1790)` onto WB bus and `RD` write-enable on `R7` (`5970,1760`). | *Not yet tested* | **Not yet tested** |
| **MEM-04** | `STS` & `LDS` Round-Trip | `R6=0x1234`, `R8=0x0000` | `STS R6, 0x0010` (`0xC0600010`)<br>+ 2 `NOP`s +<br>`LDS R8, 0x0010` (`0xB8000010`) | `DATA_RAM[0x0010] <- 0x1234`, then `R8 <- 0x1234` | Note `Rs=6` in `[23:20]` for `STS` and `Rd=8` in `[27:24]` for `LDS`. | *Not yet tested* | **Not yet tested** |
| **MEM-05** | `CFG` Special Register Write | `STACK_POINTER=0x0000` | `CFG SP, 0xFF00` (`0x0F70FF00`) | `STACK_POINTER (5970,1370) <- 0xFF00` | `[31:24]=0x0F` (`CFG`), `[23:20]=0x7` (`CFG_REG[7] = STACK_POINTER`), `[15:0]=0xFF00`. | *Not yet tested* | **Not yet tested** |

### 4.5 Control-Flow & System Operations (`JMP`, `CALL`, `RET`, `BRE`, `HLT`, `RST`)

| Test ID | Category / Case | Initial State | Instruction & Hex Encoding | Expected Architectural Behavior | Circuit-Specific Notes / Edge Cases | Observed Result | Status |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :---: |
| **CTRL-01** | `JMP` Unconditional | `PC=0x000005` | `JMP 0x000001` (`0xF0000001`) | `PC <- 0x000001`, flush `IF1` & `IF2` | Double bit-reversal across `(2970,2760)` and `(2930,2680)` preserves `0x000001`. | *Not yet tested* | **Not yet tested** |
| **CTRL-02** | `CALL` Unconditional (`0xE`) | `PC=0x000002` | `CALL 0x000010` (`0xE0000010`) | Loads `CALL_RET_ADD` into `PC`, flushes `IF1` & `IF2` | Note: Single bit-reversal on `(3090,2680)` maps `CALL_RET_BR_ADD[b]` to `PC[23-b]`. | *Not yet tested* | **Not yet tested** |
| **CTRL-03** | `BRE` Conditional Branch | `ZERO_F=1` (`FLAGS[3]=1`) | `BRE 0x000010` (`0x05000010`) | Branch taken when `ZERO_F == 1` | Must follow flag-setting instruction by 2 `NOP`s so `Reg (3650,1280)` has latched `FLAGS`. | *Not yet tested* | **Not yet tested** |
| **CTRL-04** | `HLT` Processor Halt | Running | `HLT` (`0x0E020000`) | Asserts `HLT` -> switches Clock MUX `(1270,1070)` to `0` | Freezes `Tunnel: CLK` at `0` until manual simulation reset. | *Not yet tested* | **Not yet tested** |
| **CTRL-05** | `RST` Software Reset | Registers non-zero | `RST` (`0x0E030000`) | Asserts `RST_MAIN` & `DFR_RST` | Clears `PC`, pipeline registers, `R0`–`R15`, and `FLAGS` to `0`. | *Not yet tested* | **Not yet tested** |
