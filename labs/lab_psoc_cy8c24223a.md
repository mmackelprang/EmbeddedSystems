# Lab: Programmable System-on-Chip Design with the Cypress CY8C24223A (PSoC 1)

**Course:** EENG 5XX — Advanced Embedded Systems Design
**Lab Number:** TBD
**Target Device:** Cypress (Infineon) **CY8C24223A-24PVXIT** — PSoC 1, 20-pin SSOP
**Estimated Duration:** Two 3-hour lab sessions

---

## 1. Header & Overview

### Abstract

The Cypress **CY8C24223A** is a member of the original **PSoC 1** family — a class of *Programmable System-on-Chip* devices that pair a modest 8-bit **M8C** Harvard-architecture core with a fabric of user-configurable **analog** and **digital** blocks. Unlike a traditional microcontroller whose peripherals (UART, ADC, timers, opamps) are fixed in silicon, PSoC 1 lets the engineer *route and instantiate* peripherals at design time using **PSoC Designer**, much like placing IP cores in a small FPGA. In this lab, students will design, breadboard, and program a complete mixed-signal embedded system around the CY8C24223A. You will configure digital blocks to implement timers, PWMs, and a UART; configure analog blocks to instantiate a delta-sigma ADC and a DAC; and write M8C C firmware that ties these soft peripherals to physical I/O. By the end of the lab you will appreciate both the unique flexibility of PSoC 1 and the design discipline (decoupling, layout, configuration management) it demands.

### Learning Objectives

By the end of this lab the student will be able to:

1. Navigate the **PSoC Designer IDE**, place user modules in analog/digital block slots, and route them to physical pins.
2. Describe the **M8C CPU** architecture (Harvard, 8-bit, 24 MHz max) and its instruction-set and memory-map implications for embedded software.
3. Distinguish between **switched-capacitor (SC)** and **continuous-time (CT)** PSoC analog blocks and select the appropriate type for a given signal-conditioning task.
4. Configure on-chip mixed-signal peripherals — **Delta-Sigma ADC, DAC, PWM, Timer, UART** — entirely in firmware/configuration without external silicon.
5. Apply correct **GPIO drive-mode** selection (strong, open-drain, pull-up, high-Z) for digital interfacing.
6. Practice professional **mixed-signal breadboarding** discipline: decoupling, ground topology, VREF treatment, and ESD-safe SMD handling.
7. Use the **MiniProg** programmer with the **ISSP** protocol to flash and debug a target board.

---

## 2. Background: Electronics Discussion

### 2.1 The M8C CPU Core

The M8C is an **8-bit, Harvard-architecture** processor with separate program (Flash) and data (RAM) memory spaces. Key parameters for the CY8C24223A:

| Parameter | Value |
|---|---|
| Architecture | Harvard, 8-bit |
| Max CPU clock | **24 MHz** (internal main oscillator, ±2.5% accuracy at 5 V) |
| Flash program memory | **4 KB** |
| SRAM | **256 bytes** |
| Register space | 8-bit A (accumulator), X (index), SP, PC (16-bit), F (flags) |
| Instruction set | ~38 instructions: `MOV`, `ADD`, `SUB`, `AND`, `OR`, `XOR`, `CMP`, `JMP`, `CALL`, `RET`, `INDEX`, `ROMX`, etc. |
| Stack | RAM-resident, grows upward from `SP` |
| Interrupts | Vectored, with priority encoder; one vector per analog/digital block + GPIO/sleep/supply |

The M8C has **no hardware multiplier and no barrel shifter** — bit-banged math is expensive, which influences peripheral choice (offload to digital blocks!). Pointers are 8-bit into RAM and 16-bit into Flash (via `ROMX`). With only **256 B of RAM**, students must be deliberate about stack depth, buffer sizes, and `const` placement in Flash.

#### Memory Map (CY8C24223A)

```
Program (Flash):  0x0000 – 0x0FFF   (4 KB, includes interrupt vector table at 0x0000)
Data (SRAM):      0x00   – 0xFF     (256 B; page 0 only on this part)
I/O Registers:    Bank 0 / Bank 1, selected via XIO bit of CPU_F
```

### 2.2 PSoC 1 Programmable Architecture

The defining feature of PSoC 1 is that **peripherals are not fixed**. Instead, the silicon provides a small array of generic blocks:

