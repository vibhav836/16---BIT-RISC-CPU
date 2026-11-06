# Memory Architecture (`PRG_ROM` and `DATA_RAM`)

## 1. Harvard Memory Organization Overview

The processor implements a strict Harvard memory architecture with physically separate storage arrays, address buses, and data buses for instructions (`PRG_ROM`) and data (`DATA_RAM`).

| Memory Parameter | Program ROM (`PRG_ROM`) | Data RAM (`DATA_RAM`) |
| :--- | :--- | :--- |
| **Logisim Component Type** | `ROM` (`lib="5"`, `appearance="logisim_evolution"`) | `RAM` (`lib="5"`, `appearance="logisim_evolution"`) |
| **Circuit Coordinates in `main`** | `(1730,1300)` | `(6610,1300)` |
| **Address Bus Width (`addrWidth`)** | **24 bits** (`0x000000` – `0xFFFFFF`) | **16 bits** (`0x0000` – `0xFFFF`) |
| **Data Bus Width (`dataWidth`)** | **32 bits** (`0x00000000` – `0xFFFFFFFF`) | **16 bits** (`0x0000` – `0xFFFF`) |
| **Addressing Granularity** | **32-bit Word-Addressed** (1 instruction per address) | **16-bit Word-Addressed** (1 data word per address) |
| **Theoretical Addressable Words** | $2^{24} = 16,777,216$ words ($64	ext{ MiB}$) | $2^{16} = 65,536$ words ($128	ext{ KiB}$) |
| **Pipeline Stage Accessed** | Stage 2 (`IF2`, driven by Fetch Reg `1560,1280`) | Stage 5 (`MEM/WB`, driven by EX/WB Regs at `x=3980`) |
| **Persistence** | Non-volatile (saved in `.circ` XML `contents` attribute) | Volatile (cleared to `0x0000` upon simulation reset) |

---

## 2. Instruction Memory (`PRG_ROM` at `1730,1300`)

### 2.1 Addressing and Read Timing
- `PRG_ROM` is a combinational read-only memory module whose 24-bit address input (`A` at `1730,1310`) is driven directly by the 24-bit Fetch Address Register (`1560,1280`).
- Because `PRG_ROM` has `dataWidth="32"`, each address `A` stores one complete 32-bit instruction word. Advancing the Program Counter by `+1` moves to the next 32-bit instruction.
- On the rising edge of `CLK`, the 32-bit output `D` (`1970,1360`) of `PRG_ROM` is latched into the 32-bit Instruction Register (`2070,1280`).

### 2.2 Pre-Loaded ROM Image in `CPU_Harvard_Arc.circ`
Inspection of the `<a name="contents">` attribute of `PRG_ROM` (`1730,1300`) reveals the following 6-instruction test program stored in the circuit file:

```text
addr/data: 24 32
d0000001 d1000001 10100000 c0100001 b2000000 f0000001
```

| ROM Word Address | Hex Instruction | Decoded Assembly Mnemonic | Architectural Meaning |
| :---: | :---: | :--- | :--- |
| `0x000000` | `0xD0000001` | `LDI R0, 0x0001` | Load immediate constant `0x0001` into `R0` |
| `0x000001` | `0xD1000001` | `LDI R1, 0x0001` | Load immediate constant `0x0001` into `R1` |
| `0x000002` | `0x10100000` | `ADD R0, R1` | Add `R0` and `R1`, store sum in `R0`, update flags |
| `0x000003` | `0xC0100001` | `STS R1, 0x0001` | Store `R1` (`Rs=1` in `[23:20]`) to `DATA_RAM[0x0001]` |
| `0x000004` | `0xB2000000` | `LDS R2, 0x0000` | Load `R2` (`Rd=2` in `[27:24]`) from `DATA_RAM[0x0000]` |
| `0x000005` | `0xF0000001` | `JMP 0x000001` | Unconditional jump back to ROM word address `0x000001` |

---

## 3. Data Memory (`DATA_RAM` at `6610,1300`)

### 3.1 Port Connections and Control Signals
`DATA_RAM` (`6610,1300`) is a synchronous-write, combinational-read 16-bit RAM with separate data-in and data-out ports:
- **16-bit Address Port (`A` at `6610,1310`)**:
  - Driven by 16-bit Controlled Buffer `(6270,1310)`, which passes `Tunnel: ADD_OUT` (latched in EX/WB Register `3980,2080` from instruction bits `[15:0]`) whenever `OR Gate (6260,1360)` (`LDS1 | STS1`) is `1`.
  - When neither `LDS1` nor `STS1` is active in Stage 5, Controlled Buffer `(6270,1310)` is high-impedance (`Z`), and Pull Resistor `(6420,1310)` pulls the 16-bit address bus to `0x0000`.
