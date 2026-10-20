# Undocumented commands: working notes

One entry per command not described in the Programming Guide. Addresses are E4418B; "handler" is the parser callback and the address after the arrow is the routine doing the work. Behaviour comes from the decompiled code (`Analysis/E4418/decompiled.c`) checked against `listing.asm`.

Internal error returns translate through the table at 0x0C656C (see NOTES.md). Commands that fail on a sensor check queue a message with the error; the message text is quoted where known.

## IEEE 488.2 common commands

### *_TRG?

Code 0x086, handler 0x02E06E. Query only.

Runs the same trigger routine as `*TRG` (0x089B70: arms channel A/B according to the trigger source and state variables 0x8A to 0x8C), then has 0x08969A write the resulting reading into the response. In effect `*TRG` followed by a reading, in one query.

Not reachable over the bus as typed: the keyword entry is literally `_TRG`, with a leading underscore, which is not a legal SCPI program-header character. Sending `*_TRG?` to an E4419B returns `-101 Invalid character` (verified on hardware). The entry is dormant; use `*TRG` and then read. See BENCH-VERIFICATION.md.

## SERVice subsystem

### SERVice:BIST:BMONitor

Code 0x00A, handler 0x02986A → 0x0A9AD0. Command only, no parameter.

Writes 0x12345678 to the battery check word at RAM 0xE05070. The "RAM Battery" self-test (0x0ACA60) reports Passed only while that word still holds 0x12345678, and prints `address %X, batteryCheckValue` on the debug stream. The word lives in the battery-backed header that survives power-off and firmware loads, so this command clears a "RAM Battery Fault" after the backup battery has been replaced.

### SERVice:BIST:CALibrator

Code 0x014. Accepts its parameter and does nothing (no-op handler, no bound variable).

### SERVice:SENSor[1|2]:TNUMber?

Code 0x011, handler 0x029BD0 → 0x029AFC. Query only.

Returns the 16-character test number string held at offset 0xF1E of the sensor data structure (copied from the E-series sensor EEPROM). If the field is empty, a default string from 0x029AF4 is returned. Only valid when an E-series sensor (type 6) is connected; otherwise the error is queued with the text described below.

### SERVice:SENSor[1|2]:FREQuency:MAXimum? and :MINimum?

Codes 0x015 and 0x016, handlers 0x029F46 and 0x029FEE. Query only.

Return a 64-bit float from the sensor data structure (maximum at offset 0x3BA6). Valid only for an E-series sensor whose EEPROM carries the extended data block (flag at offset 0x3B7C). Messages queued otherwise: "No sensor connected", "Sensor connected is not E-series", "Sensor connected does not contain this information".

### SERVice:SENSor[1|2]:POWer:AVERage:MAXimum?, :POWer:PEAK:MAXimum?, :POWer:USABle:MAXimum?, :POWer:USABle:MINimum?

Codes 0x017 to 0x01A, handlers 0x02A096, 0x02A13E, 0x02A1E6, 0x02A28E. Query only.

Same pattern as the frequency limits: each returns a 64-bit float from the E-series EEPROM data (average maximum at offset 0x3B8E, peak maximum at 0x3B96), with the same three error messages.

## STATus subsystem

### STATus:DEBug:HAPpenning <n>

Code 0x077, handler 0x02D230. Command only, one integer.

Returns its argument as the handler status, so the parser reports internal error code *n* exactly as if a real command had failed with it. Use it to exercise the error queue and status reporting. `STAT:DEB:HAPP -11`, for example, should queue -222 "Data out of range" (via the table at 0x0C656C).

### STATus:DEBug:ENBALL

Code 0x078, handler 0x02D32E → 0x02D24A. Command only.

Sets the enable mask of every status register set (internal ids 2 to 12) to 0x7FFF, so every event propagates up to the status byte.

## Other subsystems

### [SENSe[1]]|SENSe2:CORRection:STATe <bool> and :STATe?

Code 0x146. Set handler 0x03C570 (channel check 0x03C53E), query 0x03C602. Not in the Programming Guide, which documents the CSET, DCYCle, GAIN2 and LOSS2 `:STATe` commands under the same node but not this one.

The set form stores a per-channel boolean in internal variable 0x2A and returns 0x7D (-114 "Header suffix out of range") for a channel the model does not have. The query returns that boolean. No other code reads variable 0x2A through the variable interface, so the setting appears to change nothing. On the E4419B, `SENS:CORR:STAT?` and `SENS2:CORR:STAT?` both return 0.

