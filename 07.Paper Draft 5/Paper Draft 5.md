# Software-Controlled Dynamic Voltage Scaling Using the STPMIC1/1L PMIC for STM32MP1-Based Embedded Systems

## Abstract
Modern embedded processors require multiple regulated voltage domains, deterministic power sequencing, and software-controlled transitions between operating states. Discrete regulator-based power architectures can satisfy basic supply requirements, but they offer limited coordination across rails and complicate runtime voltage management as system complexity increases. This paper presents an experimental evaluation of software-controlled power-state management using the STPMIC1/1L Power Management Integrated Circuit (PMIC) on an STM32MP135F-DK platform. The implementation uses the STM32Cube bare-metal firmware environment and Board Support Package (BSP) PMIC APIs to initialize the PMIC, configure voltage rails, and enter Run, Sleep, Stop, Low-Power Stop (LP-Stop), Low-Power Low-Voltage Stop (LPLV-Stop), Standby, and Switch-Off states. Oscilloscope measurements show that Run Mode maintains the measured rails near 1.31 V and 1.20 V, while LPLV-Stop reduces the second measured rail to approximately 842 mV, with the first measured rail remaining near 1.31 V. Standby and Switch-Off measurements show rail collapse toward the oscilloscope noise floor. The results demonstrate that the STPMIC1/1L BSP-controlled PMIC path can perform stable software-driven voltage scaling and controlled rail shutdown on the evaluated STM32MP1 platform.

## Keywords
Power Management Integrated Circuit, STPMIC1, STPMIC1L, STM32MP1, dynamic voltage scaling, low-power modes, embedded systems, I2C, BSP

## I. Introduction
Embedded systems increasingly rely on heterogeneous processors, high-speed interfaces, memory subsystems, and mixed-signal peripherals that operate from multiple voltage domains. Supplying these domains with independent discrete regulators is possible, but this approach increases board-level complexity and places sequencing, fault handling, and runtime voltage coordination across several separate devices. These constraints become more important in energy-constrained platforms, where the system must reduce voltage and disable rails during low-power states without compromising wake-up behavior or rail stability.

Multi-output PMICs address this problem by integrating DC-DC converters, low-dropout regulators (LDOs), load switches, sequencing logic, and digital control interfaces into a single coordinated power device. For STM32MP1-class microprocessor units (MPUs), the STPMIC1 and STPMIC1L provide a PMIC architecture that can be configured through I2C and managed through board-level firmware. This enables software to participate directly in power-state transitions rather than treating voltage regulation as a fixed hardware function.

The remaining gap is not whether PMICs can provide regulated supplies, but how the software-controlled PMIC path behaves during practical low-power transitions on an STM32MP1 evaluation platform. In particular, embedded developers need evidence that BSP-level APIs can configure rails, apply Dynamic Voltage Scaling (DVS), and transition between Run, low-power, and shutdown states while maintaining observable rail stability.

This paper addresses that gap through a structured implementation and measurement study of the STPMIC1/1L driver path on the STM32MP135F-DK discovery kit. The contributions of this work are threefold:

1. It documents a bare-metal STM32Cube PMIC control flow using BSP initialization, rail configuration, and power-mode transition APIs.
2. It experimentally characterizes measured rail behavior in Run, LPLV-Stop, Standby, and Switch-Off modes using oscilloscope captures.
3. It separates software implementation behavior from measurement results and discussion, providing a clearer basis for evaluating PMIC-assisted low-power design on STM32MP1-based systems.

## II. Related Work and Technical Context
Power-management strategies for embedded systems have evolved from discrete regulator arrangements toward integrated PMIC-based architectures. Prior work on multi-rail embedded platforms reports that systems using separate buck converters and LDOs can face challenges in board density, thermal distribution, sequencing coordination, and runtime flexibility [1]–[4]. PMIC-oriented studies and application materials describe how integrated regulators, protection logic, and sequencing engines can reduce power-delivery complexity and improve coordination across rails [5]–[8].

The STPMIC1 family is positioned for STM32MP1-class systems and includes multiple DC-DC converters, LDOs, load switches, programmable sequencing, and an I2C control interface [9]–[11]. These capabilities make it suitable for software-managed operating states in which firmware changes rail configuration according to platform activity.

Software-based power management is also a central technique in embedded low-power design. Dynamic voltage and frequency scaling (DVFS) and software-controlled low-power states can reduce energy consumption when workload or retention requirements permit lower voltage or rail shutdown [12]–[15]. This paper builds on that context by focusing on the board-level PMIC driver path and measured rail behavior during selected low-power transitions on an STM32MP135F-DK platform.

## III. Literature Review

### Scope

This literature review covers verified sources relevant to software-controlled power management for STM32MP1-class embedded systems using STPMIC1 and STPMIC1L power-management ICs [11], [12], [14], [20]. It draws on peer-reviewed IEEE/ACM work on dynamic voltage scaling (DVS), dynamic voltage and frequency scaling (DVFS), and dynamic power management (DPM), together with STMicroelectronics reference manuals, board documentation, datasheets, and application notes [1]–[25]. The listed sources are limited to papers with DOI links and vendor documents with official STMicroelectronics URLs [1]–[25].

### Synthesis of the Literature

Software-controlled low-power design for embedded systems is supported by prior work on variable-speed scheduling, DVS, DVFS, and DPM [1]–[10]. CMOS dynamic power is approximately proportional to $CV^2f$, so voltage reduction is commonly treated as a mechanism for reducing dynamic energy when performance constraints permit it.

