# Analysis notes

Findings verified against the code so far. Addresses are E4418B unless marked. Anything not yet confirmed is listed under "Open items".

## Image layout

- A `.dload` file is a sequence of load records, each a 10-byte header (`WL`, u32 load address, u32 inclusive end address) followed by the data, and ends with a 2-byte trailer (`48 30` in both files). Both files hold two records: the code at 0x020000 and the initialised data segment straight after it (E4418B 0x0E9170 to 0x0F462B, E4419B 0x0E9BC4 to 0x0F1D5B). Load them with `Tools/dload.py`. The first pass read the file as one block from offset 0x0A, which put the whole data record 10 bytes too high; see REVIEW.md.
- The image checks itself. A header at 0x020030, written byte-for-byte to both flash chips, gives the device count (2), the image length in words (0x6A316 on the E4418B, 0x68EAE on the E4419B) and, at 0x020054, a 16-bit sum of each byte lane (high lane 0x9EB5 and low lane 0x9D5D on the E4418B; 0x4774 and 0xF963 on the E4419B). The ROM Checksum self test (0x0AC1D2, worker 0x0AFC9A) recomputes the lane sums from 0x020000 to 0x020000 + 2 x length, which is exactly the end of the data record. Both sums match the image built by `Tools/dload.py` and fail on the flat load.
- The 2-byte trailer is the ASCII record tag `H0`, the boot ROM loader's end-of-download record ("Terminating due to H0"). The loader also knows a `GU` (run address) record. See BOOTROM.md.
- Entry: 0x020004 jumps to 0x020084, which calls 0x0202C4 (data segment setup) and then 0x020598 (stack at 0xE7FFF0, start pSOS).
- Code runs from 0x020000 to about 0x0E9000. The SCPI parser tables sit between 0x043000 and 0x05B000.
- Initialised data is copied from ROM to RAM at boot:

| Model | ROM source | RAM range |
|---|---|---|
| E4418B | 0x0E9170 | 0xE05000 to 0xE104BB |
| E4419B | 0x0E9BC4 | 0xE05000 to 0xE0D197 |

  BSS follows. A RAM address A has its initial value at ROM (A - 0xE05000 + ROM source). The data record starts with an image of the battery-backed header. When its first long reads 0x12345678, 0x0202C4 copies from ROM source + 0x7E to RAM 0xE0507E, so the live header at 0xE05000 to 0xE0507D survives a firmware download; otherwise it copies 0xB4BC bytes from 0x0EA000 to 0xE05000.
- Battery-backed RAM check at 0x0202C4: the magic long at 0xE05000 (0x12345678 / 0x87654321) plus the build date and time strings decide between "first time power-up" and "not first time". A separate marker, 0x18273645 at 0xE0507A, flags the first boot after a download onto older "MII" firmware.
- Source module names appear in assertion strings: `disp.c`, `channel.c`, `stingray.c`, `isb_driver.c`, `sed1330.c`, `ddt.c`, `saveRecall.c`, `tpu_timer.c`, `scpi_diag.yuc`, `fpanel.yuc` and others. The build path is `/proj/merlin/...`; "merlin" is the project codename.
- Debug output: 0x0E3F88 is fprintf to the stream at *(*(0xE104B8)+0x0C). Handlers print diagnostic text there as well as returning the SCPI response.

## DSP image

Full structure, vector table, memory map and disassembly are in `DSP/STRUCTURE.md` and
`DSP/listing.asm` (regenerate with `Tools/dsp_analyze.py`). Summary:

- 64 KB = 16384 words = eight 2048-word boot pages. Word = `b0<<16 | b1<<8 | b2`
  (big-endian), b3 a fourth byte (0xFF over code). This byte order is the only one that
  produces any RTS (`0x0A000F`); RTI = `0x0A001F`.
