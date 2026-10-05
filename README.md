# QuaOS — about the project

A hand-written educational operating system: a bootloader, a kernel and a command
shell built from scratch, with no third-party libraries. It runs on two
architectures — **x86 (32-bit)** and **ARM (Cortex-A15, QEMU `virt`)** — and
is launched in the QEMU emulator.

## What it is written in

| Language | Where | Size |
|---|---|---|
| **C** (freestanding, no libc) | the kernel: scheduler, file systems, shell, drivers | ~4,500 lines (35 files) + ~970 lines of headers |
| **x86 assembly (NASM)** | bootloader, kernel entry, interrupts, context switching, ring3 | ~560 lines (7 files) |
| **ARM assembly (GAS, `.S`)** | startup, vector table, context switching | ~130 lines (3 files) |
| **Linker script (`.ld`)** | kernel memory layout (x86 — at the 1 MB address, ARM — its own) | 2 files |
| **Make / Bash / Batch** | build and launch (`Makefile`, `build_and_run.sh`, `run.bat`, `run_wsl1.bat`) | — |

Build tools: `nasm`, `gcc -m32`, `ld` (x86); `arm-none-eabi-gcc` (ARM);
launched with `qemu-system-i386` / `qemu-system-arm`.

## What it can already do

- **x86 boot:** its own bootloader (stage1): A20, the BIOS memory map (E820),
  protected mode, relocating the kernel to 1 MB.
- **Memory:** a physical page manager (bitmap), paged memory (paging),
  its own `kmalloc`/`kfree` (free-list).
- **Interrupts and hardware (x86):** GDT/IDT, PIC, a 100 Hz PIT timer, PS/2 keyboard
  and mouse, VGA text mode with scrolling, COM1, CMOS RTC, PCI, ATA/IDE (PIO),
  ACPI (reboot / shutdown).
- **Multitasking:** a preemptive timer-driven round-robin scheduler, each
  task has its own kernel stack.
- **Processes:** ring3 (user mode), `int 0x80` system calls,
  a separate page directory per process; a crash of a user process does not
  kill the kernel.
- **File systems:** a VFS layer, a RAM disk, initrd, and its own **QuaFS**
  (flat, up to 64 files, name up to 23 characters) on an ATA disk.
- **Shell:** a line editor (cursor, history, Home/End/Delete), colored output.
  Commands mentioned in the documentation and history: `help`, `clear`, `echo`,
  `uptime`, `reboot`, `shutdown`, `date`, `lspci`, `format`, `ls`, `write`, `cat`,
  `rm`, `ps`, `disktest`, `diskdump`, `ptest`, `proctest`, `frames`.
- **ARM port:** PL011 UART, GICv2, the ARM generic timer, PSCI (reset/off),
  an interactive shell and multitasking. The console is right in the terminal (UART).

## What is not there yet

- ARM has no paging/PMM, ring3 or system calls (stubs).
- QuaFS: no directories, and deleting a file does not free space on the disk.
- No networking, graphics mode or sound.

## The architecture in one picture

```
boot/boot.asm (x86 only)  ->  kernel/arch/<arch>/  (entry + drivers)
                                        |
                                        v
                    kernel/src/kernel.c : kernel_main()   (common code)
                    shell, task, kheap, kprintf, VFS, QuaFS
                                        |
                          hal_* interface (kernel/hal/hal.h)
                                        |
                         x86 (kernel/arch/x86)  |  ARM (kernel/arch/arm)
```

The main rule: the common code in `kernel/src/` does not touch hardware directly, only
through the functions from `kernel/hal/hal.h`. See `docs/architecture.md` and
`docs/hal-design.md` for details.

## Folder structure

```
boot/                 the x86 bootloader
kernel/src/           common code (shell, scheduler, heap, file systems)
kernel/include/       common headers
kernel/hal/hal.h      the HAL interface
kernel/arch/x86/      everything x86-specific
kernel/arch/arm/      everything ARM-specific
docs/                 documentation (architecture.md, hal-design.md)
```

## How to build and run

| Method | File | When |
|---|---|---|
| WSL2 + a QEMU window from WSL | `run.bat` (`run.bat arm` — ARM) | WSL2 works (needs "Virtual Machine Platform") |
| WSL1 for building + QEMU for Windows | `run_wsl1.bat` | WSL2 is unavailable; needs `qemu-system-i386.exe` for Windows |

Manually in WSL: `make`, `make run`, `make run-headless`, `make arm`, `make run-arm`.

## Run logs

The `logs/` folder next to the scripts: `last-build.log` (the full output),
`last-errors.log` (errors only), `last-qemu.log` (QEMU's internal log),
`last-arm-serial.log` (kernel output to the UART, ARM only), `run-bat.log`
(the actions of the `.bat` itself). The 20 most recent runs are kept.

## Development history (git, 12 commits)

Stage 6 (HAL + ARM + ACPI/PSCI + documentation) → ATA/IDE, RTC, PCI, a terminal with
colors and scrolling → QuaFS v0 → Phase 0 (mouse, VFS) → Phase 1 (open/read/
write/close, RAM disk, initrd) → Phase 2.1 (preemptive multitasking) →
Phase 2.2 (separate process address spaces, the current stage).