> **TODO: Verify citation for the CMOS $CV^2f$ dynamic-power relationship.** Candidate references (author to confirm before submission): (a) A. P. Chandrakasan, S. Sheng, and R. W. Brodersen, "Low-power CMOS digital design," *IEEE J. Solid-State Circuits*, vol. 27, no. 4, pp. 473–484, Apr. 1992, DOI: https://doi.org/10.1109/4.126534; (b) J. M. Rabaey, A. Chandrakasan, and B. Nikolic, *Digital Integrated Circuits: A Design Perspective*, 2nd ed., Prentice Hall, 2003 (Ch. 5).

Yao, Demers, and Shenker formalized a scheduling model in which variable processor speed can reduce CPU energy when timing constraints are considered [1]. Pillai and Shin showed that operating-system software can manage dynamic voltage scaling for embedded real-time workloads [2]. Pering, Burd, and Brodersen evaluated DVS algorithms and showed that workload conditions and policy choices affect energy outcomes [3]. Burd, Pering, Stratakos, and Brodersen demonstrated a dynamic-voltage-scaled microprocessor system, supporting the claim that DVS can be implemented in real processor hardware [4].

DPM extends the literature beyond CPU voltage scaling by addressing component and subsystem power states [5]–[7]. Benini, Bogliolo, and De Micheli surveyed system-level DPM techniques and identified policies and hardware state transitions as requirements for managing active and low-power states [5]. Simunic, Benini, Glynn, and De Micheli showed that portable-system energy management depends on software policies that account for idle periods and state-transition costs [6]. Simunic, Benini, Acquaviva, Glynn, and De Micheli further showed that DVFS and DPM can be combined in portable systems, so voltage scaling should be considered together with broader power-state management [7].

Embedded and real-time systems require DVFS and DPM claims to be evaluated in relation to timing, workload, and schedulability constraints [8]–[10]. Pouwelse, Langendoen, and Sips evaluated DVS on a low-power microprocessor and supported the claim that practical low-power processors can expose software-controllable voltage-scaling behavior [8]. Zhong and Xu addressed energy-aware modeling and scheduling for real-time tasks under DVS, showing that real-time DVFS evaluation must include task timing and schedulability constraints [9]. Bhatti, Belleudy, and Auguin examined hybrid power management in real-time embedded systems and treated DVFS and DPM as interacting techniques rather than independent optimizations [10]. These sources support a conservative interpretation of STM32MP1 voltage-rail experiments: a voltage trace can support a rail-level DVS observation, but system-level energy claims require measurements and analysis beyond rail voltage alone [3], [7], [9], [10].

The STM32MP1 platform introduces a multi-domain power-control context for PMIC-based experiments [11], [12]. STMicroelectronics documentation describes STM32MP1-class MPUs as Arm-based 32-bit devices with platform-level power-control and low-power-mode architecture that software must coordinate [11], [12]. The STM32MP135F-DK board documentation identifies the STM32MP135FA MPU, STPMIC1 PMIC, DDR3L memory, USB, Ethernet, wireless connectivity, and onboard current-measurement support as board-level features [13]. This documentation supports using the STM32MP135F-DK as a relevant target for PMIC-controlled STM32MP13x experiments, but it does not by itself establish whole-board energy reduction [13].

STPMIC1 documentation provides the PMIC-specific basis for host-controlled power management on STM32MP1-class systems [14]–[19]. The STPMIC1 datasheet describes a highly integrated PMIC for microprocessor units with multiple buck converters, LDOs, boost and power-switch functions, programmable non-volatile memory, and I2C/digital I/O control [14]. The STPMIC1 I2C programming guide supports claims about software-visible PMIC control through an I2C programming model [15]. ST application notes provide integration guidance for STM32MP151/153/157 battery-powered applications, STM32MP13x wall-adapter-powered applications using STPMIC1 variants, STPMIC1 auto turn-on behavior, and STPMIC1 PCB layout practice [16]–[19]. These sources support claims about STPMIC1 integration paths, sequencing-related behavior, and board-design requirements, but they remain vendor documentation rather than peer-reviewed evidence [14]–[19].

STPMIC1L documentation supports claims about a newer ST PMIC option for STM32MP1x applications [20]–[25]. ST describes the STPMIC1L as a power-management IC for MPUs with two buck converters, four LDOs, programmable non-volatile memory, I2C and digital I/O control, programmable output voltages, programmable turn-on and turn-off sequences, and immediate alternate output settings through dedicated power-control pins [20]. STPMIC1L application notes provide official STM32MP13 wall-adapter guidance, non-volatile-memory configuration information, application hints, bill-of-material details, and PCB layout guidelines [21]–[25]. These documents support claims about STPMIC1L rail programmability, sequencing support, configuration, external-component requirements, and layout-dependent regulator integration [20]–[25].

Taken together, the reviewed sources support a careful framing of the STM32MP1/STPMIC1 paper [1]–[25]. The peer-reviewed DVS, DVFS, and DPM literature supports the relevance of software-controlled voltage scaling, power-state control, workload-aware policy, and timing-aware evaluation [1]–[10]. The STMicroelectronics manuals, board documents, datasheets, and application notes support the claim that STM32MP1/STPMIC1-class hardware exposes PMIC, rail-control, sequencing, low-power-mode, and integration mechanisms relevant to software-controlled experiments [11]–[25]. The same source set does not support a claim that voltage measurements alone prove system-level energy savings, because the listed sources connect energy outcomes to workload behavior, timing constraints, idle intervals, state-transition costs, and broader DVFS/DPM interactions [3], [6], [7], [9], [10].

### Source Matrix