- Page 0 is the boot page (sparse: ~11 words of code at PM 0x0000 plus a small block near
  word 1267). Page 1 (words 2048-4095) is the main code, ~2021 words, loaded at PM 0x0000 as
  an overlay; it opens with the interrupt vector table and holds all the RTS/RTI. Pages 2-6
  are empty. Bytes 0xF0F0 to 0xFFFF at the top of page 7 hold the configuration bitstream for
  the measurement board's XC3042 gate array (see "Measurement-board gate array" below).
- JUMP = `0x18xxxx`, CALL = `0x1Cxxxx` (14-bit direct address in bits 17-4, 4-bit condition
  in bits 3-0; 0xF = always).
- The code page is fully disassembled in `DSP/listing.asm` (disassembler `Tools/adsp2101.py`,
  opcode table ported from MAME). It is a sensor-acquisition loop. The main CPU exchanges
  commands and results with it through a data-memory mailbox at DM 0x3800+: the main CPU
  writes commands with bus command 0xF804 (e.g. 0x3815, which `DIAG:DETector:THIRd` sets to
  0x105 and the DSP reads at PM 0x0249) and reads status with 0xF002 (e.g. 0x381e/0x381f).
  Low DM (0x0002/0x0006/0x0007 status, 0x0800/0x0803 control) is the board hardware. Full map
  in `DSP/STRUCTURE.md`.
- Seven subroutines open with `0x0D03CF`, a register move from group-3 register 15 into MR1.
  The matching epilogue does `I6 = MR1; JUMP (I6)`, so the instruction pops the return address
  off the PC stack (the register later ADSP-21xx parts call TOPPCSTACK). It is code, not data.

### Measurement-board gate array

The CLIP places the XC3042A (HP 1821-1672, U20) and the DSP flash (E4418-80023, U13) on the
same board, the Average CCA E4418-60003, which is the A6 measurement assembly (one per
channel). The last 3,856 bytes of the DSP flash (0xF0F0 to 0xFFFF) are the gate array's
bitstream. Read from 0xFFFF downward, least significant bit first, they give an XC3000-family
serial stream:

- eight 1s, then the preamble `0010`, a 24-bit length count of 30,825 and `1111`;
- 285 frames of 108 bits, each starting with 0 and ending with 111 (285 x 108 = 30,780, the
  XC3042 frame layout);
- a postamble of 1s.

The schematic (CLIP page 66) confirms the mode. U20's mode pins are M0 (pin 54) on net FPGA_M0,
pulled to ground by R71-E (4.7K), and M1 (pin 52) and M2 (pin 56) on pull-ups PUP9 and PUP8.
M0 M1 M2 = 0 1 1 is the XC3000 master-parallel-down mode: the gate array drives its own address
lines from 0xFFFF downward and reads a byte at a time. Its A0-A11 sit on the DSP address bus
(ADDR0-ADDR11) and D0-D7 on DATA8-DATA15, the same lines the flash uses, so the gate array loads
itself from the top of the shared flash at power-up with no help from either processor. CCLK
(pin 2) goes to FPGA_CCLK and DONE to D_NP. An unfitted header, J21, brings out
CCLK, D/P, DIN and FPGA_M0, so a download cable could override M0 and load the part serially.

The Version screen's per-channel "Gate Array Rev A.01.03" does not come from the gate array. The
DSP sends it, from its own initialised data (see the DSP section).

## Memory map (as used by the main firmware)

The boot ROM sets up the MC68332 chip selects; the main firmware never touches CSBAR/CSOR or the SCI. Addresses below are the ones the firmware uses.

