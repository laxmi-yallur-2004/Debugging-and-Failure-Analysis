# Debugging & Failure Analysis on STM32F401 (ST-Link SWD)

Five deliberately injected bugs on an STM32F401CCU6 Black Pill. Each bug causes a real Cortex-M fault. The fault handler captures the crash details, and after an ST-Link SWD reset the board reports the cause over USB serial.

## How It Works

1. **Inject:** send `1` to `5` over the serial monitor. The matching buggy function runs.
2. **Detect:** the CPU raises a fault (HardFault, BusFault or UsageFault). A common handler, `Fault_Capture`, reads the stacked PC and LR plus the CFSR and HFSR fault registers.
3. **Save:** the details are stored in `.noinit` RAM, which survives a reset.
4. **Recover:** the handler does not reset by itself. If an ST-Link debugger is attached it halts with a breakpoint; otherwise the LED blinks and it waits. Recovery is an ST-Link SWD reset.
5. **Report:** after reset, `setup()` sees the saved magic value (`0xDEADBEEF`) and prints the full fault report.

## Hardware

- Board: STM32F401CCU6 Black Pill
- Debug and programming: ST-Link over SWD (PA13 SWDIO, PA14 SWCLK)
- Serial: USB serial, 9600 baud (read-only reporting)
- LED: PA5 (blinks every second; keeps blinking in fault state)

## The 5 Bugs

| # | Bug | What is wrong in the code | CFSR | Fault |
|---|-----|---------------------------|------|-------|
| 1 | Loop Overrun | 200 writes into a 10-byte stack buffer | `0x100` | Instruction bus error |
| 2 | Integer Overflow | `uint8_t` 200+56 wraps to 0, then 100/0 | `0x2000000` | Divide by zero |
| 3 | Division By Zero | 100 divided by `time = 0` | `0x2000000` | Divide by zero |
| 4 | Null Pointer | Write through a pointer set to address 0 | `0x10000` | Undefined instruction (trap) |
| 5 | Unaligned Access | 32-bit write to `0x20000001` | `0x1000000` | Unaligned access |

## Test Results

| Bug | PC | LR | CFSR | HFSR |
|-----|----|----|------|------|
| 1 | `0x64646464` | `0x80004B9` | `0x100` | `0x0` |
| 2 | `0x800027C` | `0x80004B3` | `0x2000000` | `0x0` |
| 3 | `0x80002A8` | `0x80004B3` | `0x2000000` | `0x0` |
| 4 | `0x80002CA` | `0x80004B3` | `0x10000` | `0x0` |
| 5 | `0x80002E4` | `0x80004B3` | `0x1000000` | `0x0` |

Total faults counted: 5. The board recovered after every fault.

Sample report:

```
=== RECOVERED FROM FAULT ===
[FAULT] BUG CODE=3
[ROOT CAUSE] Division By Zero - time=0
[FAULT] PC=0x80002A8
[FAULT] LR=0x80004B3
[FAULT] CFSR=0x2000000
[FAULT] HFSR=0x0
[INFO] Total Faults=3
[INFO] RECOVERY: Recovered via SWD Reset
```

## Notes on the Results

- **Bug 1:** the PC is `0x64646464`. `0x64` is 100, the value the loop writes, so the overrun overwrote the saved return address. When the function returned, the CPU jumped to that address and could not fetch code. This is direct proof of a stack overflow.
- **Bugs 2 and 3:** different root causes (silent overflow vs. direct zero divisor) but the same fault type. Their PCs differ, showing they happened in different functions.
- **Bug 4:** the fault is an undefined-instruction trap with the PC inside the null-pointer function. This is most likely a trap inserted by the compiler for a write to address 0, not a hardware bus error. Confirm by disassembling the `.elf` and checking for a `udf` instruction at the PC.
- **Bug 5:** needs the unaligned-access trap enabled (`UNALIGN_TRP`), which `setup()` does.
- **Debugger check:** the breakpoint only runs when a debugger is attached (`CoreDebug->DHCSR & C_DEBUGEN`). Without this check the breakpoint causes a second HardFault that overwrites PC and LR with wrong values (`LR=0xFFFFFFF9`, `HFSR=0x80000000`).
- **Find the buggy line:** `arm-none-eabi-addr2line -e yourfile.elf -f -C 0x<PC>`

## How to Run

1. Open `debug_fault_analysis.ino` in Arduino IDE with STM32 board support (Generic STM32F4 series, STM32F401CCU6).
2. Upload with ST-Link (SWD).
3. Open the Serial Monitor at 9600 baud.
4. Send `1` to `5` to inject a bug.
5. Reset the board through ST-Link SWD to see the fault report.
