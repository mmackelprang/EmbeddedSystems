# Lab: Microchip Atmel ATF16V8C High-Performance EE PLD

## Overview

This senior-level laboratory introduces the Atmel ATF16V8C, a high-performance electrically erasable programmable logic device (EE PLD) in the classic 20-pin architecture used for glue logic, state decoding, and compact combinational or registered logic design. The lab emphasizes hardware logic synthesis, fuse-map thinking, timing-aware design, and verification of programmable digital logic at the device level.[web:19][web:20]

Students will design, program, and verify a reusable control logic subsystem that combines combinational decoding, registered outputs, and mode-dependent behavior. The lab is structured to highlight the conceptual boundary between microcontroller firmware and programmable logic hardware.[web:19][web:20]

## Learning Objectives

By the end of this lab, students will be able to:

- Describe the role of the ATF16V8C as an erasable programmable logic device used for implementing compact digital logic functions.[web:19]
- Distinguish combinational logic implementation from registered logic implementation in a PLD.[web:19][web:20]
- Create logic equations or CUPL-style designs for decoding and control tasks.
- Program and test a PLD using a suitable programmer and verification process.
- Analyze propagation delay, product-term limitations, and output configuration choices.[web:19][web:20]
- Compare a hardware logic implementation with an equivalent software-driven implementation on a microcontroller.

## Required Equipment

- Atmel ATF16V8C-7JC device or equivalent supported part.
- PLD programmer supporting the target device.
- Breadboard or digital logic trainer.
- 5 V regulated power source.
- DIP switches or logic input source.
- LEDs with current-limiting resistors for outputs.
- Oscilloscope or logic analyzer.
- PLD development software supporting equations, fuse maps, or CUPL-like entry.

## Background

The ATF16V8C belongs to the family of programmable array logic and generic array logic style devices that allow compact implementation of digital logic without a full microcontroller. Such devices are commonly used for address decoding, bus control, state qualification, and timing-sensitive logic reduction.[web:19][web:20]

Unlike firmware, a PLD expresses behavior as hardware logic. That makes it attractive when a design requires predictable propagation characteristics, low overhead, and simple always-on decision logic. In this lab, students will implement both combinational and registered functions and compare them against a software-based approach.[web:19][web:20]

## Design Scenario

A retro-computing peripheral interface needs a compact hardware controller with these features:

- Input decode of command and mode lines.
- Output generation for `ENABLE`, `WRITE_OK`, and `FAULT` indicators.
- Optional registered behavior for one synchronized status output.
- Fast response without burdening a host microcontroller.

Students will create a PLD design that serves as discrete control hardware for a larger embedded subsystem.

## Pre-Lab Questions

1. What engineering problem is solved more effectively by a PLD than by discrete logic gates?
2. How does a registered output differ from a combinational output in practical system behavior?
3. Why do propagation delay and product-term limits matter in programmable logic design?
4. When would a microcontroller be a poor substitute for a PLD?
5. What verification steps are necessary after programming a logic device?

## Lab Tasks

### Part 1: Device Familiarization

1. Review the pinout, power requirements, and output configuration model of the ATF16V8C.
2. Identify which pins will be used for input conditions and which for outputs.
3. Create a signal table for the target logic subsystem.
4. Define whether each output should be combinational or registered.

### Part 2: Truth Table and Equation Development

1. Build a truth table for at least three outputs driven by several input conditions.
2. Minimize or simplify the logic where appropriate.
3. Translate the truth table into equations or a PLD design entry format.
4. Document assumptions about active-high or active-low behavior.

### Part 3: Combinational Logic Implementation

1. Implement a first version with purely combinational outputs.
2. Program the device and verify each output against the truth table.
3. Use LEDs and measurement tools to validate operation.
4. Record any hazards or unexpected transitions.

### Part 4: Registered Output Implementation

1. Convert one output into a registered output synchronized to a clock.
2. Demonstrate the difference between immediate combinational response and clocked response.
3. Measure or estimate the effect on timing and observable behavior.
4. Explain when registered outputs improve system robustness.

### Part 5: Integrated Control Logic Design

Produce a final PLD design with these minimum features:

- At least four input signals.
- At least three output signals.
- One output that changes only on a clock edge.
- One decode condition that forces a fault or inhibit state.
- Clear documentation of equations and test cases.

### Part 6: Firmware Comparison

1. Describe how the same function could be implemented on a microcontroller.
2. Compare latency, determinism, flexibility, and complexity.
3. Identify which implementation is preferable for the given scenario and justify the choice.

## Deliverables

Students must submit:

- Truth table and logic equations.
- PLD source file or equation listing.
- A pin assignment table.
- Verification notes showing expected versus observed behavior.
- A brief timing discussion covering propagation or clocked behavior.
- A short comparison between PLD and microcontroller implementation.

## Assessment Criteria

| Criterion | Excellent | Satisfactory | Needs Improvement |
|---|---|---|---|
| Logic design | Equations are correct, efficient, and well justified | Mostly correct with minor issues | Logic is incomplete or incorrect |
| Device programming | Device programmed and verified systematically | Device works with limited verification | Programming flow is unclear or incomplete |
| Timing awareness | Student discusses timing and output modes clearly | Basic timing awareness is present | Timing considerations are missing |
| Registered logic use | Registered behavior is used correctly and explained | Registered output works with limited analysis | Registered logic is missing or misunderstood |
| Documentation | Tables, equations, and tests are complete | Documentation is adequate | Documentation is weak or missing |
| Comparative analysis | PLD vs MCU trade-offs are precise and thoughtful | Comparison is basic | Comparison lacks technical depth |

## Discussion Prompts

- Why do programmable logic devices remain relevant even in systems that already include microcontrollers?
- Which parts of a digital control path are best implemented in hardware rather than firmware?
- How do device-level timing guarantees affect system architecture?
- What is lost, and what is gained, when replacing programmable logic with software?

## Extension Opportunities

- Implement a more complex address decoder.
- Add state-based output qualification.
- Explore active-low bus control conventions.
- Compare hand-minimized equations with CAD-generated logic.
- Incorporate the PLD into a larger mixed hardware/firmware design.

## References

Vendor and distributor documentation describes the ATF16V8C as a high-performance erasable PLD suitable for combinational and registered logic implementation in compact digital designs.[web:19][web:20]
