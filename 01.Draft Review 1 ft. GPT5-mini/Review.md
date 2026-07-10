
# Draft Review: STPMIC1/1L Software-Defined Power Management (STM32MP1)

Reviewer task: assess whether the draft paper presents a publishable technical contribution and list concrete gaps and improvements.

1. Claimed contribution
- The paper claims a software implementation of an STPMIC1/1L PMIC driver (BSP/HAL level) enabling Dynamic Voltage Scaling (DVS), coordinated power‑state transitions, and system‑level optimization on STM32MP1-class platforms.
- It also claims experimental oscilloscope characterization of low‑power modes (Sleep, Stop, LP‑Stop, LPLV‑Stop) and observed voltage‑rail behavior during mode transitions.

2. Novelty assessment
- Verdict: incremental / application‑note style. The work documents an engineering integration and measurement campaign for a known PMIC/SoC pairing rather than introducing new algorithms, models, or broadly generalizable theory.
- Classification: useful engineering demonstration and characterization; better suited for an industry track, workshop, or application note unless extended with novel control methods or deeper, generalizable analysis.

3. Missing evidence required for a strong conference submission
- System‑level energy/power numbers: whole‑board or SoC power (mW) in each mode and per‑mode energy savings (not just rail voltages).
- Latency metrics: voltage transition latency, settling time, and wake‑up latency (time-to-ready) for each low‑power mode.
- Repeatability and statistics: number of trials, mean±std or confidence intervals for all measurements.
- Workload impact: power vs. performance or energy‑per‑task for representative workloads demonstrating the benefit of DVS.
- Driver internals and safety: concrete implementation details (APIs, sequencing/state machine, failure handling, I2C timing implications, concurrency) to justify portability and correctness claims.
- Transient analysis: quantified overshoot/undershoot, peak deviations during transitions, and interaction with SoC regulators under load steps.
- Baselines and comparisons: static-rail baseline, and ideally a vendor or alternative PMIC/driver baseline.
- Reproducibility artifacts: code, scripts, oscilloscope settings, and raw traces as supplementary material.

4. Recommended experiments, tables, and baselines (prioritized)
- High priority:
	- Measure whole‑board/system power (mW) in Run, Sleep, Stop, LP‑Stop, and LPLV‑Stop with statistics.
	- Measure energy and time to wake for each mode (energy-per-wakeup cycle) and report tradeoffs across duty cycles.
	- Measure voltage transition latency, settling time, and peak overshoot for each rail under nominal and loaded conditions.
	- Baseline: static-rail (no DVS) power/perf comparison.
- Medium priority:
	- Workload experiments: idle, CPU-bound, memory‑intensive, and representative real tasks showing energy‑per‑task and performance impact.
	- Stability/noise characterization: ripple amplitude, PSD plots or spectral analysis, and probe/bandwidth settings.
	- Repeatability: N≥5 trials with mean±std and explicit measurement setup (probe attenuation, bandwidth, sampling rate).
- Lower priority (but valuable):
	- Long‑term cycling or thermal stress for reliability claims
	- Comparison to an alternate PMIC or vendor-provided driver where possible.

Suggested tables/figures:
- Table: per-mode summary — commanded voltages, measured mean voltages, ripple, transition latency, system power, wake energy.
- Plot: power vs. performance or energy-per-task for workloads, with/without DVS.
- Plot: annotated voltage/time traces showing overshoot and settling (zoomed view) with measurement bandwidth noted.
- Comparison table: this work vs. baseline(s) showing quantified improvements.

5. Unsupported or overstated claims
- "Significant energy savings" and "nearly 40%": the manuscript reports a ~30–40% voltage reduction on a secondary rail but does not present system power/energy measurements — percentage claims about energy savings are therefore unsubstantiated.
- "Voltage transitions occur without overshoot or instability": unsubstantiated without quantified transient/overshoot/settling metrics and load‑step tests.
- Generalization to future platforms ("well‑positioned for next‑generation STM32N6"): speculative unless supported by architectural/compatibility analysis.
- Any claim that BSP‑mode behavior inherently delivers system‑level effectiveness should be tempered unless supported by workload‑level or energy metrics.

6. Reframing recommendations to improve acceptance
- If retaining current scope: reframe as a rigorous measurement/characterization and engineering guide. Title example: "Characterization of STPMIC1/1L Dynamic Behavior and BSP Integration on STM32MP1: Measurements and Engineering Recommendations." Emphasize methods, reproducible dataset, oscilloscope/DAQ settings, and practical integration lessons.
- If aiming for research track: add a novel software technique (e.g., predictive DVS policy, closed‑loop voltage control, sequencing optimization algorithm) with quantitative comparison to the baseline BSP implementation.
- If comparative study is feasible: reframe as a quantitative comparison of PMIC‑driven DVS approaches including baselines and general design lessons.

7. Concrete next steps (practical checklist)
- Add whole‑board/system power measurements for all modes (with statistics).
- Instrument wake‑latency and per‑rail transition latency and report settling/overshoot values.
- Run representative workloads and report energy‑per‑task and performance tradeoffs with and without DVS.
- Add a table that consolidates voltage, power, latency, ripple, and wake energy per mode.
- Soften speculative language and add a dedicated "Limitations" paragraph.
- Publish code/scripts/traces as supplementary material or online artifact for reproducibility.

Summary verdict: The draft provides a useful engineering demonstration and oscilloscope traces, but as written it is not yet a strong candidate for a high‑impact research track. With targeted additional measurements (system energy, latency, repeatability), clearer baselines, and either a novel control contribution or a reproducible measurement dataset plus engineering recommendations, the work could become a solid submission to an industry/engineering track or a workshop.