| No. | Full citation | DOI or official URL | Claim supported | Source type |
|---:|---|---|---|---|
| [1] | F. Yao, A. Demers, and S. Shenker, "A scheduling model for reduced CPU energy," in *Proceedings of IEEE 36th Annual Foundations of Computer Science*, 1995, pp. 374-382. | https://doi.org/10.1109/SFCS.1995.492493 | Variable-speed/voltage-aware scheduling can reduce CPU energy when timing constraints are considered. | Peer-reviewed conference paper |
| [2] | P. Pillai and K. G. Shin, "Real-time dynamic voltage scaling for low-power embedded operating systems," in *Proceedings of the 18th ACM Symposium on Operating Systems Principles*, 2001, pp. 89-102. | https://doi.org/10.1145/502034.502044 | Operating-system software can manage voltage scaling for embedded real-time workloads. | Peer-reviewed conference paper |
| [3] | T. Pering, T. Burd, and R. Brodersen, "The simulation and evaluation of dynamic voltage scaling algorithms," in *Proceedings of the 1998 International Symposium on Low Power Electronics and Design*, 1998, pp. 76-81. | https://doi.org/10.1145/280756.280790 | DVS algorithms must be evaluated under workload-dependent conditions; policy choice affects energy outcomes. | Peer-reviewed conference paper |
| [4] | T. D. Burd, T. A. Pering, A. J. Stratakos, and R. W. Brodersen, "A dynamic voltage scaled microprocessor system," *IEEE Journal of Solid-State Circuits*, vol. 35, no. 11, pp. 1571-1580, Nov. 2000. | https://doi.org/10.1109/4.881202 | Dynamic voltage scaling can be implemented in real microprocessor hardware, not only simulated at policy level. | Peer-reviewed journal article |
| [5] | L. Benini, A. Bogliolo, and G. De Micheli, "A survey of design techniques for system-level dynamic power management," *IEEE Transactions on Very Large Scale Integration (VLSI) Systems*, vol. 8, no. 3, pp. 299-316, Jun. 2000. | https://doi.org/10.1109/92.845896 | System-level DPM requires policies and hardware state transitions for components and subsystems. | Peer-reviewed journal article |
| [6] | T. Simunic, L. Benini, P. Glynn, and G. De Micheli, "Dynamic power management for portable systems," in *Proceedings of the 6th Annual International Conference on Mobile Computing and Networking*, 2000, pp. 11-19. | https://doi.org/10.1145/345910.345914 | Portable-system energy management depends on software policies that account for idle periods and state-transition costs. | Peer-reviewed conference paper |
| [7] | T. Simunic, L. Benini, A. Acquaviva, P. Glynn, and G. De Micheli, "Dynamic voltage scaling and power management for portable systems," in *Proceedings of the 38th Design Automation Conference*, 2001. | https://doi.org/10.1109/DAC.2001.935564 | DVFS and DPM can be combined in portable systems; voltage scaling should be considered with broader power-state management. | Peer-reviewed conference paper |
| [8] | J. Pouwelse, K. Langendoen, and H. Sips, "Dynamic voltage scaling on a low-power microprocessor," in *Proceedings of the 7th Annual International Conference on Mobile Computing and Networking*, 2001, pp. 251-259. | https://doi.org/10.1145/381677.381701 | Practical low-power microprocessors can expose software-controllable voltage scaling behavior. | Peer-reviewed conference paper |
| [9] | X. Zhong and C.-Z. Xu, "Energy-aware modeling and scheduling of real-time tasks for dynamic voltage scaling," in *26th IEEE International Real-Time Systems Symposium*, 2005, pp. 366-375. | https://doi.org/10.1109/RTSS.2005.17 | Real-time DVFS evaluation must include task timing and schedulability constraints. | Peer-reviewed conference paper |
| [10] | M. K. Bhatti, C. Belleudy, and M. Auguin, "Hybrid power management in real time embedded systems: an interplay of DVFS and DPM techniques," *Real-Time Systems*, vol. 47, no. 2, pp. 143-162, 2011. | https://doi.org/10.1007/s11241-011-9116-y | DVFS and DPM interact in embedded systems; evaluating one without the other can miss important power-mode tradeoffs. | Peer-reviewed journal article |
| [11] | STMicroelectronics, "STM32MP157 advanced Arm-based 32-bit MPUs," Reference Manual RM0436. | https://www.st.com/en/microcontrollers-microprocessors/stm32mp157.html | STM32MP1-class MPUs contain power-control and low-power-mode architecture that software must coordinate. | Reference manual / vendor documentation |
| [12] | STMicroelectronics, "STM32MP13xx advanced Arm-based 32-bit MPUs," Reference Manual RM0475. | https://www.st.com/en/microcontrollers-microprocessors/stm32mp135.html | STM32MP13x devices provide the platform-level power architecture relevant to STM32MP135F-DK experiments. | Reference manual / vendor documentation |
| [13] | STMicroelectronics, "STM32MP135F-DK: Discovery kit with STM32MP135F MPU," Databrief DB4677, version 1.0, Feb. 7, 2023; User Manual UM2993, version 2.0, Mar. 7, 2023. | https://www.st.com/en/evaluation-tools/stm32mp135f-dk.html | The STM32MP135F-DK includes an STM32MP135FA MPU, STPMIC1 PMIC, DDR3L memory, and onboard current-measurement support. | Board documentation / databrief / user manual |
| [14] | STMicroelectronics, "STPMIC1: Highly integrated power management IC for micro processor units," Datasheet DS12792, version 10.0, May 18, 2022. | https://www.st.com/en/power-management/stpmic1.html | STPMIC1 integrates multiple buck converters, LDOs, boost/power switches, programmable NVM, and I2C/digital I/O control for host-controlled power management. | Datasheet |
| [15] | STMicroelectronics, "AN5440: The STPMIC1 I2C programming guide," Application Note, version 1.0, Feb. 7, 2020. | https://www.st.com/en/power-management/stpmic1.html#documentation | STPMIC1 exposes an I2C programming model for software control of PMIC functions. | Application note |
| [16] | STMicroelectronics, "AN5260: STM32MP151/153/157 MPU lines and STPMIC1 integration on a battery powered application," Application Note, version 2.0, Jan. 28, 2021. | https://www.st.com/en/power-management/stpmic1.html#documentation | ST provides integration guidance for STM32MP15x MPUs with STPMIC1 in battery-powered designs. | Application note |
| [17] | STMicroelectronics, "AN5587: Integration of STM32MP13x MPU product lines and STPMIC1D / STPMIC1A on a wall adapter supply," Application Note, version 2.0, Jun. 27, 2023. | https://www.st.com/en/power-management/stpmic1.html#documentation | STPMIC1 variants are officially documented for STM32MP13x integration on wall-adapter-powered systems. | Application note |
| [18] | STMicroelectronics, "AN5861: STPMIC1 auto turn-on," Application Note, version 1.0, Nov. 30, 2022. | https://www.st.com/en/power-management/stpmic1.html#documentation | STPMIC1 includes documented startup/turn-on behavior relevant to PMIC sequencing. | Application note |
| [19] | STMicroelectronics, "AN5431: The STPMIC1 PCB layout guidelines," Application Note, version 1.1, Dec. 12, 2019. | https://www.st.com/en/power-management/stpmic1.html#documentation | PMIC-based power design requires layout guidance for stable regulator behavior and integration. | Application note |
| [20] | STMicroelectronics, "STPMIC1L: Power management IC for MPU: 2 buck converters and 4 LDOs," Datasheet DS14839, version 4.0, Apr. 30, 2026. | https://www.st.com/en/power-management/stpmic1l.html | STPMIC1L supports STM32MP1x applications with two buck converters, four LDOs, programmable NVM, I2C/digital I/O control, sequencing, and alternate output settings. | Datasheet |
| [21] | STMicroelectronics, "AN6312: How to use STPMIC1L to develop a wall adapter powered application on STM32MP13 MPUs," Application Note, version 3.0, Apr. 13, 2026. | https://www.st.com/en/power-management/stpmic1l.html#documentation | STPMIC1L has official STM32MP13 application guidance for wall-adapter-powered designs. | Application note |
| [22] | STMicroelectronics, "AN6421: STPMIC1L NVM configuration," Application Note, version 1.0, Dec. 2, 2025. | https://www.st.com/en/power-management/stpmic1l.html#documentation | STPMIC1L behavior can be configured through non-volatile settings, supporting scalable PMIC configuration. | Application note |
| [23] | STMicroelectronics, "AN6429: STPMIC1L application hints," Application Note, version 1.0, Dec. 3, 2025. | https://www.st.com/en/power-management/stpmic1l.html#documentation | STPMIC1L integration has vendor-specified design and application considerations. | Application note |
| [24] | STMicroelectronics, "AN6438: The STPMIC1L BOM details," Application Note, version 1.0, Dec. 19, 2025. | https://www.st.com/en/power-management/stpmic1l.html#documentation | STPMIC1L designs require defined external components, supporting claims about PMIC-based board-level integration. | Application note |
| [25] | STMicroelectronics, "AN6384: The STPMIC1L PCB layout guidelines," Application Note, version 1.0, Oct. 22, 2025. | https://www.st.com/en/power-management/stpmic1l.html#documentation | STPMIC1L regulator performance and integration depend on board-layout practice. | Application note |