| Range | Use |
|---|---|
| 0x000000 to 0x01FFFF | Boot ROM A.02.00 (dumped over GPIB; see BOOTROM.md). VBR is set to 0x0000000A |
| 0x020000 to 0x0FFFFF | Main firmware flash: the download image (to 0x0F462B on the E4418B, 0x0F1D5B on the E4419B), erased above (read out on the E4419B) |
| 0xE00000 to 0xE3FFFF | Battery-backed RAM (inferred: the check at 0x0202DE prints "RUN OUT OF BATTERY-BACKED RAM" if BSS ends above 0xE40000): persistent header, data segment, BSS |
| 0xE40000 to 0xE7FFFF | Further RAM: pSOS regions, heap, stack (top 0xE7FFF0) |
| 0xF00000 to 0xF00007 | GPIB controller, 8 byte registers (also driven directly by the 436A/437B drivers) |
| 0xF00800 to 0xF00807 | 16550-type UART for the RS-232/RS-422 interface (IER, LCR, MCR, LSR, MSR, SCR, DLL, DLM) |
| 0xF01180, 0xF011C0 | PLD mode register: write to 0xF01180 for mode 0, 0xF011C0 for mode 1 |
| 0xF01800, 0xF01802 | Recorder output DACs |
| 0xFFFA11 to 0xFFFA17 | SIM port E/F data and direction |
| 0xFFFC15 to 0xFFFC1F | QSM port QS pins and QSPI registers |
| 0xFFFE00 to 0xFFFE20, 0xFFFF00 | TPU registers and parameter RAM |

## Battery-backed header (RAM 0xE05000)

These locations survive power-off and firmware downloads; first-power-up defaults are set at 0x020554.

| Address | Content | Default | Command |
|---|---|---|---|
| 0xE05000 | Power-up magic 0x12345678 | | DIAG:CPF clears it |
| 0xE05004 | System version string (variable 0xF9) | empty | SERV:VERS:SYST |
| 0xE05018 | Processor board version string (variable 0xF8) | empty | SERV:VERS:PROC |
| 0xE0502C | Serial number string (variable 0xF6) | empty | SERV:SNUM, front-panel Edit Serial Number |
| 0xE0504A | Options string (variable 0xF7) | empty | SERV:OPT |
| 0xE05068 | Backlight drive setting | 0x46 (70) | DIAG:DISP:BRIG:SET?, Service > Display > Set Brightness |
| 0xE0506C | Contrast setting | 0x21 (33) | DIAG:DISP:CONT:SET, Set Contrast |
| 0xE05070 | RAM battery check word 0x12345678 | | SERV:BIST:BMON |
| 0xE05074 | GPIB address (word) | 13 | SYST:COMM:GPIB:ADDR (warning printed if 0) |
| 0xE05076 | Variable 0xAE (serial port selector used by the RS-232/RS-422 tests) | 0 | |
| 0xE0507A | Download marker 0x18273645 | | Set by the data-segment copy |
| 0xE0507E | Build date and time strings | | Compared at power-up |

## Boot ROM services

The boot ROM (address 0; dumped and described in BOOTROM.md) owns the exception vectors; the main firmware never writes VBR. It calls the boot ROM through traps:

| Trap | Caller | Purpose |
|---|---|---|
| `TRAP #0` | 0x0362DA (DIAG:MON), 0x0D3C1E (Ctrl-C on the console) | Enter the boot ROM monitor (see BOOTROM.md) |
| `TRAP #11` | 0x0A2764 | Read VBR / status register |
| `TRAP #14` | 0x0A2A14, 0x0A2A44 | Boot ROM services selected by the pushed number: the version pointer, and service 10, which starts a download (used by `DIAGnostic:DOWNload:FW`) |

On the E4419B (S/N MY45103686) `DIAGnostic:VERSion:BOOT?` returns `A.02.00`, so the boot ROM
is version A.02.00. Several findings here are confirmed against that unit over GPIB; see
`BENCH-VERIFICATION.md`.

## Measurement board bus protocol

Every operation on the measurement board (ADSP-2101 plus sensor interface) goes through 0x064364: `(board, command word, tx length, tx buffer, rx length, rx buffer)`. Bits 15 to 11 of the command word are the opcode, bit 10 the channel, and the low bits a payload length or sub-function.

