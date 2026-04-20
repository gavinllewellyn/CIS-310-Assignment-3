# CIS 310 — Assignment 3: 8-bit Processor

An 8-bit CPU built in the Digital logic simulator by H. Neemann. The processor integrates the components from Assignment 1 (Program Counter, Instruction Memory, Instruction Register) 
and Assignment 2 (Register File, ALU) and is driven by a four-state ring-counter FSM implementing the classic FETCH → DECODE → EXECUTE → WRITEBACK cycle.

**Author:** Gavin Llewellyn
**Course:** CIS 310
**Status:** ✅ Passes the provided test script (`test_8bit_processor.txt`)

---

## Quick Start

1. Install Digital.
2. Clone or download this repository.
3. Open **`8-bit-Processor.dig`** in Digital. All subcircuits must stay in the same directory as the top-level file — they are already located there in this repo.
4. To verify correctness, run the provided test script: **Analysis → Test** and load `test_8bit_processor.txt`.

---

## Repository Layout

```
.
├── 8-bit-Processor.dig         ← top-level CPU circuit (open this one)
├── ControlUnit.dig             ← 4-state one-hot ring-counter FSM
├── 8-bit_ProgramCounter.dig    ← 8-bit Program Counter with 2-bit CTRL
├── 8-Bit_Memory.dig            ← 8-word × 8-bit instruction memory
├── 8-bit_IR.dig                ← 8-bit Instruction Register
├── RegisterFile.dig            ← R0–R3 register file
├── ALU.dig                     ← 8-bit ALU
├── 8-bit-Register.dig          ← generic 8-bit register (used inside IR/RegFile)
├── 8-bit_Adder.dig             ← used by the Program Counter
├── 8-bit-full-adder.dig        ← used by the ALU
├── 8-bit Mem Imp - In Class.dig ← memory cell used inside 8-Bit_Memory.dig
├── test_8bit_processor.txt     ← test script for Digital's test runner
├── 8-Bit_Processor_Test_Result.csv ← exported test waveform
├── docs/
│   └── Assignment3_Report.pdf  ← written report (also the Canvas submission)
└── screenshots/
    ├── top_level_processor.jpg
    ├── control_unit.jpg
    ├── test_result_1_programming.jpg
    ├── test_result_2_run_start.jpg
    └── test_result_3_sub_and_wrap.jpg
```

---

## Design Overview

### Instruction Format

Each instruction is a single 8-bit word:

| Bits | Field  | Width | Purpose                                                      |
| :--: | :----- | :---: | :----------------------------------------------------------- |
| 7:5  | opcode |   3   | ALU control {S1, S0, Cin} — wired directly to the ALU        |
| 4:3  | op1    |   2   | Destination and first source register (R0–R3)                |
| 2:1  | op2    |   2   | Second source register (R-type) or 2-bit immediate (I-type)  |
|  0   | type   |   1   | 0 = R-type, 1 = I-type                                       |

### Supported Instructions

| Opcode | R-type (T=0)   | I-type (T=1)       |
| :----: | :------------- | :----------------- |
| 000    | ADD Rd, Rs     |                    |
| 001    | ADDC Rd, Rs    |                    |
| 010    | SUBB Rd, Rs    |                    |
| 011    | SUB Rd, Rs     |                    |
| 100    | PASS Rd        | **LDI Rd, #imm**   |
| 101    | INC Rd         |                    |
| 110    | DEC Rd         |                    |
| 111    | PASS Rd        |                    |

### Four-State FSM

A one-hot ring counter of four D flip-flops. The rightmost flip-flop (Inst_Fetch) is initialized to `1`; all others default to `0`. On each rising clock edge the single `1` shifts one position to the left, so exactly one state signal is active at any time.

| State     | Inst_Fetch | IR_ctrl | Write_Back | What Happens                                                       |
| :-------- | :--------: | :-----: | :--------: | :----------------------------------------------------------------- |
| FETCH     |     1      |   01    |     0      | IR captures instruction from memory at PC address                  |
| DECODE    |     0      |   00    |     0      | Splitters decode fields; register file provides operands           |
| EXECUTE   |     0      |   00    |     0      | ALU computes combinationally                                       |
| WRITEBACK |     0      |   00    |     1      | Selected register stores ALU output; PC increments                 |

---

## Test Results

Running `test_8bit_processor.txt` through Digital's **Analysis → Test** menu produces a `passed` result across all 152 reference lines.

### Programming Phase

The test writes 7 instructions into memory while holding `InstrMem_ctrl = 1`:

| Address | Instruction | Encoding | Purpose                        |
| :-----: | :---------- | :------: | :----------------------------- |
|    0    | INC R1      |   0xA8   | R1 ← R1 + 1                    |
|    1    | INC R1      |   0xA8   | R1 ← R1 + 1                    |
|    2    | INC R1      |   0xA8   | R1 ← R1 + 1                    |
|    3    | INC R1      |   0xA8   | R1 ← R1 + 1                    |
|    4    | INC R2      |   0xB0   | R2 ← R2 + 1                    |
|    5    | INC R2      |   0xB0   | R2 ← R2 + 1                    |
|    6    | SUB R1, R2  |   0x6C   | R1 ← R1 − R2                   |
|    7    | (empty)     |   0x00   |                                |


### Run Phase

After programming, 40 run cycles execute the program. Expected behavior:

- R1: 0 → 1 → 2 → 3 → 4 → 2  *(four increments, then SUB R1, R2)*
- R2: 0 → 1 → 2
- R0, R3: unchanged at 0

The trace confirms every register write occurs only during the WRITEBACK state, the `IR_ctrl` signal pulses to 1 exactly once per instruction at the start of the FETCH state, and the PC wraps from 7 back to 0 as expected.

---

## Key Design Decisions

- **Opcode → ALU control (no decoder).** The ISA is defined so that each 3-bit opcode value is the corresponding 3-bit ALU control code `{S1, S0, Cin}`. The three opcode bits are wired directly to the ALU with no intermediate logic.
- **Single zero-extension mux for I-type.** A 2-to-1 mux selected by the `type` bit picks between `RegFile.Read_Data1` (R-type) and a zero-extended `{6'b0, op2}` (I-type) as ALU input A. This is all the extra hardware LDI requires.
- **One-hot ring counter FSM.** Four D flip-flops in a ring with one default-1 flip-flop provide a dead-simple state machine where each control signal is just a direct flip-flop output — no state-decode logic needed.
- **Write gating at two levels.** The register-file write bus is built from a 4-to-1 "which register" mux feeding a Write_Back-gated 2-to-1 mux; outside WRITEBACK every register's control bits are forced to `00` (hold).
