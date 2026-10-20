# Bench verification

Findings checked against the real instrument over GPIB (pyVISA). Unit:
`Agilent Technologies,E4419B,MY45103686,A2.09.01` at GPIB address 13 - the same
firmware as `Firmware/E4419-A2-09-01.dload`. Writes were limited to settings that were
restored afterwards (the CUSTOM_0 user table, the 8481A cal factor), the table-edit selection,
and normal zeroing and calibration. Nothing that writes the sensor EEPROM, the battery-backed
header or the firmware, enters download mode or switches language was run.

## Confirmed

| Command (sent) | Result | Confirms |
|---|---|---|
| `*IDN?` | `Agilent Technologies,E4419B,MY45103686,A2.09.01` | the analysed image is this unit |
| `DIAG:VERS:FW?` | `May 12 2009 10:17:44` | build timestamp from version.c (0x021094/0x0210DC) |
| `DIAG:VERS:BOOT?` | `A.02.00` | boot ROM version via TRAP #14 (new fact: boot ROM is A.02.00) |
| `DIAG:PEEK? 0xE05074` | `+13` | GPIB address word in the battery header; PEEK + memory map |
| `DIAG:PEEK? 0xE05000` | `+4660` (0x1234) | power-up magic high word |
| `DIAG:PEEK:BLOCK? 0xE05000,4` | bytes `12 34 56 78 ...` | magic 0x12345678, words returned high-byte-first |
| `DIAG:DSP? 0x381E / 0x381F / 0x3815` | `9 / 0 / 0` | the DSP data-memory mailbox responds (DSP side of the board protocol) |
| `DIAG:UART:REGS?` | `IER=0, LCR=3, MCR=e, LSR=60, MSR=0, SCR=fe, DLL=6d, DLM=0` | 16550 register dump at 0xF00800 |
| `DIAG:SETT?` | `CHANNEL 0 ***... cal_factor, cal_rcf, meas_freq, conf_frequency, meas_units, init_continuous, trig_source, trig_delay_auto, window resolution` | the DIAG:SETTings? field block |
| `DIAG:STAT?` | `CHANNEL 1 ***... MODE / WAITING / TRIGGER STATE / TRIGGER NEXT STATE / SENSOR / SENSOR POSITION` | the DIAG:STATes? field block |
| `DIAG:BATT:FIT? / ABS? / CAP? / CHAR? / TIME?` | `0 / 1 / 0 / 0 / 65535` | battery-pack queries (no pack fitted) |
| `DIAG:DISP:DUMP? 0` | 40 comma-separated `0x00` values | one 320-pixel display line (40 bytes) |
| `STAT:DEB:HAP -11 / -57 / -8` then `SYST:ERR?` | `-222 / -224 / -200` | the internal-code -> SCPI error table at 0x0C656C, live |

`STAT:DEB:HAP -1` and `-100` queue no error, consistent with those table entries being
status-bit (type 0/1/2) rather than queued-error (type 3) entries.

## Corrections and notes

- **`*_TRG?` (code 0x086) is not invokable over the bus.** The keyword entry is literally
  `_TRG` - a leading underscore, which is not a legal SCPI program-header character. Sending
  `*_TRG?` returns `-101 Invalid character`. The entry is dormant; use `*TRG` plus a read.
  (COMMANDS.md updated.)
- The positive device-error table lives at a model-specific address: RAM 0xE0EC38 on the
  E4418B, RAM 0xE0B8EC on the E4419B (in initialised data on both). A PEEK at 0xE0EC38 on the
  E4419B reads BSS, which is why it returned 0. Corrected 18 October 2026; see "Re-check" below.
- SCPI short forms must be exact: `STAT:DEB:HAP` (not HAPP), `DIAG:BATT:FIT` (not FITT). An
  intermediate truncation returns `-113 Undefined header`.

## Confirmed with a sensor (N8482A on Channel A, power reference)

Sensor reported as "Stinger" (E-series, internal type 6): N8482A, S/N MY51520008, cal date
2011-12-27, cal place PG,MYS, stinger rev 52.48.