- **Digital PSoC Blocks:** The CY8C24223A contains **4 digital blocks**, each an 8-bit datapath (counter, shift register, comparator, mux). Blocks are concatenated to form Timers, Counters, PWMs, UARTs (TX+RX = 2 blocks), SPI, CRC generators, and pseudo-random sequence generators. The configuration tool **PSoC Designer** abstracts this into drag-and-drop **User Modules**.
- **Analog PSoC Blocks:** The CY8C24223A contains **6 analog blocks** arranged as **2 continuous-time (CT)** and **4 switched-capacitor (SC)** blocks. These can be configured as opamps, programmable-gain amplifiers (PGA), comparators, instrumentation amplifiers, sample-and-hold, integrators, and the building blocks of **delta-sigma** and **incremental** ADCs and **6/8-bit DACs**.

Compared with a traditional fixed-peripheral MCU (e.g., an 8051 derivative or a small AVR), PSoC 1 trades raw CPU throughput for a *mixed-signal fabric*: you can synthesize the exact peripheral set your application requires, eliminating external opamps, comparators, and even simple ADC/DAC chips. The trade-off is that block count is fixed; if your design needs a UART + SPI + I²C + 2 PWMs you may exceed the 4 digital blocks and must time-multiplex or pick a larger PSoC part.

#### CT vs. SC Analog Blocks — Quick Reference

| Aspect | Continuous-Time (CT) | Switched-Capacitor (SC) |
|---|---|---|
| Signal path | Always-on opamp + resistor network | Clocked, sampled capacitor charge transfer |
| Bandwidth | Higher (limited by opamp GBW) | Limited by sample clock (kHz range typ.) |
| Typical use | Front-end gain, low-noise instrumentation amp, comparator | Integrators (ΔΣ-ADC), filters, DAC, sample-and-hold |
| Noise vs DC accuracy | Continuous noise, no aliasing | Low-offset (auto-zeroed), but introduces sampled noise |

### 2.3 On-Chip Analog Capabilities

The CY8C24223A supports the following user-configurable analog functions:

- **Programmable-gain amplifiers** (CT blocks) with software-selectable gain.
- **Comparators** with hysteresis (CT or SC).
- **Delta-Sigma ADC** up to ~14-bit resolution (resolution traded against sample rate).
- **Incremental ADC** for higher-rate, lower-resolution sampling.
- **6/8-bit voltage-output DACs** built from SC blocks.
- **Internal voltage reference (VREF)** brought out to a pin for bypass capacitor.

These functions are configured *graphically* in PSoC Designer; the tool generates initialization C functions (`ADCINC_Start()`, `PGA_SetGain()`, etc.) which the student calls from `main.c`.

### 2.4 Power, I/O, and Package Notes

| Item | Specification |
|---|---|
| Package | **20-pin SSOP** (0.635 mm / 25 mil pitch) — *not breadboard-compatible without adapter* |
| Operating voltage | **2.4 V – 5.25 V** (3.3 V or 5 V typical) |
| Operating temperature | **Industrial, −40 °C to +85 °C** |
| GPIO drive modes | Strong (push-pull), Open-Drain Hi, Open-Drain Lo, Pull-Up, Pull-Down, High-Z analog, High-Z digital, Strong Slow |
| GPIO current | Up to 20 mA sink/source per pin (with aggregate limits) |
| Programming interface | **ISSP** (In-System Serial Programming) — 5-wire: VDD, VSS, SDATA, SCLK, XRES |

GPIO drive modes are **per-pin** and selected in software — this is unusual compared with most MCUs and is essential to configure correctly (e.g., an I²C pin must be **Open-Drain Hi**, while a logic output should be **Strong**).

### 2.5 CY8C24223A 20-pin SSOP Pinout

The following pinout table reflects the **20-pin SSOP (PVXIT)** package. *Always cross-check the latest Infineon/Cypress datasheet before pin-critical design.*

