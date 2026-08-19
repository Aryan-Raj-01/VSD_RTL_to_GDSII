# ASIC Design Flow — Architecture to Post-Silicon Validation
### A Complete, Stage-by-Stage Reference (with Detailed Physical Design)

This document walks through the full ASIC design flow: what happens at each stage,
what commonly goes wrong, which files move in and out, and the interview questions
typically asked around each stage. Physical Design (PD) is broken into its full set
of sub-stages — floorplanning, power planning, placement, CTS, routing, and ECO —
since that's where most of the day-to-day PD/STA engineering work actually happens.

---

## PART A — FRONT-END (Logical Domain)

### 1. Architecture & Specification
**What happens:** Architects define the microarchitecture — pipeline depth, cache
hierarchy, bus/interconnect protocols, interfaces, memory map — and set
Power-Performance-Area (PPA) targets and the verification/timing budget for the whole
chip. This stage produces documents and models, not RTL.

**Common problems**
- PPA targets set before enough floorplan/technology analysis exists, so they turn out
  physically unrealistic later
- Ambiguous corner-case behavior left undefined in the spec, causing disputes during
  RTL coding and re-spins of verification plans
- Spec churn mid-project cascades rework into RTL, verification, and timing budgets
- Interface mismatches between IP blocks designed by different teams/vendors

**Files used/produced:** architecture specification, microarchitecture specification,
IP/interface specification (Word/PDF), register map (Excel or IP-XACT), high-level
power/performance models (spreadsheets, C/SystemC models), verification plan draft

---

### 2. RTL Design (Logic Design)
**What happens:** Engineers implement the specification in Verilog, VHDL, or
SystemVerilog — writing synthesizable RTL for datapaths, control logic, and
interconnect, following coding guidelines that keep the design synthesis-friendly.

**Common problems**
- Incomplete sensitivity lists causing simulation-vs-synthesis mismatch
- Unintended latch inference from incomplete `if`/`case` branches
- Blocking (`=`) vs. non-blocking (`<=`) assignment misuse in sequential logic
- Poor hierarchy/partitioning creating oversized blocks that are hard to synthesize,
  place, or debug
- Combinational feedback loops accidentally created

**Files used/produced:** `.v` / `.sv` / `.vhd` RTL source, design-spec cross-reference
documents, lint waivers, coding-guideline checklists

---

### 3. RTL / Functional Verification
**What happens:** Testbenches (commonly UVM-based) drive the RTL and check functional
correctness against the specification, independent of synthesis. Lint and
clock-domain-crossing (CDC) checks also happen here, along with formal property
checking for critical logic.

**Common problems**
- Coverage gaps — untested corner cases escape all the way to silicon
- Testbench bugs masking real design bugs (false pass)
- CDC issues not caught until post-silicon bring-up
- Simulation runtime too slow to run full regression before schedule deadlines
- Formal proofs that don't converge on large state spaces

**Files used/produced:** testbench source (UVM environments, sequences), waveform
dumps (`.vcd`, `.fsdb`, `.shm`), coverage databases (`.ucdb`), lint reports, CDC
reports, formal verification reports

---

### 4. Logic Synthesis
**What happens:** RTL + technology timing library + SDC timing constraints are fed
into a synthesis tool, which maps the RTL to a technology-specific gate-level netlist,
optimizing for the target timing, area, and power. Design-for-Test (DFT) scan chains
are typically inserted at this stage so the design becomes testable after fabrication.

**Common problems**
- Over-constraining causes unnecessary area/power bloat ("vertical logic")
- Under-constraining leaves real timing paths unmet ("horizontal logic")
- Pre-layout wireload/interconnect estimates are inaccurate, forcing later
  re-synthesis once real parasitics are known
- Scan-chain ordering done without physical awareness creates routing congestion
  downstream
- Multiple instantiations of the same RTL module needing independent optimization
  (resolved via "uniquify")

**Files used/produced:** gate-level netlist (`.v`), `.lib`/`.db` timing-power library,
`.sdc` timing constraints, scan-insertion reports, ATPG-ready netlist, area/timing/
power reports