### MEMory:TABLe:LOSS[:MAGNitude] <value>{,<value>} and :LOSS[:MAGNitude]?

Code 0x170. Set handler 0x0425BC (worker 0x042404), query 0x0426F0 (worker 0x0425FE). `MEMory:TABLe:LOSS:MAGNitude:POINts?` shares code 0x172 with the documented `GAIN:POINts?`. The Programming Guide documents only the GAIN form.

The set form is meant as the loss view of `MEMory:TABLe:GAIN[:MAGNitude]` (0x171), the table counterpart of `SENSe:CORRection:LOSS1`, which the Programming Guide mentions only as "specifically for calibration factors". It takes up to 80 values (GAIN takes 81) for the selected sensor-calibration table or frequency-dependent offset table. With no table selected both forms return 0x7E (-221 "Settings conflict").

It does not work. Three things go wrong:

- The values are not converted from percent. GAIN values arrive as fractions (98 % as 0.98) and are checked against variables 0xCC and 0xCD (1.5 and 0.01). LOSS values arrive as typed and are checked against 0xD5 and 0xD6 (0.99 and -0.5), the LOSS1 limits for a fractional loss, LOSS = 1 - cal factor. So `MEM:TABL:LOSS 2` fails with -222 "Data out of range", and a 2 % loss has to be sent as 0.02.
- Each value v is stored as 100 - v (0x0E2A6C is double subtract) where 1 - v was intended. The table store (0x075CFC, 0x078D14) keeps round(1000 x value) in 16 bits, so 99.98 becomes 99980, which wraps to 34444.
- The query (0x0425FE) reports 100 + t for each value t that `GAIN?` would report (0x0E2A72 is double add), where 100 - t was intended.

Confirmed on the E4419B. With the built-in HP8482A table, `GAIN?` returns 98.0, 98.0, 99.5, ... and `LOSS?` returns 198.0, 198.0, 199.5, .... On the empty CUSTOM_0 table, `MEM:TABL:LOSS 2,50` was rejected with -222 and stored nothing; `MEM:TABL:LOSS 0.02,0.5` was accepted, after which `GAIN?` returned 3444.4, 3396.4 and `LOSS?` 3544.4, 3496.4, exactly as the arithmetic above predicts. `MEM:CLE:TABL` then emptied CUSTOM_0 again. Use `GAIN` with cal factors in percent; do not use `LOSS`.

`MEMory:TABLe:SELect ""` is rejected with -224, so once a table has been selected there is no SCPI way to return to "no table selected". The selection only sets the target for `MEMory:TABLe` commands; it does not apply a correction.

### TCONF <parameter>

Code 0x0B1, handler 0x030D94. Root-level command. Takes one parameter and returns success without doing anything.

### FORMat:SREGister

Code 0x0BB, handler 0x032568. Reads the FORMat:DATA setting (variable 0x75) and returns success; no other effect found.

### RTEST<n>:ONE?, :TWO?, :FOUR?, :FIVE?, :TEST?

Code 0x12F, handler 0x037C80. Root keyword `RTEST` with a numeric suffix. All five leaves share the code used by `FETCh[:SCALar][:POWer:AC]?` and run the same handler; the leaf only sets the channel number the handler sees (1, 2, 4, 5, or the suffix for `TEST`). It looks like a test hook for checking channel-number range handling: channels 4 and 5 do not exist. Use FETCh? instead.

### READD (E4419B only)

Accepted by the E4419B as an alias of the `READ` root keyword (same command codes). Not present in the E4418B tree. Confirmed on an E4419B: `READD?` behaves exactly like `READ?` (both return -213 "Init ignored" with INIT:CONT ON), where an unknown header would give -113. See BENCH-VERIFICATION.md.

### SYSTem:LANGuage option list

| Model | Accepted values |
|---|---|
| E4418B | SCPI, HP436A, HP437, HP437B |
| E4419B | SCPI, HP438, HP438A |

The manuals list only SCPI plus HP437B or HP438A. HP437 and HP438 are short aliases. HP436A selects a separate HP 436A emulation (see NOTES.md, open items).

## HP 436A language (E4418B only)

Selected with `SYSTem:LANGuage HP436A` or from the front panel (Command Set menu, action 0x05CEEC passes language index 1). The E4418B manuals do not mention it. In this mode the GPIB controller is driven by a separate HP-IB driver ("Initialising hp436A HPIB driver", 0x083EFE) and each received character goes through a small state machine (0x084D1C). Every 436A code is translated into SCPI commands that the normal parser executes (0x085D8A), so the emulation behaves exactly like the equivalent SCPI settings.

