# Optical Angular Position Encoder – Design Review (rev C)

**Sources:**
- `optic angular_final.pdf`: EE316 P4 report. Fig. 2 is the analog front end.
- `Counter_gurup2.TXT`: SDF netlist exported from Proteus `Counter_gurup2.pdsprj`. This is the **primary connectivity source for the counter/display**.
- `circuit_diagram.jpeg` / `schematics_rep.jpeg`: block-level only.

Rev C is the complete system on one board: optical front end, direction logic, BCD counters, boundary/preset logic and the 3-digit display.

## 1. Blocks (schematic sheet, A0)
| # | Block | Parts |
|---|---|---|
| 1 | Optical front end: zero-bias photodiodes, TIA, Schmitt | U1 LM358, U2 LM393, R1–R14, C2–C3, J2 (LEDs), J3 (photodiodes), J4 (test/ext) |
| 2 | Direction detection and glitch blocker (report blocks 1, 2, 4, 7, 8, 14) | U22, U23 74HC74; U24–U28 CD4081; U29–U30 CD4071; U31–U32 74HC04; U33 74HC86 |
| 3 | Tens/hundreds counters | U7, U8 CD4029 |
| 4 | Boundary comparators (blocks 9–11) | U11, U12, U14, U16, U18, U19: CD4063 |
| 5 | Preset multiplexers (block 5/6) + jam AND | U3, U9, U10, U13, U15, U17, U20, U21: 74HC157; U34, U35: 74HC21 |
| 6 | Display (block 13) | U4–U6 74LS47, R15–R35 330 Ω, DS1–DS3 SA39-11EWA (common anode) |
| 7 | Supply | J1 (+5 V), C1 220 µF, C4–C38 100 nF (one per IC) |
| 8 | Reset (block 12) | SW1, C100 1 µF, R100 10 k |

The Proteus-to-KiCad reference table is in `xref.json` (for example Proteus U5 → KiCad U7, U13 AND → U24A, U1_UP_DOWN_DIRECTION → U22A).

## 2. Deviations from the Proteus netlist (each one justified)
1. **Jam lines that were wired-AND, now an explicit AND (critical).** The four 74LS157 outputs U9/U15/U40/U52 all drove the same four jam inputs of the tens counter, all enabled, and so did U10/U53 for the hundreds counter. Proteus resolves that contention. Real totem-pole outputs fight each other, the logic level becomes undefined, and the ICs can overheat. Each unselected mux outputs its A inputs, which are tied high, so the intended function is an AND of the mux outputs. It is now implemented explicitly with 2× 74HC21 (4-input AND, tens) and 4 gates of CD4081 (hundreds). The logic is the same.
2. **Floating inputs tied.** Proteus treats an open TTL input as HIGH. On the board:
   - Mux A inputs (U9/U10/U15/U40/U52/U53, pins 2, 5, 11, 14) and U27's NOT input are tied to +5 V, matching the simulation.
   - U25 CLK1, U11 pin 4 (A>B cascade input) and the unused mux sections are tied to GND.
   - Spare gates have their inputs on GND.
   - 74LS47 BI/RBO is left open (it has an internal pull-up).
