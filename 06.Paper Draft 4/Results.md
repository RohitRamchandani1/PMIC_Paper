# Experimental Design Critique: STPMIC1/1L Rail Measurement Study

---

## 1. Missing Baseline Comparisons

**Rail identity is never established.** "Channel 1" and "Channel 2" appear throughout all results without ever being mapped to a named PMIC output (BUCK1, BUCK2, LDO1, etc.) or a named STM32MP13x power domain (VDDCORE, VDD_DDR, VDDIO, etc.). Without this mapping, a reviewer cannot verify that the observed LPLV-Stop reduction is the intended DVS rail or an incidental one.

**Commanded voltage vs. measured voltage is never compared.** The BSP writes a target voltage over I2C; the paper should show the register-programmed target alongside the oscilloscope mean. A 1.31 V measurement is meaningless without knowing whether the PMIC was commanded to 1.30 V, 1.35 V, or some NVM default. This delta is the actual accuracy claim.

**No NVM-default baseline.** If the board powers up with a pre-programmed NVM state, the paper cannot distinguish "BSP software changed the rail" from "this is the rail's hardware default." A capture before `BSP_PMIC_Power_Mode_Init()` executes would establish this.

**Sleep, Stop, and LP-Stop are described in the implementation section but entirely absent from results.** The abstract lists seven modes; only four are measured. Reviewers will notice the gap immediately. Either measure all seven or explicitly scope the paper to the four modes with documented justification.

---

## 2. Missing Quantitative Metrics

| Missing metric | Why it matters | Realistic source |
|---|---|---|
| Rail ripple (mV peak-to-peak or RMS) | Switching converter quality claim requires a number, not "low-amplitude ripple" | Oscilloscope measurement statistics or AC-coupled capture |
| Transition duration (Run → LPLV-Stop and reverse) | DVS implies a latency cost; not quantified anywhere | Oscilloscope cursors on the transition edge |
| Wake-up latency from each LP state | Critical for any real-time or latency-sensitive framing | Oscilloscope trigger from wake event to rail restoration |
| Current on primary supply rail | Without current, "power reduction" cannot be stated; voltage alone is insufficient | STM32MP135F-DK has an onboard IDD measurement header (CN16) documented in UM2993 |
| Number of repeated captures | N=1 per mode provides no statistical confidence | Minimum 5–10 captures per mode; report mean ± std deviation |
| Rail settling time after transition | Relevant to DVS implementation quality | Time from I2C command to rail within ±2% of target |

The most serious gap is the absence of any current measurement. The Discussion section uses the phrase "voltage reduction of roughly 30%," which implicitly suggests energy reduction, but dynamic power scales as $CV^2f$, not $CV$. Without current data, no energy claim—even a qualified one—can be made.

---

## 3. Missing Measurement Conditions

All of the following are unspecified in the current draft:

- **Oscilloscope settings**: probe type, attenuation ratio (1× vs. 10×), input coupling (DC or AC), bandwidth limit, time/div, voltage/div, sampling rate, trigger source and level, trigger mode. Without these, the measurements are not reproducible.
- **Load conditions during each capture**: Is the CPU executing code, in WFI, or halted by the debugger? Is DDR in self-refresh or fully active? Are USB, Ethernet, or wireless peripherals clocked? These directly affect rail voltage and current.
- **Ambient temperature**: Regulator output voltage, efficiency, and ripple are temperature-dependent.
- **Board power source**: USB-C at 5 V, external 5 V barrel, or bench supply? Supply impedance affects rail regulation.
- **Probe ground connection point**: On a switching-converter output, probe ground placement relative to the output capacitor ground can add or subtract measured ripple artifacts.
- **Wake-up source configuration**: The firmware mentions wake-up must be configured before LP entry, but the actual source (RTC, GPIO, EXTI) used during the measurement is never stated.

---

## 4. Confounding Factors

**Ground loop and probe artifact risk.** Single-ended oscilloscope probing of SMPS outputs is sensitive to ground lead inductance. The negative residuals measured in Standby (-25 mV to -35 mV) are reported as "oscilloscope noise floor," but undisclosed probe ground routing could introduce a systematic offset. A differential probe or a ground spring tip should be used, or the probe ground arrangement should be explicitly described so reviewers can assess this.

**PMIC device identity is ambiguous throughout.** The title, abstract, and conclusion use "STPMIC1/1L" as if both were tested. The STM32MP135F-DK board carries a physically fixed PMIC. The paper should state once, definitively, which device is populated on the test board and treat the other as a note or future scope. Conflating them throughout the paper makes every measurement claim ambiguous.

**Debugger connection state during measurement.** An attached ST-LINK debugger activates debug power domains and can prevent the CPU from entering certain low-power states or can hold clocks active. If measurements were taken with the debugger connected, the power state may not represent production behavior. This must be disclosed.