Decoder at 0x084346:

| Code | 436A meaning | SCPI issued |
|---|---|---|
| `+` | Cal factor off | `SENSE:CORR:GAIN1 100` |
| `-` | Cal factor on | `SENSE:CORR:GAIN1 <n>`, n computed from the value stored at instance offset 0xCE |
| `1` to `5` | Manual range 1 to 5 | Range kept in the instance state; 0x08418A updates the recorder output scaling |
| `9` | Autorange | Range state set to auto |
| `A` | Watts | `CALC:REL:STAT OFF`, `UNIT:POW W` |
| `B` | dB relative | `UNIT:POW DBM`, `CALC:REL:STAT ON` |
| `C` | dB reference (capture reference) | `UNIT:POW DBM`, `CALC:REL:MAGN:AUTO ONCE` |
| `D` | dBm | `CALC:REL:STAT OFF` if needed, `UNIT:POW DBM` |
| `H` | Hold | `TRIG:SOURCE HOLD` |
| `I` | Trigger immediate | `SENSE:AVER:STAT OFF`, `TRIG:SOUR BUS`, `TRIG:DEL:AUTO OFF`, `INIT:CONT ON`, `TRIG:IMM` |
| `R` | Free run | `SENSE:AVER:STAT ON`, `TRIG:SOUR IMM`, `TRIG:DEL:AUTO OFF`, `INIT:CONT ON` |
| `T` | Trigger with delay | `SENSE:AVER:STAT ON`, `INIT:CONT OFF`, `TRIG:SOUR IMM`, `TRIG:DEL:AUTO ON`, `INIT:IMM` |
| `V` | Free run with delay | same as `R` |
| `Z` | Zero | `UNIT:POW W`, `CAL:ZERO:AUTO ONCE` |

Codes the original 436A did not have:

| Code | Effect |
|---|---|
| `FR <value> KZ\|MZ\|GZ` | Sets the measurement frequency (`SENSE:FREQ`). Only with a sensor that uses frequency correction (check at 0x083FD8). Same syntax as the 437B. Handler 0x0848DE. |
| `X1`, `X2` | `SENSE:SPEED 20` or `SENSE:SPEED 40` (0x084A4E) |
| `SYST:LANG SCPI`, `SYST:LANG HP437B`, `SYST:LANG HP436A`, `SYST:LANG?` | Recognised by a character matcher (0x0847FC) so the mode can be left without the front panel. Long forms SYSTEM:LANGUAGE also accepted. |

The reading output uses the 436A status, range and mode characters kept in the instance structure (offsets 0xA6, 0xA7, 0xA8). The front panel has an "HP436A Modes" menu with Zero Mode and Filter Mode softkeys for this mode.

## Front panel

Items in the firmware that the User's Guide does not describe:

- **Edit Serial Number** softkey. Its action (0x05EBC2) copies the entry buffer (variable 0x61) into the serial-number variable 0xF6, stored at 0xE0502C. There is no format check; the "Invalid Serial Number" branch is unreachable because the check at 0x09E1E6 always passes. It sits under the Service menu; see FRONTPANEL.md.
- **HP 436A** as a Command Set choice, and the **HP436A Modes** menu (Zero Mode, Filter Mode).
- **Test1** to **Test6** call `freqInfoTest(1)` to `freqInfoTest(6)` (0x099D98) through thunks at 0x05DCD4 to 0x05DD40. Each writes a full-width sample string such as `CF: 199.9%` or `999.9GHz(M)` into the frequency and cal-factor display fields, to check the field layout; they are not instrument self tests. The softkey at 0x05DCC2/0x05DD1C searches the options string for `001` and opens menu 0x059AD0 if found, 0x059A14 if not. See FRONTPANEL.md.
- **Individual self tests**: the firmware has two softkey sets. The one the E4419B shows under Service > Self Test > Individual is Memory, Lithium Battery, Measure Assembly, Calibrator, then Keyboard, Fan, Display and Serial Interface. The other starts ROM Checksum, RAM, RAM Battery, Measure Assembly, with Calibrator, Keyboard and the serial items on its second page. Both run the same BIST workers as `DIAGnostic:BIST:*?`, plus a fan test (0x0AEA22) and serial loop-back tests; the mapping is in FRONTPANEL.md.
- A DAC test that writes sine and waveform patterns to the recorder DACs at 0xF01800/0xF01802 ("DACtest entered: reset to stop.", 0x0AF3FA and 0x0AF546). Nothing references these routines, so it cannot be started in this build.