**Core synthesis concepts worth knowing**
- **WNS (Worst Negative Slack):** the slack of the single worst-violating timing path.
- **TNS (Total Negative Slack):** the sum of slack across every violating path.
  Optimizing for TNS (not just WNS) generally produces more balanced, higher-yield
  timing closure.
- **Compile strategies:** top-down hierarchical (whole design at once, slower, sees
  cross-boundary opportunities), time-budgeted (sub-blocks compiled independently to
  allotted timing budgets, faster for large SoCs), and iterative
  compile-characterize-recompile loops.
- **Flattening vs. structuring:** flattening removes hierarchy for better cross-block
  optimization (slower runtime, harder debug); structuring preserves hierarchy for
  faster, more predictable, sometimes lower-QoR results.

---

## PART B — BACK-END: PHYSICAL DESIGN (Full Detail)

Physical design converts the gate-level netlist into manufacturable geometry. It is
usually the longest and most iteration-heavy part of the flow, so it's broken out
sub-stage by sub-stage below.

### 5.1 Floorplanning
**What happens:** Defines the chip/core die area and aspect ratio, places large macros
(memories, analog/mixed-signal blocks, hard IP), assigns I/O pad/pin locations,
partitions the design into physical/power domains, and reserves blockages for future
routing and macro keep-out areas. For low-power designs, power-domain floorplanning
(always-on islands, level shifters, isolation cells per UPF/CPF) is decided here too.

**Common problems**
- Poor macro placement locks in routing congestion and long timing paths that can't be
  fixed later without a re-floorplan
- Aspect ratio or utilization chosen too aggressively for the design's connectivity
- I/O placement conflicts with package/bump assignment (for flip-chip/BGA parts)
- Insufficient blockages around macros causing routing/DRC problems in later stages

**Files used/produced:** floorplan database (`.def`/tool-native), macro/IP `.lef`
abstracts, I/O placement file, power-domain/UPF or CPF files, blockage definitions

---

### 5.2 Power Planning
**What happens:** Builds the power delivery network — power rings around the core and
macros, power straps across the die, and standard-cell power rails — sized to keep
IR drop and electromigration (EM) within spec under worst-case switching activity.
Decoupling capacitors are inserted to dampen dynamic voltage droop.

**Common problems**
- IR drop (static and dynamic) causing voltage droop that slows down cells and creates
  new timing violations that didn't exist at the netlist level
- Electromigration violations on power straps/vias carrying more current than their
  metal width supports over the chip's lifetime
- Insufficient decap placement leading to localized voltage collapse during high
  switching-activity bursts
- Power grid too dense, eating into routing resources for signal nets

**Files used/produced:** power grid/strap definitions, IR-drop analysis reports, EM
analysis reports, decap placement data, updated `.def`

---

### 5.3 Placement
**What happens:** Standard cells are placed within the floorplan in three passes —
global placement (coarse, wirelength/timing-driven), legalization (snaps cells onto
the manufacturing grid/rows without overlap), and detailed placement (local
optimization for congestion and timing). Placement-stage optimization also includes
buffer insertion and cell sizing/re-mapping to fix early timing violations.

**Common problems**
- Congestion hotspots from uneven cell density, especially near macros and blockages
- Timing degradation from unrealistic pre-placement (pre-CTS) clock latency
  assumptions
- Cell legalization pushing cells far from their optimal global-placement position,
  hurting timing/congestion that looked fine before legalization
- Excessive buffer insertion bloating area and power

**Files used/produced:** placed `.def`, congestion maps, pre-CTS timing reports,
placement QoR (quality of results) reports

---

### 5.4 Clock Tree Synthesis (CTS)
**What happens:** Builds a balanced (or intentionally "useful-skewed") distribution
network from each clock source to every sequential element, using buffers/inverters
sized to control insertion delay, transition time, and skew. Clock gating cells and
clock-domain boundaries are handled carefully to avoid glitches.

**Common problems**
- Clock skew after CTS introduces new hold violations that didn't exist pre-CTS
  (because pre-CTS timing assumed ideal/zero clock latency)
- Excessive clock-tree power from over-buffering to hit an aggressive skew target
- Useful-skew optimization (intentionally unbalancing skew to help a critical path)
  applied incorrectly, creating hold problems elsewhere