| Pin # | Name | Type | Primary Function | Notes / Alternate Functions |
|------:|------|------|------------------|------------------------------|
| 1  | P0[7] | I/O   | GPIO Port 0 bit 7 | Analog mux input, ADC channel |
| 2  | P0[5] | I/O   | GPIO Port 0 bit 5 | Analog input to column 1 |
| 3  | P0[3] | I/O   | GPIO Port 0 bit 3 | Analog input to column 0 / AGND ref option |
| 4  | P0[1] | I/O   | GPIO Port 0 bit 1 | Analog column 0 output / external VREF option |
| 5  | P2[7] | I/O   | GPIO Port 2 bit 7 | Digital block I/O |
| 6  | P2[5] | I/O   | GPIO Port 2 bit 5 | Digital block I/O |
| 7  | P2[3] | I/O   | GPIO Port 2 bit 3 | External AGND reference |
| 8  | P2[1] | I/O   | GPIO Port 2 bit 1 | External VREF input (bypass cap recommended) |
| 9  | SMP   | Out   | Switch-Mode Pump | Optional boost output (leave open if unused) |
| 10 | P1[7] | I/O   | GPIO Port 1 bit 7 | **ISSP SCLK** during programming |
| 11 | P1[5] | I/O   | GPIO Port 1 bit 5 | I²C SCL (digital block) |
| 12 | P1[3] | I/O   | GPIO Port 1 bit 3 | Digital block I/O |
| 13 | P1[1] | I/O   | GPIO Port 1 bit 1 | I²C SDA / **ISSP SCLK alt** |
| 14 | VSS   | Power | Ground (digital + analog) | Tie all VSS pins to a single low-impedance ground plane |
| 15 | P1[0] | I/O   | GPIO Port 1 bit 0 | I²C SDA alt / **ISSP SDATA** |
| 16 | P1[2] | I/O   | GPIO Port 1 bit 2 | Digital block I/O |
| 17 | P1[4] | I/O   | GPIO Port 1 bit 4 | Optional external 32.768 kHz crystal in |
| 18 | P1[6] | I/O   | GPIO Port 1 bit 6 | Optional external crystal out |
| 19 | XRES  | In    | Active-high external reset | **Required for ISSP programming**; pull low (with weak pulldown) in normal operation |
| 20 | VDD   | Power | +2.4 V to +5.25 V supply | **100 nF + 10 µF decoupling required** |

> **💡 Tip:** The CY8C24223A has only a single VDD/VSS pair, but a *clean analog ground* is achieved by careful star-grounding on the breadboard, not by separate pins. Treat the area around pin 14 (VSS) and pin 20 (VDD) as your highest-priority bypass node.

---

## 3. Components Needed

| Qty | Component | Why Needed |
|----:|-----------|------------|
| 1 | **Cypress CY8C24223A-24PVXIT** (20-pin SSOP) | The target PSoC 1 device under study; provides the M8C core and the configurable analog/digital fabric used in every exercise. |
| 1 | **Cypress MiniProg1 or MiniProg3** programmer | Drives the ISSP protocol (SDATA/SCLK/XRES) needed to flash firmware and supply target VDD; without it the part cannot be programmed in-circuit. |
| 1 | **20-pin SSOP-to-DIP breakout adapter** (0.65 mm pitch) | The SSOP package has 0.635 mm pin pitch and **cannot** be inserted directly into a 2.54 mm breadboard; the breakout converts to DIP for prototyping. |
| 1 | **830-point full-size breadboard** | Provides power rails plus enough tie-points for the PSoC adapter, MiniProg header, peripherals, and analog/digital separation. |
| 4 | **100 nF (0.1 µF) ceramic capacitors** (X7R, ≥10 V) | High-frequency bypass on VDD and VREF; place within ~5 mm of the pins. Critical because the SC analog blocks switch at MHz rates and inject noise on the supply if poorly decoupled. |
| 1 | **10 µF electrolytic or tantalum capacitor** (≥10 V) | Bulk decoupling for the VDD rail, handling lower-frequency current transients from the M8C core and digital blocks. |
| 1 | **3.3 V or 5 V regulated supply** (bench supply or USB power module) | Provides the operating rail. 5 V gives the widest analog dynamic range; 3.3 V is closer to modern system levels. *MiniProg3 can supply 5 V or 3.3 V directly during development.* |
| 4 | **220 Ω resistors** (¼ W, 5 %) | Current-limiting for LEDs; sized so that ((VDD − Vf)/R) ≈ 10–15 mA per LED, safely within GPIO sink/source limits. |
| 2 | **10 kΩ resistors** | Pull-ups for the push button input and (if I²C is used) the SDA/SCL lines. |
| 2 | **LEDs**, two colors (e.g., red + green), 3 mm or 5 mm | Visible indicators for GPIO output and PWM brightness — used in Exercises 1 and 2. |
| 1 | **Tactile push-button switch** (SPST momentary) | Digital input for testing GPIO drive modes, debouncing logic, and interrupt-on-change. |
| 1 | **10 kΩ linear potentiometer** | Variable analog input source for the Delta-Sigma ADC in Exercise 2; a known, hand-adjustable voltage between VSS and VDD. |
| 1 | **Small 8 Ω speaker or piezo buzzer** | Audible verification of the PWM output and of the DAC waveform generator in Exercise 4. |
| 1 | **1 µF ceramic capacitor** (for VREF bypass) | Stabilizes the internal voltage reference at pin 8 (P2[1]); poor VREF decoupling directly degrades ADC ENOB and DAC linearity. |
| ~25 | **Solid-core 22 AWG jumper wires** | Point-to-point wiring for power, signals, and the ISSP header. |
| 1 | **Logic analyzer or 2-channel oscilloscope** *(optional but strongly recommended)* | Required to verify UART framing (Exercise 3) and the DAC waveform (Exercise 4); a Saleae-class 24 MHz logic analyzer is sufficient for digital signals. |
| 1 | **Antistatic wrist strap + ESD mat** | CMOS analog blocks and fine-pitch SSOP pins are highly ESD-sensitive during soldering and handling. |