The key codes returned by `DIAG:BIST:KEYB:KEY?` follow the table at 0x0AEEA0: 0 ZERO_CAL, 1 FREQ_CAL_FAC, 2 DOWN_ARROW, 3 RIGHT_ARROW, 4 UP_ARROW, 5 MORE, 6 SOFTKEY4, 7 SOFTKEY3, 8 SOFTKEY2, 9 LEFT_ARROW, 10 dBm_WATTS, 11 REL_OFFSET, 12 MEAS_SETUP, 13 SAVE_RECALL, 14 SYSTEM_INPUTS, 15 PREV, 16 PRESET_LOCAL, 17 SOFTKEY1, 18 ZOOM, 19 SPLIT_WINDOW.

## Side effect of SERVice:VERSion:PROCessor

The processor-board version string (variable 0xF8, at 0xE05018) also switches features. FUN_000212E8 upper-cases it and looks it up in a version table at 0xE05144: an empty string or a version below "C" gives 0, "C" or later gives 1 (strings compare by length, then alphabetically). When it is 1:

- Service > Self Test > Individual shows the Memory, Lithium Battery, Measure Assembly, Calibrator, Keyboard, Fan set instead of the ROM Checksum, RAM, RAM Battery set (0x05DAAE, 0x05BBD6, 0x05E08A);
- the two lists of self tests built at 0x0A9D20 and 0x0AA2D0 gain a "Fan Fault" entry, and the Fan softkey's worker (0x0AEA22) runs the test; with 0 it does nothing.

The E4419B on the bench reports board version C. Setting the string to an earlier letter would hide the fan test; set it back to the original value if you change it.

## Side effect of SERVice:VERSion:SYSTem

Writing the system version string (stored at 0xE05004) also compares it with "2" (0x022CEC). The result sets the PLD mode flag (variable 0x1D) and reprograms the PLD: "2" selects mode 1 (write to 0xF011C0), anything else mode 0 (write to 0xF01180). This looks like a processor-board variant switch; set the string back to its original value if you change it.

## DIAGnostic subsystem

The keyword is `DIAGnostic[1|2]`; the suffix selects the channel (default 1). On the E4418B use 1 or nothing. Every command below exists in both models unless noted. Many handlers also print a line on the debug console (`diag_xxx_q ch %d result ...`).

Parameter names in angle brackets follow the converter types: `<num>` numeric (MIN/MAX/DEF accepted), `<bool>` ON|OFF|1|0, `<char>` one of the listed words.

### Memory access

| Command | Code | Handler | Behaviour |
|---|---|---|---|
| `DIAGnostic:POKE <addr>,<word>` | 0x12A | 0x0377BC | Writes a 16-bit word. Odd address: -224. |
| `DIAGnostic:PEEK? <addr>` | 0x12B | 0x037836 | Reads a 16-bit word. Odd address: -224. |
| `DIAGnostic:PEEK:BLOCK? <addr>,<n>` | 0x12C | 0x037968 | Reads n words (n ≤ 256, else -222); returns 2n bytes, high byte first. |
| `DIAGnostic:PEEK:BYTE? <addr>` | 0x12D | 0x0379E6 | Reads one byte, any address. |
| `DIAGnostic:POKE:BYTE <addr>,<byte>` | 0x12E | 0x037A56 | Writes one byte, any address. |

Addresses are the CPU's physical addresses (see NOTES.md for the map). A bus error from an unmapped address is not trapped by these handlers.

### Battery-backed memory

- `DIAGnostic:CPF` (0x0BF, 0x0328E0) clears the power-up magic long at 0xE05000. The next power-up is treated as a first power-up: the BSS area (0xE104BC to 0xE2D244 on the E4418B) is zeroed, which drops anything kept there across power cycles. Command only.
- `DIAGnostic:BIST:BMONitor` (0x110) is the same routine as `SERVice:BIST:BMONitor`: re-arms the RAM battery check word.

### Firmware download entry

`DIAGnostic:DOWNload:FW:HPIB`, `:RS232`, `:RS422` (0x0E7 to 0x0E9, handlers 0x035204, 0x03536A, 0x0354CA). Command only, no parameter.

Each one prints `Executing DIAG:DOWN:FW:<port>`, stops the PIT timer, suspends the display process and the other tasks, shows "Downloading over <port>..." on the display, stores a boot-monitor command line (`hm HPIB`, `hm SERIAL_RS232`, `hm SERIAL_RS422`; see BOOTROM.md) and a return address in low RAM (0xE007FC, 0xE007F8), then calls 0x0A2A44 and never returns. The instrument is now waiting for a firmware image on that port. The RS-232 and RS-422 forms return error 0x80 if the serial interface is not fitted (check at 0x02137C). Only a power cycle gets the instrument out of this state if no download follows.

