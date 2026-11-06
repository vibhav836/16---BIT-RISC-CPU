# Logisim Evolution Circuit Assets

## Primary Circuit File

- **Filename**: `CPU_Harvard_Arc.circ`
- **Tool Version**: Logisim Evolution v4.1.0 (`project source="4.1.0"`)
- **Integrity Policy**: This file is an exact, unmodified copy of the author's original processor implementation. All technical documentation in [`../documentation/`](../documentation/) is derived directly from inspecting its XML netlist and component attributes.

## Subcircuits Inside `CPU_Harvard_Arc.circ`

| Subcircuit Name | Component Count | Wire Segments | Role in the Project |
| :--- | :---: | :---: | :--- |
| `main` | 276 | 1,204 | Top-level pipelined 16-bit Harvard CPU datapath, control routing, `PRG_ROM`, `DATA_RAM`, 16 general-purpose registers (`R0`–`R15`), 8 special/I/O configuration registers, and pipeline stage registers. |
| `ALU` | 615 | 5,715 | Dedicated 16-bit Arithmetic Logic Unit containing a custom 16x16 combinational array multiplier (256 AND gates + 256 1-bit full adders), a 16-bit ripple-carry adder/subtractor, a 16-bit subtractor/comparator, 16-bit bitwise logic (`AND`, `OR`, `XOR`), 1-bit right/left shift-and-rotate splitters, an 8-to-3 priority encoder, an 8-to-1 result multiplexer, and an 8-bit status flag register (`13000,6730`). |
| `INST_DECODER` | 90 | 449 | Combinational instruction decoder accepting the 32-bit instruction word (`DATA_IN[31:0]`) and 8-bit flags (`FLAGS_IN[7:0]`), built from eight 4-to-16 decoders, 12 conditional branch/call/return AND gates, and bit-reversal splitters. |
| `test` | 26 | 75 | Standalone auxiliary component sandbox (testing a Priority Encoder, a Counter driving a RAM with 4 Hex Digit Displays, and a 7-bit ASCII TTY). It does not instantiate `main`, `ALU`, or `INST_DECODER`. |

## External Library Reference Notice (`ARRAY_MUL.circ`)

Line 16 of `CPU_Harvard_Arc.circ` declares an external Logisim file library:

```xml
<lib desc="file#ARRAY_MUL.circ" name="13"/>
```

When opening `CPU_Harvard_Arc.circ` in Logisim Evolution on a new machine, Logisim may prompt to locate `ARRAY_MUL.circ` if that file is not present in the same directory. However, netlist inspection confirms that **zero components in `main`, `ALU`, `INST_DECODER`, or `test` instantiate `lib="13"`**—the entire 16x16 array multiplier is flattened directly inside the `ALU` subcircuit using standard `#Gates` (`lib="1"`) and `#Arithmetic` (`lib="3"`) primitives. If prompted by Logisim Evolution, you can either place `ARRAY_MUL.circ` alongside `CPU_Harvard_Arc.circ` or remove the unused `<lib desc="file#ARRAY_MUL.circ" name="13"/>` declaration in a backup copy.
