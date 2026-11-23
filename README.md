# Design and PPA Evaluation of a Hamming SEC-DED (13,8) Encoder/Decoder with Error Injection in CMOS Using Cadence Virtuoso

**VLSI Design Course Project** implementing a full-custom, transistor-level **Hamming SEC-DED (13,8)** encoder/decoder integrated with a programmable **Error Injection** block and **Mode MUX** in **90 nm CMOS** technology using **Cadence Virtuoso**. The project focuses on functional reliability verification against soft errors and comprehensive **PPA (Power, Performance/Latency, Area)** evaluation.

---

## 📌 1. Architecture Overview & Research Objects

This project designs, verifies, and evaluates three core objects at the transistor level:

1. **Technology & Design Flow:** **90 nm CMOS** technology using a full-custom transistor-level flow on **Cadence Virtuoso** (`Schematic → Simulation → Functional Verification`), with optional **Layout** and clean **DRC/LVS** for selected core cells or blocks (layout scope finalized at Checkpoint 1).
2. **VLSI Digital System:** A **Hamming SEC-DED (13,8)** encoder/decoder (8 data bits + 4 Hamming parity bits for Single Error Correction + 1 extended overall parity bit for Double Error Detection, defined by a $5 \times 13$ parity-check matrix $H$), coupled with a controllable **Error Injection** block (1, 2, or 3-bit programmable error injection) and **Mode MUX**.
3. **Shared Standard Cell Library:** Basic shared cells—`INV`, `NAND`, `NOR`, `MUX2`—and three **XOR/XNOR** circuit styles (**Static CMOS**, **Transmission Gate**, and **Pass Transistor**), since the XOR tree accounts for the majority of the EDAC circuit area, critical path depth, and cascaded voltage-degradation behavior.

### System Block Diagram

```text
                       ┌────────────────────────── BYPASS ──────────────────────────┐
                       │                                                            ▼
D[7:0] ──► [ Hamming SEC-DED (13,8) Encoder ] ──► CW[12:0]                   ┌────────────┐
                         │                              │                    │            │
                         │                              ▼                    │            │
                         │               [ Error Injector (ERR[12:0]) ]      │  Mode MUX  ├──► Y[7:0]
                         │               (Programmable 1, 2, 3-bit)          │ MODE[1:0]  │
                         │                              │ CW'[12:0]          │            │
                         │             ┌───── RAW ──────┼───────────────────►│            │
                         ▼             │                ▼                    │            │
                 (CODEC path) ─────────┴─► [ Hamming SEC-DED (13,8) Decoder ]┴────────────┘
                                           (Syndrome → Error Locator → Bit Corrector)
                                                        │
                                                        ▼
                                            Status Flags: SEC, DED
```

### Operating Modes (`MODE[1:0]`)

| Mode | Description | Purpose |
| :--- | :--- | :--- |
| **BYPASS** | Passes `D[7:0]` directly to `Y[7:0]`, bypassing encoding/decoding | Baseline reference for delay and power overhead |
| **RAW** | Outputs raw corrupted data directly from `CW'[12:0]` without correction | Visual/functional baseline to compare against `PROTECTED` |
| **CODEC** | Routes data through the Encoder–Decoder chain | Standalone verification of the Hamming SEC-DED codec |
| **PROTECTED** | Full protection flow (Encode → Error Inject → Decode & Correct) with `SEC` and `DED` flags | Primary fault-tolerant operating mode |

### Comparison Configurations

System overhead and reliability are evaluated progressively across four configurations:
* **`Baseline`:** 8-bit unprotected datapath.
* **`+Parity`:** Single-bit parity check (error detection only).
* **`+Hamming`:** Hamming $(12,8)$ Single Error Correction (**SEC**).
* **`+SEC-DED`:** Extended Hamming $(13,8)$ Single Error Correction and Double Error Detection (**SEC-DED**).

---

