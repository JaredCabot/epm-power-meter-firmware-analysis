# Review of the analysis, 18 October 2026

A second pass over the findings in this folder: each claim checked against the code (Ghidra
output and the 68k listing), the DSP image, the CLIP and the E4419B on the bench (S/N
MY45103686, firmware A2.09.01, N8482A on Channel A). Corrections have been made in place in the
files named below. This file records what changed and why.

## Errors found and corrected

### 1. The `.dload` files were loaded 10 bytes off above 0x0E9170

Each `.dload` file holds two load records, not one. The second record header sits at file offset
0x0C917A (E4418B) or 0x0C9BCE (E4419B):

    57 4C 00 0E 91 70 00 0F 46 2B     'WL', load 0x0E9170, end 0x0F462B   (E4418B)

Every tool, and the Ghidra import, read the file as one block from offset 0x0A, so the whole
initialised data segment (ROM 0x0E9170 up, RAM 0xE05000 to 0xE104BC) sat 10 bytes too high. Code
is unaffected because it all lies below 0x0E9170. What was affected: anything that read an
initial value from the data segment, the addresses in `strings.txt` above 0x0E9170, string
annotations on data-segment addresses in `listing.asm`, and Ghidra's RAM_DATA block.

Fix: `Tools/dload.py` loads the records properly, and every tool now uses it. `run_all.py` also
writes the Ghidra inputs (`Tools/ghidra_inputs.py`). `listing.asm`, `strings.txt`, the pickles and
both Ghidra projects and `decompiled.c` files were regenerated. The SCPI tree, entries, error
and enum tables did not change.

Check: on the E4419B, `DIAG:PEEK:BLOCK?` at RAM 0xE0BBC8 returns the bytes the corrected loader
puts there.

### 2. The positive device-error table was two codes off

A consequence of item 1. NOTES.md listed 0x7C to 0x8D with 0x7D = -213 and 0x82 = -241. The
correct table runs 0x7A to 0x8B: 0x7B = -213 "Init ignored", 0x7D = -114 "Header suffix out of
range", 0x7E = -221 "Settings conflict", 0x80 = -241 "Hardware missing", 0x82 = -256 "File name
not found". The E4419B copy is at RAM 0xE0B8EC, not BSS. Confirmed on the E4419B with
`STAT:DEB:HAP` for codes 0x7A to 0x82 and 0x86; the rest are from the table. NOTES.md, BENCH-VERIFICATION.md and HANDOFF.md corrected.

### 3. Sensor EEPROM block severity was reversed

NOTES.md called types 6 (Basic Params) and 8 (Auto-Filter) critical and types 1 to 5 normal.
The parser at 0x06F088 does the opposite: types 1 to 5 (severity 1) are required, and a missing
one ends in "Read Failed - critical"; types 6 and 8 (severity 2) are optional and may be absent
without any message. The N8482A has neither 6 nor 8 and reads cleanly.

### 4. The DSP flash's page 7 is the XC3042 bitstream

STRUCTURE.md called bytes 0xF0F0 to 0xFFFF a characterisation or board-ID table, and NOTES.md put
the FPGA bitstream in the boot ROM. Read from the top down, least significant bit first, the
block is a complete XC3000-family stream: length count 30,825, then 285 frames of 108 bits, each
framed 0...111, which is the XC3042 layout. The CLIP puts the XC3042A (1821-1672) on the
measurement board next to the DSP flash. NOTES.md also gave the FPGA as CLIP part 1821-4401,
which is the SP334 RS-232 transceiver, and quoted the CLIP as saying the main processor
configures the FPGA; the CLIP says no such thing. README.md had the same claim.

### 5. The seven DSP `??` words are code

STRUCTURE.md suggested `0x0D03CF` was data in the instruction stream. Each one starts a routine
that pushes MR1 to a software stack, and each routine exits with `I6 = MR1; JUMP (I6)`: the
instruction pops the return address off the PC stack. Also corrected: page 0 copies to DM
0x3802 to 0x3B0E, not 0x3801.

### 6. Two undocumented commands were missing

HANDOFF.md said `coverage.py` proved nothing undocumented was missing. The script treats every
command under SENSe, MEMory, SYSTem and the other standard subsystems as documented, so it could
not see these two:

- `[SENSe[1]]|SENSe2:CORRection:STATe` (0x146): stores and returns a per-channel boolean that
  nothing else reads.
- `MEMory:TABLe:LOSS[:MAGNitude]` (0x170): a twin of `GAIN[:MAGNitude]` that stores 100 - x;
  its query reports 100 + t, so values do not round-trip (probably a firmware bug). The query
  side is confirmed on the E4419B (HP8482A table: GAIN 98.0, LOSS 198.0).

Both are now in COMMANDS.md and both were seen on the E4419B.

### 7. Front-panel details

- Test1 to Test6 call `freqInfoTest(1..6)` through six thunks; FRONTPANEL.md listed three
  actions and gave `freqInfoTest` as 0x099DE0 (it starts at 0x099D98).
- 0x05DD1C does not pick a caption. It compares the options string with `001` and opens one of
  two menus.