| Command | Result | Confirms |
|---|---|---|
| `SERV:SENS:TYPE?` | `N8482A` | sensor model from the EEPROM |
| `SERV:SENS:SNUM? / TNUM? / CDAT? / CPL?` | `MY51520008 / MY51520008 / 2011,12,27 / PG,MYS` | sensor identity queries (TNUMber is the undocumented one) |
| `DIAG:DET:IMAG:HEADER?` | `model number : 'N8482A', serial number : 'MY51520008MY51520008', place : 'PG,MYS', date : '6346072840', stinger rev : 52.48` | the HEADER? format (note: serial prints doubled; the date field is a raw packed value, not formatted) |
| `DIAG:DET:IMAG:BLOC?` | `(Block type 1, format 1) / (2,2) / (3,1) / (4,1) / (5,2)` | the EEPROM block-type registry: 1 Linearity, 2 Temp-Comp, 3 Cal Factor, 4 Header, 5 Misc (types 6/8 not present in this sensor) |
| `DIAG:DET:MRES?` | `+16986` | mount-resistor reading (sensor-type ID) |
| `DIAG:DET:TEMP?` | `+6648` | sensor thermistor reading |
| `DIAG:RADC?` | `+177309` | raw ADC reading |
| `FETC?` | `-3.58623935E+001` | a live power reading (uncalibrated) - the measurement path |
| `READ?` vs `READD?` (CONT ON) | both time out with `-213,"Init ignored"` | **`READD` is a true alias of `READ` on the E4419B** (a non-alias would give -113) |

Two positive device error codes seen live (code numbers corrected 18 October 2026):

- `READ?` with INIT:CONT ON -> `-213 Init ignored` = device code 0x7B.
- `SERV:SENS:FREQ:MAX?` on this sensor -> `-241 Hardware missing; Sensor connected does not
  contain this information` = device code 0x80. This sensor's EEPROM has no extended
  frequency/power-limit block, so those undocumented SERV:SENSor queries report -241.

## Calibration against the power reference (N8482A on Channel A)

The full zero/calibrate chain was run and passed:

- `UNIT:POW DBM`, `SENS:FREQ 50MHZ`. Pre-zero reading about -36 dBm (reference off, unzeroed).
- `CAL:ZERO:AUTO ONCE` -> `*OPC?` = +1, no error; reading dropped to -45.4 dBm (zeroed floor).
- `CAL:AUTO ONCE` -> no error. The instrument energised the 50 MHz / 1 mW reference, the
  sensor measured it and the reference cal factor was computed; afterwards the reading was
  -51.1 dBm (reference off again). A cal failure would have queued an error here - it did not.
- `SENS:CORR:GAIN2?` = 0 with `:STAT?` = 0 (the user frequency-dependent cal factor is off).
- `CAL:RCF?` returns `-241 "Hardware missing"` in this state (the reference cal factor is not
  rereadable with the reference off; -241 is device code 0x80).

So the zero -> calibrate -> reference-oscillator -> measurement chain is confirmed end to end.
A direct reading of the reference was taken later with the 8481A, using the documented
`OUTP:ROSC ON` to hold the reference on (see "8481A on channel B").

## Version screen and front-panel menu map

The front-panel menus were read over GPIB (`DIAG:DISP:DUMP?` rendered to an image) while
stepping through the panel. This mapped the keypress paths to the undocumented items
(FRONTPANEL.md, "Menu navigation") and the Service > Version screen confirmed several things at
once:

| Field (Version screen) | Value | Confirms |
|---|---|---|
| Model / Serial | E4419B / MY45103686 | the unit |
| Main F/W Rev / Date | A2.09.01 / May 12 2009 10:17 | matches the image and `DIAG:VERS:FW?` |
| Bootrom Rev | A.02.00 | matches `DIAG:VERS:BOOT?` |
| Proc. Bd. Version | C | the SERVice:VERSion:PROCessor value |
| System Version | 2 | the `SERV:VERS:SYST = "2"` case (this board runs PLD mode 1) |
| DSP Revision (ChA/ChB) | A.01.11 / A.01.11 | exactly the DSP image disassembled in `DSP/` |
| Gate Array Rev (ChA/ChB) | A.01.03 / A.01.03 | the XC3042 FPGA is a real versioned part |

Confirmed keypress paths: Edit Serial Number = System/Inputs -> More -> Service -> Version ->
Edit Serial Number; Individual self tests = ...Service -> Self Test -> Individual; Command Set
/ language = System/Inputs -> Remote Interface -> Command Set. The front panel also has a
`Power Ref Off/On` softkey (System/Inputs -> More) that toggles the reference oscillator.
Screenshot kept at `menu-shots/version-screen.png`.

## Re-check, 18 October 2026

A second pass over the same unit, with the N8482A still on Channel A. Only queries, `*CLS`,
`STAT:DEB:HAP` and `MEMory:TABLe:SELect` were sent; the meter was returned to local
afterwards. The table-edit target was left at HP8482A (see below).

