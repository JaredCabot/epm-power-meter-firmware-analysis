# Boot ROM

Read out of the E4419B (S/N MY45103686) over GPIB on 18 October 2026 with
`DIAGnostic:PEEK:BLOCK?`, using `Tools/dump_bootrom.py`. Two passes, identical (SHA-256
`2956A2AFC4FBBA59FA9BD606F9E99922AE0480BA0D7B1C5BED91F22170E1F504`). The
image is `Firmware/bootrom-A.02.00-MY45103686.bin` (128 KB, address 0x000000 to 0x01FFFF); a
linear-sweep disassembly and the strings are in `BOOTROM/`.

## Identification

- `Bootrom Revision A.02.00` (0x0C00), the same value `DIAGnostic:VERSion:BOOT?` returns.
- `@(#)LDS Rev: 2.03 ; Bootrom Rev 1.02` and `Copyright 1988-1996, Hewlett-Packard Company, all
  rights reserved.` (0x5338 to 0x53A0).
- Code and data run from 0x0000 to 0xA1AF. The rest of the 128 KB is erased (0xFF).
- The rest of the flash, 0x020000 to 0x0FFFFF, was read the same way: it is the A2.09.01 download
  image byte for byte, with 0xFF above 0x0F1D5B (BENCH-VERIFICATION.md).
- The boot ROM lives in the bottom of the same two 4 Mbit flash chips as the main firmware
  (E4418-80019 and -80020 in the CLIP, one chip per byte lane). Its flash programmer recognises
  three device IDs: SGS-Thomson 29F040, Fujitsu 29F040A and AMD 29F040.

## Reset and vectors

The reset vectors at 0x0000 are SSP 0x00000000 and PC 0x00004E18. The reset code sets VBR to
0x0000000A, so the boot ROM's exception table starts at 0x000A: vector 2 (bus error) is at
0x0012 and holds 0x040E, and so on. Each vector points at an 8-byte stub at 0x040E + 8n that loads
the vector number into D0 and jumps through the user stack pointer, which the boot code sets to
the current handler. The main firmware calls in through `TRAP #0` (monitor), `TRAP #11` (VBR and
status register) and `TRAP #14` (services such as the version pointer and service 10, download;
see NOTES.md).

## Power-up decision (0x0F4C)

1. FUN 0x0D24 scans four front-panel key rows (3, 1, 0, 2) through the port table in boot RAM
   and tests bit 15 of the column read-back (0x4BB8) for each:

   | Key held at power-up | Action |
   |---|---|
   | row 3 | "Waiting for HPIB download..." |
   | row 1 | "Clearing memory from 0x%x to 0x%x..." (wipes RAM, including the battery-backed area) |
   | row 0 | "Waiting for RS232 download..." |
   | row 2 | "Waiting for RS422 download..." |

   Which physical keys sit on these rows has not been identified; the CLIP shows the
   KEYROW0-4 / KEYCOL0-3 matrix lines but not the key positions. Do not experiment with held keys
   without knowing which one is the memory-clear key.
2. FUN 0x0CBE polls the console up to 1,000 times. ESC (0x1B) or Ctrl-F (0x06) received in that
   window prints "Starting boot monitor..." and enters the ROM monitor.
3. Otherwise it prints "Starting main firmware..." and jumps to 0x020000.

The console is the 68332's on-chip SCI (registers 0xFFFC08 to 0xFFFC0F), not the 16550 UART the
main firmware uses for its RS-232/RS-422 interface. The SCI pins leave the processor board on
P4-D20/E20 as B_RS232_RXD/B_RS232_TXD; where they come out has not been traced. Baud rate: not
yet determined.

## ROM monitor

Prompt "ROM Monitor / Enter ? for help." Commands (table at 0x1194):

| Command | Help text |
|---|---|
| `bootvars` | display bootrom variables |
| `bc [<hex boot config>]` | set the bootrom configuration |
| `gu [<hex start addr>]` | go to start address |
| `bs` | force a breakpoint when starting |
| `gbreak` | force a gdb breakpoint |
| `gdb` | enable gdb trapping of exceptions |
| `dlong`, `dword`, `dbyte`, `dmem [<hex start> [num bytes]]` | display memory as longs, words or bytes |
| `slong`, `sword`, `sbyte`, `smem <hex start> <hexchars>` | set memory as longs, words or bytes |
| `hmon [device]` | download into memory |
| `version` | display bootrom version |

`bootvars` prints bootromRevision, compositeErrors, bootConfig, the boot ROM's data/BSS/stack
addresses and four RAM test ranges with their error counts. The boot-config store (`bc`) is
disabled in this build: its area start and end (0x9854, 0x9858) are both 0, so any write ends in
"bootConfigAreaSize is zero". A gdb remote stub is included ("gdb communication enabled", read and
write memory packets, "bad checksum" handling). The main firmware enters the monitor with
`DIAGnostic:MONitor` (TRAP #0) and from Ctrl-C on its console.

## Download format (0x65DA)

The loader reads two-character record tags from the selected port. The power-up download paths
run the monitor command lines `hm HPIB`, `hm SERIAL_RS232` and `hm SERIAL_RS422` (that is, `hmon`
with a device name); the main firmware's `DIAGnostic:DOWNload:FW` commands pass the same strings
when they hand over through `TRAP #14` service 10.

| Tag | Payload | Action |
|---|---|---|
| `WL` | u32 start, u32 end (inclusive), then end - start + 1 data bytes | Write and load. Addresses in flash go through the flash programmer ("Programming..."); others are copied to RAM. |
| `GU` | u32 address | Run address, kept with the download state. |
| `H0` | none | End of download ("Terminating due to H0"). |

Other tags are skipped. So the `.dload` files are a `WL` record for the code, a `WL` record for the
initialised data and the `H0` terminator, which is the `48 30` trailer.

## Self-check

The boot ROM has a table-driven checksum routine (0x53C2 onward) that sums every Nth byte for N
byte lanes and compares the result with a stored 16-bit value, the same scheme as the header at
0x020030 that the main firmware's ROM Checksum test uses (NOTES.md). Its messages are "ROM
Checksum Failure. Bad Checksum / Bad Table / Bad ROM Id / Bad ROM size" (0x5624 to 0x56D9).
Which tables it is run against at power-up has not been traced.