## 🛠️ 2. Design & Verification Methodology

1. **Circuit Design & Cell Characterization:**
   * Design `Schematic`, `Symbol`, and characterization testbenches for `INV`, `NAND`, `NOR`, `MUX2`, and three `XOR`/`XNOR` topologies (`Static CMOS`, `Transmission Gate`, `Pass Transistor`), including 4-to-6-stage cascaded XOR chains to analyze voltage degradation and balanced XOR tree optimization.
   * Build sub-blocks (`Hamming SEC-DED Encoder`, `Programmable 1, 2, 3-bit Error Injector`, `Hamming Decoder`—`Syndrome Generator`, `Error Locator Decoder`, `Bit Corrector`, `Flag Logic`—and `Mode MUX`) and integrate the top-level system.
   * *(Optional)* Complete physical **Layout** with clean **DRC/LVS** for the selected XOR cell or the Hamming block (prioritized at Checkpoint 1).
2. **Functional Verification & Evaluation:**
   * Implement a **Python Golden Model** and **Verilog Reference Model** aligned 100%, and generate the verification test vectors.
   * Execute **8 simulation scenarios** on Cadence Virtuoso covering error-free transmission across all 4 modes, 1-bit Error Injection (verifying single-bit correction and `SEC` flag), 2-bit Error Injection (verifying double-bit detection and `DED` flag), 3-bit Error Injection, and direct `RAW` vs. `PROTECTED` comparison.
   * Compare circuit outputs against the golden model across all test vectors and compute statistics for 1-bit correction rate, 2-bit detection rate, and 3-bit miss/miscorrection rate.
3. **PPA & Robustness Analysis:**
   * Measure propagation delay ($t_{pLH}$, $t_{pHL}$), critical path latency (`Encoding → Error Injection → Decoding → Error Correction`), static and dynamic/average power, and **PDP** for each cell, block, and full system.
   * Evaluate area via **transistor count** (supplemented by layout area for laid-out components).
   * Perform **PVT corner** simulations (schematic level) on the critical path, plus **Monte Carlo** statistical analysis if time permits.

---

## 📊 3. Expected Results, Evaluation Tables & Hypotheses

### Design Deliverables
* Complete standard cell library (`Schematic`, `Symbol`, simulation waveforms, and characterization table covering delay, power, PDP, and transistor count).
* **Hamming SEC-DED (13,8) + Error Injection** system operating properly across all 4 modes (`BYPASS`, `RAW`, `CODEC`, `PROTECTED`).
* Top-level schematic simulation matching the Python/Verilog golden model and test vectors **100%**.
* *(Optional)* Clean **DRC/LVS** layout for selected core cells (chosen XOR cell) or the Hamming block.

### Target PPA & Reliability Metrics
* **Power:** Average power, static power, and PDP per block and for the entire system.
* **Latency:** Critical path delay (`Encoding → Error Injection → Decoding → Error Correction`) across PVT corners (schematic level).
* **Area:** Area and transistor count, including the overhead of Hamming SEC-DED relative to the unprotected Baseline.
* **Reliability:** Correction and detection capability for 1-bit, 2-bit, and 3-bit errors.

### Evaluation Tables (Tables A – E)

| Table | Evaluation Scope | Key Metrics |
| :--- | :--- | :--- |
| **Table A** | **Protection Overhead** | Area (transistor count / layout area), delay, power, and PDP across `Unprotected Baseline`, `+Parity`, `+Hamming`, and `+SEC-DED` |
| **Table B** | **XOR/XNOR Topology Comparison** | Transistor count, delay, power, PDP, and output voltage degradation when cascading 4 to 6 stages across the 3 XOR/XNOR styles |
| **Table C** | **Error Coverage & Functional Eval** | Correction rate (1-bit), detection rate (2-bit), and miss/miscorrection rate (3-bit) vs. golden model; comparison of `RAW` vs. `PROTECTED` |
| **Table D** | **PVT Corners** | Critical path delay and power at best / typical / worst corners (schematic level) |
| **Table E** | **Monte Carlo** *(if time permits)* | Mean ($\mu$) and standard deviation ($\sigma$) of delay and power on the critical path |