### Implications for the STM32MP1/STPMIC1 Paper

The strongest supportable technical claim is that STM32MP1/STPMIC1-class hardware provides a software-accessible path for PMIC control, rail configuration, sequencing-related behavior, and low-power-mode coordination [11]–[19]. The reviewed peer-reviewed literature supports the relevance of DVS, DVFS, and DPM as established low-power techniques, while the STMicroelectronics sources support the specific STM32MP1, STM32MP135F-DK, STPMIC1, and STPMIC1L hardware context [1]–[25]. Oscilloscope captures can validate whether selected rails change as commanded, but such captures alone support rail-behavior claims rather than whole-board energy-savings claims [3], [7], [9], [10], [13].

The paper should avoid claiming system-level energy savings unless current, power, workload, timing, and state-transition evidence are measured or otherwise supported [3], [6], [7], [9], [10], [13]. A voltage reduction on one rail can support a DVS or PMIC-command observation for that rail when tied to the relevant hardware documentation and measurement evidence [8], [14], [15], [20]. Total energy interpretation requires attention to workload behavior, timing constraints, idle intervals, state-transition costs, and interactions between DVFS and DPM [3], [6], [7], [9], [10]. For conference readiness, the reviewed source set supports adding evidence on whole-board current or power, transition and wake latency, repeatability, and workload-level energy/performance behavior [3], [6], [7], [9], [10], [13].

### Appendix: Unmet Claims

- "CMOS dynamic power is approximately proportional to $CV^2f$, so voltage reduction is commonly treated as a mechanism for reducing dynamic energy when performance constraints permit it." The source matrix supports DVS/DVFS generally but does not explicitly cite a source for the CMOS $CV^2f$ relationship. **Suggested remedy:** add one of the following candidate references (author to verify before submission): (a) A. P. Chandrakasan, S. Sheng, and R. W. Brodersen, "Low-power CMOS digital design," *IEEE J. Solid-State Circuits*, vol. 27, no. 4, pp. 473–484, Apr. 1992, DOI: https://doi.org/10.1109/4.126534; (b) J. M. Rabaey, A. Chandrakasan, and B. Nikolic, *Digital Integrated Circuits: A Design Perspective*, 2nd ed., Prentice Hall, 2003 (Ch. 5); or remove the formula-level claim and cite the supporting DVS sources directly.

