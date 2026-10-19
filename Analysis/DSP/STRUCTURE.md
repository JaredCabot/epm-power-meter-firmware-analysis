# Measurement-board DSP image

`Firmware/E4418-9A EPM-441-2A EPROM DSP Rev A.01.11 E4418-80023.bin`, 65536 bytes.
The DSP lives on the measurement board, separate from the main firmware. The image
structure, interrupt vector table and full code disassembly (`listing.asm`) are
established and checked against the image.

Regenerate with `python Tools/dsp_analyze.py` (uses `Tools/adsp2101.py`).

## Byte order (confirmed)

Each 24-bit word is stored in 4 bytes: `word = b0<<16 | b1<<8 | b2`, big-endian, with b3
a fourth byte that is 0xFF over code and otherwise mostly 0x00. This is the only ordering
of the four bytes under which `0x0A000F` (RTS) appears at all; every other ordering yields
zero RTS. 65536 bytes = 16384 words = eight 2048-word (0x800) boot pages, matching the
ADSP-2101's eight-page boot structure.

## Page map

| Page | Words | Bytes | Content |
|---|---|---|---|
| 0 | 0-2047 | 0x00000 | Boot page: 11 words of code at the start (PM 0x0000) plus a small table/code block around word 1267; the rest empty. Loaded by the on-chip boot logic at reset. |
| 1 | 2048-4095 | 0x02000 | Main code, ~2021 words. Starts with the interrupt vector table; all 17 RTS/RTI are here. Loaded at PM 0x0000 (overlays page 0). |
| 2-6 | 4096-14335 | 0x04000 | Empty (all zero). |
| 7 | 14336-16383 | 0x0E000 | Empty up to byte 0xF0EF; bytes 0xF0F0 to 0xFFFF are the XC3042 gate-array bitstream (see "Page 7: gate-array bitstream"). No DSP code. |

So the DSP firmware is page 1; page 0 is the bootstrap that brings page 1 in as an overlay.
Pages 2-6 are unused, and the top of page 7 belongs to the gate array, not the DSP.

### Boot sequence (page 0)

The ADSP-2101 auto-loads page 0 into PM at reset and runs from PM 0x0000. The first 11 words
are a loader (`page0_boot.txt`):

```
M4 = 0x3fff (-1)      ; decrement modifier
L4 = 0, L5 = 0        ; linear addressing
I4 = 0x07ff           ; PM source = top of page 0
I5 = 0x3b0e           ; DM destination
CNTR = 0x030d (781)
DO ... UNTIL CE: AX1 = PM(I4,M4); DM(I5,M4) = AX1   ; copy 781 words PM->DM, descending
SI = 0x065f
DM(0x3fff) = SI       ; write the System Control Register
```

It copies a 781-word data/coefficient block from the top of page-0 program memory (PM
0x04F3-0x07FF, which disassembles as data, not code; only the upper 16 bits of each PM word
reach DM) down into data memory at 0x3802-0x3B0E,
then writes 0x065F to the ADSP-2101 System Control Register at DM 0x3FFF: BFORCE (bit 9) set,
BPAGE (bits 8-6) = 1, BWAIT = 3, PWAIT = 7. That write forces the boot load of page 1 (the main
overlay), which then runs from PM 0x0000. In each page the fourth byte of word 0 is the boot
length field (0xFF on page 0, 0xFE on page 1).

### Page 7: gate-array bitstream

Bytes 0xF0F0 to 0xFFFF (3,856 bytes) are not DSP data. Read from 0xFFFF downward, taking each
byte least significant bit first, they form an XC3000-family configuration stream:

- eight 1s, the preamble `0010`, a 24-bit length count of 30,825, and `1111`;
- 285 frames of 108 bits, every one starting with 0 and ending with 111 (the XC3042 layout,
  285 x 108 = 30,780 bits);
- a postamble of 1s, then unused bytes at the low end of the block.

The CLIP puts the XC3042A (1821-1672, U20) on the same measurement board as this flash
(E4418-80023, U13). The address and bit order match the XC3000 master-parallel-down mode, in
which the gate array reads a byte-wide PROM from the top address down at power-up, so the gate
array most likely loads itself from this flash while the DSP boots from the bottom pages. The
mode-pin wiring has not been checked on the schematic. The Version screen reports this part as
"Gate Array Rev A.01.03" on each channel.

