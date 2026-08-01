# Raspberry Pi Pico Arbitrary Waveform Generator (AWG)

<div align="center">

  ![Microcontroller](https://img.shields.io/badge/MCU-RP2040%20Dual--Core%20ARM%20Cortex--M0%2B-blue?style=for-the-badge&logo=raspberrypi)
  ![Firmware](https://img.shields.io/badge/Firmware-MicroPython%20%2B%20PIO-green?style=for-the-badge&logo=python)
  ![DAC Architecture](https://img.shields.io/badge/DAC-11--Bit%20R--2R%20Resistor%20Ladder-orange?style=for-the-badge)
  ![Waveforms](https://img.shields.io/badge/Waveforms-9%20Synthesized%20Geometries-purple?style=for-the-badge)
  ![License](https://img.shields.io/badge/License-All%20Rights%20Reserved-red?style=for-the-badge)

  <br>

  <h3>A High-Performance, Low-Cost Dual-Core Embedded Signal Generator Demonstrating RP2040 Hardware PIO State Machines, Multi-Bit R-2R Resistor Ladder DAC Synthesis, and Event-Driven LCD User Interfaces.</h3>

  <br>

  <img src="media/prototype_photo.jpeg" width="800" alt="Raspberry Pi Pico Arbitrary Waveform Generator Prototype Build">
  <br>
  <sub><b>Figure 1:</b> Assembled hardware prototype on bench setup showing Raspberry Pi Pico (RP2040), 16x2 I2C LCD, tactile buttons, and 11-bit R-2R DAC array.</sub>

</div>

---

## 📋 Table of Contents
- [Executive Summary](#-executive-summary)
- [System Architecture](#-system-architecture)
- [Key Features](#-key-features)
- [Hardware Architecture & Electronics](#-hardware-architecture--electronics)
  - [GPIO Allocation Table](#gpio-allocation-table)
  - [R-2R DAC Circuit Theory](#r-2r-dac-circuit-theory)
  - [Bill of Materials (BOM)](#bill-of-materials-bom)
- [Firmware & Software Architecture](#-firmware--software-architecture)
  - [Dual-Mode Signal Generation Pipeline](#dual-mode-signal-generation-pipeline)
  - [On-Device User Interface](#on-device-user-interface)
- [Testing & Signal Validation](#-testing--signal-validation)
- [Engineering Lessons & Architectural Insights](#-engineering-lessons--architectural-insights)
- [Applications](#-applications)
- [Future System Roadmap](#-future-system-roadmap)
- [Repository Structure Overview](#-repository-structure-overview)
- [Showcase License & Rights Statement](#-showcase-license--rights-statement)

---

## 📌 Executive Summary

Commercial arbitrary waveform generators (AWGs) typically range from hundreds to thousands of dollars. This engineering project explores building a versatile, desk-friendly signal generator around the **Raspberry Pi Pico (RP2040)** microcontroller for sub-$15. 

The instrument combines **hardware-accelerated Programmable I/O (PIO)** for high-frequency clock/pulse generation with a **11-bit parallel R-2R resistor-ladder DAC** for multi-bit arbitrary analog waveform streaming. User control is provided via an event-driven 16x2 LCD interface and debounced tactile navigation push-buttons.

<div align="center">
  <img src="media/Clean%20overview%20photo.png" width="750" alt="Clean overview of assembled AWG prototype">
  <br>
  <sub><b>Figure 2:</b> Bench overview of the assembled functional prototype.</sub>
</div>

---

## 📐 System Architecture

The AWG system separates digital control logic, high-speed waveform synthesis execution, and analog signal reconstruction into decoupled functional layers:

<div align="center">
  <img src="media/Block%20diagram%201.png" width="820" alt="System Block Diagram">
  <br>
  <sub><b>Figure 3:</b> High-level hardware and firmware block diagram.</sub>
</div>

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                            USER INTERFACE LAYER                             │
│   [ 4x Tactile Push Buttons ]  ────────►  [ 16x2 Character LCD Display ]    │
│   (UP / DOWN / OK / BACK)                 (PCF8574 I2C Adapter @ 0x27)      │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ Event-Driven State Navigation
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RP2040 DUAL-CORE MICROCONTROLLER                         │
│   ┌───────────────────────────────────┬─────────────────────────────────┐   │
│   │ Core 0: Control Loop & UI Engine  │ Core 1: DAC Streaming / Timer   │   │
│   └───────────────────────────────────┴─────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ PIO State Machine 0: High-Speed Square Wave & Pulse Synthesizer     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ Parallel 11-Bit Digital Bus (GP0-GP10)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ANALOG OUTPUT CONDITIONING                              │
│   [ Parallel GPIO Bus (GP0-10) ] ───► [ 11-Bit R-2R Resistor Ladder DAC ]   │
│                                                   │                         │
│                                                   ▼                         │
│                                      [ Analog Output Terminal / BNC ]       │
└──────────────────────────────────────┬──────────────────────────────────────┘
```

---

## ✨ Key Features

<div align="center">
  <img src="media/Features.png" width="750" alt="Core System Features Overview">
  <br>
  <sub><b>Figure 4:</b> Core system features and functional highlights.</sub>
</div>

- **9 Synthesized Waveform Geometries:** Supports Sine, Square, Pulse, Triangle, Sinc, Gaussian, Exponential, Noise, and DC output.
- **Dual-Mode Signal Generation Engine:**
  - *PIO State Machine Mode:* High-speed pulse/square generation offloaded entirely to RP2040 PIO state machines.
  - *Timer DMA/CPU Mode:* Multi-bit arbitrary analog signal streaming via lookup tables feeding the parallel DAC.
- **On-Device Parameter Control:** Real-time adjustment of waveform geometry, output frequency, step resolution, amplitude scaling, DC offset, phase inversion, and pulse rise/fall times.
- **Menu-Driven Character LCD:** 16x2 LCD visual feedback powered by an I2C expander for minimal GPIO pin footprint.
- **Debounced Tactile Navigation:** 4-button menu state machine handling navigation (`UP`, `DOWN`, `OK/Menu`, `BACK`).

---

## ⚡ Hardware Architecture & Electronics

### GPIO Allocation Table

The RP2040 GPIO pins are allocated to maximize hardware parallel bus throughput for the DAC while minimizing pin count for the UI:

| Pin Range | Peripheral Interface | Signal Type | Function / Description |
|:---|:---|:---|:---|
| **GP0 – GP10** | 11-Bit R-2R DAC Array | Parallel Digital Out | Bit 0 (LSB) through Bit 10 (MSB) binary-weighted outputs |
| **GP18** | Push Button 1 (`UP`) | Digital Input (Pull-Up) | Increment frequency / parameter menu item |
| **GP19** | Push Button 2 (`OK/MENU`) | Digital Input (Pull-Up) | Confirm selection / cycle edit parameter |
| **GP20** | Push Button 3 (`DOWN`) | Digital Input (Pull-Up) | Decrement frequency / parameter menu item |
| **GP21** | Push Button 4 (`BACK`) | Digital Input (Pull-Up) | Return to main menu / cancel parameter edit |
| **GP26** | I2C0 SDA | Open-Drain / Digital | Serial Data line for 16x2 LCD PCF8574 expander |
| **GP27** | I2C0 SCL | Open-Drain / Digital | Serial Clock line for 16x2 LCD PCF8574 expander |

---

### R-2R DAC Circuit Theory

The analog signal synthesis relies on an **11-bit R-2R resistor ladder DAC**. The circuit converts parallel digital GPIO logic levels ($V_{DD} = 3.3\text{V}$) into discrete analog voltage levels according to the ideal binary-weighted transfer function:

$$V_{\text{out}} = V_{\text{ref}} \times \sum_{i=0}^{10} \left( D_i \times 2^{i-11} \right) = V_{\text{ref}} \times \frac{\text{DAC}_{10}}{2048}$$

Where:
- $V_{\text{ref}} = 3.3\text{V}$ (Pico GPIO output voltage)
- $D_i \in \{0, 1\}$ represents the digital logic state of GPIO pin $i$
- $\text{DAC}_{10} \in [0, 2047]$ is the 11-bit integer value driven onto `GP0`–`GP10`

---

### Bill of Materials (BOM)

| Component | Quantity | Form Factor | Primary Engineering Role |
|:---|:---:|:---:|:---|
| **Raspberry Pi Pico** | 1 | Module (DIP-40) | Dual-core ARM Cortex-M0+ microcontroller @ 133 MHz |
| **16x2 Character LCD** | 1 | HD44780 | Main user interface display screen |
| **PCF8574 I2C Expander** | 1 | Backpack Board | Converts parallel LCD interface to 2-wire I2C (`0x27`) |
| **1kΩ Resistors** | 22 | Through-Hole (1/4W) | R-2R resistor ladder DAC array ($R = 1\text{k}\Omega, 2R = 2\text{k}\Omega$) |
| **Tactile Push Buttons** | 4 | 6mm Momentary | Menu navigation input switches |
| **BNC / Terminal Posts** | 1 | Terminal Post | Analog waveform output connection |
| **Breadboard / Wire** | 1 | Prototyping Board | Hardware prototyping interconnect bus |

---

## 💻 Firmware & Software Architecture

### Dual-Mode Signal Generation Pipeline

The firmware utilizes a dual-path execution pipeline to balance high-frequency digital clock output with multi-bit arbitrary analog synthesis:

```text
                        ┌────────────────────────┐
                        │   Waveform Selection   │
                        └───────────┬────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
       [ Square / Pulse Mode ]           [ Arbitrary Analog Mode ]
                    │                               │
                    ▼                               ▼
      ┌───────────────────────────┐   ┌───────────────────────────┐
      │ RP2040 PIO State Machine  │   │ 256-Sample Lookup Tables  │
      │ Hardware Pulse Generator  │   │ (Sine, Sinc, Gaussian...) │
      └─────────────┬─────────────┘   └─────────────┬─────────────┘
                    │                               │
                    ▼                               ▼
       [ High-Frequency Clock ]         [ Parallel Bus Stream (GP0-10) ]
                    │                               │
                    ▼                               ▼
         [ Output Terminal ]             [ 11-Bit R-2R DAC Ladder ]
```

---

### On-Device User Interface

Visual parameter management is executed via the 16x2 character LCD and a 4-button debounced menu state machine:

<div align="center">
  <img src="media/Screen.png" width="700" alt="16x2 LCD Menu Interface Screen">
  <br>
  <sub><b>Figure 5:</b> On-device 16x2 LCD screen showing waveform geometry and active frequency parameters.</sub>
</div>

---

## 📊 Technical Specifications & Design Targets

> ℹ️ **Design Target Notice:** Specifications listed below represent architectural design targets established during hardware modeling and prototyping.

| Technical Parameter | Target Specification | Implementation Notes |
|:---|:---|:---|
| **Microcontroller Silicon** | RP2040 (Dual ARM Cortex-M0+ @ 133 MHz) | Raspberry Pi Foundation Silicon |
| **Waveform Geometries** | 9 Types | Sine, Square, Pulse, Triangle, Sinc, Gaussian, Exp, Noise, DC |
| **Target Frequency (Square Wave)** | 1 Hz to 10 MHz | High-speed clock generation via RP2040 PIO State Machine |
| **Target Frequency (Arbitrary Waves)** | ~1 Hz – 300 Hz *(Estimated, unverified)* | CPU/Timer-paced sample table streaming via R-2R DAC |
| **Digital-to-Analog Resolution** | 11-Bit Parallel DAC ($2^{11} = 2048$ steps) | R-2R Resistor Ladder Network across `GP0`–`GP10` |
| **User Interface Display** | 16x2 Character LCD via I2C (`0x27`) | Event-driven UI update loop (~10 Hz update rate) |
| **System Power Input** | USB 5V Bus Power | Regulated to 3.3V on Pico board |

---

## 🧪 Testing & Signal Validation

<div align="center">
  <img src="media/Real%20test%202.png" width="650" alt="Oscilloscope Signal Validation Capture">
  <br>
  <sub><b>Figure 6:</b> Live signal output validation on a Digital Storage Oscilloscope (DSO).</sub>
</div>

Empirical oscilloscope captures, per-waveform validation records, and measurement notes are compiled in the **[Complete AWG Waveform Validation Report](Test%20Result/Complete_AWG_Waveform_Validation_Report.pdf)**. The raw validation documents are stored within the [`Test Result/`](Test%20Result) folder.

> 🛠️ **Documentation Status Note:** Output linearity, total harmonic distortion (THD), and Signal-to-Noise Ratio (SNR) audits are currently being compiled for future publication.

---

## 💡 Engineering Lessons & Architectural Insights

1. **Offloading Timing to PIO:** Executing high-frequency clock generation inside standard CPU software loops causes output timing jitter whenever display update interrupts fire. Offloading pulse synthesis to dedicated RP2040 PIO state machines guarantees jitter-free signal timing regardless of main CPU load.
2. **R-2R Ladder Topology Requirements:** Binary-weighted digital-to-analog conversion requires strict 1:2 resistor value ratioing ($R$ and $2R$ values, such as $1\text{k}\Omega$ and $2\text{k}\Omega$). Using identical resistor values across all branches distorts voltage output steps regardless of resistor precision tolerances.

---

## 🔧 Applications

<div align="center">
  <img src="media/It%27s%20applications.png" width="750" alt="Example Applications of the AWG">
  <br>
  <sub><b>Figure 7:</b> Bench and educational use cases for an arbitrary waveform generator.</sub>
</div>

This instrument is designed for a range of bench testing, embedded development, and laboratory applications:
- **Analog Circuit & Filter Testing:** Characterizing frequency response, gain, and transient response.
- **Sensor & Transducer Emulation:** Simulating real-world sensor outputs for control system validation.
- **Educational Demonstrations:** Visualizing waveform mathematics, Fourier synthesis, and signal theory.
- **Embedded System Stimulus:** Supplying external clock, reference, and pulse signals during hardware debugging.

---

## 🚀 Future System Roadmap

- [ ] **Active Op-Amp Output Buffer:** Adding an operational amplifier buffer stage to lower output impedance and prevent signal attenuation under load.
- [ ] **Adjustable Gain & Offset Stage:** Integrating digital potentiometers for variable peak-to-peak amplitude and DC offset tuning.
- [ ] **Custom 2-Layer PCB Enclosure:** Transitioning from breadboard prototyping to a custom PCB and desktop instrument enclosure.

---

## 📁 Repository Structure Overview

```text
Arbitrary Waveform Generator - Public Showcase/
├── README.md                            # Primary engineering showcase documentation
├── LICENSE                              # Showcase rights statement
├── .gitignore                           # Public git ignore rules
├── media/                               # Image & graphic visual assets
│   ├── prototype_photo.jpeg             # Bench hardware prototype photo
│   ├── Clean overview photo.png         # Assembled build overview graphic
│   ├── Features.png                     # Core feature highlight banner
│   ├── Block diagram 1.png              # System block diagram graphic
│   ├── Screen.png                       # 16x2 LCD UI menu screenshot
│   ├── Real test 2.png                  # Oscilloscope validation capture
│   └── It's applications.png            # Applications overview visual card
└── Test Result/                         # Technical validation documents
    └── Complete_AWG_Waveform_Validation_Report.pdf  # 37-page waveform validation report
```

---

## 📄 Showcase License & Rights Statement

This public showcase repository is published strictly for demonstration, architectural review, and engineering portfolio evaluation purposes. All rights reserved. Refer to the [LICENSE](LICENSE) file for complete details.