**The 842 mV value is not verified against any PMIC documentation.** Is 842 mV the programmed LP voltage for LPLV-Stop on this rail as specified in the BSP or NVM? Or is it a sagging rail under residual load? Without the target register value and a load-current measurement, a reviewer cannot determine whether this is correct DVS operation or a partially loaded rail misbehaving.

**Only 2 of 6+ PMIC outputs are measured.** STPMIC1/1L provides BUCK1, BUCK2, LDO1–LDO4, and load switches. Unmeasured outputs may remain active at full voltage during the "low-power" modes, making the power state characterization incomplete. The paper should either measure all active outputs or justify the two-rail selection with reference to the BSP-defined LP configuration.

---

## 5. Tables and Plots Required for Conference Submission

**Table: Rail-to-domain mapping and mode coverage**
For each PMIC output: rail name, nominal voltage, LP voltage (if applicable), associated STM32MP13x power domain, and whether the rail is enabled/scaled/disabled in each of the seven modes.

**Table: Oscilloscope measurement conditions**
One row per capture: channel, probe model, attenuation, coupling, time/div, voltage/div, bandwidth limit, trigger source, trigger level, number of captures averaged.

**Table: Commanded vs. measured voltage per rail per mode**
BSP/NVM-programmed target, oscilloscope measured mean, delta (mV and %), number of captures.

**Plot: Transition waveform**
A time-domain capture spanning the Run → LPLV-Stop transition showing both channels, including the actual voltage step. Currently only steady-state captures are shown. A reviewer evaluating DVS latency needs to see the transition, not just the before and after states.

**Plot: Mode-by-mode summary bar chart or box plot**
Rail voltage for both channels across all measured modes, with error bars from repeated captures. This replaces the current prose-only comparison with a visual that is standard in measurement papers.

---

## 6. Likely Reviewer Challenges

1. **"Which PMIC was actually on the board?"** The STPMIC1/1L dual labeling without a definitive statement will generate an immediate reviewer question.

2. **"Why are four of seven described modes missing from the results?"** Sleep, Stop, and LP-Stop are in the abstract and implementation section, which creates an expectation reviewers will check.

3. **"What is the energy reduction?"** Any paper titled with "Dynamic Voltage Scaling" that omits current measurement or power data will face this challenge. The DVS claim is rail-level only, and that needs to be stated precisely in the contribution statement.

4. **"What was the CPU doing during each measurement?"** Without firmware state disclosure (executing benchmark loop, WFI, halted by debugger), the results are not reproducible and may not represent the intended operating mode.

5. **"Why should we trust a single oscilloscope capture?"** Reviewers for IEEE conference proceedings expect repeated measurements and statistical characterization, even for voltage rails.

6. **"How does 1.31 V / 1.20 V relate to the PMIC datasheet nominal values?"** The paper must show that the BSP-configured values match the intended targets. Uncorroborated absolute values raise questions about calibration and probe accuracy.

7. **"What is the wake latency from Standby?"** If the paper motivates LPLV-Stop vs. Standby as a tradeoff, reviewers will expect wake latency to be part of that comparison.

---

## 7. Minimum Additional Experiments Before Submission

Ranked by impact on reviewer credibility:

1. **Document channel-to-rail mapping.** Identify PMIC output names for Channel 1 and Channel 2 from the board schematic (STM32MP135F-DK schematics are publicly available from ST). This is a documentation task, not a new measurement.

2. **Add current measurement using the onboard IDD connector.** UM2993 documents CN16 as an onboard current-sense path. A bench ammeter or oscilloscope current probe on this path provides supply current in each mode, enabling a watts-level comparison across modes.

3. **Capture at least three repeats per mode.** Report mean ± standard deviation for voltage on each channel. This establishes measurement reproducibility at minimal additional cost.

4. **Capture the Run → LPLV-Stop transition waveform** with time cursors showing the voltage step duration. One capture with the oscilloscope triggered on the transition is sufficient.

5. **Record the programmed PMIC register target** for each rail in each mode (from BSP source or register log) and compare to the measured mean. This directly validates the BSP command path.

6. **Measure and report wake latency** from LPLV-Stop back to Run Mode using the oscilloscope trigger from the wake event to the rail settling point. This is the key DVS cost metric that is currently absent.

7. **Measure the three omitted modes (Sleep, Stop, LP-Stop), or explicitly narrow the paper scope** to the four measured modes with a documented rationale for the omission.

---

## Summary Assessment

The core experimental apparatus is appropriate for the stated scope: the STM32MP135F-DK provides a relevant platform, the BSP APIs expose the right control path, and an oscilloscope is sufficient for voltage-domain characterization. The fundamental problem is that the measurements as currently reported document that *some* rails changed during mode transitions on *an unspecified* board configuration with *unspecified* load conditions, captured once per mode. That is adequate for a proof-of-concept demonstration but falls short of the reproducibility, completeness, and quantitative rigor expected at IEEE conference level. The experiments needed to close this gap—current measurement, repeated captures, transition waveforms, and rail identity documentation—are all achievable from the described hardware without new equipment.