### Benchmark from References
* Paper **P2** (`arXiv:2002.07507`) reports ASIC synthesis results for multiple SEC-DED codes; notably, the `(14,8)` code reports `Area: 1150.95`, `Power: 138.02`, and `Delay: 292.7` (in the paper's units). Since the team's `(13,8)` code is of comparable complexity, measured results should fall within the same order of magnitude, with any large deviations analyzed. This serves strictly as an order-of-magnitude sanity check rather than a hard target due to full-custom design on a different PDK.

### Hypotheses to Verify & Known Limitations
* **Hypothesis 1:** Hamming SEC-DED significantly increases area and power compared to the unprotected Baseline; **Table A** quantifies this overhead.
* **Hypothesis 2:** The system critical path resides in the deep XOR chain inside the decoder (`syndrome → error location decoding → correction XOR`) rather than in encoding or error injection; a balanced XOR tree reduces logic depth.
* **Hypothesis 3:** Pass-transistor XOR logic saves transistors but suffers voltage degradation across cascaded stages, whereas transmission-gate XOR maintains stronger voltage swing at the cost of larger area.
* **Hypothesis 4:** Worst-case PVT corners noticeably increase critical path delay; if Monte Carlo is run, the distribution spread provides basis for proposing a design safety margin.
* **Known Limitations:**
  1. Hamming SEC-DED may miscorrect or fail to detect errors of $\ge 3$ bits (without supplementary CRC), which will be quantified in **Table C**.
  2. Primary evaluations stop at the schematic level and do not yet capture post-layout parasitic resistance/capacitance effects.

---

## 📅 4. Action Plan (Week 4 → Week 9)

| Week | Main Tasks | In Charge | End-of-Week Deliverable |
| :--- | :--- | :--- | :--- |
| **W4** | Finalize PDK, references, simulation tools (and DRC/LVS if doing layout); get instructor approval for revised proposal; entire team redoes inverter task (schematic, sim, layout, DRC/LVS); set up Cadence library and Git repo; read and summarize P1–P10; initial Python golden model | **M1 (Khoa)** finalize, repo, golden model; **All** inverter, reading | Approved proposal; each member achieves clean DRC/LVS for an inverter; literature summaries |
| **W5 – CHECKPOINT 1** | Complete cell library: schematic, symbol, sim, characterization for `INV`, `NAND`, `NOR`, `MUX2`, and 3 XOR types (layout, DRC/LVS only for selected cell; finalize layout scope at this checkpoint); align golden model and Verilog; generate test vectors | **M2 (Hoàn)** cells, XOR compare; **M3 (Nhân), M4 (Ngân)** `INV`, `NAND`, `NOR`, `MUX2`; **M5 (Uyên)** error injection cells; **M1 (Khoa)** golden model, vectors | W5 Report: measured cell library, **Table B** (XOR comparison), `(13,8)` Hamming equations validated by golden model, layout scope decided |
| **W6** | Block schematics: Hamming encoder/decoder, error injection block, Mode MUX; simulate each block with vectors | **M3 (Nhân) & M4 (Ngân)** Hamming jointly; **M5 (Uyên)** error injection; **M1 (Khoa)** MUX, interfaces | Each block passes 100% of vectors at schematic level, obtaining delay and power data |
| **W7 – CHECKPOINT 2** | Top-level integration; run 8 verification scenarios (including 1-bit and 2-bit Error Injection, `RAW` vs. `PROTECTED`); functional evaluation per **Table C** criteria; measure critical path and power | **M1 (Khoa)** top-level, functional eval; **All** scenarios; **M3 (Nhân) & M4 (Ngân)** decoder critical path | W7 Report: top-level schematic working correctly, demo waveforms, functional evaluation results (initial **Table C**), initial **Table A** (schematic level) |
| **W8** | PVT corners on critical path (schematic level), Monte Carlo if time permits; finalize **Table C** (1, 2, 3-bit error coverage); *(optional)* layout of chosen XOR cell or Hamming block, DRC/LVS | **M2 (Hoàn)** PVT, Monte Carlo; **M1 (Khoa), M5 (Uyên)** Table C; **M3 (Nhân), M4 (Ngân)** layout (if done) | Complete **Table C**, initial **Tables D & E**; clean DRC/LVS (if layout is done) |
| **W9 – CHECKPOINT 3 (Submission)** | Finalize **Tables A through E**; area/power distribution charts; write report; create slides; record demo video; practice presentation; package submission | **M5 (Uyên)** synthesis, editing; **Each member** writes their chapters; **M1 (Khoa)** reviews | Submission: report, slides, video, design files, member contribution table |

---

## 📑 5. Expected Project Report Outline

The final project report is structured into five chapters:
1. **Chapter 1 – Project Overview and Theoretical Background:** Soft errors, EDAC motivation, 90 nm CMOS full-custom scope, Hamming SEC-DED `(13,8)` theory, and transistor-level CMOS logic gate characteristics.
2. **Chapter 2 – System Architecture and Design Methodology:** Top-level architecture, 4 operating modes, 90 nm standard cell library design, and sub-block circuit implementations.
3. **Chapter 3 – Verification Environment and Simulation Scenarios:** Python/Verilog golden models, Cadence Virtuoso testbench & test vectors, 8 verification scenarios, and PPA/PVT/Monte Carlo measurement methodologies.
4. **Chapter 4 – Simulation Results, PPA Evaluation, and Discussion:** XOR/XNOR comparison (**Table B**), functional verification & error coverage (**Table C**), configuration PPA overhead (**Table A**), critical path analysis, PVT/Monte Carlo stability (**Tables D & E**), and ASIC benchmarking (`P2`).
5. **Chapter 5 – Conclusion and Future Work:** Summary of findings, hypothesis verification, schematic-level and $\ge 3$-bit error limitations, and future layout/post-layout directions.

<details>
<summary>Expand full chapter-by-chapter outline</summary>

* **CHAPTER 1: PROJECT OVERVIEW AND THEORETICAL BACKGROUND**
  * 1.1. Problem statement and necessity (Soft Errors in digital circuits and the need for data protection using EDAC codes)
  * 1.2. Research objectives and scope (90 nm CMOS technology, full-custom transistor-level design on Cadence Virtuoso)
  * 1.3. Theoretical background of Hamming SEC-DED `(13,8)` error correction code
  * 1.4. Characteristics of transistor-level CMOS logic gate structures
* **CHAPTER 2: SYSTEM ARCHITECTURE AND DESIGN METHODOLOGY**
  * 2.1. Specification and Top-level Architecture
    * 2.1.1. Full system block diagram: `Encoder → Error Injection → Decoder → Mode MUX`
    * 2.1.2. Analysis of 4 operating modes: `BYPASS`, `RAW`, `CODEC`, `PROTECTED`
  * 2.2. Standard Cell Library Design on Cadence Virtuoso (90 nm)
  * 2.3. Detailed Design of Sub-blocks
    * 2.3.1. Encoding block (`Hamming SEC-DED Encoder`)
    * 2.3.2. Controllable error injection block (`Programmable 1, 2, 3-bit Error Injector`)
    * 2.3.3. Decoding and error correction block (`Syndrome Generator`, `Error Locator Decoder`, `Bit Corrector`, `Flag Logic`)
    * 2.3.4. Mode control block (`Mode MUX`) and comparison configurations (`Baseline`, `+Parity`, `+Hamming`, `+SEC-DED`)
* **CHAPTER 3: VERIFICATION ENVIRONMENT AND SIMULATION SCENARIOS**
  * 3.1. Building reference models (*Golden Model*) using Python and Verilog
  * 3.2. Testbench setup and test vectors on Cadence Virtuoso
  * 3.3. Description of 8 functional verification scenarios
  * 3.4. PPA measurement methods, PVT corner, and Monte Carlo analysis
* **CHAPTER 4: SIMULATION RESULTS, PPA EVALUATION, AND DISCUSSION**
  * 4.1. Comparison of transistor count, delay, power, PDP, and voltage degradation levels among the 3 `XOR`/`XNOR` types (**Table B**)
  * 4.2. Functional verification and error coverage results (**Table C**)
    * 4.2.1. Simulation waveforms of 8 scenarios and 100% matching with the Golden Model
    * 4.2.2. Statistics on correct correction rate (1-bit error), detection rate (2-bit error), and miss/miscorrection rate (3-bit error)
  * 4.3. PPA overhead evaluation among protection configurations (**Table A**)
    * 4.3.1. Comparing Area (transistor count / layout area), Latency, Power, and PDP among `Baseline`, `+Parity`, `+Hamming`, and `+SEC-DED`
    * 4.3.2. Critical path analysis of the system (`Encoder → Error Injection → Decoder`)
  * 4.4. Stability analysis across PVT corners and Monte Carlo (**Table D** & **Table E**)
  * 4.5. Benchmarking against reference works (ASIC synthesis benchmarks for SEC-DED `(14,8)` code and related CMOS studies)
* **CHAPTER 5: CONCLUSION AND FUTURE WORK**
  * 5.1. Summary of achieved results and verification of initial hypotheses
  * 5.2. Project limitations (behavior with $\ge 3$-bit errors, post-layout parasitic capacitance/resistance effects)
  * 5.3. Future development directions (completing full-chip layout/post-layout simulation, integrating RAM codes, expanding data bus width)

</details>

---

## 📂 6. Repository Structure

```text
VLSI/
├── DE_CUONG_DU_AN_VLSI_EDAC.docx   # Project proposal document
├── README.md                       # Project overview, architecture, action plan, and references
├── Reference_materials/            # Reference textbooks and papers (PDF)
│   ├── Book/                       # CMOS VLSI Design, Digital Integrated Circuits, Error Control Coding
│   └── Paper/                      # Papers on XOR/XNOR topologies, Hamming, SEC-DED, Monte Carlo
│
├── model/       # (planned) Python golden model (parity-check matrix H, vector generation, error stats)
├── verilog/     # (planned) Verilog reference model for functional cross-checking
├── virtuoso/    # (planned) Cadence Virtuoso libraries (Schematic, Symbol, Layout)
├── vectors/     # (planned) Stimulus vectors and expected golden outputs
├── scripts/     # (planned) Stimulus generators and OCEAN scripts for automated PPA/PVT measurements
├── results/     # (planned) Raw CSV measurements, Tables A–E, and simulation waveforms
└── report/      # (planned) Final project report, presentation slides, and demo video
```

---

## 👥 7. Team Members

| Member | Role | Main Contribution | Assigned Paper |
| :--- | :--- | :--- | :--- |
| **Khoa (M1)** | Architecture, integration, verification | Spec, Python/Verilog golden model, test vectors, 8 verification scenarios and functional evaluation, top-level schematic, Mode MUX, top-level error flags, PVT corners (and Monte Carlo if time permits) | P2, P3 |
| **Hoàn (M2)** | Cell library | `INV`, `NAND`, `NOR`, `MUX2`, 3 types of `XOR`/`XNOR`; cell characterization (delay, power, PDP); selecting XOR type; layout for chosen XOR cell if layout is decided | P4, P5, P6 |
| **Nhân (M3)** | Hamming SEC-DED (Decoder) | Decoding side: syndrome tree, error location decoding, bit correction circuit, `SEC`/`DED` flag logic; measuring decoder critical path | P1, P9 |
| **Ngân (M4)** | Hamming SEC-DED (Encoder & Layout) | Encoding side: parity-check matrix $H$ (13,8), encoder; integrating encoder–decoder with Nhân; Hamming block layout if selected (top priority) | P7, P8 |
| **Uyên (M5)** | Error Injection, synthesis | Error injection block (1, 2, 3 bits, error location selection); synthesizing data into comparison tables; report, slides, demo video | P10 |

---

## 📚 8. References

Local PDF copies of the textbooks and papers are stored in [`Reference_materials/`](./Reference_materials/).

### Main Textbooks (`B1` – `B5`)

| # | Document | Used for |
| :--- | :--- | :--- |
| **B1** | N. Weste, D. Harris, *CMOS VLSI Design: A Circuits and Systems Perspective*, Pearson/Addison-Wesley | CMOS logic, delay, power, layout, design rules |
| **B2** | J. Rabaey, A. Chandrakasan, B. Nikolić, *Digital Integrated Circuits: A Design Perspective*, Pearson | Delay/power models, logic styles (pass transistor, transmission gate) |
| **B3** | S. Lin, D. J. Costello, *Error Control Coding*, Pearson | Hamming code theory, SEC-DED |
| **B4** | S.-M. Kang, Y. Leblebici, *CMOS Digital Integrated Circuits: Analysis and Design*, McGraw-Hill | CMOS circuit analysis, layout design |
| **B5** | *VLSI design of systematic odd-weight-column byte error detecting SEC-DED code* | Origins of Hamming code |

### Papers (`P1` – `P10`)

| # | Member | Paper / Source | Used for |
| :--- | :--- | :--- | :--- |
| **P1** | Nhân | M. Y. Hsiao, *"A class of optimal minimum odd-weight-column SEC-DED codes"*, IBM J. Res. Develop., 1970 | SEC-DED fundamentals (Hsiao code), compared with our extended Hamming |
| **P2** | Khoa | *"Design of SEC-DED and SEC-DED-DAEC Codes of different lengths"*, `arXiv:2002.07507` | ASIC PPA summary table of various SEC-DED codes, used as a benchmark |
| **P3** | Khoa | *IBM Research Report RC23586* | Setting up Monte Carlo for delay and power |
| **P4** | Hoàn | Makara Tang, *"Comparison of CMOS XOR and XNOR Gate Design"*, Master thesis, OHSU (c. 2002) | Comparing various XOR/XNOR structures, testing cascaded XOR chains |
| **P5** | Hoàn | J.-M. Wang et al., *"New efficient designs for XOR and XNOR functions on the transistor level"*, IEEE JSSC, 1994 | Classic paper on transistor-level XOR/XNOR |
| **P6** | Hoàn | *"Revisiting the Utility of Transmission Gate and Passtransistor Logic Styles in CMOS VLSI Design"*, IEEE Xplore | Area, power, delay comparison of TG and pass transistor, including XOR gates |
| **P7** | Ngân | F. Noorbasha et al., *"VLSI Implementation of Encryption and Decryption System Using Hamming Code Algorithm"*, IJERA, 2014 | CMOS Hamming layout at 90/70/50 nm, transistor count |
| **P8** | Ngân | R. Rai et al., *"VLSI Implementation of Hamming Code for Molecular Communication"*, Springer, 2021 | CMOS Hamming encoder, power and delay analysis at 250 nm |
| **P9** | Nhân | *"15-bit Novel Hamming Codec using HSPICE 22nm CMOS Technology based on GDI Technique"*, IJES | Hamming codec using GDI, transistor count comparison |
| **P10** | Uyên | *"A Statistical Model of Logic Gates for Monte Carlo Simulation..."*, PATMOS 2007, Springer LNCS | Statistical model for Monte Carlo, supporting data synthesis |
