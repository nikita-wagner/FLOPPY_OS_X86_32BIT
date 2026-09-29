# Floppy OS (x86, 32-bit)

> ⚠️ **Work in progress.** This is an unfinished, educational bootloader project, not a usable operating system.

A multi-stage x86 bootloader written in NASM assembly. It boots from a 1.44 MB floppy image, loads further stages from disk with BIOS interrupts, verifies them with a simple XOR checksum, and can switch the CPU from 16-bit real mode into 32-bit protected mode. The project ships with an interactive GDB debugger script for stepping through the boot process in QEMU.

## Features

- **Multi-stage loader**: the 512-byte boot sector loads stages `B0`–`B5` from the floppy via `int 0x13` and jumps to them one at a time (press a key to advance).
- **Stage hashing (`B0`, "sumchecker")**: XOR-folds the boot sector and the following stages into 16-bit hashes, prints them in hex, and compares each against the checksum word stored at the end of its sector (`OK` / `NOT`).
- **Protected mode (`B4`)**: enables the A20 line (fast method, keyboard-controller fallback), loads a GDT with flat 4 GB code/data segments, sets `CR0.PE`, far-jumps into 32-bit code, and writes text directly to VGA memory (`0xB8000`).
- **Interactive debugger (`debugger.gdb`)**: a key-driven GDB menu for single-stepping, dumping registers, setting breakpoints, and reading or writing memory while QEMU is halted at `0x7C00`.
- **Notes in `description/`**: write-ups on floppy geometry, CHS sectors, the stack pointer, registers, and string instructions.

## Status

| Area | State |
| --- | --- |
| Boot sector, sequential stage loading | Working |
| XOR checksum verification of stages | Working, but the checksum algorithm is basic and the code is due for cleanup |
| Real → protected mode switch | Working |
| Protected → real mode return | **Not implemented** (`B4` currently does `jmp 0x7C00` from 32-bit code) |
| Stages `B1`, `B2`, `B3`, `B5` | Placeholders that only print their name |
| Kernel / filesystem / drivers | Not started |

## Requirements

Install [MSYS2](https://www.msys2.org/) and use the **UCRT64** shell. Then:

```bash
pacman -S nasm qemu gdb
```

The debugger script uses `msvcrt` and `mintty`, so it currently works on **Windows only**.

## Quick Start

```bash
git clone https://github.com/NikitaKonkov/FLOPPY_OS_X86_32BIT.git
cd FLOPPY_OS_X86_32BIT
chmod +x build.sh

./build.sh -n    # build and run in QEMU
./build.sh -d    # build and run in QEMU, then open the GDB debugger
```

`build.sh` assembles every stage with NASM, creates a blank 2880-sector image (`floppy.img`), and writes each binary to its sector with `dd`. It requires either `-n` or `-d`.

### Debugger keys

QEMU starts paused (`-s -S`) and GDB attaches on `localhost:1234`, stopping at `0x7C00`.

| Key | Action |
| --- | --- |
| `s` | Step one instruction (prints it first) |
| `f` | Continue |
| `r` / `a` | Show main / all registers |
| `b` | Set a breakpoint at an address |
| `m` | Examine memory (address, length, format) |
| `w` | Write memory (byte, half-word, word, giant word) |
| `c` | Clear the screen |
| `q` | Quit the menu |

## Disk Layout

| Sector (LBA) | File | Loaded at | Purpose |
| --- | --- | --- | --- |
| 0 | `boot.asm` | `0x7C00` (by BIOS) | Boot sector: loads stages and dispatches to them |
| 1 | `B0.asm` | `0x8000` | Hash checker |
| 2 | `B1.asm` | `0x8200` | Placeholder (prints `B1`) |
| 3 | `B2.asm` | `0x8400` | Placeholder (prints `B2`) |
| 4–5 | `B3.asm` | `0x8600` | Placeholder (prints `B3`) |
| 6–7 | `B4.asm` | `0x8A00` | A20, GDT, protected-mode switch |
| 8–9 | `B5.asm` | `0x8E00` | Placeholder (prints `B5`) |

Stages are loaded to `0x7C00 + 512 × (LBA + 1)`, and the BIOS reads them by 1-based CHS sector number (LBA + 1). The sector, count, and address tables live in the data section of `boot.asm`. The 512 bytes at `0x7E00` are never loaded, so `B0` hashes them as "empty".

## Project Structure

```
.
├── boot.asm            # Stage 1: boot sector and stage loader
├── B0.asm – B5.asm     # Stages 2–7
├── build.sh            # Build, image creation, QEMU / GDB launcher
├── debugger.gdb        # Interactive GDB menu
├── sc.py               # CHS ↔ linear sector helper
├── description/        # Notes on floppy, registers, stack, strings
├── Floppy_disk_structure.gif
└── floppy.img          # Generated disk image
```

## How It Works

1. The BIOS loads sector 0 to `0x7C00` and jumps to it.
2. `boot.asm` waits for a key, sets up the stack, and reads the stages into RAM.
3. On each pass it jumps to the next stage in its table. On the first pass this is `B0`, which prints the hashes and returns to `0x7C00`.
4. Each stage prints its marker and returns control to the boot sector, which then launches the next one.
5. `B4` prepares and enters 32-bit protected mode.

## Known Limitations

- The 16-bit XOR checksum is only a basic integrity check, and several sector values are hard-coded.
- Stage return paths and the checker (see the comments at the end of `B0.asm`) need a rewrite.
- Only tested on QEMU, not real hardware.

## Author

Created by **Nikita Konkov** as a learning project in x86 assembly and bootloader development.
