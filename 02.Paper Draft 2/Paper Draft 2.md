# Software-Controlled Dynamic Voltage Scaling Using the STPMIC1/1L PMIC for STM32MP1-Based Embedded Systems

## Abstract
Modern embedded processors require multiple regulated voltage domains, deterministic power sequencing, and software-controlled transitions between operating states. Discrete regulator-based power architectures can satisfy basic supply requirements, but they offer limited coordination across rails and make runtime voltage management difficult as system complexity increases. This paper presents an experimental evaluation of software-controlled power-state management using the STPMIC1/1L Power Management Integrated Circuit (PMIC) on an STM32MP135F-DK platform. The implementation uses the STM32Cube bare-metal firmware environment and Board Support Package (BSP) PMIC APIs to initialize the PMIC, configure voltage rails, and enter Run, Sleep, Stop, Low-Power Stop (LP-Stop), Low-Power Low-Voltage Stop (LPLV-Stop), Standby, and Switch-Off states. Oscilloscope measurements show that Run Mode maintains measured rails near 1.31 V and 1.20 V, while LPLV-Stop reduces the second measured rail to approximately 842 mV with the first measured rail remaining near 1.31 V. Standby and Switch-Off measurements show rail collapse toward the oscilloscope noise floor. The results demonstrate that the STPMIC1/1L BSP-controlled PMIC path can perform stable software-driven voltage scaling and controlled rail shutdown on the evaluated STM32MP1 platform.

## Keywords
Power Management Integrated Circuit, STPMIC1, STPMIC1L, STM32MP1, dynamic voltage scaling, low-power modes, embedded systems, I2C, BSP

## I. Introduction
Embedded systems increasingly rely on heterogeneous processors, high-speed interfaces, memory subsystems, and mixed-signal peripherals that operate from multiple voltage domains. Supplying these domains with independent discrete regulators is possible, but this approach increases board-level complexity and places sequencing, fault handling, and runtime voltage coordination across several separate devices. These constraints become more important in energy-constrained platforms, where the system must reduce voltage and disable rails during low-power states without compromising wake-up behavior or rail stability.

Multi-output PMICs address this problem by integrating DC-DC converters, low-dropout regulators (LDOs), load switches, sequencing logic, and digital control interfaces into a single coordinated power device. For STM32MP1-class microprocessor units (MPUs), the STPMIC1 and STPMIC1L provide a PMIC architecture that can be configured through I2C and managed through board-level firmware. This enables software to participate directly in power-state transitions rather than treating voltage regulation as a fixed hardware function.

The remaining gap is not whether PMICs can provide regulated supplies, but how the software-controlled PMIC path behaves during practical low-power transitions on an STM32MP1 evaluation platform. In particular, embedded developers need evidence that BSP-level APIs can configure rails, apply Dynamic Voltage Scaling (DVS), and transition between Run, low-power, and shutdown states while maintaining observable rail stability.

This paper addresses that gap through a structured implementation and measurement study of the STPMIC1/1L driver path on the STM32MP135F-DK discovery kit. The contribution of this work is threefold:

1. It documents a bare-metal STM32Cube PMIC control flow using BSP initialization, rail configuration, and power-mode transition APIs.
2. It experimentally characterizes measured rail behavior in Run, LPLV-Stop, Standby, and Switch-Off modes using oscilloscope captures.
3. It separates software implementation behavior from measurement results and discussion, providing a clearer basis for evaluating PMIC-assisted low-power design on STM32MP1-based systems.

## II. Related Work and Technical Context
Power-management strategies for embedded systems have evolved from discrete regulator arrangements toward integrated PMIC-based architectures. Prior work on multi-rail embedded platforms reports that systems using separate buck converters and LDOs can face challenges in board density, thermal distribution, sequencing coordination, and runtime flexibility [1]-[4]. PMIC-oriented studies and application materials describe how integrated regulators, protection logic, and sequencing engines can reduce power-delivery complexity and improve coordination across rails [5]-[8].

The STPMIC1 family is positioned for STM32MP1-class systems and includes multiple DC-DC converters, LDOs, load switches, programmable sequencing, and an I2C control interface [9]-[11]. These capabilities make it suitable for software-managed operating states in which firmware changes rail configuration according to platform activity.

Software-based power management is also a central technique in embedded low-power design. Dynamic voltage and frequency scaling and software-controlled low-power states can reduce energy use when workload or retention requirements permit lower voltage or rail shutdown [12]-[15]. This paper builds on that context by focusing on the board-level PMIC driver path and measured rail behavior during selected low-power transitions on an STM32MP135F-DK platform.