---

## 4. Hardware Setup: Breadboarding Instructions

### 4.1 SSOP-to-DIP Adapter Preparation

The CY8C24223A-24PVXIT comes in a **20-pin SSOP** with **0.635 mm (25 mil) pin pitch** — roughly one-quarter of a breadboard's 2.54 mm pitch. **Do not attempt to bridge SSOP pins onto a breadboard with wire wrap.** You must first solder the part to a 20-pin SSOP-to-DIP breakout (e.g., a SchmartBoard or generic 0.65 mm SSOP adapter).

> **⚠️ Warning — ESD:** Always wear a grounded wrist strap and work on a dissipative mat. PSoC 1 analog blocks include high-impedance opamp inputs that are particularly vulnerable to ESD damage that *may not* show up as an obvious "dead chip" — instead you get degraded offset/noise on analog blocks.

> **⚠️ Warning — Soldering:** Use a fine (0.4 mm or smaller) chisel tip, **flux paste**, and the *drag-solder* technique. Inspect all 20 pins under magnification for bridges before applying power. A bridged pair on the supply (pins 14/15 or 19/20 region) can damage the part instantly.

> **💡 Tip:** Pre-tin **pin 1** (P0[7]) only, align the chip carefully (confirm the dot/notch matches the silkscreen), reflow pin 1 to tack the part, then verify alignment of pin 11 (opposite corner) before continuing.

After soldering, use a continuity beep test from VDD (pin 20) to VSS (pin 14) — **must read open**, not a short.

### 4.2 Power Rail Setup

1. Connect the breadboard's red rail to your 5 V (or 3.3 V) supply and the blue rail to GND. Wire **both** sides of the breadboard's rails together to give a low-impedance distribution.
2. Place the **10 µF bulk capacitor** between the red and blue rails near the supply entry point.
3. Place a **100 nF ceramic** between rows adjacent to **pin 20 (VDD)** and the GND rail, with leads as short as physically possible. Repeat for any second VDD pin if present (not on this 20-SSOP, but maintain the discipline for analog VREF).
4. **VREF treatment:** Place a **1 µF ceramic** from **pin 8 (P2[1] / VREF)** to GND. Do *not* drive this pin externally unless you explicitly want to override the internal reference and have configured PSoC Designer accordingly.

### 4.3 ISSP Programming Header

Wire a 5-pin header (or 5 dedicated jumpers) to the MiniProg connector:

| MiniProg signal | PSoC pin | Notes |
|---|---|---|
| **VDD**   | Pin 20 (VDD) | MiniProg can source target VDD during dev. |
| **GND**   | Pin 14 (VSS) | Star ground at this point. |
| **XRES**  | Pin 19 (XRES) | Active-high reset. **Do not** add a pull-up here; PSoC Designer expects to drive this line. A weak (100 kΩ) pulldown is acceptable for noise immunity. |
| **SDATA** | Pin 15 (P1[0]) | Serial data; do not load with capacitance > a few pF. |
| **SCLK**  | Pin 10 (P1[7]) | Serial clock from MiniProg. |

> **💡 Tip:** Keep ISSP wires under 10 cm and route them away from the LED / speaker signals — programming can fail under "Power Cycle" mode if SDATA sees crosstalk during the brief programming-entry window.

### 4.4 GPIO Connections