### Debug monitor and RTOS reports

- `DIAGnostic:MONitor` (also `:MON`, 0x0F9, 0x0362E4) executes `TRAP #0`. The firmware never sets the vector base register, so the handler belongs to the boot ROM (not in these images); it is presumably the boot ROM's debug monitor. The console input routine at 0x0D3BCE executes the same `TRAP #0` when it receives Ctrl-C.
- `DIAGnostic:PSOS:PS`, `:SREP`, `:MREP`, `:MHREP`, `:SEMREP` (0x103 to 0x107) print pSOS+ reports on the debug console: task list, system report, memory region report, memory heap report and semaphore report (routines 0x0CD8C4, 0x0CE0C6, 0x0CD9B0, 0x0CDC1E, 0x0CF8A6). Interrupts are masked while each report prints. Nothing is returned over the remote interface.

### Version queries

- `DIAGnostic:VERSion:FW?` (0x11A, 0x037248) returns the firmware build date and time as one string, for example "May 12 2009 10:15:44" (strings at 0x021094 and 0x0210DC, the compile time of version.c).
- `DIAGnostic:VERSion:BOOT?` (0x11B, 0x0372AC) returns the boot ROM version. 0x021120 calls the boot ROM with `TRAP #14`, divides the returned number by 100 and formats it as `A.%05.2f` (a value of 102 gives "A.01.02").

### Sensor EEPROM

E-series sensors (internal type 6) carry an EEPROM that the measurement board reads over the instrument bus. These commands access it directly.

| Command | Code | Behaviour |
|---|---|---|
| `DIAGnostic:DETector:EEPRom <addr>,<byte>` | 0x0DA set, 0x033742 | **Writes one byte to the sensor EEPROM** (bus command (ch<<10)\|0x3004). No range or type check. |
| `DIAGnostic:DETector:EEPRom? <addr>,<n>` | 0x0DA query, 0x0338DA | Reads n bytes (1 to 16) from addr and returns them as decimal numbers printed with `%d` and no separator, so a reply for n > 1 cannot be split reliably; read one byte per query (`Tools/dump_sensor_eeprom.py`). Confirmed on the E4419B: `DIAG:DET:EEPR? 0,16` returns `60007054110195187024922`. n=0 or n>16 returns the text "Min block size is 1" / "Max block size is 16". |
| `DIAGnostic:DETector:EEPRom:TEST` and `:TEST?` | 0x0C5 | Runs the EEPROM write test (0x06C022) on addresses 3000 to 3255 and 3500 to 3755: save each window, write a descending pattern, read back, restore. Query form returns 1 (pass) or 0. Prints progress on the debug console. E-series only, else error 0x80. |
| `DIAGnostic:DETector:EEPRom:READ` | 0x0DF | Pauses measurement, re-reads the sensor data from the EEPROM (sensor method at +0x3BE2), resumes. E-series only. |
| `DIAGnostic:DETector:READEEPRom?` | 0x0E0 | Re-parses the EEPROM image held at offset 0x288 of the sensor structure and returns the parser status (0x06F088). |
| `DIAGnostic:DETector:IMAGe:HEADER?` | 0x0E1 | Returns "model number : '%s', serial number : '%s', place : '%s', date : '%s', stinger rev : %d.%d" from the sensor header (offsets 0xF04, 0xF14, 0xF2E, 0xF36, 0xF3E, 0xF3F). |
| `DIAGnostic:DETector:IMAGe:BLOCks?` | 0x0E2 | Lists the data blocks present in the image: one "Block type %d, format %d" line per block slot (21 slots of 0x8C bytes from offset 0x29C) whose tag is 6. |
| `DIAGnostic:DETector:EEPRom:RDI` | 0x0C4 | Accepts and ignores its parameter. |
| `DIAGnostic:EEPROM:DUMMY:READALL?` | 0x0C0 | Parses a built-in dummy EEPROM image (RAM 0xE105BC) and returns the parser status. |
| `DIAGnostic:EEPROM:DUMMY:READDATA? <n>` | 0x0C1 | Returns its argument; placeholder. |
| `DIAGnostic:EEPROM:DUMMY:WRITEDATA`, `:AUTOFILL` | 0x0C2, 0x0C3 | Accept two parameters and do nothing. |