| Command word | Wrapper | Purpose (from callers) |
|---|---|---|
| 0x0800 | 0x0644D4 | Board identify / BIST status (board scan at power-up) |
| `(ch<<10)\|0x0801` | 0x064500 | Read channel status word (sensor connected, interrupting) |
| `(ch<<10)\|0x1000` | 0x06453E | Channel command, no data |
| `(ch<<10)\|0x1800` | 0x064578 | Read 4-byte raw measurement (zero, cal, DIAG:RADC?) |
| `(ch<<10)\|0x2002` | 0x0645B6 | Write 2-byte channel setting (sensor connect/disconnect, init) |
| `(ch<<10)\|0x2800\|cfg` | 0x06460A, 0x064668, 0x064A5E, 0x064ABA | Channel configuration bits: speed, RCAL path, reference cal, gain path |
| `(ch<<10)\|0x3004` | 0x064846 | Write one sensor EEPROM byte (address, data) |
| `(ch<<10)\|0x3800\|n` | 0x0648B2 | Read n sensor EEPROM bytes |
| 0xC000\|(n+2) | 0x064746 | Block write, n data bytes (no direct callers; the DSP ignores opcode 24) |
| 0xC800 | 0x0647C6 | No direct callers; ignored by the DSP |
| 0xD000, 0xD001 | 0x0647EA, 0x064818 | 2-byte reads (no direct callers; ignored by the DSP) |
| 0xD800 | 0x064462 | 10-byte read during board scan: the DSP and gate-array revision strings |
| 0xE000\|n | 0x0649EE | Analog measurement: mount resistor |
| 0xE001 | 0x064A30 | Analog multiplexer reading (analogueMeasurement) |
| 0xE005 | 0x0649C0 | Sensor temperature |
| 0xE800, 0xE800\|(n+2) | 0x0646C4, 0x0646E8 | Bus transfer control / block write |
| 0xF002 | 0x064B18 | Read ADSP-2101 data-memory word |
| 0xF804 | 0x064B54 | Write ADSP-2101 data-memory word (address, value) |

The channel-configuration command (opcode 0x2800) carries a config byte in the low 8 bits: `(ch<<10) | 0x2800 | cfg`. Each wrapper sets or clears one bit of `cfg` and resends the whole byte, so the bits accumulate:

| Bit | Mask | Wrapper | SCPI command | Meaning |
|---|---|---|---|---|
| 0 | 0x01 | 0x06460A | `DIAG:DETector:SPEed` | measurement speed select (param 1 clears, else sets) |
| 1 | 0x02 | 0x064668 | `DIAG:RCALibrate` | RCAL signal path |
| 2 | 0x04 | 0x064A5E | `DIAG:DETector:RCALibrate` | reference calibrator on |
| 3 | 0x08 | 0x064ABA | `DIAG:DETector:GSWitch` | detector gain path (param 1 = path 1) |

Data memory addresses written through 0xF804 (for example 0x3815) fall in the ADSP-2101's internal data RAM (0x3800 to 0x3BFF). The DSP side is confirmed in the DSP disassembly: the DSP polls that DM mailbox (it reads 0x3815 at PM 0x0249) and writes status to 0x381e/0x381f. See the DSP image section and `DSP/STRUCTURE.md`.

### DSP side of the bus

The link to the DSP is its serial port 0. The SPORT0 receive interrupt (vector at PM 0x000C) stores
each incoming word at DM 0x396D; the main loop at PM 0x0197 splits it into the opcode (bits 15-11,
to DM 0x383D) and the low 10 bits (to DM 0x383F; the channel bit is dropped because each
measurement board serves one channel), then jumps through a 31-entry table at DM 0x385E indexed by
opcode - 1. Replies go out through TX0.

