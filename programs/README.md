# Example Machine-Code Programs (`PRG_ROM`)

## 1. How to Enter and Run Programs in Logisim Evolution

No external assembler is bundled with the circuit; programs are entered directly as 32-bit hexadecimal machine words into the **`PRG_ROM`** component at `(1730,1300)` in `main`:

1. Open [`../circuit/CPU_Harvard_Arc.circ`](../circuit/CPU_Harvard_Arc.circ) in Logisim Evolution v4.1.0.
2. In the `main` circuit, right-click **`PRG_ROM`** (`1730,1300`) and select **Edit Contents...** (or load a Logisim `v2.0 raw` hex image file).
3. Enter the 32-bit hex words starting at address `000000`.
4. Press `Ctrl+R` (**Simulate -> Reset Simulation**) to clear the Program Counter, pipeline registers, `R0`–`R15`, `FLAGS`, and `DATA_RAM`.
5. Advance the clock manually (`Ctrl+T` or Poke `Clock (1200,1060)`) and observe the register bank (`x=4500..5970`) and `DATA_RAM` (`6610,1300`).

> **Pipeline Scheduling Rule (2-Cycle RAW Delay)**:
> Because the 4-stage pipeline register hierarchy (`IF1`, `IF2`, `ID`, `EX`, `WB`) does not include hardware forwarding multiplexers, an instruction writes its result into `Rd` (or `FLAGS`) at the end of Stage 5 (`WB`), whereas a subsequent instruction reads `Rd`/`Rs` in Stage 4 (`EX`) or evaluates `FLAGS_IN` in Stage 3 (`ID`). Always place **two `NOP` (`0x00000000`)** instructions (or two independent instructions) between a producer instruction and a consumer instruction when you want the consumer to read the newly written value on the very first pass.

---

## Program 1: Original Pre-Loaded `PRG_ROM` Loop (Baseline Circuit Program)

- **Purpose**: Demonstrates immediate loading (`LDI`), register addition (`ADD`), memory store (`STS`), memory load (`LDS`), and unconditional loop jump (`JMP`) exactly as saved in `CPU_Harvard_Arc.circ`.
- **Initial Processor State**: All registers `R0`–`R15 = 0x0000`, `DATA_RAM = 0x0000`, `PC = 0x000000`.

| ROM Address | Assembly Instruction | Binary Fields (`Op \| Rd \| Rs \| Type \| Imm16`) | 32-bit Hex Word |
| :---: | :--- | :---: | :---: |
| `0x000000` | `LDI R0, 0x0001` | `1101 \| 0000 \| 0000 \| 0000 \| 0000000000000001` | `0xD0000001` |
| `0x000001` | `LDI R1, 0x0001` | `1101 \| 0001 \| 0000 \| 0000 \| 0000000000000001` | `0xD1000001` |
| `0x000002` | `ADD R0, R1` | `0001 \| 0000 \| 0001 \| 0000 \| 0000000000000000` | `0x10100000` |
| `0x000003` | `STS R1, 0x0001` | `1100 \| 0000 \| 0001 \| 0000 \| 0000000000000001` | `0xC0100001` |
| `0x000004` | `LDS R2, 0x0000` | `1011 \| 0010 \| 0000 \| 0000 \| 0000000000000000` | `0xB2000000` |
| `0x000005` | `JMP 0x000001` | `1111 \| 0000 \| 0000 \| 0000 \| 0000000000000001` | `0xF0000001` |

- **Expected State Changes**:
  - `R0` is initialized to `0x0001` and increments by `R1` (`0x0001`) on each iteration after `R1` settles (`0x0001 -> 0x0002 -> 0x0003 ...`).
  - `DATA_RAM[0x0001]` receives `0x0001` from `R1`.
  - `R2` loads `DATA_RAM[0x0000]` (`0x0000`).
  - `JMP 0x000001` flushes the pipeline (`DFR_RST1`, `DFR_RST2`) and restarts execution from `0x000001`.

---

## Program 2: Hazard-Free Addition, Register Copy, and Memory Round-Trip

- **Purpose**: Computes `R2 = 0x0015 + 0x002A = 0x003F`, copies `R2` into `R3` (`MOV`), stores `R3` into `DATA_RAM[0x0010]` (`STS`), loads it back into `R4` (`LDS`), and halts (`HLT`).
- **Initial Processor State**: All registers `0x0000`, `DATA_RAM[0x0010] = 0x0000`, `PC = 0x000000`.