## IV. System Platform and PMIC Architecture

### A. Hardware Platform
The experimental platform is the STM32MP135F-DK discovery kit, which uses an STM32MP1 MPU as the primary processing device. The board is powered through an STPMIC1 or STPMIC1L PMIC connected via I2C Instance 4. The PMIC supplies major MPU voltage domains through BUCK regulators, LDOs, and load switches. The measured rails include outputs associated with system domains such as the processing core, DDR interface, GPIO banks, communication peripherals, and internal analog circuitry.

The PMIC consolidates voltage generation and sequencing so that startup, voltage scaling, low-power entry, and rail shutdown can be coordinated through PMIC configuration and firmware-level control. External rail behavior was measured using a Tektronix Mixed Domain Oscilloscope.

![STPMIC1L connected to STM32MP135F-DK board](./01.Hardware_Setup.png)

**Fig. 1.** STPMIC1L connected to the STM32MP135F-DK board.

### B. PMIC-Controlled Voltage Domains
The PMIC provides multiple regulated outputs, including BUCK1, BUCK2, and several LDO rails. In the evaluated setup, oscilloscope probes were connected to selected PMIC output rails to observe steady-state voltages, voltage reduction during low-power operation, and rail collapse during shutdown-oriented states.

> **TODO — Table I (Rail-to-Load Mapping).** When the final board configuration is confirmed, add a table listing each PMIC output, nominal Run Mode voltage, low-power voltage (if applicable), associated STM32MP1 power domain, and rail state in each evaluated mode. Derive from STM32MP135F-DK schematic (UM2993) and STPMIC1/1L datasheets [13], [14], [20].

## V. Software Implementation

### A. Firmware Environment
The implementation was developed in a bare-metal environment using the STM32Cube firmware ecosystem on a Windows development workstation. The PMIC software path contains three main elements: a register-map abstraction for PMIC register access, I2C communication routines, and voltage-setting APIs used to modify rail configuration at runtime.

### B. BSP PMIC Control Flow
The BSP hierarchy provides the firmware entry points used for PMIC control:

1. `BSP_PMIC_Init()` loads PMIC parameters stored in the PMIC non-volatile memory (NVM).
2. `BSP_PMIC_Power_Mode_Init()` configures initial rail voltages and enables or disables the required BUCK and LDO outputs.
3. `BSP_PMIC_Set_Power_Mode()` transitions the platform among Run, Sleep, Stop, LP-Stop, LPLV-Stop, Standby, and related low-power states.

The platform powers up in Run Mode by default. Before entering deeper low-power states, wake-up sources must be configured so that the system can return to Run Mode after the low-power interval. The PMIC power-management flow is summarized in Fig. 2.

![STPMIC1/1L BSP power-management flow](./01.Software_Setup.png)

**Fig. 2.** BSP-controlled STPMIC1/1L power-management flow: initialization, low-power entry, wake-up handling, and return to Run Mode.

### C. Power-Mode Sequence
Each evaluated mode was invoked through the BSP PMIC functions. Voltage transitions were initiated by writing new rail configurations to the PMIC over I2C. For LP-Stop and LPLV-Stop, reduced voltage levels were applied to selected domains to support DVS. Standby and Switch-Off modes were evaluated to observe rail shutdown behavior and residual voltage levels at the oscilloscope inputs.

> **TODO — Fig. 3 (Timing Diagram).** Add a concise timing diagram showing the software sequence from Run Mode through I2C rail update, low-power entry, wake-up event, and return to Run Mode, making the firmware-to-PMIC control dependency explicit. *Note: inserting this figure as Fig. 3 will shift the oscilloscope figure (currently Fig. 3) to Fig. 4 and all subsequent figure numbers by one.*

## VI. Measurement Methodology

### A. Instrumentation
Voltage rail behavior was captured using a Tektronix Mixed Domain Oscilloscope. The probes were connected to selected PMIC output rails to measure steady-state voltage, ripple characteristics, transition behavior, and rail collapse during low-power and shutdown-oriented states.

![Experimental oscilloscope setup, view 1](./02.Experimental_procedure_01.png)

![Experimental oscilloscope setup, view 2](./03.Experimental_procedure_02.png)

**Fig. 3.** Oscilloscope-based measurement setup for PMIC rail observation during software-controlled power-mode transitions.

### B. Experimental Procedure
The evaluation began with Run Mode as the baseline operating condition. Low-power modes were then invoked sequentially through the BSP PMIC control path. During each mode, oscilloscope captures were recorded for the selected rails. The analysis compares mean rail voltages, visible ripple behavior, and qualitative rail stability across Run, LPLV-Stop, Standby, and Switch-Off modes.

The available measurements support a voltage-focused comparison. Current consumption, energy per transition, and full transition timing were not reported in the source measurements and are therefore not quantified in this paper.

> **TODO — Table II (Oscilloscope Measurement Conditions).** For reproducibility, add a table listing probe model, attenuation ratio, input coupling (DC/AC), bandwidth limit, time/div, voltage/div, sampling rate, trigger source, and trigger level for each channel and each mode capture.

## VII. Measurement Results

### A. Run Mode

Fig. 4 shows two rails measured simultaneously under Run Mode conditions on the STM32MP135F-DK board. The oscilloscope reports a mean voltage of approximately 1.31 V on Channel 1 and approximately 1.20 V on Channel 2. Both channels remain near their respective mean values across the captured interval.

