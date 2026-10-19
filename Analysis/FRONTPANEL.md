# Front-panel menu system

How the softkey menus are built and what the service and test items do. Addresses
are E4418B. Traced from `fpanel.yuc` code and the menu data tables; the data tables
are walked by a generic engine, so no code points directly at an individual menu node.

## Engine

The current screen is held in a state variable at `DAT_00e1eb8c[0x74]` (a menu id).
`FUN_0009bbd8` is the softkey builder: a switch on that id, one case per screen, each
case installing a softkey descriptor through the menu-manager object `DAT_00e1eaec`:

- method `+0x70` installs a softkey set: `(descriptor, count, flags)`.
- method `+0x74` finishes the set / returns to the caller's screen.

A softkey press runs the descriptor's action function or pushes a child screen id.
Screen ids seen in the switch include 0x21, 0x2d, 0x2e, 0x2f, 0x36, 0x37, 0x3e, 0x40,
0x50, 0x5b, 0x5c, 0x9c, 0xa1, 0xa2, 0xa4, 0xa7, 0xa8, 0xaa.

## Label table

A flat array of 32-bit string pointers indexed by label id. Ids start at 1: the string for id n
is `*(0x5f5c6 + 4*n)` (E4418B), so id 1 is at `0x5f5ca`. The table runs to at least `0x5faee`.
(An earlier version used `0x5f5ca + 4*n`, which is one id off; the slot addresses below are
unaffected.) The
softkey descriptors store the label id, not the string. The service and test captions, with the
address of each table slot (not of the string):

| String | Addr | | String | Addr |
|---|---|---|---|---|
| `Service` (`Service  `) | 0x5f9fa | | `Self Test` | 0x5fa0a |
| `Self Test\|` | 0x5f9fe | | `Instrument\nSelf Test` | 0x5fa0e |
| `Display\|` | 0x5fa02 | | `Confidence\nCheck` | 0x5fa12 |
| `Version` | 0x5fa06 | | `Individual\|` | 0x5fa16 |
| `Display  ` | 0x5fa1a | | `Set\n Contrast` | 0x5fa1e |
| `Set\n Brightness` | 0x5fa22 | | `Edit Serial\n Number` | 0x5fa26 |
| `Serial^\nDiagnostic@` | 0x5fa2a | | `Serial Diag` | 0x5fa32 |
| `Battery  ` | 0x5fa46 | | `Backlight\n%s` | 0x5fa4a |
| `Test` | 0x5fa4e | | `Test1`..`Test6` | 0x5fa52..0x5fa66 |
| `Conf. Check` | 0x5fa6a | | `Tests  ` | 0x5fa76 |
| `ROM\nChecksum` | 0x5fa7a | | `RAM` | 0x5fa7e |
| `RAM\nBattery` | 0x5fa82 | | `Measure\nAssembly` | 0x5fa86 |
| `Calibrator` | 0x5fa8a | | `Keyboard` | 0x5fa8e |
| `Memory` | 0x5fa92 | | `Lithium\nBattery` | 0x5fa96 |
| `Fan` | 0x5fa9a | | `Serial Test` | 0x5fa9e |
| `UART\nConfig` | 0x5faa2 | | `Local Loop\nBack` | 0x5faa6 |
| `RS232\nLoop Back` | 0x5faaa | | `Run\nTest` | 0x5fada |
| `Command^\nSet@` | 0x5f93a | | `HP436A^\nModes@` | 0x5f942 |
| `436A Modes` | 0x5f99a | | `Zero Mode\n%s` | 0x5f99e |
| `Filter Mode\n%s` | 0x5f9a2 | | | |

## Edit Serial Number

Softkey action `0x05ebc2`. Copies the entry buffer (variable 0x61) into the serial-number
variable 0xF6 at RAM 0xE0502C, the same store `SERVice:SNUMber` writes. No format check: the
"Invalid Serial Number" branch is unreachable because the check at `0x09e1e6` always passes.
The caption lives under the Service menu alongside Serial Diagnostic, Display and Battery.

## Test1 to Test6

Each softkey calls `freqInfoTest(n)` (0x099D98) through its own thunk:

| Thunk | Call |
|---|---|
| 0x05DCD4 | `freqInfoTest(1)`: `CF: 199.9%` |
| 0x05DCE6 | `freqInfoTest(2)`: `CF: 199.9%` / `999.9GHz(M)` |
| 0x05DCF8 | `freqInfoTest(3)`: `999.9GHz` / `(20)` |
| 0x05DD0A | `freqInfoTest(4)`: `999.9GHz` / `(20,M)` |
| 0x05DD2E | `freqInfoTest(5)`: `999.9GHz` |
| 0x05DD40 | `freqInfoTest(6)`: `999.9GHz` / `(M)` |

`freqInfoTest` prints `freqInfoTest: %d` on the debug stream, clears one display field and
writes the sample string into another. The strings are the widest forms of the frequency and
cal-factor annunciators, so these are display-layout checks, not instrument tests. The labels
Test1 to Test6 sit in the same descriptor block (around 0x59E00 to 0x5A000). A `Conf. Check`
label (id 0x129) exists in the table, but no item in this block uses it.

The items at 0x05DCC2 and 0x05DD1C (descriptor items 0x059E2E and 0x059F60, label id 0, so
unlabelled) call 0x08F59C, which reads the options string (variable
0xF7), searches it for `001` (0x08F598) with `strstr` (0x0E40AC), and passes menu 0x059AD0 if
found or 0x059A14 if not to the menu manager, which opens that menu.

## Individual self-test menus

