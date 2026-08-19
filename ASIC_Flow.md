# ASIC Design Flow — Architecture to Post-Silicon Validation

A stage-by-stage reference: what happens, what commonly goes wrong, and what files
move in and out of each stage. Compiled from general industry practice; cross-references
to *Advanced ASIC Chip Synthesis* (Bhatnagar) note where that book's chapters fit in.

---

## Front-end (logical domain)

### 1. Architecture & Specification
**What happens:** Architects define microarchitecture — pipeline depth, cache hierarchy,
bus protocols, interfaces — and set power/performance/area (PPA) targets. Output is a
specification, not code.

**Common problems**
- Unrealistic PPA targets set before enough analysis is done
- Ambiguous corner-case behavior left undefined, causing disputes during RTL coding
- Spec churn mid-project cascading rework into every later stage

**Files:** architecture spec, microarchitecture spec, IP/interface spec (Word/PDF),
register map (Excel or IP-XACT)

---

### 2. RTL Design
**What happens:** Engineers write Verilog / VHDL / SystemVerilog implementing the spec.
*(Book reference: Chapter 5, "Partitioning and Coding Styles" — the coding rules exist
so this stage produces synthesizable, well-behaved logic.)*

**Common problems**
- Incomplete sensitivity lists → simulation-vs-synthesis mismatch
- Unintended latch inference
- Blocking vs. non-blocking assignment misuse in Verilog
- Poor partitioning creating huge, hard-to-optimize blocks

**Files:** `.v` / `.sv` / `.vhd` source, design spec cross-reference docs

---

### 3. RTL Verification
**What happens:** Testbenches (often UVM) drive the RTL and check functional correctness
against spec, independent of synthesis. Lint and clock-domain-crossing (CDC) checks
happen here too.

**Common problems**
- Coverage gaps — untested corner cases escape to silicon
- Testbench bugs masking real design bugs
- CDC issues not caught until post-silicon
- Simulation runtime too slow for full regression before deadlines

**Files:** testbench source, waveform dumps (`.vcd`, `.fsdb`, `.shm`), coverage
databases (`.ucdb`), lint/CDC reports

---

### 4. Synthesis & DFT Insertion
**What happens:** RTL + timing library + SDC constraints → Design Compiler → gate-level
netlist. Scan chains inserted for post-fab testability.
*(Book reference: Chapters 3, 4, 6, 7 — synthesis environment, technology library,
constraining designs, optimization. Chapter 8, "Design for Test," covers scan insertion
in the full book, though not present in the uploaded excerpt.)*

**Common problems**
- Over-constraining causes area bloat (Ch. 7 §7.1's exact topic)
- Pre-layout wireload estimates are wrong, forcing later re-synthesis
- Scan chain ordering creates routing congestion later

**Files:** gate-level netlist (`.v`), `.lib`/`.db` technology library, `.sdc`
constraints, ATPG test patterns (`.stil`), area/timing/power reports

---

## Back-end (physical + manufacturing domain)

### 5. Physical Design
**What happens:** Netlist becomes geometry — floorplanning (chip area, macro placement,
I/O), power planning (grid, IR-drop-aware straps), placement (standard cells positioned),
clock tree synthesis (balanced clock distribution), routing (global then detailed).
*(Book reference: Chapter 9, "Links to Layout & Post Layout Opt." — title matches this
stage; body text not present in the uploaded excerpt.)*

**Common problems**
- Routing congestion in dense areas
- IR drop causing voltage droop under switching load
- Clock skew after CTS causing new hold violations
- Crosstalk/signal-integrity noise between adjacent routed nets

**Files:** `.lef` (cell/tech abstracts), `.def` (placed/routed design), updated `.sdc`,
technology files, floorplan files

---

### 6. Physical Verification & Signoff
**What happens:** DRC (do the shapes obey foundry manufacturing rules?), LVS (does the
layout implement the intended netlist?), antenna checks (does a long metal wire
accumulate damaging charge during plasma etch?), and post-layout STA signoff using real
extracted parasitics instead of pre-layout estimates.
*(Book reference: Chapters 11–13 — SDF generation, PrimeTime basics, static timing
analysis — this is exactly where PrimeTime earns its keep.)*

**Common problems**
- DRC violations from tool/rule-deck gaps needing manual fixes
- LVS mismatches from missing well-taps or antenna diodes
- Post-layout STA reveals violations invisible pre-layout, forcing ECO loops

**Files:** extracted parasitics (`.spef` / `.dspf` / `.rspf`), DRC/LVS rule decks,
signoff STA reports

---

### 7. Tapeout
**What happens:** Final verified layout converts to GDSII (or OASIS) — the geometric
database the foundry uses to make photomasks. "Tapeout" is the point this file is
frozen and sent out.

**Common problems**
- Last-minute ECOs discovered right before the deadline
- IP/macro integration mismatches (a macro's LEF doesn't match its actual GDS)
- Mask cost is enormous, so errors here are very expensive

**Files:** GDSII/OASIS stream file, mask data prep files, tapeout signoff checklist

---

### 8. Fabrication
**What happens:** The foundry generates photomasks from the GDSII, then runs silicon
wafers through hundreds of process steps — photolithography, etching, ion
implantation/doping, metal deposition layer by layer.

**Common problems**
- Process variation across the wafer (why STA is done across PVT — process/voltage/
  temperature — corners, not a single number)
- Defect density affecting yield
- Mask misalignment

**Files:** largely foundry-internal/proprietary — output is wafers, not files

---

### 9. Test & Post-Silicon Validation
**What happens:** Wafer sort/probe test checks each die before dicing, using the ATPG
patterns from stage 4. Good dice are packaged, then run through final test on automated
test equipment (ATE). Separately, post-silicon validation engineers characterize real
chips against simulation predictions and debug surprises.

**Common problems**
- Yield loss from process defects
- Test escapes (a bad die passes test but fails in the field)
- Silicon running at a different speed than STA predicted (correlation gap)
- Power/thermal issues only visible under real workloads

**Files:** ATPG patterns (`.stil`/`.wgl`), ATE test programs, bin/yield reports, silicon
characterization/debug logs

---

## Quick-reference: file formats by purpose

| Format | Purpose | Stage |
|---|---|---|
| `.v` / `.sv` / `.vhd` | RTL / netlist source | RTL, synthesis |
| `.lib` / `.db` | Timing/power library | Synthesis, STA |
| `.sdc` | Timing constraints | Synthesis onward |
| `.stil` / `.wgl` | Test patterns | DFT, ATE |
| `.lef` | Cell/tech abstracts | Physical design |
| `.def` | Placed/routed design | Physical design |
| `.spef`/`.dspf`/`.rspf` | Extracted parasitics | Post-layout STA |
| `.gds` / OASIS | Final mask geometry | Tapeout, fabrication |
| `.vcd`/`.fsdb`/`.shm` | Simulation waveforms | RTL verification |
| `.ucdb` | Coverage database | RTL verification |

---

*Note: this document covers the full industry-standard ASIC flow using general
knowledge. The uploaded book, "Advanced ASIC Chip Synthesis" (Bhatnagar), covers stages
4 and 6 in the most depth (Synopsys Design Compiler, Physical Compiler, PrimeTime), with
stages 1, 2, 3, 5, 8, and 9 outside its scope or only listed in its table of contents
without body text present in the uploaded file.*