- Clock routing congestion in dense areas competing with signal routing

**Files used/produced:** clock tree netlist/`.def` update, skew/latency reports,
post-CTS timing reports, clock gating verification reports

---

### 5.5 Routing
**What happens:** Connects all placed cells and macros with metal wires across the
available layers — global routing (coarse resource allocation), track assignment,
then detailed routing (exact wire/via geometry, DRC-clean). Metal fill is added
afterward across otherwise-empty areas to keep metal density uniform for chemical-
mechanical polishing (CMP) during fabrication.

**Common problems**
- Routing congestion in dense regions, sometimes unresolvable without going back to
  placement or floorplanning
- Crosstalk / signal-integrity noise between adjacent parallel-routed nets, especially
  on long, tightly-spaced wires
- DRC violations from the router being unable to legally close every net (open nets,
  via violations)
- Antenna violations — a long metal wire segment left unconnected to its final gate
  during fabrication accumulates plasma-etch charge and can damage the gate oxide
- Timing degradation from real (post-route) parasitics differing from earlier estimates

**Files used/produced:** fully routed `.def`, extracted parasitics (`.spef`/`.dspf`/
`.rspf`), DRC reports, antenna-violation reports, metal-fill data

---

### 5.6 Post-Route Optimization & ECO
**What happens:** After routing, remaining timing, DRC, or IR-drop violations are
fixed with Engineering Change Orders (ECOs) — small, surgical netlist edits (gate
resizing, buffer insertion/removal, spare-cell reuse) that avoid a full re-run of
placement and routing wherever possible.

**Common problems**
- Late-stage ECOs that ripple into new violations elsewhere ("whack-a-mole" fixing)
- Limited routing resources left for ECO-inserted buffers in already-congested areas
- Functional ECOs (fixing a logic bug this late) being far more expensive and riskier
  than timing/DRC ECOs
- Re-verification (STA, DRC, LVS, formal equivalence) needed after every ECO round

**Files used/produced:** ECO netlist patches, updated `.def`, incremental timing/DRC
reports

---

## PART C — SIGNOFF, TAPEOUT, AND MANUFACTURING

### 6. Physical Verification & Signoff
**What happens:** A battery of independent checks confirms the layout is both
manufacturable and functionally faithful to the netlist before it can be released:
- **DRC (Design Rule Check):** do the drawn shapes obey the foundry's manufacturing
  geometry rules (minimum width, spacing, enclosure, etc.)?
- **LVS (Layout vs. Schematic):** does the extracted layout implement exactly the
  intended netlist, device-for-device and connection-for-connection?
- **LEC (Logical/Formal Equivalence Check):** does the post-layout netlist remain
  logically equivalent to the original RTL/gate-level netlist after all ECOs?
- **Antenna checks:** flagged wires from the routing stage are verified/fixed with
  antenna diodes or jumpers.
- **ERC (Electrical Rule Check):** floating nets, drive-strength conflicts,
  well/substrate connectivity issues.
- **Signoff STA:** static timing analysis re-run using real extracted parasitics
  (not pre-layout estimates) across multiple corners and modes — Multi-Corner
  Multi-Mode (MCMM) analysis covering best-case/worst-case process, voltage, and
  temperature (PVT), plus on-chip variation (OCV/AOCV) margins.
- **IR drop and EM signoff:** final confirmation the power grid holds up under
  real switching activity.

**Common problems**
- DRC violations from tool/rule-deck mismatches needing manual fixing near die edges
  or macro boundaries
- LVS mismatches from missing well-taps, antenna diodes, or unconnected pins
- Post-layout STA reveals violations invisible pre-layout (real coupling capacitance,
  real clock latency), forcing further ECO loops
- Reconciling all of DRC, LVS, STA, IR-drop, and EM signoff simultaneously, since
  fixing one can reopen another

**Files used/produced:** DRC/LVS/ERC rule decks and reports, extracted parasitics,
signoff STA/MCMM reports, IR-drop/EM signoff reports, formal equivalence reports

---

