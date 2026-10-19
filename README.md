# EPM series power meter firmware (E4418B / E4419B)

Firmware analysis of the Agilent E4418B (single channel) and E4419B (dual channel) EPM power
meters: the undocumented SCPI commands, service functions and internals that the official manuals
leave out. The findings are written up as a supplementary manual in the HP/Agilent service-manual
style, and checked where possible on a real E4419B (firmware A2.09.01, boot ROM A.02.00) over GPIB.

## Contents

|File|What it is|
|-|-|
|`Docs/EPM_Firmware_Supplement.pdf`|The supplement (31 pages): undocumented commands, HP 436A emulation, the measurement board and DSP, the sensor EEPROM format, firmware and memory, the boot ROM, front-panel service functions, device error codes and a verification record|
|`Analysis/COMMANDS.md`|Every undocumented command, with syntax, behaviour, risks and code addresses|
|`Analysis/NOTES.md`|Image layout, memory map, battery-backed header, measurement-board bus protocol, SCPI parser structures, sensor EEPROM format, device error codes|
|`Analysis/FRONTPANEL.md`|The softkey menu engine, the label table, and the actions behind the service and test items|
|`Analysis/BOOTROM.md`|The boot ROM (read out over GPIB): power-up modes, ROM monitor commands, download record format|
|`Analysis/DSP/STRUCTURE.md`|The measurement-board ADSP-2101 image: page layout, boot sequence, bus-opcode dispatch, data-memory mailbox, and the XC3042 gate-array bitstream|
|`Analysis/BENCH-VERIFICATION.md`|Every check made on the E4419B over GPIB, with the commands sent and the replies|
|`Analysis/REVIEW.md`|The second-pass review: errors found in the first analysis and how they were corrected|
|`Firmware/E4418-A1-09-01.dload`, `Firmware/E4419-A2-09-01.dload`|Agilent main-processor firmware for the E4418B and E4419B (both built 12 May 2009)|
|`Firmware/E4418-9A EPM-441-2A EPROM DSP Rev A.01.11 E4418-80023.bin`|Measurement-board DSP flash image (E4418-80023, revision A.01.11)|

Addresses in the notes are E4418B unless marked. The notes also mention the scripts, generated
listings and Ghidra projects used to produce them; those are not part of this repository.

## Licence

The supplement, the analysis notes and this README are MIT licensed (see `LICENSE`). The firmware
images are Agilent's and are not covered by that licence.

## The `.dload` format

A `.dload` file holds two load records, each a 10-byte header (`WL`, a 32-bit load address and a
32-bit inclusive end address) followed by the data: the code at 0x020000, then the initialised
data segment. The file ends with the two-byte `H0` end-of-download record. Reading it as one flat
block puts the data segment 10 bytes too high.

## Hardware summary

* Main processor MC68332 (CPU32) running pSOS+. The boot ROM and the main firmware share two
4 Mbit flash chips, one per byte lane; the firmware loads at 0x020000.
* One measurement board per channel, each with an ADSP-2101 DSP booted from its own 512 Kbit flash
(E4418-80023) and a Xilinx XC3042A gate array that loads its bitstream from the top 3,856 bytes
of that flash.
* LCD on an SED1330 controller.