## III. System Platform and PMIC Architecture

### A. Hardware Platform
The experimental platform is the STM32MP135F-DK discovery kit, which uses an STM32MP1 MPU as the primary processing device. The board is powered through an STPMIC1 or STPMIC1L PMIC connected through I2C Instance 4. The PMIC supplies major MPU voltage domains through BUCK regulators, LDOs, and load switches. The measured rails include outputs associated with system domains such as the processing core, DDR interface, GPIO banks, communication peripherals, and internal analog circuitry.

The PMIC consolidates voltage generation and sequencing so that startup, voltage scaling, low-power entry, and rail shutdown can be coordinated through PMIC configuration and firmware-level control. External rail behavior was measured using a Tektronix Mixed Domain Oscilloscope.

![STPMIC1L connected to STM32MP135F-DK board](../Paper%20Draft%201/01.Hardware_Setup.png)

**Fig. 1.** STPMIC1L connected to the STM32MP135F-DK board.

### B. PMIC-Controlled Voltage Domains
The PMIC provides multiple regulated outputs, including BUCK1, BUCK2, and several LDO rails. In the evaluated setup, oscilloscope probes were connected to selected PMIC output rails to observe steady-state voltages, voltage reduction during low-power operation, and rail collapse during shutdown-oriented states.

**Recommended Table I.** Add a complete rail-to-load mapping table when the final board configuration is available. The table should list each PMIC output, nominal Run Mode voltage, low-power voltage if applicable, associated STM32MP1 domain, and whether the rail remains active in each low-power mode.

## IV. Software Implementation

### A. Firmware Environment
The implementation was developed in a bare-metal environment using the STM32Cube firmware ecosystem on a Windows development workstation. The PMIC software path contains three main elements: a register-map abstraction for PMIC register access, I2C communication routines, and voltage-setting APIs used to modify rail configuration at runtime.

### B. BSP PMIC Control Flow
The BSP hierarchy provides the firmware entry points used for PMIC control:

1. `BSP_PMIC_Init()` loads PMIC parameters stored in the PMIC non-volatile memory (NVM).
2. `BSP_PMIC_Power_Mode_Init()` configures initial rail voltages and enables or disables the required BUCK and LDO outputs.
3. `BSP_PMIC_Set_Power_Mode()` transitions the platform among Run, Sleep, Stop, LP-Stop, LPLV-Stop, Standby, and related low-power states.

The platform powers up in Run Mode by default. Before entering deeper low-power states, wake-up sources must be configured so that the system can return to Run Mode after the low-power interval. The PMIC power-management flow is summarized in Fig. 2.

![STPMIC1/1L BSP power-management flow](../Paper%20Draft%201/01.Software_Setup.png)

**Fig. 2.** BSP-controlled STPMIC1/1L power-management flow: initialization, low-power entry, wake-up handling, and return to Run Mode.

### C. Power-Mode Sequence
Each evaluated mode was invoked through the BSP PMIC functions. Voltage transitions were initiated by writing new rail configurations to the PMIC over I2C. For LP-Stop and LPLV-Stop, reduced voltage levels were applied to selected domains to support DVS. Standby and Switch-Off modes were evaluated to observe rail shutdown behavior and residual voltage levels at the oscilloscope inputs.

**Recommended Fig. 3.** Add a concise timing diagram showing the software sequence from Run Mode through I2C rail update, low-power entry, wake-up event, and return to Run Mode. This figure should make the control dependency between firmware and PMIC state explicit.

## V. Measurement Methodology

### A. Instrumentation
Voltage rail behavior was captured using a Tektronix Mixed Domain Oscilloscope. The probes were connected to selected PMIC output rails to measure steady-state voltage, ripple characteristics, transition behavior, and rail collapse during low-power and shutdown-oriented states.

![Experimental oscilloscope setup, view 1](../Paper%20Draft%201/02.Experimental_procedure_01.png)

![Experimental oscilloscope setup, view 2](../Paper%20Draft%201/03.Experimental_procedure_02.png)

**Fig. 3.** Oscilloscope-based measurement setup for PMIC rail observation during software-controlled power-mode transitions.

### B. Experimental Procedure
The evaluation began with Run Mode as the baseline operating condition. Low-power modes were then invoked sequentially through the BSP PMIC control path. During each mode, oscilloscope captures were recorded for the selected rails. The analysis compares mean rail voltages, visible ripple behavior, and qualitative rail stability across Run, LPLV-Stop, Standby, and Switch-Off modes.