- **LED1 (red, "heartbeat"):** Pin 5 (P2[7]) → 220 Ω resistor → LED anode; LED cathode → GND. Drive mode: **Strong**.
- **LED2 (green, "PWM"):** Pin 6 (P2[5]) → 220 Ω resistor → LED anode; LED cathode → GND. Drive mode: **Strong**.
- **Push button (PB1):** Pin 11 (P1[5]) → one switch terminal; the other terminal to GND. Also place a **10 kΩ** from pin 11 to VDD (external pull-up), **or** configure the pin as internal **Pull-Up** in PSoC Designer and omit the external resistor.
- **Potentiometer (POT1):** Wire ends to VDD and GND. Wiper → pin 2 (P0[5]). Configure pin 2 as **High-Z analog** for ADC routing.
- **Speaker/Buzzer:** Pin 12 (P1[3]) → series 100 Ω resistor → piezo (+); piezo (−) → GND. Drive mode: **Strong**. (For a coil speaker, add a series capacitor to block DC.)

### 4.5 Textual Wiring Diagram (Node-by-Node)

```
+5V ──┬── [10 µF] ── GND
      │
      ├── [100 nF] ── GND           (close to U1 pin 20)
      │
      ├── U1.pin20 (VDD)
      │
      └── POT1.HI ─── (wiper → U1.pin2) ─── POT1.LO ── GND

GND ──┬── U1.pin14 (VSS)
      │
      ├── [1 µF] ── U1.pin8 (VREF)
      │
      ├── LED1.cath
      ├── LED2.cath
      ├── PB1.term2
      └── SPK1.−

U1.pin5  (P2[7]) ── [220 Ω] ── LED1.anode
U1.pin6  (P2[5]) ── [220 Ω] ── LED2.anode
U1.pin11 (P1[5]) ──┬── PB1.term1
                   └── [10 kΩ] ── +5V
U1.pin12 (P1[3]) ── [100 Ω] ── SPK1.+

ISSP header (to MiniProg):
   Hdr.VDD   ── U1.pin20
   Hdr.GND   ── U1.pin14
   Hdr.XRES  ── U1.pin19
   Hdr.SDATA ── U1.pin15 (P1[0])
   Hdr.SCLK  ── U1.pin10 (P1[7])
```

### 4.6 Pre-Power Sanity Checklist

> **⚠️ Warning:** Complete *every* item before connecting the MiniProg or applying VDD.

- [ ] **DMM continuity from VDD rail to GND rail = OPEN** (no short).
- [ ] DMM continuity from U1.pin20 to VDD rail = closed.
- [ ] DMM continuity from U1.pin14 to GND rail = closed.
- [ ] 100 nF decoupling cap is within ~5 mm of pins 14/20.
- [ ] 1 µF VREF cap installed on pin 8.
- [ ] XRES (pin 19) is not tied directly to VDD or GND with a hard wire.
- [ ] LED polarity correct (long lead = anode, to resistor).
- [ ] Pot wiper wired to pin 2 (not VDD or GND swapped).
- [ ] No solder bridges visible on SSOP adapter under magnification.
- [ ] Supply set to **5.0 V ± 0.1 V** (or 3.3 V if that is your chosen rail) **before** powering on.

---

## 5. Lab Exercises

Each exercise builds on the prior one. Save each as a separate **PSoC Designer project** (`lab1_blink`, `lab2_adc_pwm`, `lab3_uart`, `lab4_dac`).

### Exercise 1 — Interrupt-Driven LED Blink (Timer User Module)

**Objective:** Blink LED1 at exactly **2 Hz** (250 ms ON, 250 ms OFF) using a hardware **Timer** PSoC block and its terminal-count ISR. **No software `delay()` loops are permitted.**

**Background:** The Timer user module occupies one digital PSoC block configured as an 8- or 16-bit down-counter. When the count reaches zero it asserts a *Terminal Count (TC)* interrupt and reloads. Using the ISR for the toggle decouples LED timing from any `main()` workload — fundamental to all real embedded firmware.

**Procedure:**

