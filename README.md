# Design of a Low-Power Fully Differential Folded-Cascode Amplifier in 130 nm CMOS

A PMOS-input, fully differential folded-cascode amplifier designed and simulated with the open-source **SkyWater SKY130** PDK (`sky130_fd_pr__pfet_01v8_lvt` / `sky130_fd_pr__nfet_01v8`), using **xschem** for the schematic and **ngspice** for simulation.

| | |
|---|---|
| **Supply** | 1.8 V |
| **Power** | ≈ 36 µW (20.2 µA) |
| **Differential gain (Ad)** | 60.1 dB (≈ 1000 V/V) |
| **Unity-gain frequency** | ≈ 10 MHz (CL = 1 pF per output) |
| **Process / temperature** | SKY130 TT, 27 °C |

---

## Table of contents

1. [Overview](#overview)
2. [How the circuit works](#how-the-circuit-works)
3. [Device sizes and biasing](#device-sizes-and-biasing)
4. [Specifications](#specifications-simulated)
5. [Simulation results and waveforms](#simulation-results-and-waveforms)
6. [Hand analysis vs simulation](#hand-analysis-vs-simulation)
7. [How to reproduce](#how-to-reproduce)
8. [Pros of the design](#pros-of-the-design)
9. [Limitations](#limitations)
10. [Possible applications](#possible-applications)
11. [Future work](#future-work)
12. [Repository structure](#repository-structure)

---

## Overview

This project is a single-stage, fully differential **folded-cascode OTA** with a PMOS input pair. The goal is a low-power, moderate-gain amplifier whose input common-mode level sits low (≈ 0.89 V), with every transistor held in saturation at the operating point.

The design was biased by hand with ideal DC sources and verified with `.op`, `.ac`, `.tran` and `.dc` analyses.

## How the circuit works

```
                         VDD = 1.8 V
        ┌─────────────┬──────┴──────┬─────────────┐
        │             │             │             │
       M4 (Vb4)     M1 (Vt)       M5 (Vb4)        │
        │             │             │             │
      net3        ┌───┴───┐       net4             │
        │         │ net1  │         │              │
       M6 (Vb3)  M2      M3       M7 (Vb3)         │
        │       (vin1)  (vin2)      │              │
      Vout1       │       │       Vout2            │
        │        Vx      net5       │              │
       M8 (Vb2)───┘       └────── M9 (Vb2)         │
        │                           │              │
        └─ Vx ── M10 (Vb1)   M11 (Vb1) ── net5 ────┘
                    │           │
                   GND         GND
```

*(Simplified sketch. The exact connectivity is in the schematic and netlist. M8 joins Vout1 to Vx, and M9 joins Vout2 to net5.)*

**Signal path**

1. **M1** (PMOS tail, gate = Vt) sets the input stage current at ≈ 10 µA.
2. **M2/M3** (PMOS differential pair) split the tail current, about 5 µA each at balance. A differential input steers current between the two sides.
3. The pair's drain currents are injected into the **fold nodes** (Vx and net5), where the NMOS sinks **M10/M11** pull ≈ 10 µA each.
4. Since M10 sinks 10 µA and M2 supplies 5 µA, the remaining ≈ 5 µA flows up through the **cascode devices M8/M9** from the output branch. The signal current from M2/M3 therefore flows into the output nodes **Vout1/Vout2**.
5. The **top PMOS sources M4/M5** (gate = Vb4) supply the 5 µA for the branch, and **PMOS cascodes M6/M7** (gate = Vb3) raise the output resistance from above.
6. Output current into the high output resistance becomes the output voltage.

**Polarity:** Vout1 is inverted with respect to `vin1` and in phase with `vin2`. The DC transfer and transient plots below both show this.

**Gain**

$$A_d = g_{m2}\,R_{out}, \qquad R_{out} = R_{up}\parallel R_{down}$$

$$R_{up}\approx g_{m6}\,r_{o6}\,r_{o4}, \qquad R_{down}\approx g_{m8}\,r_{o8}\,(r_{o10}\parallel r_{o2})$$

The gm of the cascode device (M6, M8) multiplies the ro of the device it sits on. The NMOS side (M8/M10) has the lower output resistance, so it sets the gain.

**Current budget**

| Branch | Current |
|---|---|
| M1 (tail) | 10 µA |
| M2, M3 (each) | 5 µA |
| M4–M9 (each fold leg) | 5 µA |
| M10, M11 (each) | 10 µA |
| **Total from VDD** | **≈ 20 µA** |

## Device sizes and biasing

### Transistor sizes (from the schematic netlist)

| Device | Type | Role | L (µm) | W (µm) | m | Total W (µm) |
|---|---|---|---|---|---|---|
| M1 | PMOS LVT | Tail current source | 10 | 10 | 11 | 110 |
| M2, M3 | PMOS LVT | Input differential pair | 10 | 5.55 | 10 | 55.5 |
| M4, M5 | PMOS LVT | Top current sources | 10 | 5.55 | 10 | 55.5 |
| M6, M7 | PMOS LVT | PMOS cascodes | 10 | 5.55 | 10 | 55.5 |
| M8, M9 | NMOS | NMOS cascodes | 5 | 1.5 | 5 | 7.5 |
| M10, M11 | NMOS | Fold-node current sinks | 10 | 30 | 1 | 30 |

Load: **C1 = C2 = 1 pF** from each output to ground.

### Bias voltages

| Node | Voltage | Drives |
|---|---|---|
| VDD | 1.8 V | Supply |
| Vt | 1.125 V | M1 gate |
| Vb1 | 0.69 V | M10/M11 gates |
| Vb2 | 1.05 V | M8/M9 gates |
| Vb3 | 0.775 V | M6/M7 gates |
| Vb4 | **1.1325 V** | M4/M5 gates (critical, see [Limitations](#limitations)) |
| Input common mode | 0.89 V | vin1, vin2 |

## Specifications (simulated)

Conditions: VDD = 1.8 V, TT corner, 27 °C, CL = 1 pF on each output.

| Parameter | Value |
|---|---|
| Supply current | 20.18 µA |
| Power dissipation | 36.3 µW |
| Tail current (M1) | 10.06 µA |
| Input pair current (each) | 5.03 µA |
| Input pair gm (gm2) | 47.0 µS |
| **Differential gain, Ad (low frequency)** | **60.06 dB (≈ 1007 V/V)** |
| Common-mode gain, Acm | 36.49 dB (≈ 67 V/V) |
| **CMRR (Ad / Acm)** | **23.57 dB** |
| Ad roll-off (−3 dB) | ≈ 7–8 kHz *(estimated from Rout·CL; the plot shows ≈ 55 dB at 10 kHz)* |
| Unity-gain frequency | ≈ 10 MHz *(read from the AC plot)* |
| Input common mode | 0.89 V |
| Output common mode | 0.472 V |

> **Note on CMRR:** Acm here is the common-mode-to-common-mode gain, so CMRR = Ad / Acm. It is not the common-mode-to-differential conversion that a mismatch analysis would give.

### DC operating point

| Device | Id | Vds | Vdsat | Vds − Vdsat | Region |
|---|---|---|---|---|---|
| M1 | 10.06 µA | 0.203 V | 0.181 V | 22 mV | Saturation (thin margin) |
| M2/M3 | 5.03 µA | 1.325 V | 0.176 V | 1.15 V | Saturation |
| M4/M5 | 5.06 µA | 0.298 V | 0.181 V | 117 mV | Saturation |
| M6/M7 | 5.06 µA | 1.030 V | 0.175 V | 0.86 V | Saturation |
| M8/M9 | 5.06 µA | 0.200 V | 0.166 V | 34 mV | Saturation (thin margin) |
| M10/M11 | 10.09 µA | 0.272 V | 0.154 V | 118 mV | Saturation |

## Simulation results and waveforms

### 1. DC transfer characteristic

![DC transfer](images/dc_transfer.png)

`vin1` is held at 0.89 V and `vin2` is swept from 0 to 1.8 V. The outputs switch when `vin2` crosses `vin1` (≈ 0.9 V): Vout1 rises and Vout2 falls, which confirms the polarity described above. The outputs saturate near 0.08 V and 1.77 V. Those flat ends are regions where the cascodes have left saturation, so they are not usable linear range. The sweep step is 10 mV, so the transition itself is coarsely resolved.

### 2. Transient response

![Transient](images/transient.png)

A 1 kHz differential sine of ±10 µV on each input (20 µV peak differential) gives ±10 mV on each output (20 mV peak differential), a gain of about 1000 V/V. This matches the 60 dB from the AC analysis. The outputs are 180° out of phase and sit on a common mode of ≈ 472 mV. A microvolt-level input is used on purpose, because at 1000 V/V a millivolt-level input would clip the outputs.

### 3. AC response: Ad, Acm and CMRR

![AC response](images/ac_response.png)

- **Ad (red):** flat at 60 dB, then a single-pole roll-off to about 10 MHz at 0 dB. At very high frequency it flattens near −14 dB because of capacitive feedthrough.
- **Acm (blue):** flat at 36.5 dB, rolling off along the same pole.
- **CMRR (orange):** about 23.6 dB at low frequency, staying near 24 dB with a dip to about 20 dB around 10 MHz.

### 4. Bias sensitivity: Vout1 vs Vb4

![Vb4 sweep](images/vb4_sweep.png)

This sweep shows the main weakness of the open-loop bias. Vout1 swings from about 1.4 V to 0.4 V over roughly **0.2 mV** of Vb4. The circuit has no common-mode feedback, so a tiny mismatch between the PMOS supply current and the NMOS sink current pushes the outputs toward a rail. The kinks in the curve mark where the cascodes leave saturation. The chosen Vb4 = 1.1325 V puts every device in saturation, but with Vout1 at the low end of its valid range.

## Hand analysis vs simulation

| Method | Assumptions | Ad |
|---|---|---|
| Ideal hand calculation | λ = 0.01 V⁻¹, Vov = 0.2 V, constant ro | ≈ 108 dB (≈ 250,000 V/V) |
| Hand calculation, simulated gm and gds | gm and gds from the operating point | ≈ 58.6 dB (≈ 855 V/V) |
| **Simulation (AC)** | | **60.06 dB (≈ 1007 V/V)** |

The simulation agrees with the hand calculation once the real small-signal values are used. The ideal figure is far higher because it assumes a constant Early voltage, and several devices do not have one. "In saturation" only means Vds > Vdsat. A device just inside saturation still has a large gds. For example, M8 has gds = 2.8 µS with 34 mV of margin, against about 0.2 µS when it had more than 1 V of Vds in earlier runs. Its ro is therefore only ≈ 0.35 MΩ, and the NMOS side limits Rout to about 18 MΩ against about 880 MΩ for the PMOS side.

The remaining ≈ 1.4 dB between the 58.6 dB hand calculation and the 60.06 dB simulation is probably body effect (gmb) on M8, which the formula ignores.

## How to reproduce

**Tools:** xschem, ngspice, the SKY130 PDK (for example via the IIC-OSIC-TOOLS container).

1. Open `diff_folded_cascade.sch` in xschem and generate the netlist.
2. Make sure the testbench code block points `.lib` at your local `sky130.lib.spice` and uses the `tt` corner:

```spice
.lib <path-to-pdk>/sky130A/libs.tech/ngspice/sky130.lib.spice tt

.control
  option temp = 27
  alter vdd dc = 1.8

  * ---- Operating point ----
  op
  print @m.xm1.msky130_fd_pr__pfet_01v8_lvt[id]  @m.xm1.msky130_fd_pr__pfet_01v8_lvt[vds] @m.xm1.msky130_fd_pr__pfet_01v8_lvt[vdsat]
  * (repeat the print lines for M2 ... M11)
  let i_vdd  = -i(vdd)
  let p_diss = 1.8 * i_vdd
  print i_vdd p_diss

  * ---- Differential gain: vin1 = +0.5, vin2 = -0.5 (Vid = 1 V) ----
  ac dec 20 1 10g

  * ---- Common-mode gain: vin1 = vin2 = 1 V AC ----
  alter vin1 acmag = 1
  alter vin2 acmag = 1
  ac dec 20 1 10g

  let Ad      = ac1.vout1 - ac1.vout2
  let Acm     = (ac2.vout1 + ac2.vout2)/2
  let Ad_dB   = db(Ad)
  let Acm_dB  = db(Acm)
  let CMRR_dB = Ad_dB - Acm_dB
  print Ad_dB[0] Acm_dB[0] CMRR_dB[0]
  plot Ad_dB Acm_dB CMRR_dB xlog title 'Ad, Acm and CMRR (dB)'

  * ---- Transient ----
  tran 1u 5m

  * ---- DC transfer ----
  dc vin2 0 1.8 0.01

  * ---- Bias sweep (shows the Vb4 sensitivity) ----
  * dc vb4 1.1315 1.1335 0.00005

  write diff_folded_cascade.raw
.endc
```

3. Run `ngspice` on the netlist. The `.op` printout should show all 11 devices with Vds > Vdsat.

**Tip:** if the plot names `ac1`/`ac2` are not recognised, type `display` in the ngspice console to see the actual names.

## Pros of the design

- **Low power.** About 36 µW from 1.8 V, with a 10 µA tail and 5 µA in each folded branch.
- **Fully differential.** Rejects supply and substrate noise common to both sides, doubles the usable differential swing, and cancels even-order distortion.
- **PMOS input pair.** Allows a low input common mode (0.89 V here), which suits sources that sit near ground. PMOS inputs also typically have lower 1/f noise than NMOS inputs.
- **Folded-cascode structure.** The input common-mode range is decoupled from the output swing, and cascoding raises gain without stacking the whole circuit between the rails.
- **Long channels (L = 5–10 µm).** Low gds and good matching. gm2 = 47 µS with a 5 µA bias gives gm/Id ≈ 9.3 V⁻¹, which is moderate inversion.
- **Single stage.** One dominant pole at the output (load-compensated), so no compensation capacitor is needed and the roll-off is clean, as the AC plot shows.
- **Verified operating point.** All 11 transistors are in saturation, and the currents match the design intent (10 / 5 / 5 / 10 µA).
- **Open-source flow.** Fully reproducible with free tools and the public SKY130 PDK.

## Limitations

These are known gaps in the current version.

- **No common-mode feedback (CMFB).** The output common mode is set by a fixed Vb4 that must be tuned to a window of roughly 0.1 mV (see the Vb4 sweep). It will not survive temperature, supply or process variation without a real CMFB circuit.
- **Moderate gain (60 dB).** The output common mode sits at 0.47 V, near the bottom of the valid range, so M8/M9 have only 34 mV of Vds margin and a high gds. Centring the outputs near 0.9 V should raise Ad. A rough estimate is 75–80 dB, but this has not been simulated.
- **Low CMRR (23.6 dB).** The tail M1 has only 22 mV of Vds margin, so its output conductance is high. Lowering the input common mode, for example from 0.89 V towards 0.8 V, should give it more headroom.
- **Thin saturation margins** on M1 (22 mV) and M8/M9 (34 mV).
- **Ideal bias sources.** Vt, Vb1–Vb4 are ideal voltage sources, not generated on-chip.
- **Limited characterisation.** Only the TT corner at 27 °C has been simulated. Noise, PSRR, input common-mode range, output swing, slew rate and settling, and process/voltage/temperature variation are not yet characterised.

## Possible applications

With the limitations above in mind, this topology suits:

- Low-frequency, low-power sensor and biomedical front ends (ECG/EEG-class signals)
- The first stage of a larger multi-stage op-amp
- An OTA for gm-C filters, or an integrator in a switched-capacitor circuit (a CMFB circuit would be needed first)
- A teaching example of folded-cascode biasing and the gain/saturation trade-offs

## Future work

- [ ] Add a CMFB circuit (switched-capacitor or continuous-time) to replace the hand-tuned Vb4
- [ ] Centre the output common mode near VDD/2 and re-measure Ad
- [ ] Improve the tail and bottom cascode margins to raise CMRR and gain
- [ ] Generate the bias voltages with an on-chip bias network
- [ ] Run process corners (ff/ss/fs/sf), temperature and supply sweeps
- [ ] Measure input-referred noise, PSRR, ICMR, output swing, slew rate and settling
- [ ] Draw the layout and run post-layout simulation

## Repository structure

```
.
├── README.md
├── diff_folded_cascade.sch      # xschem schematic
├── diff_folded_cascade.spice    # netlist and testbench with the .control block
└── images/
    ├── dc_transfer.png
    ├── transient.png
    ├── ac_response.png
    └── vb4_sweep.png
```

## Author

[Your Name]

## License

Add a license of your choice (for example MIT or Apache-2.0). Note that the SKY130 PDK has its own license terms.