> **TODO: Quantify switching-regulator ripple amplitude.** A peak-to-peak or RMS ripple value (in mV, AC-coupled) is required before any statement about ripple magnitude can appear in the paper. No ripple value is reported in the source captures.

![Run Mode rail waveform](./04.Run_Mode.png)

**Fig. 4.** Run Mode rail voltages on the STM32MP135F-DK board. Channel 1: 1.31 V mean. Channel 2: 1.20 V mean.

### B. LPLV-Stop Mode

Fig. 5 shows rail voltages after the BSP transitions the platform to LPLV-Stop Mode. Channel 1 remains at approximately 1.31 V. Channel 2 is reduced to approximately 842 mV. No oscillation or visible instability is present in the captured interval.

> **TODO: Provide BSP-programmed target voltage for Channel 2 in LPLV-Stop Mode.** The measured 842 mV cannot be verified as correct DVS operation without comparison to the commanded target value. The identity of the PMIC output (BUCK1, BUCK2, or LDO) that Channel 2 represents must also be established from the board schematic (UM2993) to interpret the rail reduction in the STM32MP13x power-domain context.

> **TODO: Capture the Run-to-LPLV-Stop voltage transition waveform.** The transition duration and settling behavior are not in the current measurement set. A cursor-measured transition waveform is required to characterize DVS latency.

![LPLV-Stop Mode rail waveform](./05.LPLV_Stop_Mode.png)

**Fig. 5.** LPLV-Stop Mode rail voltages. Channel 1: 1.31 V mean. Channel 2: 842 mV mean.

### C. Standby Mode

Fig. 6 shows both channels at near-zero levels in Standby Mode. The measured values fall in the range of −25 mV to −35 mV on both channels. These values lie within the oscilloscope measurement baseline and indicate that the probed rails are deactivated in this state. Retention or backup supply rails present on the board are outside the two measured channels and are not characterized here.

![Standby Mode rail waveform](./06.Stand_By_Mode.png)

**Fig. 6.** Standby Mode rail voltages. Both channels: −25 mV to −35 mV, within the oscilloscope noise floor.

### D. Switch-Off Mode

Fig. 7 shows near-zero measured voltage on both channels in Switch-Off Mode. No distinguishable ripple or switching activity is present on the measured outputs.

![Switch-Off Mode rail waveform](./07.Switch_OFF_Mode.png)

**Fig. 7.** Switch-Off Mode rail voltages. Both channels near zero; no ripple detected on measured outputs.

### E. Summary of Observed Rail Voltages

Table I reports the mean rail voltages recorded from the oscilloscope captures. Only values present in the source measurements are included. Ripple amplitude, voltage transition duration, wake latency, and current consumption were not measured in this study and are omitted.

**Table I — Observed Rail Voltages Across Evaluated PMIC Power Modes**

| Mode | Channel 1 mean (V) | Channel 2 mean (V) | Observed rail state |
|---|---:|---:|---|
| Run | ≈ 1.31 | ≈ 1.20 | Both rails active; no instability in captured interval. |
| LPLV-Stop | ≈ 1.31 | ≈ 0.842 | Channel 2 reduced; Channel 1 unchanged. |
| Standby | −0.025 to −0.035 | −0.025 to −0.035 | Both rails at oscilloscope noise floor; rails deactivated. |
| Switch-Off | ≈ 0 | ≈ 0 | Both rails deactivated; no switching activity on measured outputs. |

> **TODO — Table II (Commanded vs. Measured Voltage).** When BSP register targets are available, add a table with columns: Mode, Channel, PMIC output name (BUCK/LDO), BSP-programmed target voltage (V), oscilloscope-measured mean (V), and deviation (mV and %). This table is the minimum evidence required to substantiate the DVS accuracy claim.

> **TODO — Table III (Comprehensive Mode Characterization).** Add a table with the following columns for each evaluated mode: peak-to-peak ripple (mV), Run-to-mode transition duration (µs or ms), mode-to-Run wake latency (µs or ms), and PMIC outputs active/scaled/disabled. This consolidates voltage, ripple, and timing evidence required for conference-level characterization.

> **TODO — Table IV (Oscilloscope Measurement Conditions).** For reproducibility, add a table listing probe model, attenuation ratio, input coupling (DC/AC), bandwidth limit, time/div, voltage/div, sampling rate, trigger source, and trigger level for each channel and each mode capture.

> **TODO — Table V (Mode-Feature Matrix).** Add a table listing all seven BSP-defined modes (Run, Sleep, Stop, LP-Stop, LPLV-Stop, Standby, Switch-Off), the PMIC outputs enabled or disabled in each, and the wake-up source requirements. Sleep, Stop, and LP-Stop are described in Section IV but are absent from the measurements; this table should either include measured data for those modes or explicitly document them as out of scope with a stated justification.

## VIII. Discussion

The four captured modes—Run, LPLV-Stop, Standby, and Switch-Off—show that the BSP-controlled STPMIC1/1L driver produces measurably distinct, software-selectable rail states on the STM32MP135F-DK platform. Channel 1 remains at 1.31 V across Run and LPLV-Stop modes, indicating that the BSP low-voltage stop configuration applies a selective per-rail reduction rather than a platform-wide voltage change. Channel 2 transitions from 1.20 V in Run Mode to 842 mV in LPLV-Stop Mode, a reduction of 358 mV (29.8%). In Standby and Switch-Off modes, both measured rails are deactivated; the residuals of −25 mV to −35 mV on both channels are within the oscilloscope noise floor and do not indicate residual supply activity on the measured outputs.