| Opcode | Command word | DSP handler | Notes |
|---|---|---|---|
| 1 | 0x0800 | 0x0387 | board identify / BIST status |
| 2 | 0x1000 | 0x01BA | channel command |
| 3 | 0x1800 | 0x01C7 | 4-byte raw measurement |
| 4 | 0x2000 | 0x01E8 | 2-byte channel setting |
| 5 | 0x2800 | 0x01F6 | configuration: the low bits (0 to 19) index a second table at DM 0x389C |
| 6 | 0x3000 | 0x0274 | sensor EEPROM byte write |
| 7 | 0x3800 | 0x02D2 | sensor EEPROM read |
| 8 to 26 | 0x4000 to 0xD000 | 0x03B2 | no action (returns to the loop) |
| 27 | 0xD800 | 0x0368 | sends the 10 bytes at DM 0x3807 to 0x380B, "a0111a0103" |
| 28 | 0xE000 | 0x035B | analogue measurement |
| 29 | 0xE800 | 0x0399 | transfer control |
| 30 | 0xF000 | 0x03A0 | reads the next word as a DM address and returns DM(address) |
| 31 | 0xF800 | 0x03AB | reads an address and a value and writes DM(address) |

The main CPU's board scan (0x064462) treats the 10 bytes as two revision strings: FUN_00064008
turns "a0111" into "A.01.11" (and the number 1.11), FUN_00064138 turns "a0103" into "A.01.03".
If the first five bytes are all the same, the slot is taken as empty. These are the DSP Revision
and Gate Array Rev lines on the Version screen; both come from the DSP's data, not from the gate
array.

## Sensor EEPROM format

The E-series sensor EEPROM is read over the measurement bus by `FUN_0006ee40(bus, addr, dest,
len)` (16 bytes at a time, with an inter-page delay) and parsed by `FUN_0006F088`. Layout:

- Addr 0: format/revision byte. The parser accepts 1, 2, 3, 6 or 7 and retries up to five times
  otherwise; 0, 4, 5 and 8-or-above are treated as unreadable.
- Addr 1: one byte (sub-revision). Addr 2-3: one word.
- Addr 4: block count N.
- Addr 5 onward: N directory entries, 7 bytes each:
  `{u16 blockAddr, u8 type, u8 format, u16 length, u8 checksum}`. Type is 1-20, format 1-5.
- Each block's payload lives at `blockAddr` in the EEPROM, `length` bytes, protected by an
  8-bit additive checksum (the sum of the payload bytes must equal the entry's checksum byte;
  a mismatch prints `EEPROM - data block failed checksum %d vs %d`).

The parser builds a `[type][format]` directory in the RAM sensor image: slot at
`sensor + 0x29C + type*0x8C + format*0x14`, with a state byte at `+0x23`: 0 for a format with no
handler, 1 once registered, then 2 (listed in the directory), 5 (checksum passed) and
6 (installed). `DIAGnostic:DETector:IMAGe:
BLOCks?` lists the slots that reached state 6. Each installed block runs a per-block handler
(a function pointer in the slot) that copies its data into the working sensor structure.

Failures queue `Sensor EEPROM data not found or unreadable`, `...Read Failed - unknown` or
`...Read Failed - critical`. The header fields (model, serial, place, date, stinger rev) read
by `IMAGe:HEADER?` sit at sensor offsets 0xF04/0xF14/0xF2E/0xF36/0xF3E-F (see COMMANDS.md).

The block types and their install handlers are registered by `FUN_00071b66`, each with a name
string. The complete set:

| Type | Name | Formats (handler) | Severity |
|---|---|---|---|
| 1 | Linearity | 1 (0x06F9AE) | 1, required |
| 2 | Temp-Comp | 1 (0x06FCDA), 2 (0x07001E) | 1, required |
| 3 | Cal Factor | 1 (0x0704A6), 2 (0x070782) | 1, required |
| 4 | Header | 1 (0x06FAF2) | 1, required |
| 5 | Misc | 1 (0x070AD6), 2 (0x070C7E), 3 (0x070FB8) | 1, required |
| 6 | Basic Params | 1 (0x071580), 2 (0x0715C6) | 2, optional |
| 8 | Auto-Filter | 1 (0x071850) | 2, optional |

