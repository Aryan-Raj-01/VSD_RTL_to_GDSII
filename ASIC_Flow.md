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

---

## Optimization — what the book covers (Ch. 7) and how it extends

### From the book directly (Chapter 7, §7.1 "Design Space Exploration")
- Optimization = smallest area while meeting all timing requirements. Achieving it needs
  understanding *how* the synthesis engine behaves, not just running it.
- Since DC98, Design Compiler prioritizes **timing over area**, and optimizes to reduce
  **total negative slack (TNS)** rather than **worst negative slack (WNS)** — meaning it
  spends effort fixing many small violations, not just the single worst path.
- Area increases considerably as constraints tighten — tightening past a point produces
  "vertical logic" (over-built, area-heavy) with no further timing benefit; over-relaxed
  constraints produce "horizontal logic" that violates real timing needs.
- Practical rule of thumb the book gives: over-constrain by roughly **10% tighter than
  required** — enough margin to avoid repeated synthesis-layout iterations, without so
  much that DC bloats the area.

### General knowledge — the rest of Ch. 7's table of contents (body text not in the
uploaded file, but these are the standard techniques the titles refer to)
- **Total Negative Slack (TNS) vs. Worst Negative Slack (WNS):** WNS is the single worst
  path's slack; TNS sums every violating path's slack. Optimizing for TNS produces more
  balanced, higher-yield timing closure than chasing WNS alone.
- **Compilation strategies:**
  - *Top-down hierarchical compile* — compile the whole design at once, letting DC see
    cross-boundary optimization opportunities; slow on large designs.
  - *Time-budgeting compile* — allocate timing budgets to sub-blocks based on
    estimated critical-path contribution, then compile blocks independently — faster,
    common for large SoCs.
  - *Compile-characterize-write script-recompile* — an iterative loop: compile once,
    characterize actual delays, feed those back as tighter/looser constraints, recompile.
  - *Design budgeting* — similar to time-budgeting but extended to area/power budgets
    per block.
- **Resolving multiple instances (uniquify):** shared RTL modules instantiated multiple
  times get separate, uniquely-named netlist copies so each instance can be optimized
  for its own local timing/area context instead of one compromise solution.
- **Optimization techniques:**
  - *Flattening* — removes hierarchy boundaries so DC can optimize logic across module
    boundaries (better QoR, much slower runtime, harder to debug).
  - *Structuring* — the opposite: preserves/creates hierarchy so DC works on
    smaller sub-problems (faster, more predictable, sometimes worse QoR).
  - *Removing hierarchy* selectively at compile time is a middle ground.
  - *Optimizing clock networks* — minimizing insertion delay/skew via balanced buffering
    even before real CTS happens in physical design.
  - *Optimizing for area* — cleanup passes after timing closure, since DC prioritizes
    timing during main compile.

---

## Formulas for manual approximation (physical design / STA sanity checks)

Your book explains delay through the **non-linear delay model (NLDM)** — a
characterized lookup table indexed by input transition and output load capacitance,
interpolated by DC (Ch. 4, §4.3.1) — not a closed-form equation. That's accurate for
tool use, but PD/STA engineers still keep a few back-of-envelope formulas for sanity
checks. These are standard industry formulas, not from the book itself.

**1. Setup check** (data must arrive before the next clock edge)
```
Data Required Time  = T_period + T_skew(setup) − T_setup(capture FF)
Data Arrival Time    = T_clk-to-Q(launch FF) + T_comb(max path)
Setup Slack          = Data Required Time − Data Arrival Time      (must be ≥ 0)
```

**2. Hold check** (data must not change too soon after the clock edge)
```
Data Required Time  = T_hold(capture FF) + T_skew(hold)
Data Arrival Time    = T_clk-to-Q(min, launch FF) + T_comb(min path)
Hold Slack           = Data Arrival Time − Data Required Time      (must be ≥ 0)
```
Note the book's own `set_clock_uncertainty –setup / –hold` commands (Ch. 6, §6.3) are
exactly how these skew/uncertainty terms get fed into DC's version of this check.

**3. Maximum operating frequency (rough estimate)**
```
F_max ≈ 1 / (T_clk-to-Q + T_comb(max) + T_setup + margin)
```

**4. Clock skew effect (quick sign-check)**
- Positive skew (capture clock arrives later than launch) → relaxes setup, tightens hold
- Negative skew (capture arrives earlier) → tightens setup, relaxes hold