1. In PSoC Designer, create a new project targeting `CY8C24223A`, 24 MHz SysClk, 5 V.
2. Place a **Timer16** user module in `DBB00`+`DBB01` (chains two 8-bit digital blocks).
3. Configure clock source = `VC3` (e.g., 24 MHz / 16 / 256 ≈ 5.86 kHz). Set the Timer16 **Period Register (reload value)** to `1464` — i.e. `(5860 / 4) − 1` — so the terminal-count fires at 4 Hz, giving a 2 Hz LED toggle.
4. Route the Timer16 interrupt to the global ISR table.
5. In `Device Editor`, set **P2[7]** drive mode to **Strong**.
6. In `main.c`, enable the Timer interrupt and global interrupts, then sleep `main` in a `while(1){}`.
7. In `timer16int.asm` or your C ISR, toggle the LED:

```c
#include <m8c.h>
#include "PSoCAPI.h"

#pragma interrupt_handler Timer16_ISR
void Timer16_ISR(void) {
    PRT2DR ^= 0x80;   // toggle P2[7]
}

void main(void) {
    M8C_EnableGInt;
    Timer16_EnableInt();
    Timer16_Start();
    while (1) {
        // Idle — all work in ISR
    }
}
```

**Expected Output:** LED1 toggles at a visually steady 2 Hz. Verified with a stopwatch: 10 toggles in 5.00 s ± 0.1 s.

**Discussion Questions:**
1. Why is `PRT2DR ^= 0x80` not strictly atomic on the M8C, and when could that matter?
2. What happens if you forget `M8C_EnableGInt`? What if you forget `Timer16_EnableInt()` but call `Timer16_Start()`?

---

### Exercise 2 — Potentiometer → ADC → PWM Brightness

**Objective:** Read POT1 with the **Delta-Sigma ADC (ADCINC)** user module and use its output to set the duty cycle of an 8-bit **PWM** driving LED2. Above a threshold (e.g., 75 % of full scale), also light LED1 as a "high" indicator.

**Background:** This exercise demonstrates the *mixed-signal flow* unique to PSoC 1: the analog signal enters via a high-Z analog GPIO, is routed through the analog mux into an SC-block-based ADC, and the ADC result is consumed in software, which then loads a PWM digital-block compare register — all on one chip, no external opamps or DACs.

**Procedure:**

1. Open the Lab 1 project, *Save As* `lab2_adc_pwm`.
2. Place an **ADCINC** user module (uses 2 SC analog blocks). Resolution = 8-bit, sample rate ≈ 1 kS/s. Input = `AnalogColumn_InputSelect_0`.
3. Route **P0[5]** (pin 2) into Analog Column 0 via the Analog Column Input MUX.
4. Place a **PWM8** user module in a free digital block. Period = 255, initial compare = 0, clock = `VC2`. Route its output to **Row_0_Output_0 → P2[5]** (pin 6).
5. In `main.c`:

```c
#include <m8c.h>
#include "PSoCAPI.h"

void main(void) {
    int adc_val;

    M8C_EnableGInt;
    PWM8_Start();
    ADCINC_Start(ADCINC_HIGHPOWER);
    ADCINC_GetSamples(0);          // continuous

    while (1) {
        while (ADCINC_fIsDataAvailable() == 0) { /* wait */ }
        adc_val = ADCINC_iGetData();
        ADCINC_ClearFlag();

        if (adc_val < 0)   adc_val = 0;
        if (adc_val > 255) adc_val = 255;

        PWM8_WritePulseWidth((BYTE)adc_val);

        if (adc_val > 191) PRT2DR |=  0x80;   // LED1 ON
        else               PRT2DR &= ~0x80;   // LED1 OFF
    }
}
```

**Expected Output:** Rotating POT1 smoothly changes LED2 brightness from off to full. Past roughly the 3-o'clock pot position, LED1 illuminates.

**Discussion Questions:**
1. Why does the ADC's *effective* number of bits (ENOB) drop if you remove the VREF capacitor?
2. PWM dimming exploits the eye's persistence of vision. What minimum PWM frequency avoids visible flicker, and how does that constrain the PWM clock divider?

---

### Exercise 3 — UART Telemetry of ADC Readings

**Objective:** Configure a **UART** user module (TX-only is acceptable) at **9600 baud, 8-N-1** and stream the ADC reading every 100 ms to a host PC terminal (PuTTY, Tera Term, or `screen`).

**Background:** The UART user module consumes **two digital blocks** (transmitter + receiver) — or one if TX-only — and a 9600-baud byte clock derived from a clock-divider chain. This exercise illustrates the PSoC 1 budgeting issue: with ADCINC, PWM, and Timer16 already instantiated, the engineer must verify that enough analog/digital blocks remain.

**Procedure:**