Types 7 and 9-20 are not used. The severity byte (type slot +0x16, the last argument to the
registration at 0x06ED50) controls what the parser at 0x06F088 does when a type does not install:

- Severity 1 (types 1 to 5): a missing or bad block queues `Sensor EEPROM <name> data not found
  or unreadable`, sets the read status to 2 and ends with `Sensor EEPROM Read Failed - critical`.
- Severity 2 (types 6 and 8): if the directory does not list the type, the status becomes 1 and
  nothing is reported. Only a listed block that fails its checksum is treated as critical.

The N8482A on the bench carries no type 6 or 8 block and reads without error, which fits. Its raw EEPROM (read out 21 October 2026, BENCH-VERIFICATION.md) is format 6 with seven directory entries: types 1 to 5 plus types 18 and 19, which the firmware does not register and so ignores. The Cal Factor block is a 9-byte header with the point count at byte 5, then records of u32 frequency in kHz and two u16 factors with 0x4000 = 100 %. The Linearity, Temp-Comp and Cal Factor blocks are the cal
tables exposed by the `DIAGnostic:DETector:IMAGe` commands in COMMANDS.md. Confirmed on an
N8482A, where `DIAG:DET:IMAG:BLOC?` lists types 1 (Linearity), 2 (Temp-Comp, format 2),
3 (Cal Factor), 4 (Header) and 5 (Misc) - see BENCH-VERIFICATION.md.

## Internal variable store

Commands and the front panel keep settings in numbered variables (0 to 0x11D) held by the
"istate" object, created in the root process (call at 0x020982) by FUN_000CA7FE from a descriptor block at 0x0450B4:
{descriptor table 0x044C3C, area A size 0x5FA, area B size 0x55C, area A defaults 0x043124, area B
defaults 0x0440C8, count 0x11E}. Each 14-byte descriptor holds a signed offset, a size and a type
byte (0x0A for a double). `get(id)` (FUN_000CA01E, method +0x2C) returns area A + offset for a
positive offset and area B + ~offset for a negative one, so a variable's default can be read
straight from the image. Examples: 0xCC = 1.5 and 0xCD = 0.01 (the cal-factor limits, 150 % and
1 % held as fractions), 0xD5 = 0.99 and 0xD6 = -0.5 (the LOSS1 limits, LOSS = 1 - cal factor),
0x91 = 0.1 and 0x93 = 1E10 (MIN/MAX companions used by `DIAGnostic:UNIT`).

## SCPI parser structures

- Keyword entry, 42 bytes: +0x00 pointer to keyword (`SHORT\0suffix\0`, long form = short + suffix); +0x04 child group; +0x18 s16 channel flag (1 or 2 = fixed channel, -1 none, -2 accepts a numeric suffix); +0x1A pointer to the u16 command code.
- Group: u16 id, u16 count, s16 default, then `count` entry pointers. Main root: 0x0541EC (E4418B), 0x05507E (E4419B). Common-command root: 0x053258.
- Command code to descriptor: the table base pointer is stored at 0x0288C6 and points to 0x027FEC; descriptor D = *(0x027FEC + 4 x code). *(D+0) is the set record and *(D+4) the query record.
- Record {u32 tag, u32 x}: tag 1 means x is the handler; tag 0 means x points to {0, handler, u16 nparams, param descriptor}; tag 3 means the query returns a bound internal variable.
- 0x0234F8 is a handler that only returns success. Commands bound to it store their parameter in an internal variable, read elsewhere through the context's `get(varId)` method (vtable offset 0x2C).
- Handlers return a small internal code (-n), held as constants at 0x020F50 + 2n. The table at 0x0C656C (stride 6) translates it to a SCPI error. Examples: -11 gives -222 "Data out of range", -57 gives -224 "Illegal parameter value", -8 gives -200 "Execution error".
- Parameter converters: numeric 0x0B5332 to 0x0B698C (accepts MIN/MAX/DEF), boolean 0x0B6BDC to 0x0B73B0, character 0x0B76AA to 0x0B8978, string 0x0BA1EA.