**Rail-voltage reduction versus power reduction.** The 29.8% Channel 2 voltage reduction is a rail-level observation. Dynamic CMOS power scales as $P \propto CV^2f$ (see Section III, Appendix: Unmet Claims for citation TODO), so a reduction from 1.20 V to 0.842 V would reduce the $V^2$-dependent power component on that rail by approximately 50.6% if load capacitance and switching frequency were held constant. However, no current measurement is reported in this study. > **TODO:** Any statement about power or energy reduction requires measured supply current or power-monitor data and must account for load state, regulator efficiency, and retained-domain activity on unmeasured rails. The voltage data alone supports a rail-level DVS observation; it does not support a system-level energy savings claim [3], [7], [9], [10].

**Channel identity and rail verification.** The PMIC output (BUCK1, BUCK2, LDO1–LDO4) and the STM32MP13x power domain (VDDCORE, VDD_DDR, VDDIO, etc.) corresponding to each oscilloscope channel are not identified in the current measurement set. Without this mapping, the LPLV-Stop reduction cannot be attributed to the intended DVS rail or verified against the BSP-programmed target. > **TODO:** Channel-to-rail mapping must be established from the STM32MP135F-DK schematic (UM2993) and documented in Table II before the result can be interpreted in a domain-specific context.

**Mode coverage.** Sleep, Stop, and LP-Stop modes are described in the firmware implementation (Section V) and enumerated in the abstract, but are absent from the measurement results. The current dataset covers four of seven stated operating modes. > **TODO:** Either measure and report the three omitted modes, or explicitly narrow the paper scope to the four evaluated modes with a documented rationale for the omission.

**Measurement reproducibility.** Each reported value is drawn from a single oscilloscope capture per mode. Single-capture data provides no statistical characterization of rail behavior across repeated transitions. > **TODO:** A minimum of five repeated captures per mode, with reported mean and standard deviation, is required before the tabulated values can be treated as characterization data rather than single-point observations.

**Transition behavior.** The captures show only steady-state behavior before and after mode transitions. The Run-to-LPLV-Stop voltage transition waveform—including step amplitude, overshoot, and settling time—is not captured. > **TODO:** Transition duration is the primary latency cost of DVS operation. Without a cursor-measured transition waveform, no timing claim about the DVS step can be made.

**Debugger connection state.** If the ST-LINK debugger was connected during measurement, it may have held debug power domains active and prevented the CPU from reaching the target low-power state. This condition must be disclosed, as measurements taken with an attached debugger may not represent production-representative power behavior.

The results support the following bounded conclusion: the BSP-controlled STPMIC1/1L driver produces software-selectable, distinct rail states across the four evaluated modes on the STM32MP135F-DK board. The scope of valid inference from the current data is limited to (1) Channel 2 voltage is reduced from 1.20 V to 842 mV when the BSP enters LPLV-Stop Mode, as observed in a single oscilloscope capture; and (2) both measured rails are deactivated in Standby and Switch-Off modes, as observed in single captures. Quantified energy savings, ripple characterization, transition timing, wake latency, and full seven-mode coverage each require additional measurements before they can be claimed.

## IX. Limitations and Evidence Needed Before Submission
Several parts of the paper require additional evidence before the work can be submitted as a complete IEEE conference paper:

1. The literature references must be verified against real publications and formatted consistently according to IEEE style.
2. Claims about improved conversion efficiency, reduced PCB footprint, and thermal benefits require either citations, measurements, or narrower wording.
3. Claims of system-level energy savings require current or power measurements; voltage-only data supports voltage scaling, not total energy reduction.
4. Transition-time claims require measured timing values from oscilloscope cursors or logs.
5. Ripple and noise claims should include quantitative peak-to-peak or RMS values if they are central to the argument.
6. The exact PMIC rail names, channel-to-rail mapping, load domains, and low-power retention behavior should be documented in a table.
7. The distinction between STPMIC1 and STPMIC1L should be clarified if both devices were not physically tested.
8. The applicability to future STM32 platforms should be either supported by documentation or removed from the core conclusion.

## X. Conclusion
This paper presented a software-controlled PMIC power-management implementation for the STM32MP135F-DK platform using the STPMIC1/1L PMIC. The BSP control path initializes PMIC parameters, configures voltage rails, and transitions the platform through low-power operating states. Oscilloscope measurements show stable Run Mode rail behavior near 1.31 V and 1.20 V, LPLV-Stop voltage reduction of the second measured rail to approximately 842 mV, and measured rail collapse toward the oscilloscope noise floor in Standby and Switch-Off modes. The results support the use of the STPMIC1/1L and BSP driver path for software-managed voltage scaling and rail shutdown on STM32MP1-based embedded platforms. Future work should add current measurements, transition timing, complete rail mapping, and verified citation support to quantify energy benefits and strengthen conference-paper readiness.

## References

[1] F. Yao, A. Demers, and S. Shenker, "A scheduling model for reduced CPU energy," in *Proc. IEEE 36th Annu. Found. Comput. Sci.*, 1995, pp. 374–382. DOI: https://doi.org/10.1109/SFCS.1995.492493

[2] P. Pillai and K. G. Shin, "Real-time dynamic voltage scaling for low-power embedded operating systems," in *Proc. 18th ACM Symp. Oper. Syst. Princ.*, 2001, pp. 89–102. DOI: https://doi.org/10.1145/502034.502044

[3] T. Pering, T. Burd, and R. Brodersen, "The simulation and evaluation of dynamic voltage scaling algorithms," in *Proc. Int. Symp. Low Power Electron. Design*, 1998, pp. 76–81. DOI: https://doi.org/10.1145/280756.280790