To extract the bitstream in normal (first-bit-first) order: take bytes 0xF0F0 to 0xFFFF, reverse
the byte order, and reverse the bits within each byte.

## Interrupt vector table (page 1, PM 0x0000)

ADSP-2101 vectors are four words apart. The first entries:

| PM | Words | Note |
|---|---|---|
| 0x0000 | 1c7e2f 1c7e5f 1c01cf 18003f | restart: CALL 0x7e2, CALL 0x7e5, CALL 0x01c, JUMP 0x003 |
| 0x0004 | 185a1f 000000 000000 000000 | IRQ2: JUMP 0x5a1 |
| 0x0008 | 0a001f ... | RTI (unused interrupt) |
| 0x000c | 9f96d8 18581f 000000 000000 | JUMP 0x581 handler |
| 0x0010 | 0a001f ... | RTI |
| 0x0014 | 0c0030 1c5ccf 0c0020 0a001f | handler then RTI |

JUMP (`0x18xxxx`) and CALL (`0x1cxxxx`) targets all land inside the 0-0x7ff code page,
which confirms the 14-bit direct-address field (bits 17-4) and the 4-bit condition (bits
3-0, 0xF = always). RTS = `0x0A000F`, RTI = `0x0A001F`.

## Disassembly

`listing.asm` is the full disassembly of the code page, with labels for every JUMP/CALL/DO
target (`sub_xxxx` for call targets, `L_xxxx` otherwise). `Tools/adsp2101.py` is the
disassembler; its opcode map, tables and bit fields are ported from MAME's ADSP-2100
disassembler (`2100dasm.cpp`). Every word in the code page decodes to a known instruction
(no unknown opcodes), which confirms the opcode table against the image. Every instruction
type is decoded, including the shift group, `MODIFY`, `DIVS`/`DIVQ`, `IDLE`, stack and
mode-control ops. Seven words, all `0x0D03CF`, print as `MR1 = ??` because they read group-3
register 15, which the 2101 documentation leaves undefined. They are code. Each starts a routine
(at 0x0049, 0x00FE, 0x016D, 0x0402, 0x0579, 0x059A, 0x05C0) whose next instructions push MR1 to a
software stack, and whose exit does `I6 = MR1; JUMP (I6)` with no RTS. So the instruction pops
the return address off the hardware PC stack, the operation later ADSP-21xx parts name
TOPPCSTACK. That reading is inferred from the prologue and epilogue; the 2101 documentation
does not define the register.

0x0049, 0x00FE and 0x0402 are reached by CALL, and 0x016D by JUMP from 0x0048. The other three
prologues are dead code: the interrupt vectors enter those routines below the prologue (at
0x0581, 0x05A1 and 0x05CC), and those paths leave by RTI or RTS.

The code is a sensor-acquisition loop: it polls memory-mapped hardware status, runs ALU/MAC
filtering, and exchanges commands and results with the main CPU through a data-memory
mailbox. `histogram.txt` has the instruction-mnemonic frequency.

## Data-memory map and the main-CPU mailbox

The DSP's data memory splits into memory-mapped board hardware (low addresses) and the
on-chip RAM used as scratch and as the mailbox (0x3800 and up):

| DM address | Use (from the code) |
|---|---|
| 0x0002, 0x0006, 0x0007 | Board hardware status, polled with AND masks in the acquisition loop |
| 0x0800, 0x0803 | Board hardware control/command registers |
| 0x3800-0x380x | Circular sample buffer (`I2`/`L2` set up in `sub_001c`) |
| 0x3813 | Set at init (`DM(0x3813) = SR1`) |
| 0x3815 | Command/parameter from the main CPU. The DSP reads it at PM 0x0249, tests it and latches it to 0x3847. This is the location `DIAGnostic:DETector:THIRd` writes (0x105) over bus command 0xF804. |
| 0x381e, 0x381f | Status/handshake the DSP writes back; the main CPU reads these over bus command 0xF002. |
| 0x3844-0x388f | Working and result area |

So the measurement-board bus commands in `../NOTES.md` land here: 0xF804 writes a DSP DM word
(e.g. the 0x3815 command), 0xF002 reads one (e.g. the 0x381e/0x381f status), and
`DIAGnostic:DSP`/`DIAGnostic:DSP?` expose that directly.

## Still open

- The meaning of the 781-word block that page 0 copies to DM 0x3802-0x3B0E.
- The gate array's mode-pin wiring, which would confirm master-parallel-down configuration.