### 7. GDSII Generation & Tapeout
**What happens:** Once every signoff check passes, the final verified layout is
streamed out to GDSII (or the newer OASIS format) — the geometric database format the
foundry's mask shop uses. Before mask generation, Design-for-Manufacturing (DFM)
checks run: metal density/CMP modeling, litho-hotspot detection, and Resolution
Enhancement Techniques (RET) such as Optical Proximity Correction (OPC), which
pre-distorts shapes on the mask to compensate for how light diffracts during
photolithography. "Tapeout" is the moment this database is frozen and released to
the foundry.

**Common problems**
- Last-minute ECOs discovered right before the deadline, forcing a rushed re-verify
- IP/macro integration mismatches — a macro's `.lef` abstract doesn't match its actual
  GDS geometry
- DFM hotspots flagged too late to fix without another routing pass
- Mask cost is enormous (multiple mask sets per design, especially at advanced nodes),
  so any error caught here is very expensive in time and money

**Files used/produced:** GDSII/OASIS stream file, mask data prep (MDP) output, OPC/RET
data, DFM sign-off reports, tapeout checklist and release documentation

---

### 8. Fabrication (Wafer Fab)
**What happens:** The foundry generates photomasks from the GDSII/OASIS data, then
runs raw silicon wafers through hundreds of sequential process steps: photolithography
(patterning each layer using the masks), etching (removing material per the pattern),
ion implantation/doping (setting transistor electrical properties), thin-film
deposition (metal/dielectric layers), and CMP (planarizing each layer before the next
one is built on top). This repeats layer by layer to build up the full 3D transistor
and interconnect stack.

**Common problems**
- Process variation across the wafer and from lot to lot — the reason STA is run
  across PVT (process/voltage/temperature) corners rather than a single nominal number
- Defect density affecting overall die yield
- Mask misalignment between layers
- Litho/etch variation causing systematic timing or leakage shifts vs. the design's
  models

**Files used/produced:** largely foundry-internal/proprietary process data; the
tangible output of this stage is patterned silicon wafers, not design files

---

### 9. Packaging, Test & Post-Silicon Validation
**What happens:** Before dicing, each die on the wafer is probed ("wafer sort") using
Automatic Test Pattern Generation (ATPG) patterns generated back at the synthesis/DFT
stage, run through scan chains via automated test equipment (ATE), to screen out bad
dice early. Good dice are diced and packaged (wire-bond or flip-chip depending on the
package type), then run through final test on the ATE again, sometimes after a
burn-in period to screen for early-life failures. In parallel, post-silicon validation
engineers characterize real chips — measuring actual max frequency, power, and
functional behavior against pre-silicon simulation/STA predictions — and debug any
surprises (a step often called silicon bring-up or debug).

**Common problems**
- Yield loss from process defects or systematic design-for-manufacturing issues
- Test escapes — a defective die passes test but fails later in the field
- Silicon running at a different frequency than STA predicted (the "correlation gap"
  between models and real silicon)
- Power/thermal issues that only appear under real application workloads, not
  synthetic test patterns
- Package-induced signal integrity or thermal problems not visible at the die level
  alone

**Files used/produced:** ATPG test patterns (`.stil`/`.wgl`), ATE test programs,
bin/yield reports, burn-in logs, silicon characterization and debug logs, correlation
reports comparing silicon to pre-silicon STA/power models

---

## Quick-Reference: File Formats by Purpose

| Format | Purpose | Stage |
|---|---|---|
| `.v` / `.sv` / `.vhd` | RTL source / gate-level netlist | RTL, Synthesis |
| `.lib` / `.db` | Timing/power characterization library | Synthesis, STA |
| `.sdc` | Timing constraints | Synthesis onward |
| `.ucdb` | Functional coverage database | RTL verification |
| `.vcd` / `.fsdb` / `.shm` | Simulation waveforms | RTL verification |
| `.lef` | Cell/macro/technology abstracts | Physical design |
| `.def` | Placed/routed physical design | Floorplan → Route |
| UPF / CPF | Power-intent / low-power domains | Floorplan, synthesis |
| `.spef` / `.dspf` / `.rspf` | Extracted parasitics | Post-route, signoff STA |
| `.stil` / `.wgl` | ATPG test patterns | DFT, wafer sort, ATE |
| GDSII / OASIS | Final mask geometry | Tapeout, fabrication |

---

## Core Formulas for Manual/Sanity-Check Analysis

