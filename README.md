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

## 📚 4. References

Local PDF copies of the textbooks and papers are stored in [`Reference_materials/`](./Reference_materials/).

### Main Textbooks (`B1` – `B5`)

| # | Document | Used for |
| :--- | :--- | :--- |
| **B1** | N. Weste, D. Harris, *CMOS VLSI Design: A Circuits and Systems Perspective*, Pearson/Addison-Wesley | CMOS logic, delay, power, layout, design rules |
| **B2** | J. Rabaey, A. Chandrakasan, B. Nikolić, *Digital Integrated Circuits: A Design Perspective*, Pearson | Delay/power models, logic styles (pass transistor, transmission gate) |
| **B3** | S. Lin, D. J. Costello, *Error Control Coding*, Pearson | Hamming code theory, SEC-DED |
| **B4** | S.-M. Kang, Y. Leblebici, *CMOS Digital Integrated Circuits: Analysis and Design*, McGraw-Hill | CMOS circuit analysis, layout design |
| **B5** | *VLSI design of systematic odd-weight-column byte error detecting SEC-DED code* | Origins of Hamming code |