> CAUTION: `DIAG:DET:EEPR <addr>,<byte>` changes the sensor's own calibration store. Read the whole EEPROM first (`DIAG:DET:EEPR? <addr>,16` in steps) so it can be put back.

### Sensor calibration image (RAM copy)

These read and write the copy of the sensor tables held in RAM, not the EEPROM. Changes last until the sensor is re-read or disconnected.

- `DIAGnostic:DETector:IMAGe:V2P:POINter <n>` / `:ENTRy <value>` / `:ENTRy? <row>,<col>` / `:DONe` (0x0C7, 0x0C6, 0x0C8). Voltage-to-power linearity table at offset 0xF7E: rows of 65 longwords (0x104 bytes). POINter sets a running index (variable 0x118); each ENTRy write stores at the index and advances it. DONe recomputes the derived value at offset 0xF40 through the sensor method at +0x3BEE.
- `...:IMAGe:TC:POINter`, `:TC:ENTRy <word>`, `:TC:ENTRy? <n>` (0x0CD, 0x0CC). Temperature-compensation table of 16-bit words at offset 0x14EE, same pointer scheme (variable 0x119).
- `...:IMAGe:TCCoeffs:POINter`, `:TCCoeffs:ENTRy <value>`, `:TCCoeffs:ENTRy? <row>,<col>` (0x0D3, 0x0D1). Coefficient table of doubles at offset 0x157A, rows of 0x28 bytes (variable 0x11A).
- `...:IMAGe:ZTC:MEASurements?`, `:ZTC:ZADJ:UPPer?`, `:ZADJ:LOWer?` (0x0C9 to 0x0CB) return channel words at offsets 0xC4, 0xC8, 0xCC.
- `...:IMAGe:TC:MEASurements?`, `:TC:CALibration:UPPer?`, `:LOWer?` (0x0CE to 0x0D0) return a double computed from two channel longs (offsets 0xD0/0xD4, 0xD8/0xDC, 0xE0/0xE4).
- `DIAGnostic:TC:UPDate?` (0x0D2) runs the sensor's temperature-compensation update (method +0x3BFA) and returns 1, or 0 if no E-series sensor.
- `...:IMAGe:CF:FREQ <f>` / `:CF:FREQ?` (0x0DB) set and read the frequency used for the cal-factor lookup (sensor method +0x3C32, value at 0x1AAA). `:CF:CORRect? <f>` returns the interpolated correction at f (method +0x3C2A). `:CF:NPTS?` returns the table length (0x1AA4). `:CF:MAKEDUMMY` builds a dummy cal-factor table (0x07AD32). These act on channel A regardless of suffix.

### Measurement board and DSP

Bus command words are listed in NOTES.md.

| Command | Code | Behaviour |
|---|---|---|
| `DIAGnostic:DSP <addr>,<value>` | 0x0E5 set | Writes a 16-bit word into ADSP-2101 data memory (bus command 0xF804). |
| `DIAGnostic:DSP? <addr>` | 0x0E5 query | Reads a DSP data-memory word (bus command 0xF002). |
| `DIAGnostic:RADC?` | 0x102 | Returns a raw ADC reading (bus command (ch<<10)\|0x1800, 4 bytes). |
| `DIAGnostic:DETector:TEMPerature?` | 0x0D8 | Waits for the channel to go idle, then returns the sensor thermistor reading (bus command 0xE005). |
| `DIAGnostic:DETector:MRESistor?` | 0x0D9 | Returns the sensor mount-resistor measurement (bus command 0xE000\|n), used to identify sensor type. |
| `DIAGnostic:DETector:GSWitch <1\|2>` | 0x0D5 | Forces detector gain path 1 or 2 (0x064ABA) and records it at sensor offset 0x46. Always returns internal code -56 (-310 "System error"), apparently on purpose so the change is flagged. |
| `DIAGnostic:DETector:SPEed <bool>` | 0x0D6 | Sets the board's speed bit (0x06460A). |
| `DIAGnostic:DETector:RCALibrate <bool>` | 0x0D7 | Sets the board's reference-calibrator bit (0x064A5E). |
| `DIAGnostic:RCALibrate <bool>` | 0x0F7 | Switches the RCAL path on the board (0x064668). |
| `DIAGnostic:DETector:THIRd` | 0x0D4 | E-series sensors with the flag at offset 0xEBA: writes DSP location 0x3815 = 0x105, runs one measurement cycle, writes 0x3815 = 0. |
| `DIAGnostic:SRATe <bool>` and `?` | 0x0FA | Selects one of two sample-period constants (0x5140 for 1, 0xB2E8 for 0) for the channel and recomputes the filter length from 750000 / period. |
| `DIAGnostic:RCALibrate:SFACtor:DTYPe` | 0x0F8 | Accepts a parameter; no effect found. |