These are standard back-of-envelope formulas PD/STA engineers keep handy to sanity
check tool output — not a substitute for signoff-tool results, which use full
non-linear delay models (NLDM/CCS) and real extracted parasitics.

**1. Setup check** (data must arrive before the next clock edge captures it)
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

**6. Wire delay approximation (Elmore delay)** — rough pre-layout sanity check before
real parasitics exist
```
T_D ≈ 0.69 × R_wire × C_wire        (single lumped RC segment)
T_D ≈ Σ R_i × C_downstream,i        (distributed RC along a routed net)
```

**7. Dynamic and static power**
```
P_dynamic = α × C_load × V² × f        (α = switching activity factor)
P_static  = I_leakage × V
```

**8. Placement/floorplan utilization sanity check**
```
Utilization % = (Total standard-cell area) / (Core area) × 100
```
Typically kept around 70–85%; pushing toward ~90%+ risks severe routing congestion.

**9. Rent's Rule** (very rough early pin/wirelength estimate before placement exists)
```
T = k × N^p
```
T = number of terminals/pins, N = number of gates in the block, k and p are empirical
constants for a given design style.

---

## Interview Questions by Stage

### RTL Design & Synthesis
- Difference between blocking and non-blocking assignments — why does mixing them
  cause simulation-vs-synthesis mismatches?
- When does a latch get inferred unintentionally in RTL, and how do you avoid it?
- What are the typical stages of logic synthesis (read → elaborate → map → optimize →
  write)?
- What inputs does a synthesis tool need to start, and what happens if RTL doesn't
  link cleanly?
- How do you break a combinational feedback loop found during synthesis?
- Difference between `==` and `===` in Verilog, and why is one not synthesizable?
- What's the difference between WNS and TNS, and why optimize for TNS?

### Static Timing Analysis (STA) & Constraints
- Difference between setup and hold violations, and how do you fix each?
- What is clock uncertainty, and how does it differ from clock skew?
- Difference between on-chip variation (OCV) and advanced OCV (AOCV)?
- What is a false path vs. a multicycle path, and why does misclassifying one waste
  optimization effort?
- What input files does STA need, and how does STA differ from circuit simulation?
- What do positive, negative, and zero slack mean physically?
- What is Multi-Corner Multi-Mode (MCMM) analysis and why is it necessary?

### Design for Test (DFT)
- What is a scan chain, and how is it built from mux-based flip-flops?
- Difference between functional verification and DFT?
- What are the common fault models (stuck-at, transition, bridging)?
- What is ATPG, and how does it generate a test pattern for a stuck-at fault?
- Difference between BIST, boundary scan (JTAG), and scan chains?
- What input/output files are needed for scan insertion and ATPG?

### Physical Design
- What are the main stages of the PD flow (floorplan → power plan → placement → CTS →
  routing → signoff)?
- What is routing congestion, and what typically causes it?
- What is metal fill, and why is it needed for CMP?
- Difference between a placement blockage and a routing blockage?
- What's in a `.lib` file vs. a `.lef` file?
- Why does macro placement matter for routing and timing before cells are even
  placed?
- What is IR drop, and how does it differ from electromigration?
- Why does clock skew after CTS sometimes create new hold violations?
- What is useful skew, and when would you intentionally use it?
- What is an antenna violation, and how is it fixed?
- What's the difference between DRC and LVS?
- What is an ECO, and why are late-stage functional ECOs riskier than timing ECOs?

### Fabrication & Post-Silicon
- Why is STA run across PVT corners instead of a single nominal number?
- What causes the "correlation gap" between STA predictions and real silicon
  performance?
- What is the difference between wafer sort/probe test and final test?
- What is burn-in, and what kind of failures is it meant to catch?
- What is a test escape, and why is it hard to eliminate entirely?
- What's the difference between wire-bond and flip-chip packaging, and how does that
  choice affect signal/power integrity?

---

*This document uses general, widely-known ASIC industry practice and is not tied to
any single textbook or vendor's tool documentation. Terminology (tool names, exact
report formats) can vary slightly between EDA vendors (Synopsys, Cadence, Siemens EDA),
but the underlying flow and concepts above are consistent across the industry.*