| Command | Result | Confirms |
|---|---|---|
| `*IDN?`, `DIAG:VERS:FW?`, `DIAG:VERS:BOOT?`, `SYST:LANG?` | `...E4419B,MY45103686,A2.09.01`, `May 12 2009 10:17:44`, `A.02.00`, `SCPI` | same unit and firmware |
| `STAT:DEB:HAP 122` to `130`, and `134`, each followed by `SYST:ERR?` | 0x7A -211, 0x7B -213, 0x7C -214, 0x7D -114, 0x7E -221, 0x7F -230, 0x80 -241, 0x81 -220, 0x82 -256, 0x86 -231 | the corrected positive-code table in NOTES.md; the earlier table was two codes off |
| `DIAG:PEEK:BLOCK? 14728136,27` (0xE0BBC8) | binary, starting `00 03 FF 2D 00 00 00 03 FF 2B 00 00` | entries {3, -211, 0} and {3, -213, 0} in live RAM, matching the data record loaded with `Tools/dload.py` |
| `*_TRG?` | timeout, then `-101 Invalid character` | dormant `_TRG` keyword |
| `SERV:SENS:TYPE?`, `DIAG:DET:IMAG:BLOC?`, `DIAG:DET:IMAG:HEADER?` | `N8482A`, block types 1/1, 2/2, 3/1, 4/1, 5/2, same header text | earlier sensor results |
| `SERV:SENS:FREQ:MAX?` | timeout, then `-241 Hardware missing; Sensor connected does not contain this information` | device code 0x80 |
| `SENS:CORR:STAT?`, `SENS2:CORR:STAT?` | `0`, `0` | the undocumented `[SENSe]:CORRection:STATe` exists on both channels |
| `MEM:TABL:SEL?` | empty | no table selected |
| `MEM:TABL:LOSS?`, `MEM:TABL:LOSS:POIN?` | `-221 Settings conflict`; `+9.91E+37` then `-221` | the undocumented `MEMory:TABLe:LOSS` exists and returns 0x7E with no table selected |
| `MEM:TABL:SEL "HP8482A"`, then `MEM:TABL:GAIN?` and `MEM:TABL:LOSS?` | GAIN `98.0, 98.0, 99.5, 99.3, 98.5, ...`; LOSS `198.0, 198.0, 199.5, 199.3, 198.5, ...`; both 13 points | `LOSS?` reports 100 + each stored value (the readback bug in COMMANDS.md) |
| `MEM:TABL:SEL ""` | `-224 Illegal parameter value`; selection stays `HP8482A` | the selection cannot be cleared over SCPI. `SENS:CORR:CSET1:STAT?` is 0, so no table is applied to measurements |
| `MEM:CAT:TABL?` | `1178,15220,"DEFAULT,TABL,14","HP8481A,TABL,116",...,"CUSTOM_0,TABL,0",...` | built-in sensor tables |

## Open-item checks, 18 October 2026

| Command | Result | Confirms |
|---|---|---|
| `SENS:CORR:CFAC?`, `:CFAC? MIN`, `:CFAC? MAX`, `:GAIN1?`, `:LOSS?`, `:LOSS1?`, `:LOSS? MIN/MAX` | timeouts, then `-241 Hardware missing` | the cal-factor and LOSS1 handlers only serve 8480-series sensors; not testable with the N8482A |
| `SENS:CORR:GAIN2?`, `SENS:CORR:LOSS2?` | `0`, `0` | |
| `MEM:TABL:SEL "CUSTOM_0"`, `MEM:TABL:GAIN:POIN?`, `MEM:TABL:FREQ:POIN?` | `0`, `0` | CUSTOM_0 empty before the test |
| `MEM:TABL:LOSS 2,50` | `-222 Data out of range`; points still 0 | LOSS values are not converted from percent; limits are 0.99 / -0.5 |
| `MEM:TABL:LOSS 0.02,0.5`, then `GAIN?`, `LOSS?` | no error; GAIN `3444.4, 3396.4`; LOSS `3544.4, 3496.4`; 2 points | stored as 100 - v, 16-bit wrap, query adds 100: the predicted values exactly |
| `MEM:CLE:TABL`, `MEM:TABL:GAIN:POIN?` | `0`, no error | CUSTOM_0 restored to empty |
| `MEM:TABL:SEL "HP8482A"`, `MEM:CAT:TABL?` | `HP8482A`; catalog unchanged (`1178,15220`, CUSTOM_0 size 0) | edit target and table memory back as they were |

## 8481A on channel B, 18 October 2026

| Command | Result | Confirms |
|---|---|---|
| `SERV:SENS2:TYPE?` | `A` | 8480-series sensor recognised |
| `SENS2:CORR:CFAC?`, `MIN`, `MAX` | `100`, `1`, `150` | cal-factor limits 0.01/1.5 reported in percent |
| `SENS2:CORR:GAIN1?` | `100` | GAIN1 = CFACtor |
| `SENS2:CORR:LOSS1?`, `MIN`, `MAX` | `0`, `-50`, `99` | LOSS1 limits -0.5/0.99 reported in percent |
| `SENS2:CORR:LOSS1 2`, then `CFAC?`, `LOSS1?`, `GAIN1?` | `98`, `2`, `98`, no error | LOSS1 = 100 - CF, works |
| `SENS2:CORR:LOSS1 -20`, then `CFAC?` | `120` | |
| `SENS2:CORR:CFAC 100` | `CFAC?` 100, `LOSS1?` 0 | restored |
| Zero and calibrate channel B with the 8481A on the reference | no error | zero and cal with an 8480-series sensor |
| `OUTP:ROSC ON`, five readings | +0.00002 to +0.0021 dBm | 50 MHz reference at 0 dBm; `OUTP:ROSC ON` holds the reference on |
| `OUTP:ROSC OFF`, readings | about -39 to -41 dBm | noise floor with the reference off |