[4] T. D. Burd, T. A. Pering, A. J. Stratakos, and R. W. Brodersen, "A dynamic voltage scaled microprocessor system," *IEEE J. Solid-State Circuits*, vol. 35, no. 11, pp. 1571–1580, Nov. 2000. DOI: https://doi.org/10.1109/4.881202

[5] L. Benini, A. Bogliolo, and G. De Micheli, "A survey of design techniques for system-level dynamic power management," *IEEE Trans. Very Large Scale Integr. (VLSI) Syst.*, vol. 8, no. 3, pp. 299–316, Jun. 2000. DOI: https://doi.org/10.1109/92.845896

[6] T. Simunic, L. Benini, P. Glynn, and G. De Micheli, "Dynamic power management for portable systems," in *Proc. 6th Annu. Int. Conf. Mobile Comput. Netw.*, 2000, pp. 11–19. DOI: https://doi.org/10.1145/345910.345914

[7] T. Simunic, L. Benini, A. Acquaviva, P. Glynn, and G. De Micheli, "Dynamic voltage scaling and power management for portable systems," in *Proc. 38th Design Autom. Conf.*, 2001. DOI: https://doi.org/10.1109/DAC.2001.935564

[8] J. Pouwelse, K. Langendoen, and H. Sips, "Dynamic voltage scaling on a low-power microprocessor," in *Proc. 7th Annu. Int. Conf. Mobile Comput. Netw.*, 2001, pp. 251–259. DOI: https://doi.org/10.1145/381677.381701

[9] X. Zhong and C.-Z. Xu, "Energy-aware modeling and scheduling of real-time tasks for dynamic voltage scaling," in *Proc. 26th IEEE Int. Real-Time Syst. Symp.*, 2005, pp. 366–375. DOI: https://doi.org/10.1109/RTSS.2005.17

[10] M. K. Bhatti, C. Belleudy, and M. Auguin, "Hybrid power management in real time embedded systems: an interplay of DVFS and DPM techniques," *Real-Time Syst.*, vol. 47, no. 2, pp. 143–162, 2011. DOI: https://doi.org/10.1007/s11241-011-9116-y

[11] STMicroelectronics, "STM32MP157 advanced Arm-based 32-bit MPUs," Reference Manual RM0436. [Online]. Available: https://www.st.com/en/microcontrollers-microprocessors/stm32mp157.html

[12] STMicroelectronics, "STM32MP13xx advanced Arm-based 32-bit MPUs," Reference Manual RM0475. [Online]. Available: https://www.st.com/en/microcontrollers-microprocessors/stm32mp135.html

[13] STMicroelectronics, "STM32MP135F-DK: Discovery kit with STM32MP135F MPU," Databrief DB4677, ver. 1.0, Feb. 2023; User Manual UM2993, ver. 2.0, Mar. 2023. [Online]. Available: https://www.st.com/en/evaluation-tools/stm32mp135f-dk.html

[14] STMicroelectronics, "STPMIC1: Highly integrated power management IC for micro processor units," Datasheet DS12792, ver. 10.0, May 2022. [Online]. Available: https://www.st.com/en/power-management/stpmic1.html

[15] STMicroelectronics, "AN5440: The STPMIC1 I2C programming guide," Application Note, ver. 1.0, Feb. 2020. [Online]. Available: https://www.st.com/en/power-management/stpmic1.html#documentation

[16] STMicroelectronics, "AN5260: STM32MP151/153/157 MPU lines and STPMIC1 integration on a battery powered application," Application Note, ver. 2.0, Jan. 2021. [Online]. Available: https://www.st.com/en/power-management/stpmic1.html#documentation

[17] STMicroelectronics, "AN5587: Integration of STM32MP13x MPU product lines and STPMIC1D/STPMIC1A on a wall adapter supply," Application Note, ver. 2.0, Jun. 2023. [Online]. Available: https://www.st.com/en/power-management/stpmic1.html#documentation

[18] STMicroelectronics, "AN5861: STPMIC1 auto turn-on," Application Note, ver. 1.0, Nov. 2022. [Online]. Available: https://www.st.com/en/power-management/stpmic1.html#documentation

[19] STMicroelectronics, "AN5431: The STPMIC1 PCB layout guidelines," Application Note, ver. 1.1, Dec. 2019. [Online]. Available: https://www.st.com/en/power-management/stpmic1.html#documentation

[20] STMicroelectronics, "STPMIC1L: Power management IC for MPU: 2 buck converters and 4 LDOs," Datasheet DS14839, ver. 4.0, Apr. 2026. [Online]. Available: https://www.st.com/en/power-management/stpmic1l.html

[21] STMicroelectronics, "AN6312: How to use STPMIC1L to develop a wall adapter powered application on STM32MP13 MPUs," Application Note, ver. 3.0, Apr. 2026. [Online]. Available: https://www.st.com/en/power-management/stpmic1l.html#documentation

[22] STMicroelectronics, "AN6421: STPMIC1L NVM configuration," Application Note, ver. 1.0, Dec. 2025. [Online]. Available: https://www.st.com/en/power-management/stpmic1l.html#documentation

[23] STMicroelectronics, "AN6429: STPMIC1L application hints," Application Note, ver. 1.0, Dec. 2025. [Online]. Available: https://www.st.com/en/power-management/stpmic1l.html#documentation

[24] STMicroelectronics, "AN6438: The STPMIC1L BOM details," Application Note, ver. 1.0, Dec. 2025. [Online]. Available: https://www.st.com/en/power-management/stpmic1l.html#documentation

[25] STMicroelectronics, "AN6384: The STPMIC1L PCB layout guidelines," Application Note, ver. 1.0, Oct. 2025. [Online]. Available: https://www.st.com/en/power-management/stpmic1l.html#documentation