- **Write Enable (`WE` / `str` at `6610,1350`) and Output Enable (`OE` / `ld` at `6610,1360`)**:
  - `Tunnel: STS1` (`6320,1490`) and `Tunnel: LDS1` (`6320,1500`) are combined into a 2-bit bus by Splitter `(6340,1480)` and unpacked by Splitter `(6430,1340)` into:
    - `(6610,1350)` (`WE`): Driven by **`STS1`** (bit 4 of EX/WB Control Register `3980,1750`).
    - `(6610,1360)` (`OE`): Driven by **`LDS1`** (bit 3 of EX/WB Control Register `3980,1750`).
- **Clock Input (`CLK` at `6610,1370`)**:
  - Connected to `Tunnel: CLK`. Synchronous memory writes occur on the rising edge of `CLK` when `STS1 = 1`.
- **16-bit Data Input Port (`D_in` at `6610,1390`)**:
  - Driven by 16-bit Controlled Buffer `(6170,1850)` (enabled by `STS1`), which routes the shared 16-bit Write-Back Bus (`net 6180,1880`) into `DATA_RAM`. When `STS1 = 0`, Pull Resistor `(6420,1390)` pulls `D_in` to `0x0000`.
- **16-bit Data Output Port (`D_out` at `6850,1390`)**:
  - Routed to `(6200,1880)` and gated onto the shared 16-bit Write-Back Bus (`net 6180,1880`) by 16-bit Controlled Buffer `(6180,1880)` (enabled by `LDS1`).

### 3.2 `LDS` (Load from Data Space) Execution Path
For `LDS Rd, Addr16` (`[31:28] = 1011`, `[27:24] = Rd`, `[15:0] = Addr16`):
1. **`INST_DECODER`**: Asserts `LDS = 1`, decodes `Rd` onto `RD[15:0]`, and extracts `Addr16` onto `ADD_OUT[15:0]`.
2. **Stage 3 (`EX`)**: `LDS` enables Controlled Buffer `(3780,1630)` to pass `RD[15:0]` into EX/WB Register `(3980,1640)`, latches `LDS` into EX/WB Control Register `(3980,1750)` (becoming `LDS1`), and latches `ADD_OUT[15:0]` into EX/WB Register `(3980,2080)`.
3. **Stage 5 (`MEM/WB`)**:
   - `LDS1 = 1` enables Address Buffer `(6270,1310)`, driving `ADD_OUT[15:0]` onto `DATA_RAM.A`.
   - `LDS1 = 1` asserts `DATA_RAM.OE` (`6610,1360`), causing `DATA_RAM` to output the 16-bit word at `DATA_RAM[ADD_OUT]` onto `(6850,1390)`.
   - `LDS1 = 1` enables Read Buffer `(6180,1880)`, driving the read word onto the shared 16-bit Write-Back Bus (`net 6180,1880`).
   - The one-hot `RD[15:0]` signal from Reg `(3980,1640)` asserts the write-enable (`en`) of destination register `Rd`, latching the loaded 16-bit word on the rising edge of `CLK`.

### 3.3 `STS` (Store to Data Space) Execution Path
For `STS Rs, Addr16` (`[31:28] = 1100`, `[23:20] = Rs`, `[15:0] = Addr16`):
1. **`INST_DECODER`**: Asserts `STS = 1`, decodes `Rs` (`[23:20]`) onto `RS[15:0]`, and extracts `Addr16` onto `ADD_OUT[15:0]`.
2. **Stage 3 (`EX`)**: `STS` enables Controlled Buffer `(3780,2070)` to pass `RS[15:0]` into EX/WB Register `(3980,1970)`, latches `STS` into EX/WB Control Register `(3980,1750)` (becoming `STS1`), and latches `ADD_OUT[15:0]` into EX/WB Register `(3980,2080)`.
3. **Stage 5 (`MEM/WB`)**:
   - The one-hot `RS[15:0]` signal from Reg `(3980,1970)` enables the tri-state output buffer of source register `Rs` onto the shared 16-bit Write-Back Bus (`net 6180,1880`).
   - `STS1 = 1` enables Store Buffer `(6170,1850)`, routing the 16-bit value from the Write-Back Bus into `DATA_RAM.D_in` (`6610,1390`).
   - `STS1 = 1` enables Address Buffer `(6270,1310)` (`ADD_OUT -> DATA_RAM.A`) and asserts `DATA_RAM.WE` (`6610,1350`).
   - On the rising edge of `CLK`, `DATA_RAM[ADD_OUT]` latches the 16-bit value of `Rs`.

> **Reminder on Bit 6 of `ADD_OUT` and `DATA_OUT`**: As documented in [`instruction-set.md`](instruction-set.md), Splitter `(760,1800)` in `INST_DECODER` leaves `bit6` unassigned (`none`). Until `bit6="6"` is added to Splitter `(760,1800)`, effective `DATA_RAM` addresses have bit 6 pulled to `0` (`Addr16 & 0xFFBF`).