The available measurements support a voltage-focused comparison. Current consumption, energy per transition, and full transition timing were not reported in the source measurements and are therefore not quantified in this draft.

**Recommended Table II.** Include oscilloscope settings for reproducibility: probe attenuation, coupling, bandwidth limit, sampling rate, time scale, voltage scale, trigger condition, and measurement location for each channel.

## VI. Measurement Results

### A. Run Mode
Fig. 4 shows two active rails measured simultaneously in Run Mode. The oscilloscope reports mean voltages of approximately 1.31 V on Channel 1 and 1.20 V on Channel 2. Both rails remain close to their observed mean values across the sampling window, with low-amplitude high-frequency ripple consistent with switching-regulator operation under normal system activity.

![Run Mode rail waveform](../Paper%20Draft%201/04.Run_Mode.png)

**Fig. 4.** Measured PMIC rail behavior in Run Mode on the STM32MP135F-DK board.

### B. LPLV-Stop Mode
Fig. 5 shows the measured behavior in LPLV-Stop Mode. The first measured rail remains near 1.31 V, while the second measured rail is reduced to approximately 842 mV. This reduction demonstrates the DVS behavior configured through the PMIC software path. The waveform does not show visible instability or oscillation in the captured interval.

![LPLV-Stop Mode rail waveform](../Paper%20Draft%201/05.LPLV_Stop_Mode.png)

**Fig. 5.** Measured PMIC rail behavior in LPLV-Stop Mode.

### C. Standby Mode
Fig. 6 shows both measured channels collapsing toward near-zero voltage in Standby Mode. The measured values are in the approximate range of -25 mV to -35 mV, which is consistent with oscilloscope-level residual noise around the measurement baseline. This behavior indicates that the measured rails are disabled during Standby Mode, while any retention or backup supplies are outside the measured channels.

![Standby Mode rail waveform](../Paper%20Draft%201/06.Stand_By_Mode.png)

**Fig. 6.** Measured PMIC rail behavior in Standby Mode.

### D. Switch-Off Mode
Fig. 7 shows near-zero measured rail voltages in Switch-Off Mode. The captured waveform shows no distinguishable ripple or dynamic rail behavior on the measured channels, indicating that the user-domain outputs under observation are deactivated in this state.

![Switch-Off Mode rail waveform](../Paper%20Draft%201/07.Switch_OFF_Mode.png)

**Fig. 7.** Measured PMIC rail behavior in Switch-Off Mode.

### E. Summary of Measured Rail Behavior
Table I summarizes the measured values available from the oscilloscope captures. The table intentionally reports only values stated in the source draft and does not infer unmeasured current or power consumption.

**Table I**
**Observed Rail Voltages Across Evaluated PMIC Power Modes**

| Mode | Channel 1 observed mean | Channel 2 observed mean | Observed behavior |
|---|---:|---:|---|
| Run | approximately 1.31 V | approximately 1.20 V | Both measured rails active and stable in the captured interval. |
| LPLV-Stop | approximately 1.31 V | approximately 842 mV | Second measured rail reduced while first measured rail remains active. |
| Standby | approximately -25 mV to -35 mV | approximately -25 mV to -35 mV | Measured rails collapse toward the oscilloscope noise floor. |
| Switch-Off | near-zero | near-zero | Measured rails remain deactivated with no distinguishable ripple. |

**Recommended Table III.** Add a mode-feature matrix that lists Run, Sleep, Stop, LP-Stop, LPLV-Stop, Standby, and Switch-Off; PMIC outputs enabled; outputs scaled; wake-up source requirements; and expected retention behavior. This should be completed from the BSP configuration and PMIC documentation.

## VII. Discussion
The measurements show that the BSP-controlled STPMIC1/1L path can maintain active rails in Run Mode, reduce a selected measured rail in LPLV-Stop Mode, and disable measured rails in Standby and Switch-Off modes. The transition from the Run Mode Channel 2 value of approximately 1.20 V to the LPLV-Stop value of approximately 842 mV corresponds to an observed rail-voltage reduction of roughly 30% for that measured channel. This supports the intended DVS behavior for the evaluated low-voltage stop configuration.

The Standby and Switch-Off captures indicate controlled rail deactivation on the measured channels. The Standby values near -25 mV to -35 mV should be interpreted as measurement-baseline residuals rather than negative supply operation. The Switch-Off waveform similarly shows near-zero rail voltage and no distinguishable ripple on the measured outputs.

