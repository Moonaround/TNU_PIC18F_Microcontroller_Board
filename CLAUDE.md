# CLAUDE.md: TNU PIC18 board (KiCad 10.0.7)

## Project
Re-creating the Trevecca (TNU) PIC18 development board in KiCad from the original Cadence design. Final goal after this board is done: reuse it as the base for a solar panel control board.
Reference for how the finished board should look: the physical "TNU PIC18 Board V2 Rev A" (purple, 2-layer, through-hole DIP-40 PIC18F47Q84, USB via FT231XS, 4 corner mounting holes, 4 fiducials, bottom-side ground pour).

Files that matter
- `tnu_pcb_pic40.kicad_sch`: root sheet (USB-to-UART, FT231XS, USB connector)
- `microcontroller.kicad_sch`: sub-sheet (PIC18, LD1117-3.3, oscillator, PICkit3 header, test points)
- `tnu_pcb_pic40.kicad_pcb`, `tnu_pcb_pic40.kicad_pro`
- Ignore `.lck` files and `.kicad_prl`.
- Source of truth for parts: the Cadence BOM and schematic PDF. Source of truth for rules: `TNU_Design_Projects_Standards_and_Constraints` (docx). If either file is not in `docs/`, ask the user to add it. Re-check every rule below against it before relying on it.

## Working rules (important)
1. Commit before any edit session. Never edit `.kicad_sch`/`.kicad_pcb` while KiCad has the project open; ask the user to close it.
2. After every change, run ERC and DRC and report the counts before and after:
   - `kicad-cli sch erc --output erc.rpt tnu_pcb_pic40.kicad_sch`
   - `kicad-cli pcb drc --output drc.rpt tnu_pcb_pic40.kicad_pcb`
3. Never invent footprint or symbol dimensions or pinouts. A custom `.kicad_mod` or `.kicad_sym` may only be written from a datasheet the user provides (or a KiCad library part that exists locally, verified by listing the library folder). If no datasheet, say what is missing.
4. Prefer small, reviewable diffs. Do not reformat files or regenerate UUIDs. Do not change net names that other sheets rely on.
5. Wire moves, sheet pins and labels are safer in the KiCad GUI. Offer exact GUI steps instead of hand-editing S-expressions when the edit is more than a property change.

## Design standard (summary; verify against the docx)
- Two-layer FR-4, 1.6 mm, 1 oz copper, HASL; board under 100 mm in each dimension (current outline 66.2 x 99 mm).
- Traces at least 7 mil wide with 7 mil spacing; 10 mil signal traces preferred for through-hole or less dense designs; power traces wider (20-25 mil typical).
- Vias: absolute minimum 12 mil (0.3 mm) hole and 18 mil (0.45 mm) diameter; standard signal via 30 mil pad / 13 mil drill; larger vias for power.
- IC footprints follow JEDEC JEP95. USB connector: Type B USB 2.0 right angle (Molex 0670689000 or Mill-Max 897-43-004-90-000000). Preferred wire connectors: Molex KK 254 (0.1") polarized.
- Bottom layer: ground plane. Four fiducials: 1 mm copper dot with 3 mm solder mask opening.
- Passives: 0603 from the stocked-value list; 4.7K is only in the 0805 list. Electrolytics and connectors per BOM. Connectors polarized unless the device requires otherwise.
- Silkscreen: Trevecca logo, "TNU PIC18 Board V2 Rev A", every header pin labelled with DIP pin number and signal, polarity and pin-1 marks.
- Safety limits (matter for the later solar board): DC up to 24 V (48 V only with special approval); no battery charging unless a commercially certified charging module is used; current limiting above 1-2 A.

## Known problems (from ERC baseline: 26 violations)
Real wiring bugs: fix first
1. Clock on wrong MCU pin: CLK net (U4 OUT) is on U3 pin 14 (RA6/OSC2/CLKO, output). It must go to pin 13 (OSC1/CLKIN). Pin 13 is currently unconnected.
2. USB UART broken between sheets: `USB_TXD`/`USB_RXD` have hierarchical labels but the root sheet's Microcontroller sheet symbol has no matching sheet pins. Add sheet pins and wire them to U1 (check TX/RX are not swapped against the Cadence PDF). `USB_RST` on the root sheet should be a local label, not a hierarchical label.
3. Labels `SCL`, `SDA`, `RB3` attach to only one pin: R11, R12 and R8/D3 are not connected to MCU pins RC3, RC4, RB3.
4. U3 uses the PIC18F452 symbol with a PIC18F47Q84 value. A correct PIC18F47Q84 (PDIP-40) symbol is needed. Check the KiCad symbol library first; otherwise build one from the datasheet pin table.

Expected noise
- FT231XS unused pins DTR (1), RI (5), DSR (7), DCD (8): add no-connect flags.
- Stray unconnected power symbol `#PWR012` (probably a floating "usb" ground): delete.
- Missing `PWR_FLAG` on input rails: `#PWR04`, `#PWR021`, U2 pin 3 (J1 input rail, USB 5 V rail).
- "usb" ground symbols merge into GND (no separate net on the PCB). Cosmetic: switch them to GND.
- Re-enable the ignored ERC check "Assigned footprint doesn't match footprint filters".

## Footprint fix list (current PCB vs BOM/reference board)
| Ref | BOM part | Current footprint | Action |
|---|---|---|---|
| U3 | PIC18F47Q84 DIP-40 | DIP-40 | Keep footprint; replace symbol (see above) |
| U4 | CTS 636L3C016M00000 | EuroQuartz XO53 5.0x3.2 | Footprint size is right; verify pin map (1 EN, 2 GND, 3 OUT, 4 VCC) against datasheet. Symbol library entry points to a wrong Abracon 3.2x2.5 footprint |
| CN1 | Mill-Max 897-43-004-90-000000 | OST USB-B1HSxx | Needs a custom footprint from the Mill-Max datasheet (connector overhangs the top edge on the real board) |
| J1 | W237-102 | Phoenix PT-1,5-2-5.0 | Needs the correct footprint from the datasheet |
| C10 | Panasonic EEE-FK1H220P | CP_Elec_5x5.4 | Likely too small; verify body size in datasheet |
| P1-P19, TP1-TP36 | Keystone 5006 (per BOM) | TestPoint_Pad_D1.0mm | Through-hole test points in 0.1" rows on the real board; check BOM and library (KiCad has Keystone TestPoint footprints) |
| FI1-FI4 | Fiducial | 2 mm mask opening | 1 mm copper, 3 mm mask opening |
| R11, R12 | 4.7K | THT axial | Real board keeps axial; confirm with professor before changing to 0805 |
| TP16, TP18, TP44 | n/a | TP44 exists, TP16/TP18 missing | Likely a renumbering slip; restore BOM numbering |
| BOM specs | n/a | n/a | Some resistors are 5% (27, 180) and some caps 25 V; standard wants 1% and 50 V; review |

## PCB tasks (after schematic is clean)
1. Restore the Edge.Cuts outline (rectangle 66.2 x 99 mm) and mounting holes.
2. Board Setup: 2-layer stackup 1.6 mm; constraints per standard; net classes for power.
3. Re-place parts to match the reference board photo, route, add bottom GND zone.
4. Silkscreen and logo, then DRC clean, then Gerbers + drill files + BOM.