1. Save As `lab3_uart`. Confirm at least 1 digital block is free; if not, replace `Timer16` with `Timer8` to free a block, and rerun a periodic interrupt at ~10 Hz.
2. Place a **UART** (or **TX8**) user module. Clock = derived from `VC3` to yield 9600 × 8 = 76.8 kHz. Verify the achievable baud-rate error is < 2 %.
3. Route TX output to **P1[2]** (pin 16). Drive mode = **Strong**.
4. Wire pin 16 to the MiniProg3's UART RX (or to a USB-UART bridge such as a CP2102), and connect the bridge GND to the breadboard GND. **Do not** connect the bridge's 5 V to your already-powered rail.
5. Firmware:

```c
#include <m8c.h>
#include "PSoCAPI.h"
#include <stdio.h>

void main(void) {
    int adc_val;
    char buf[16];

    M8C_EnableGInt;
    UART_Start(UART_PARITY_NONE);
    UART_CmdReset();
    ADCINC_Start(ADCINC_HIGHPOWER);
    ADCINC_GetSamples(0);

    while (1) {
        while (!ADCINC_fIsDataAvailable()) { ; }
        adc_val = ADCINC_iGetData();
        ADCINC_ClearFlag();

        csprintf(buf, "ADC=%d\r\n", adc_val);
        UART_CPutString(buf);

        // crude pacing: ~100 ms via Timer8 flag — replace busy-wait
    }
}
```

6. On the host: open the COM port at 9600-8-N-1 and observe `ADC=xxx` lines.

**Expected Output:** A scrolling list of ADC values in the terminal that changes as POT1 is rotated.

**Discussion Questions:**
1. The M8C lacks a hardware divider. What does `csprintf` cost in CPU cycles, and what would a fixed-format integer-to-ASCII routine save?
2. Why must the UART TX pin's drive mode be **Strong** but the RX pin (if used) be **High-Z digital**?

---

### Exercise 4 — Capstone: Programmable Waveform Generator

**Objective:** Build a **configurable-frequency sawtooth (or stepped sine) generator** using the **6-bit DAC** user module driven by a **Timer** ISR. The frequency must be user-selectable (e.g., 100 Hz, 250 Hz, 500 Hz, 1 kHz) by short-pressing PB1, cycling through a list.

**Background:** The DAC is built from an SC block and produces a voltage-mode output on an analog column pin. Driving it from a timer ISR with a precomputed lookup table is the simplest way to bound the CPU load. With a 64-step sawtooth and a 100 Hz target, the ISR must fire at 6.4 kHz — well within the M8C's capability at 24 MHz.

**Procedure:**