**5. Total Negative Slack / Worst Negative Slack**
```
WNS = min(slack) across all paths
TNS = Σ slack_i  for every path where slack_i < 0
```

**6. Wire delay approximation (Elmore delay)** — useful for a rough pre-layout sanity
check before real parasitics exist:
```
T_D ≈ 0.69 × R_wire × C_wire        (single lumped RC segment)
T_D ≈ Σ R_i × C_downstream,i        (distributed RC along a routed net)
```

**7. Dynamic power (why tight constraints and clock gating matter)**
```
P_dynamic = α × C_load × V² × f        (α = switching activity factor)
P_static  = I_leakage × V
```

**8. Placement/floorplan utilization sanity check**
```
Utilization % = (Total standard-cell area) / (Core area) × 100
```
Typically kept around 70–85%; pushing toward ~90%+ risks the routing congestion
several of the interview sources below explicitly warn about.

**9. Rent's Rule (very rough early pin/wirelength estimate)**
```
T = k × N^p
```
T = number of terminals/pins, N = number of gates in the block, k and p are empirical
constants for a given design style — used for early floorplan/I-O estimation before
placement exists.

---

## Interview questions by stage (sourced from public VLSI interview-prep sites)

These are commonly-asked questions compiled from current interview-prep resources —
useful as a checklist, but verify answers against your own understanding rather than
memorizing them verbatim.

### RTL design / synthesis (Ch. 3–7 territory)
- What's the difference between blocking and non-blocking assignments, and why does
  mixing them cause simulation-synthesis mismatches?
- When does a latch get inferred unintentionally in RTL, and how do you avoid it?
- What are the five stages of RTL synthesis (read, elaborate, map, optimize, write)?
- What inputs does Design Compiler need to start synthesis, and what happens if the RTL
  doesn't link cleanly?
- How would you break a combinational feedback loop found during synthesis?
- What's the difference between `==` and `===` in Verilog, and why is one not
  synthesizable?
- *(Sources: vlsiweb.com/rtl-synthesis-interview-questions, chipxpert.in/rtl-design-interview-questions,
  tech.tdzire.com/synthesis-interview-questions, vlsifacts.com)*

### Constraining designs / STA (Ch. 6, 11–13 territory)
- What's the difference between setup and hold violations, and how do you fix each?
- What is clock uncertainty, and how does it differ from clock skew?
- What's the difference between on-chip variation (OCV) and advanced OCV (AOCV)?
- What is a false path vs. a multicycle path, and why does misclassifying one cause
  wasted optimization effort?
- What input files does STA need to run, and how is STA different from circuit
  simulation?
- What are positive, negative, and zero slack, and what does each mean physically?
- *(Sources: vlsiweb.com/sta-interview-questions, siliconvlsi.com/sta-interview-questions,
  vlsiuniverse.blogspot.com, velansta.blogspot.com)*

### Design for Test / DFT (Ch. 8 territory — not in your uploaded file)
- What is a scan chain, and how is it built from mux-based flip-flops?
- What's the difference between functional verification and DFT?
- What are the common fault models (stuck-at, transition, bridging)?
- What is ATPG, and how does it generate a test pattern for a given stuck-at fault?
- What's the difference between BIST, boundary scan (JTAG), and scan chains?
- What input files are needed for scan insertion and ATPG, and what output files result?
- *(Sources: maven-silicon.com/vlsi-jobs/dft-interview-questions, vlsiweb.com/dft-interview-questions,
  ecrionix.org/dft, vlsi4freshers.com)*

### Physical design (Ch. 9–10 territory — not in your uploaded file)
- What are the main stages of the PD flow (floorplan → placement → CTS → routing →
  signoff)?
- What is routing congestion, and what causes it?
- What is metal fill, and why is it needed for CMP (chemical-mechanical polishing)?
- What's the difference between a placement blockage and a routing blockage?
- What's in a `.lib` file vs. a `.lef` file?
- Why does macro placement matter for routing and timing before cells are even placed?
- *(Sources: hirist.tech/blog/top-25-physical-design-interview-questions,
  ivlsi.com/placement-interview-questions-and-answers, ivlsi.com/floorplan-interview-questions,
  learnelectronicsindia.com, physicaldesign4u.com)*

---

*As with the flow sections above: the RTL/synthesis and STA questions map directly onto
chapters actually present in your uploaded file. The DFT and physical-design questions
are included for completeness since those chapters are only in the book's table of
contents, not its body text here.*