These results validate the functional behavior of the software-controlled PMIC path for the selected modes, but they do not by themselves quantify total energy savings. Voltage reduction is an important enabling condition for lower dynamic power, yet system-level power reduction also depends on load current, clock state, regulator efficiency, retention domains, and time spent in each mode. A complete energy evaluation would therefore require current measurements or power-monitor data in addition to the voltage waveforms reported here.

## VIII. Limitations and Evidence Needed Before Submission
Several parts of the paper require additional evidence before the work can be submitted as a complete IEEE conference paper:

1. The literature references must be verified against real publications and formatted consistently according to IEEE style.
2. Claims about improved conversion efficiency, reduced PCB footprint, and thermal benefits require either citations, measurements, or narrower wording.
3. Claims of system-level energy savings require current or power measurements; voltage-only data supports voltage scaling, not total energy reduction.
4. Transition-time claims require measured timing values from oscilloscope cursors or logs.
5. Ripple and noise claims should include quantitative peak-to-peak or RMS values if they are central to the argument.
6. The exact PMIC rail names, channel-to-rail mapping, load domains, and low-power retention behavior should be documented in a table.
7. The distinction between STPMIC1 and STPMIC1L should be clarified if both devices were not physically tested.
8. The applicability to future STM32 platforms should be either supported by documentation or removed from the core conclusion.

## IX. Conclusion
This paper restructured and evaluated a software-controlled PMIC power-management implementation for an STM32MP135F-DK platform using the STPMIC1/1L PMIC. The BSP control path initializes PMIC parameters, configures voltage rails, and transitions the platform through low-power operating states. Oscilloscope measurements show stable Run Mode rail behavior near 1.31 V and 1.20 V, LPLV-Stop voltage reduction of the second measured rail to approximately 842 mV, and measured rail collapse toward the oscilloscope noise floor in Standby and Switch-Off modes. The results support the use of the STPMIC1/1L and BSP driver path for software-managed voltage scaling and rail shutdown on STM32MP1-based embedded platforms. Future work should add current measurements, transition timing, complete rail mapping, and verified citation support to quantify energy benefits and strengthen conference-paper readiness.

## References
[1] A. Kapoor and J. Rabaey, "Challenges in Multi-Rail Power Delivery for Embedded Platforms," ISLPED, 2017.

[2] L. Chen and M. Pedram, "Power Architecture Limitations in Discrete Voltage-Regulator-Based Embedded Systems," IEEE Trans. VLSI, 2018.

[3] S. Gupta et al., "Thermal and Sequencing Constraints in Multi-Domain Discrete Power Systems," Microelectronics Journal, 2019.

[4] R. Srinivasan and T. Kim, "Energy Inefficiencies in Battery-Powered Devices using Discrete Regulators," IEEE Trans. Power Electronics, 2018.

[5] A. V. Kamat and R. Li, "A Survey of PMIC Architectures for Low-Power SoCs," IEEE Comms Surveys & Tutorials, 2019.

[6] E. Tan and P. Mathew, "Integrated PMIC Solutions for Embedded and Mobile Platforms," MWSCAS, 2020.

[7] J. Park, "Embedded PMIC Sequencing and Fault-Management Techniques," IEEE TCAS-I, 2020.

[8] N. D. Patel and S. Singh, "Programmable Voltage Scaling Interfaces for Real-Time Embedded Systems," IEEE Embedded Systems Letters, 2020.

[9] STMicroelectronics, "STPMIC1 PMIC for STM32MP1 Applications," Application Note AN5308, 2021.

[10] F. Rossi et al., "Multi-Output PMICs for Heterogeneous SoCs: A Case Study with STM32MP1," DATE, 2021.

[11] STMicroelectronics, "STPMIC1L Ultra-Low-Power Variant," Datasheet DS13541, 2022.

[12] H. Huang and Y. Wang, "Software-Governed DVFS Techniques for Low-Power Embedded Computing," IEEE TCAD, 2019.

[13] P. Chandel, R. Sahoo, and L. Niu, "DVFS in Linux-Based Embedded Systems," ACM TECS, 2020.

[14] M. Y. Kim and D. Shin, "Low-Power State Management through Software-Controlled Regulators," IEEE EUC, 2019.

[15] A. Thomas, "Evaluation of Sleep and Stop Modes in Power-Constrained SoCs," IEEE Access, 2021.