- There are two individual self-test softkey sets. FRONTPANEL.md described the ROM Checksum/RAM
  set as the live menu; the E4419B shows the Memory/Lithium Battery/Fan set, and the E4418B
  image has the same labels (it is not an E4419B extra). The fan test and the serial loop-back
  tests are now listed.
- The label-table "string addresses" were table-slot addresses, and the table runs past
  0x5FA9A to at least label 329.
- The fixed 0x2C-byte node layout does not hold for the label offset; marked provisional.

## Confirmed without change

- `*_TRG?` keyword is `_TRG` and returns -101 (re-run on the E4419B).
- `READD` is the E4419B long form of `READ`.
- 0x2800 configuration bits and their SCPI commands; `DIAG:DET:THIRd` writes 0x105 to DSP word
  0x3815.
- Sensor EEPROM layout: revision byte (1, 2, 3, 6, 7 accepted, five retries), block count at 4,
  7-byte directory entries, the block-type registry and handler addresses.
- Edit Serial Number action and the self-test thunk-to-worker mapping.
- DSP byte order, page layout, opcode decoding (spot-checked by hand), the boot sequence
  (0x065F sets BFORCE with BPAGE 1) and the DM mailbox addresses.
- Bench results for versions, sensor identity, image header and blocks, and the -241 from
  `SERV:SENS:FREQ:MAX?`.

## Independent check

A separate agent, given only the raw firmware, the DSP image, the Ghidra output and the CLIP
text, re-checked items 1 to 7 above without access to this review's reasoning. It rebuilt the
image from the two records and ran the double-precision helpers in an emulator.

- Items 1, 2, 3 and 4: confirmed, including every error code in the corrected table and the full
  285 x 108 frame structure of the bitstream.
- Item 5: the boot loader is confirmed. The PC-stack reading of `0x0D03CF` is an inference from
  the prologues and epilogues, and three of the seven prologues turn out to be dead code;
  STRUCTURE.md updated.
- Item 6: confirmed (0x0E2A6C is a - b, 0x0E2A72 is a + b). It also found that the table store
  keeps values as round(1000 x v) in 16 bits, so a LOSS write may overflow; COMMANDS.md updated.
- Item 7: confirmed, with one correction to this review: the label table base is 0x5F5C6
  (ids start at 1), not 0x5F5CA. With that base every softkey label lines up with its action.
  FRONTPANEL.md updated.

One point where the checker was wrong: it read 0x0E40AC as `strcmp` and so had the options-string
menu choice reversed. 0x0E40AC returns a pointer to the match or 0, so it is `strstr`; an
options string containing `001` opens menu 0x059AD0.

The checker also reported that the "Memory" self test (0x05DFC4) runs the ROM check before the
RAM check, that the GAIN set worker loops 81 times where LOSS loops 80, and that the DSP page-0
loader copies only the upper 16 bits of each PM word.

## Open items worked on 18 October 2026 (second pass)

- Self-test menu choice: made by the processor-board version string (`SERVice:VERSion:PROCessor`),
  looked up in a version table at 0xE05144. Revision C or later gets the Memory/Lithium
  Battery/Fan set and the fan test. COMMANDS.md, FRONTPANEL.md.
- `MEMory:TABLe:LOSS` scaling: the variable store's defaults (descriptor block 0x0450B4) give the
  LOSS limits as 0.99 and -0.5, and the command takes unconverted fractions. A test on the empty
  CUSTOM_0 table reproduced the predicted wrapped values (3444.4, 3396.4) and was cleared
  afterwards. The command is unusable. COMMANDS.md, BENCH-VERIFICATION.md.
- Gate-array mode pins: traced on CLIP page 66. M0 is pulled down and M1 and M2 pulled up, which
  is master-parallel-down. NOTES.md, DSP/STRUCTURE.md.
- DSP page-0 block: the DSP's initialised data. It holds the revision string "a0111a0103" that
  the main CPU reads with opcode 0xD800 for the Version screen (so the gate-array revision comes
  from the DSP), the 31-entry bus-opcode dispatch table and a 20-entry configuration sub-table.
  The DSP side of every bus opcode is now mapped. NOTES.md, DSP/STRUCTURE.md.
- Image checksum: the header at 0x020030 holds the image length and the two byte-lane sums that
  the ROM Checksum self test checks. They match the image loaded by `Tools/dload.py` and not the
  flat load, which independently confirms the loader fix. NOTES.md.

## Boot ROM dump

The boot ROM was read out over GPIB with `DIAGnostic:PEEK:BLOCK?` (two passes, identical) and
is described in BOOTROM.md. It settled the `.dload` trailer: `48 30` is the `H0` end-of-download
record. The rest of the flash (0x020000 to 0x0FFFFF) was read the same way and is the
A2.09.01 download image exactly, loaded as two records, with nothing but 0xFF above it.

## Still open

- Which front-panel keys select the boot ROM's power-up download and memory-clear modes, the
  console baud rate, and where the SCI console pins come out.

- The meaning of the DSP configuration sub-table entries and the constants in the DSP data block.
- The `SENSe:CORRection:CFACtor` and `LOSS1` behaviour with an 8480-series sensor (they return
  -241 with the N8482A).