3. **Reset input added (block 12).** Net #00196 (OR inputs into both counters' PE and the mux select) had no driver; it was a logic toggle in Proteus. It is now RESET: push-button to +5 V, a 10 k pull-down, and a 1 µF power-on capacitor (about 10 ms). The counter now also starts from a known 000 at power-up.
4. **Isolated U19/U20 removed** (a CD4029 plus CD4063). Their outputs only fed each other, and CLK and PE were floating, so they had no effect on the rest of the circuit.
5. **Logic family:** 74LS157/LS04/XOR were changed to **74HC**. LS-TTL high-level output (min 2.7 V) is below the CD4000 input threshold at 5 V (V_IH 3.5 V), and LS outputs drive the CD4029/CD4081 inputs here. HC is pin-compatible. The 74LS47 is kept, because it needs open-collector outputs for common-anode displays.
6. **Gates packed:** 15+4 AND → 5× CD4081, 7 OR → 2× CD4071, 9 NOT → 2× 74HC04, 1 XOR → 74HC86, and the two single-FF 74HC74s were merged into U22 A/B. Spare gates are tied off.
7. **Displays:** the report BOM says common cathode, but a 74LS47 (active-low, open collector) requires **common anode**, so SA39-11EWA is used. The report's "74LS42" is a typo; the netlist uses the 74LS47.
8. **Symbols:** CD4063 uses the KiCad 74HC85 symbol (identical pinout), 74HC157 uses the 74LS157 symbol, and 74HC21 uses the 74LS21 symbol. In each case the value and MPN fields show the real part.
9. **Front end (from rev B):** photodiodes in zero-bias mode, so the TIA output is positive and works with the single 5 V LM358. PHASE_A/PHASE_B from the Schmitt outputs feed the logic directly; J4 brings them out for test or external drive.

## 3. PCB
- 200 × 160 mm, 2 layers, 1.6 mm, 4× M3 holes. All parts through-hole.
- Placement: optics/analog top-left, displays with their segment resistors and decoders top-right, logic in a 13 × 3 grid ordered by connectivity. Every IC has a 100 nF cap at its VCC pin.
- Tracks 0.25 mm at 0.2 mm clearance (JLC standard). Total +5 V current is about 250 mA, mostly the display segments, which 0.25 mm on 1 oz copper carries comfortably. GND pour on both layers.
- Routing: done by a script (grid maze router, F.Cu mostly horizontal and B.Cu mostly vertical). Results are in §4.

## 4. Verification (final)
- **ERC: 0 violations.**
- **DRC (zones refilled, schematic parity on): 0 errors, 0 unconnected, parity OK.** 36 warnings, none blocking: 33 silkscreen reference overlaps, 2 hole-to-hole (a via near a DIP pin), and 1 short dangling stub.
- **Routing:** about 3600 track segments and about 620 vias, including about 270 GND stitching vias on a 10 mm grid. All tracks are 0.25 mm at ≥0.2 mm clearance.
- **Routing problems found and fixed:**
  - Square pin-1 and terminal pads were first modelled as round; now handled.
  - Grid sampling missed the closest approach to pads; a 0.03 mm margin was added.
  - Both layers boxed in the U9 GND pins; the area was ripped up and rerouted with GND first.
  - Each leftover stub was removed with a DRC check after the removal.
- **Exports:**
  - `fab/`: Gerbers (copper, mask, silk, edge), Excellon drill, job file, STEP. There is no CPL file because every part is through-hole.
  - `docs/`: `Schematic.pdf`, plus `PCB_top.svg` and `PCB_bottom.svg` cropped to the board.

## 4a. Fabrication readiness
**Electrically complete and DRC-clean, so it can be ordered as a prototype.** It is not production-ready until the items below are confirmed on hardware, mainly optics geometry and TIA gain. The router produced it automatically, so many tracks are staircase segments. A manual tidy in KiCad (silkscreen, a few long detours) is recommended before ordering.

## 5. Open items
1. **Optics geometry is unverified:** no disk, slot pitch, 90°-electrical photodiode spacing or housing dimensions are given, so the optics are off-board on J2/J3.
2. **TIA gain:** 55 µA × 200 kΩ saturates the 5 V LM358. Measure the photocurrent; about 60 kΩ gives a linear swing.
3. **Logic sense:** more light gives a higher TIA output and a LOW comparator output. Up/down direction depends on photodiode placement; the report says it was set experimentally. Swap PHASE_A/PHASE_B at J3 wiring if it counts backwards.
4. The netlist reproduces the Proteus simulation. The report's own text says some glitches remained in simulation ("000 → 395", "steps of 10"). Those are logic-design issues, not layout issues; check them at bring-up.
5. Silkscreen overlaps and staircase routing should be tidied in KiCad before ordering.