## Boot ROM read-out, 18 October 2026

| Command | Result | Confirms |
|---|---|---|
| `DIAG:PEEK:BLOCK? 131072,16` | `00 00 00 00 4E F9 00 02 00 84 ...` | matches the firmware image at 0x020000 |
| `DIAG:PEEK:BLOCK? 0,16` | `-222 Data out of range` | the handler refuses address 0 (null check) |
| `DIAG:PEEK? 0`, then `DIAG:PEEK:BLOCK?` from 2 in 256-word blocks to 0x01FFFF | 131,072 bytes, twice, identical | boot ROM A.02.00 image; about 7 minutes per pass |
| `SYST:ERR?` afterwards | `0, "No error"` | |

## Main firmware flash read-out, 18 October 2026

| Command | Result | Confirms |
|---|---|---|
| `Tools/dump_bootrom.py flash.bin 0x20000 0x100000` (`DIAG:PEEK:BLOCK?`, 256 words per query) | 917,504 bytes in 53 minutes, `SYST:ERR?` `0, "No error"` | the whole of 0x020000 to 0x0FFFFF read |
| Compared with `Tools/dload.py` on `Firmware/E4419-A2-09-01.dload` | 0x020000 to 0x0F1D5B identical (859,484 bytes, no differences) | the installed firmware is exactly the A2.09.01 download, and the two-record load is how it sits in flash |
| 0x0F1D5C to 0x0FFFFF | 58,020 bytes, all 0xFF | nothing else is stored above the image |

One pass; the exact match with the independent download file stands in for a second pass. SHA-256 of
the read-out: `FE7A323ACE83B46358DDC05C461B81F753EA055DAD0FDB1591A536E2CB618EFB`. With the boot ROM
read-out the whole 1 MB of the two flash chips is accounted for: boot ROM 0x000000 to 0x00A1AF, erased
to 0x01FFFF, firmware 0x020000 to 0x0F1D5B, erased to 0x0FFFFF.

## Sensor EEPROM read-out, 21 October 2026 (N8482A on channel A)

| Command | Result | Confirms |
|---|---|---|
| `DIAG:DET:EEPR? 0,16` | `60007054110195187024922`, no error | the query is read-only and works; the reply is each byte as `%d` with no separator, not hex |
| `DIAG:DET:EEPR? <addr>,1` for 0 to 8191, twice | identical passes, 122 s each, `SYST:ERR?` `0, "No error"` | one byte per query is unambiguous, about 64 bytes/s |
| Directory at bytes 0 to 53 | format 6, 7 blocks: types 1 (addr 54, 195 bytes), 2 (249, 286), 3 (535, 185), 4 (2144, 92), 5 (2236, 31), 18 (2267, 5), 19 (2272, 5) | every block's 8-bit checksum matches its directory entry |
| Types 18 and 19 | five bytes of `AA` each | present in the EEPROM but not registered by the firmware, so the parser ignores them; `IMAG:BLOC?` lists only 1 to 5 |
| Cal Factor block (type 3) | 9-byte header (`00 00 0A 00 00 16 02 01 00`, 0x16 = 22 points), then 22 records of u32 frequency in kHz and two u16 factors (0x4000 = 100 %): 100 kHz 97.28 %, 50 MHz 100.00 %, 1 GHz 98.91 %, 6 GHz 94.67 %; both factors equal | `DIAG:DET:IMAG:CF:NPTS?` returns 22 |
| Bytes 720 to 751 | four more records in the same format, 15 to 18 GHz at 100 % | outside the 22-point count and the sensor's range; not read by the meter |
| Bytes 2277 to 8191 | all 0xFF | no data above the Misc and type 18/19 blocks, and no wrap below 8 KB |
| `DIAG:DET:IMAG:CF:CORR? <f>` | echoes f | not usable to read an interpolated factor |

Dump: `Work/eeprom/N8482A-MY51520008-eeprom.bin` (8192 bytes, SHA-256 E4711292D212582E6D48F000D875B3517DF19B823119A6BF09BA3C23544059BA). It is this sensor's calibration data, so it stays out of the public repository.

## Not tested (write/NV, deliberately)

- POKE/POKE:BYTE, DET:EEPR writes, CPF, DOWN:FW, SERV:SNUM/OPT/VERS, DISP:*:SET, SETVar.