### Measurement parameter overrides

`DIAGnostic:SETVar <name>,<value>` and `DIAGnostic:SETVar? <name>` (0x0E6, handlers 0x034C3C and 0x034ECE). Names and what they touch:

| Name | Index | Set | Query |
|---|---|---|---|
| MEAS | 0 | variable 0x10D (double) | same |
| G_SENSOR | 1 | sensor offset 0x0C | same |
| CAL_OFFSET | 2 | sensor offset 0x12 | same |
| RAW_ADC | 3 | variable 0x111 | same |
| SENSOR_RANGE | 4 | sensor offset 0x46 = 1 or 2 (0 → 1); returns -310 | 0 or 1 |
| TC_VALUE | 5 | none | variable 0x113 |
| ZERO_LOWER | 6 | sensor offset 0x3E | same |
| ZERO_UPPER | 7 | sensor offset 0x42 | same |
| CAL_CHAR | 8 | sensor offset 0xF40 (double) | same |
| VTHerm | 9 | variable 0x112 | same |
| RCAL_DELAY | 10 | channel offset 0xA8 | same |
| FC_VALUE | 11 | none | variable 0x114 |
| TC_THRESHOLD | 12 | variable 0xC1 | same |
| POSTAVE_MEAS | 13 | none | variable 0xC4 |
| PREAVE_MEAS | 14 | none | variable 0xC3 |

Other override commands, all per channel:

- `DIAGnostic:AVERage:ZERO:LOWer|UPPer <n>`, `:RCAL:LOWer|UPPer <n>`, `:CAL:LOWer|UPPer <n>` (0x0F1 to 0x0F6) set the filter length (a power of two) used while zeroing, during reference-cal and during calibration, separately for the lower and upper range. 0 restores the per-sensor default. The queries return the current length.
- `DIAGnostic:RPAD <dB>` and `?` (0x0EB) store a reference-pad attenuation (sensor offset 0xEBC) and its linear ratio (10^(dB/10), offset 0xEC4).
- `DIAGnostic:SDET:SPECified <bool>`, `:SDET<n>:MCLevel:UPPer|LOWer <value>`, `:SDET<n>:TSHifts:UPPer|LOWer <value>` (0x0EC to 0x0EE) store sensor-detector limit values per index n (offsets 0xECE, 0xEE0, 0xEF0).
- `DIAGnostic:ZDT:SPECified <bool>`, `DIAGnostic:ZDT <value>` (0x0EF, 0x0F0) store a zero-drift test limit (offsets 0xEF8, 0xEFA).
- `DIAGnostic:CALibration:CHECk`, `:ZERO:CHECk`, `:ZERO:DRIFt:CHECk`, `:STIMe:ANALog:FOURty` store their parameter in variables 0xBA, 0xBC, 0xBD and 0xBF; the queries read them back. `:STIMe:ANALog:TWENty`, `:STIMe:DSP`, `DIAGnostic:TEST`, `DIAGnostic:BIST:POST` accept a parameter and do nothing.

### Status reports

- `DIAGnostic:SETTings?` (0x0E3) returns a text block: `CHANNEL A ***...` followed by cal_factor, cal_rcf, meas_freq, conf_frequency, meas_units, init_continuous, trig_source, trig_delay_auto and window resolution.
- `DIAGnostic:STATes?` (0x0E4) returns MODE, WAITING, TRIGGER STATE, TRIGGER NEXT STATE, SENSOR and SENSOR POSITION for the channel.

### Display