| ROM Address | Assembly Instruction | Binary Fields (`Op \| Rd \| Rs \| Type \| Imm16`) | 32-bit Hex Word |
| :---: | :--- | :---: | :---: |
| `0x000000` | `LDI R2, 0x0015` | `1101 \| 0010 \| 0000 \| 0000 \| 0000000000010101` | `0xD2000015` |
| `0x000001` | `LDI R3, 0x002A` | `1101 \| 0011 \| 0000 \| 0000 \| 0000000000101010` | `0xD300002A` |
| `0x000002` | `NOP` | `0000 \| 0000 \| 0000 \| 0000 \| 0000000000000000` | `0x00000000` |
| `0x000003` | `NOP` | `0000 \| 0000 \| 0000 \| 0000 \| 0000000000000000` | `0x00000000` |
| `0x000004` | `ADD R2, R3` | `0001 \| 0010 \| 0011 \| 0000 \| 0000000000000000` | `0x12300000` |
| `0x000005` | `NOP` | `0000 \| 0000 \| 0000 \| 0000 \| 0000000000000000` | `0x00000000` |
| `0x000006` | `NOP` | `0000 \| 0000 \| 0000 \| 0000 \| 0000000000000000` | `0x00000000` |
| `0x000007` | `MOV R4, R2` | `1010 \| 0100 \| 0010 \| 0000 \| 0000000000000000` | `0xA4200000` |
| `0x000008` | `STS R2, 0x0010` | `1100 \| 0000 \| 0010 \| 0000 \| 0000000000010000` | `0xC0200010` |
| `0x000009` | `NOP` | `0000 \| 0000 \| 0000 \| 0000 \| 0000000000000000` | `0x00000000` |
| `0x00000A` | `LDS R5, 0x0010` | `1011 \| 0101 \| 0000 \| 0000 \| 0000000000010000` | `0xB5000010` |
| `0x00000B` | `NOP` | `0000 \| 0000 \| 0000 \| 0000 \| 0000000000000000` | `0x00000000` |
| `0x00000C` | `NOP` | `0000 \| 0000 \| 0000 \| 0000 \| 0000000000000000` | `0x00000000` |
| `0x00000D` | `HLT` | `0000 \| 1110 \| 0000 \| 0010 \| 0000000000000000` | `0x0E020000` |

- **Expected State After Halt**:
  - `R2 = 0x003F`, `R3 = 0x002A`, `R4 = 0x003F`, `R5 = 0x003F`, `DATA_RAM[0x0010] = 0x003F`.

---

## Program 3: Bitwise Logical Suite (`AND`, `OR`, `XOR`)

- **Purpose**: Exercises the 16-bit bitwise `AND` (`0x6`), `OR` (`0x5`), and `XOR` (`0x7`) functional units inside `ALU`.
- **Initial Processor State**: All registers `0x0000`, `PC = 0x000000`.

| ROM Address | Assembly Instruction | 32-bit Hex Word | Expected Result |
| :---: | :--- | :---: | :--- |
| `0x000000` | `LDI R4, 0x0F0F` | `0xD4000F0F` | `R4 <- 0x0F0F` |
| `0x000001` | `LDI R5, 0x00FF` | `0xD50000BF` *(Note: bit 6=0 -> `0x00BF`)* | `R5 <- 0x00BF` |
| `0x000002` | `LDI R6, 0x0F0F` | `0xD6000F0F` | `R6 <- 0x0F0F` |
| `0x000003` | `LDI R7, 0x0F0F` | `0xD7000F0F` | `R7 <- 0x0F0F` |
| `0x000004` | `AND R4, R5` | `0x64500000` | `R4 <- 0x0F0F & 0x00BF = 0x000F` |
| `0x000005` | `OR  R6, R5` | `0x56500000` | `R6 <- 0x0F0F \| 0x00BF = 0x0FBF` |
| `0x000006` | `XOR R7, R5` | `0x77500000` | `R7 <- 0x0F0F ^ 0x00BF = 0x0FB0` |
| `0x000007` | `NOP` | `0x00000000` | Pipeline drain |
| `0x000008` | `NOP` | `0x00000000` | Pipeline drain |
| `0x000009` | `HLT` | `0x0E020000` | Halt clock |

---

## Program 4: 16x16 Unsigned Array Multiplication (`MUL` with 32-bit Product)

- **Purpose**: Exercises the 16x16 combinational array multiplier (`0x3`) and verifies how the 32-bit product is split into `ALU_OUT[15:0]` (low word -> `Rd`) and `MUL_OUT_H[15:0]` (high word -> `R0`).
- **Initial Processor State**: All registers `0x0000`, `PC = 0x000000`.

| ROM Address | Assembly Instruction | 32-bit Hex Word | Expected Result |
| :---: | :--- | :---: | :--- |
| `0x000000` | `LDI R2, 0x1002` | `0xD2001002` | `R2 <- 0x1002` ($4098_{10}$) |
| `0x000001` | `LDI R3, 0x0020` | `0xD3000020` | `R3 <- 0x0020` ($32_{10}$) |
| `0x000002` | `NOP` | `0x00000000` | Wait for `R2`, `R3` write-back |
| `0x000003` | `NOP` | `0x00000000` | Wait for `R2`, `R3` write-back |
| `0x000004` | `MUL R2, R3` | `0x32300000` | $32	ext{-bit Product} = 0x00200040$: Low 16b `0x0040 -> R2` (and `R1`); High 16b `0x0020 -> R0` |
| `0x000005` | `NOP` | `0x00000000` | Pipeline drain |
| `0x000006` | `NOP` | `0x00000000` | Pipeline drain |
| `0x000007` | `HLT` | `0x0E020000` | Halt clock |