## Command set

- E4418B: 314 command codes reachable from the keyword tree, plus 16 common commands. E4419B: 330. The extra E4419B codes are the two-channel DIFFerence and RATio forms of CONFigure, FETCh, READ and MEASure.
- About 150 codes do not appear in any of the manuals. Most are in the `DIAGnostic[1|2]` subtree: memory access, sensor EEPROM image tools, pSOS object dumps, firmware download, display, keyboard, battery and serial-port tests.
- `*_TRG?` (code 0x086) is an extra common query, but its keyword is literally `_TRG` and the parser rejects the underscore with -101, so it cannot be sent.
- Two commands reachable in both models are in neither the manuals nor the first COMMANDS.md: `[SENSe]:CORRection:STATe` (0x146) and `MEMory:TABLe:LOSS[:MAGNitude]` (0x170). Both are now in COMMANDS.md.
- E4419B accepts `READD` as an alias for `READ`. E4418B does not.
- `SYSTem:LANGuage HP436A` selects an HP 436A emulation mode, with its own parser, HP-IB driver and front-panel menu. The manuals only describe SCPI, HP 437B (E4418B) and HP 438A (E4419B).

### DIAGnostic:PEEK and POKE (verified)

| Command | Code | Handler | Behaviour |
|---|---|---|---|
| `DIAGnostic:POKE <addr>,<word>` | 0x12A | 0x0377BC, 0x03776C | Writes a 16-bit word. Odd address returns -224. |
| `DIAGnostic:PEEK? <addr>` | 0x12B | 0x037836, 0x0377F0 | Reads a 16-bit word. Odd address returns -224. |
| `DIAGnostic:PEEK:BLOCK? <addr>,<n>` | 0x12C | 0x037968, 0x037896 | Reads n words (n up to 256, else -222). |
| `DIAGnostic:PEEK:BYTE? <addr>` | 0x12D | 0x0379E6, 0x0379B0 | Reads one byte. |
| `DIAGnostic:POKE:BYTE <addr>,<byte>` | 0x12E | 0x037A56, 0x037A20 | Writes one byte. |

Each also prints the address and value on the debug stream.

## Front-panel menu

The softkey menu engine, the label table and the actions behind the undocumented service
and test items are decoded in `FRONTPANEL.md`. In short: `FUN_0009bbd8` builds the softkeys
for the current screen (id at `DAT_00e1eb8c[0x74]`) through the menu-manager object
`DAT_00e1eaec` (install at method +0x70). Test1 to Test6 call `freqInfoTest(1..6)`, a
display-field format check, not a self test. The firmware holds two individual self-test menus,
chosen by the processor-board version string: board revision C or later gets Memory, Lithium
Battery, Measure Assembly, Calibrator, then Keyboard, Fan, Display and Serial Interface (the set
the E4419B shows); earlier boards get ROM Checksum, RAM, RAM Battery and so on. See
FRONTPANEL.md.

## Command-set coverage

`Tools/coverage.py` lists reachable commands that are neither in a documented subsystem nor in
COMMANDS.md. It treats every command under a documented top-level subsystem (SENSe, MEMory,
SYSTem, STATus and others) as documented, and counts any two- or three-digit hex number in
COMMANDS.md as covering that code, so it cannot catch an undocumented command inside a standard
subsystem. A keyword-by-keyword check against the Programming Guide text (18 October 2026) found
two such commands, `[SENSe]:CORRection:STATe` and `MEMory:TABLe:LOSS[:MAGNitude]`, now in
COMMANDS.md. Every other reachable code is in the manuals or COMMANDS.md.