1. Save As `lab4_waveform`.
2. Place a **DAC6** user module on an SC block. Route its output to **Analog Column 0 → P0[3]** (pin 3). (Verify in PSoC Designer's analog routing view.)
3. Place a **Timer8** to fire at *N × f_out* where *N* = 64 (samples per period). Configure the ISR to step a global index and write the next LUT value:

```c
#include <m8c.h>
#include "PSoCAPI.h"

static const signed char sine64[64] = {
    /* Generate offline, e.g. in Python:
       [int(round(31 * math.sin(2*math.pi*i/64))) for i in range(64)]
       Example first 8 values: 0, 3, 6, 9, 12, 15, 18, 21, ... */
      0,   3,   6,   9,  12,  15,  18,  21,
     24,  26,  28,  29,  30,  31,  31,  31,
     31,  31,  31,  31,  30,  29,  28,  26,
     24,  21,  18,  15,  12,   9,   6,   3,
      0,  -3,  -6,  -9, -12, -15, -18, -21,
    -24, -26, -28, -29, -30, -31, -31, -31,
    -31, -31, -31, -31, -30, -29, -28, -26,
    -24, -21, -18, -15, -12,  -9,  -6,  -3
};
static unsigned char idx = 0;
static unsigned char waveform_mode = 0;  // 0 = sawtooth, 1 = sine

#pragma interrupt_handler DAC_Tick_ISR
void DAC_Tick_ISR(void) {
    signed char s;
    if (waveform_mode == 0) {
        s = (signed char)(idx - 32);            // sawtooth, ±31
    } else {
        s = sine64[idx & 0x3F];                 // sine LUT
    }
    DAC6_WriteStall(s);
    idx = (idx + 1) & 0x3F;
}
```

4. In `main()`, poll PB1 (with software debounce ≥ 20 ms). On each press, advance through `{100, 250, 500, 1000} Hz` by reloading `Timer8` period.
5. Probe pin 3 with an oscilloscope; trigger on the sawtooth's rising edge.

**Expected Output:** A clean 64-step sawtooth or stepped sine at the selected frequency. Quantization steps are visible on the scope; a low-pass RC filter on the output (e.g., 1 kΩ + 100 nF) will smooth it for audio listening via the buzzer.

**Discussion Questions:**
1. The DAC6 has 6-bit resolution (64 levels). What is the theoretical SNR, and how is it affected by the chosen output filter cutoff?
2. If you increased *N* (samples per period) to 256, what happens to ISR load? Is the M8C still capable of 1 kHz output? Show the math.

---

## 6. Discussion Questions (Lab-Wide)

Answer these in the lab report. Each response should be ~150–300 words and reference at least one figure/measurement from your work.

1. **PSoC fabric vs. external op-amps:** How does PSoC 1's configurable analog fabric differ from building the same gain/filter stage from a discrete op-amp + passives? Discuss trade-offs in noise, bandwidth, drift, board area, and bill of materials.
2. **Decoupling near analog blocks:** Why is decoupling especially critical near PSoC 1's analog blocks compared with a purely digital MCU? What did you observe (in ADC noise) when you temporarily removed the VREF capacitor?
3. **M8C vs. ARM Cortex-M:** What are the limitations of the M8C core relative to a modern Cortex-M0+ device? In what scenarios would PSoC 1 still be preferred over a Cortex-M-based PSoC 4 or PSoC 6?
4. **ISSP vs. JTAG:** How does ISSP differ from JTAG mechanically and in protocol? What security implications does each have (read-protect bits, glitch-attack surface)?
5. **Block budgeting:** You ran out of digital blocks in Exercise 3. Describe two strategies (other than picking a bigger PSoC) to fit the same functionality into 4 digital blocks.
6. **Drive modes:** Why does PSoC 1 expose so many GPIO drive modes? Give a specific example where the wrong drive mode causes a functional bug (not just sub-optimal power).
7. **Sampling theory in practice:** In Exercise 4, what is your effective DAC update rate, and does it satisfy Nyquist for the highest-frequency waveform you generate?

---

## 7. Submission Requirements & Grading Rubric

### Submission Format

Submit a single **`.zip`** archive named `LASTNAME_FIRSTNAME_PSoC_Lab.zip` containing:

1. **Lab report as PDF**, including:
   - Cover page (name, course, lab #, date).
   - For each exercise: schematic excerpt, PSoC Designer block diagram screenshot, annotated source listing, expected vs. measured output. **Oscilloscope or logic-analyzer captures are required for Exercises 1 (LED toggle timing), 2 (PWM waveform on LED2 pin), 3 (UART TX framing on P1[2]), and 4 (DAC output on P0[3]).**
   - Answers to **all** in-exercise *and* lab-wide discussion questions.
2. **PSoC Designer project archive** — one folder per exercise (`lab1_blink/`, `lab2_adc_pwm/`, `lab3_uart/`, `lab4_waveform/`). Each folder must contain the `.cmx`, `.app`, and `main.c` (or main `.asm`) such that the grader can re-open and re-build the project.
3. **Photograph** of your final breadboarded circuit (top-down, in focus).

### Grading Rubric

| Category | Weight | Criteria |
|---|---:|---|
| **Hardware Setup** | **20 %** | SSOP adapter cleanly soldered (no bridges, all 20 pins connected). Decoupling correctly placed within 5 mm of pins. VREF capacitor present. ISSP header correctly wired. Sanity-check list signed off. |
| **Exercise Completion** | **40 %** | All 4 exercises function correctly on the bench. Code compiles cleanly in PSoC Designer with no unaddressed warnings. PSoC block placement and routing match the documented design. |
| **Lab Report** | **30 %** | Clear schematic and block-diagram documentation. Oscilloscope/logic-analyzer captures included where required. Code annotated with purpose, not just restated. Honest discussion of bugs encountered. |
| **Discussion Questions** | **10 %** | All in-exercise and lab-wide questions answered. Answers cite measurements, datasheet sections, or PSoC Designer screenshots — not generic boilerplate. |

> **💡 Tip:** The grader uses the same MiniProg hardware you do. If a project does not flash and run on a freshly wired board, no Exercise Completion credit will be awarded for that exercise — **always re-verify after a clean checkout**.

---

*End of Lab Document.*