| Command | Code | Behaviour |
|---|---|---|
| `DIAGnostic:DISPlay:DUMP? <row>` | 0x0FB | Returns 40 bytes of SED1330 display memory starting at row×40, as `0x%02X` values separated by commas. 40 bytes is one 320-pixel line. |
| `DIAGnostic:DISPlay:BRIGhtness:UP`, `:DOWN` | 0x0FC, 0x0FD | Step the backlight drive. |
| `DIAGnostic:DISPlay:BRIGhtness:SET?` | 0x0FE | Runs the backlight current calibration: steps the drive while reading LED_I_MON until the gradient settles, prints "current measurement and gradient" lines on the debug console, saves the result in battery-backed RAM (0xE05068) and returns it. Takes several seconds. |
| `DIAGnostic:DISPlay:CONTrast:UP`, `:DOWN` | 0x0FF, 0x100 | Step the LCD contrast. |
| `DIAGnostic:DISPlay:CONTrast:SET` | 0x101 | Saves the current contrast step to battery-backed RAM (0xE0506C) and stores it as a percentage in internal variable 0xEF. |
| `DIAGnostic:DISPlay:DBUFF <bool>` and `?` | 0x108 | Double-buffer flag. OFF also clears the buffer pointers. |
| `DIAGnostic:DISPlay:BUFF <1\|2>` | 0x109 | Selects display page 1 or 2; other values return -224. |
| `DIAGnostic:DISPlay:RBUFferring` | 0x10A | Selects page 2. |
| `DIAGnostic:BIST:DISPlay:TEST <n>` | 0x115 | Runs display test pattern n (0x0B3112). |

### Keyboard, battery pack, self tests

- `DIAGnostic:BIST:KEYBoard:DISable` / `:ENABle` (0x116, 0x117) lock and unlock the front-panel keys. `DIAGnostic:BIST:KEYBoard:KEY?` (0x119) returns the code of the last key pressed, or 0.
- `DIAGnostic:BATTery:FITted?`, `:ABSent?`, `:CAPacity?`, `:CHARge?`, `:TIME?` (0x10B to 0x10F) report the optional battery pack: present flag, absent flag, capacity byte, charge word and remaining time.
- `DIAGnostic:BIST:MEAS?` (0x111) runs the measurement-board self test and returns its result byte.
- `DIAGnostic:BIST:ANALog? <point>` (0x112) returns the measurement-board analog multiplexer reading for one test point: DUMMY1, DUMMY2, THERMA, LED_I_MON, DUMMY3, REF_TP2, REF_TP1, LCD_CONT1. Channel 1 only.
- `DIAGnostic:RECorder[1|2]:TEST?` (0x113) and `DIAGnostic:TTL:TEST?` (0x114) run the recorder and TTL output tests and return a result byte.
- `DIAGnostic:RS232:TEST?`, `DIAGnostic:RS422:TEST?` (0x122, 0x123) run the serial loopback tests (0x0B2EE2, 0x0B2FAE) and return a text result. Both run only while internal variable 0xAE is 0 (error 0x7E otherwise); the RS-422 test also needs the RS-422 interface fitted (0x0213C6).
- `DIAGnostic:UART:REGS?` (0x124) returns the serial controller register dump built by 0x022BB0.
- `DIAGnostic:RMTIO:TEST?` (0x127) always returns "IO 1 & 2 : passed / IO 3 & 4 : passed". The test is not implemented.

### Interface and hardware selection

- `DIAGnostic:PLD:MODE` (0x128) reads variable 0x1D and writes the PLD mode register: 0 to 0xF01180 or 1 to 0xF011C0. Prints "PLD mode 0|1".
- `DIAGnostic:MONGO:PORT <string>` (0x129) passes a port description string to the sensor-port manager (method +0x73E). The default string is "Sensors Connected Front and Rear". Returns -310 if the manager rejects it.
- `DIAGnostic:HADDRess <n>` is an alias of `SYSTem:COMMunicate:GPIB[:SELF]:ADDRess`.
- `DIAGnostic:RECorder[1|2]:CHANnel`, `:STATe`, `:LIMit:UPPer`, `:LIMit:LOWer` (0x029 to 0x02D) are the recorder-output commands also reachable as OUTPut:RECorder. CHANnel (not in the manual) chooses which measurement channel drives the recorder output; it reports "recorderChannel : HARDWARE MISSING" when channel B is chosen on a single-channel unit.
- `DIAGnostic:UNIT:SCPIPOWER <value>` / `?`, `DIAGnostic:UNIT:SCPIRATIO <value>` / `?` (0x0BC, 0x0BD). Set form does nothing. Query returns internal variable 0x18 (or 0x1A), or its MIN/MAX companions (0x90/0x91, 0x92/0x93) when given MIN or MAX. `DIAGnostic:UNIT:OLDWATTS` accepts a parameter and does nothing.

## Codes with handlers but no keyword

Codes 0x01B to 0x039 (except 0x029 to 0x02D), 0x088, 0x089, 0x125 and 0x126 have handlers but no keyword reaches them in this build, so they cannot be sent. They include FDO table on/off (0x026, 0x027), two alternate measurement routines for A- and D-type sensors (0x125, 0x126) and several stubs.