## Resolved since first pass

- Measurement-board configuration bits (opcode 0x2800): decoded above.
- Front-panel items Edit Serial Number, Test1 to Test6, the two unlabelled options-string items, individual self tests:
  actions traced (FRONTPANEL.md, corrected 18 October 2026).

## Device error codes

Command handlers return a small internal code. The dispatcher translates it to a SCPI error
through one of two stride-6 tables, by sign:

- Negative codes (-n): the static table at 0x0C656C, 110 entries (codes 0 to -109). Entry
  layout {u16 type, s16 scpi_error, s16 status-bit}; type 3 means "queue scpi_error".
- Positive codes (device-specific): a table in initialised data, passed as the error-table
  argument to the SCPI-context constructor 0x0C3730. E4418B: RAM 0xE0EC38 (ROM 0x0F2DA8).
  E4419B: RAM 0xE0B8EC (ROM 0x0F04B0). Same layout as the negative table, indexed by code x 6.
  Types 0 to 2 set, clear or toggle a status bit; type 3 queues the error. Codes 0x7A to 0x8B:

| Code | SCPI | Meaning | Code | SCPI | Meaning |
|---|---|---|---|---|---|
| 0x7A | -211 | Trigger ignored | 0x82 | -256 | File name not found |
| 0x7B | -213 | Init ignored | 0x83 | -257 | File name error |
| 0x7C | -214 | Trigger deadlock | 0x84 | -226 | Lists not same length |
| 0x7D | -114 | Header suffix out of range | 0x85 | -330 | Self-test failed |
| 0x7E | -221 | Settings conflict | 0x86 | -231 | Data questionable |
| 0x7F | -230 | Data corrupt or stale | 0x89 | -361 | Parity error in program message |
| 0x80 | -241 | Hardware missing | 0x8A | -362 | Framing error in program message |
| 0x81 | -220 | Parameter error | 0x8B | -363 | Input buffer overrun |

0x87 and 0x88 change status bit 2 of register 0 and queue nothing. The code backs this up:
handlers return 0x7D when a channel suffix is out of range, 0x80 when the sensor is not an
E-series part (the recorder-channel path prints "HARDWARE MISSING" before returning 0x80), and
0x7E for `MEMory:TABLe` commands with no table selected.

Checked on the E4419B (BENCH-VERIFICATION.md): `STAT:DEB:HAP` (long form `STATus:DEBug:HAPpenning`)
with each code from 0x7A to 0x82, and 0x86, queues exactly the error above (0x83 to 0x85 and 0x89
to 0x8B are from the table only), and `DIAGnostic:PEEK:BLOCK?` at 0xE0BBC8 (code
0x7A) returns `00 03 FF 2D 00 00 00 03 FF 2B 00 00`. The first version of this table, taken from
the image loaded 10 bytes off, had every code two too high.

## Open items

- (resolved 18 October 2026) The XC3042 bitstream is in the DSP flash, not the boot ROM; see
  "Measurement-board gate array". The XC3042 is HP 1821-1672 on the measurement board, not
  1821-4401 (that number is the SP334 RS-232 transceiver), and the CLIP has no statement that
  the main processor configures it. The mode pins, traced on the schematic, confirm
  master-parallel-down self-configuration.
- (resolved) Front-panel menu keypress paths: mapped on the instrument - see FRONTPANEL.md,
  "Menu navigation". The undocumented items are reached via System/Inputs -> More -> Service.
- DSP, remaining: the code page is fully disassembled, and the 7 `??` words are a PC-stack pop
  (see the DSP section). What is left is the meaning of the 781-word block that page 0 copies
  into DM 0x3802 to 0x3B0E.

The DSP side of the measurement-board protocol is now understood: the main CPU and the DSP
share a data-memory mailbox at DM 0x3800+ (commands in via 0xF804, status out via 0xF002).
See the DSP image section above and `DSP/STRUCTURE.md`.