The firmware holds two softkey sets for individual self tests. Thunk to worker:

| Thunk | Worker | Failure text |
|---|---|---|
| 0x05DE06 | 0x0AC112 | ROM Check Failed |
| 0x05DE2A | 0x0AC5F0 | RAM Check Failed |
| 0x05DE4E, 0x05DFE8 | 0x0AC9F4 | RAM Battery Fault |
| 0x05DE72, 0x05E00C | 0x0ACE7E | Measurement Channel Fault |
| 0x05DECC | 0x0AE2A6 | Calibrator Fault |
| 0x05DF26 | 0x0AED9C | Keyboard Fault |
| 0x05DF4A | 0x02137C | serial diagnostic, only with the serial interface fitted |
| 0x05DFC4 | 0x0AC5AE | Memory: runs the ROM check (0x0AC112), then the RAM check |
| 0x05E066 | 0x0AEA22 | Fan Fault |
| 0x05E0BC, 0x05E0E2 | 0x0B2808 | Serial Interface Fault (local loop-back) |
| 0x05E10A | 0x0B192C, 0x0B1994 | RS-232 loop-back; needs a rear-panel loop-back connector |
| 0x05E13A | 0x0B19FA, 0x0B1994 | RS-422 loop-back; needs a rear-panel loop-back connector |

The menu lists (item-pointer arrays in the descriptor block) group them as:

- 0x05A388 and 0x05A40A: Memory, Lithium Battery, Measure Assembly, Calibrator, then Keyboard,
  Fan and further items. This is the set the E4419B shows (see "Menu navigation").
- 0x05A1B2 and 0x05A2AE: ROM Checksum, RAM, RAM Battery, Measure Assembly, then Calibrator,
  Keyboard and the serial items.

The processor-board version string selects the set: revision C or later gets the Memory /
Lithium Battery / Fan set and the fan test (COMMANDS.md, "Side effect of
SERVice:VERSion:PROCessor"). `DIAGnostic:BIST:*?` runs the same workers over the bus.

## HP436A Modes menu

Shown only while `SYSTem:LANGuage HP436A` is selected. Two softkeys, Zero Mode and Filter
Mode, set the 436A-emulation state used by the reading formatter (see COMMANDS.md, HP 436A
language).

## Softkey node format

Softkey items sit 0x2C bytes apart in the menu-descriptor region (0x56000-0x5C000). An action
item has a word 0x000E followed by the action-function pointer and a pointer back into the
block; the label id is the 16-bit word 0x16 bytes after the 0x000E word. With the table base
above, this gives Test1 to Test6 = ids 0x123 to 0x128 on the six `freqInfoTest` thunks, and the
self-test labels on their workers (ROM Checksum 0x12D on 0x05DE06, Memory 0x133 on 0x05DFC4, Fan
0x135 on 0x05E066). Menus are item-pointer arrays followed by a header word pair such as
`012C 0006`. Other item types (submenus) have not been decoded. Labels whose text ends in
`|` or `^...@` (for example `Service|`, `Self Test|`, `Display|`, `Individual|`, `Battery|`,
`Command^\nSet@`) are the submenu-opening softkeys; the plain labels are leaf actions.

## Menu navigation (confirmed on the E4419B)

The static menu object model did not trace cleanly to a keypress table, so the paths below
were mapped on the real instrument (S/N MY45103686) by reading the LCD over GPIB
(`DIAG:DISP:DUMP?`) while stepping through the front panel. The four softkeys are the keys to
the right of the display (SK1 top .. SK4 bottom); `More` pages a multi-page menu; `Prev` goes
back. The hard keys (System/Inputs etc.) are along the bottom edge.

- **System/Inputs** (hard key) opens the Sys/Inputs menu, 3 pages:
  - 1 of 3: `Remote Interface`, `Tables`, `A Input Settings`, `B Input Settings`
  - 2 of 3: `Power Ref Off/On`, `Error List`, `Recorder Output`, **`Service`**
  - 3 of 3: (not captured)
- **Service** (Sys/Inputs 2 of 3 -> Service): `Self Test`, `Display`, `Version`,
  `Serial Diagnostic` (greyed out when the serial-interface option is not fitted).
- **Self Test** (Service -> Self Test): `Instrument Self Test`, `Confidence Check`, **`Individual`**.
- **Individual** opens the **Tests** screen (2 pages): `Memory`, `Lithium Battery`,
  `Measure Assemblies`, `Calibrator` / `Keyboard`, `Fan`, `Display`, `Serial Interface`.
  This is the Memory/Lithium Battery/Fan set described above; the E4418B image has the same
  labels.
- **Version** (Service -> Version) shows model, serial, main-F/W rev and date, boot-ROM rev,
  processor-board version, system version, and per-channel DSP and gate-array revisions; its
  softkeys are **`Edit Serial Number`** (SK1) and `Done`.

So the undocumented items are reached by:

| Item | Path |
|---|---|
| Edit Serial Number | System/Inputs -> More -> Service -> Version -> Edit Serial Number |
| Individual self tests | System/Inputs -> More -> Service -> Self Test -> Individual |
| Command Set / language (incl. HP438A) | System/Inputs -> Remote Interface -> Command Set |
| Power reference on/off | System/Inputs -> More -> Power Ref |

The developer "Test1 to Test6" screen (the `freqInfoTest` display routines) is
NOT the Individual self-test menu and does not appear in normal navigation; it looks like a
dormant engineering screen. Static confirmation of the engine, node format and label table is
above; the paths here are the empirical result. Screenshot: `menu-shots/version-screen.png`.
