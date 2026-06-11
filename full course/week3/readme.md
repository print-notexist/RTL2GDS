# Week-3: Block-Level Verification of VSDSquadron SoC

## Phase 1 - Task 1

### Cloning the VSDSquadron SoC Repository

![alt text](image.png)

Successful output.

Do note that the RISC-V compiler was not present on the machine and had to be manually downloaded before the verification environment could execute correctly.

---

# Understanding the Verification Flow

## Analysing the Makefile

The Makefile is highly modularized.

![alt text](image-6.png)

The Makefile uses the name of the folder to determine file names and paths. By following a consistent naming convention, the same Makefile can be reused across multiple designs instead of creating individual Makefiles for each test.

A single design root is defined and used as the source of truth for all paths. This is a clean modularization strategy because if the directory structure changes, only a single path needs to be modified rather than updating every reference throughout the project.

![alt text](image-7.png)

We can see that PDK paths are also included in the Makefile. However, the verification flow appears to be purely logical in nature. My assumption is that the same Makefile can be reused for later stages of verification or integrated into other EDA flows such as OpenLane or ORFS. It is often easier to inherit all required paths and selectively use them than to maintain separate configurations for each flow.

![alt text](image-8.png)

The CPU is selected through a variable, and different files are used depending on the CPU chosen.

This structure allows multiple processors to be supported with the same Makefile by simply extending the existing `ifeq` blocks.

![alt text](image-9.png)

This is the first section of the Makefile that performs an actual verification task.

Firmware files are generated and immediately verified.

The Makefile also ensures that linker scripts and startup sources exist before generation begins by running `prepare-generated`. Both processes are chained so that `prepare-generated` is always executed before `check-fw`.

We can also see flags indicating that the target environment is bare-metal silicon rather than a hosted operating system environment.

![alt text](image-10.png)

---

## What is a Linker Script?

A linker script is used to create the final memory image of compiled code.

On a desktop computer, memory management is largely handled by the operating system through virtual memory, dynamic allocation, and runtime services.

In an embedded system, none of these services are available because there is no operating system. The firmware effectively acts as the operating system and therefore must know exactly where every piece of code and data resides in memory.

To the processor, memory is simply a collection of addresses. The firmware, however, must make informed decisions about where code, data, stacks, and peripherals are located. The linker script defines this memory mapping.

---

## How Does a Linker Script Define Memory Mapping?

Integrated circuits often have strict hardware limitations.

For example:

* A UART peripheral may only respond to a specific address range.
* SRAM may only exist within a predefined address window.
* Certain peripherals may be inaccessible outside their designated memory regions.

Ignoring these constraints can lead to corrupted data, undefined behavior, or incorrect operation.

To avoid these issues, a structured memory map is created and provided to the firmware. This map serves as the authoritative reference for where information should be stored and how it should be accessed.

---

## VVP Engine

The VVP engine elaborates Verilog behavior and instantiates the modules required for simulation.

The HEX file and the testbench are also included because they form part of the complete simulation environment.

Just as different CPUs can be selected through `ifeq` blocks, the Makefile also separates simulation flows using conditional logic. Since this verification is purely RTL-based, flows such as GL and GL_SDF are not relevant here.

One notable flag is `-Ttyp`, which selects the typical timing corner. However, because this is an RTL simulation, timing corner selection has minimal impact.

![alt text](image-11.png)

Finally, the simulation is executed.

This stage:

* Toggles clocks
* Records signal activity
* Generates waveform data
* Produces VCD files
* Renames output files for easier organization and automation

---

# Module Overview

## user_pass_thru

This module creates a direct pass-through path between signals and the IO interface.

It verifies:

* IO connectivity
* Padframe routing
* Signal propagation through the external interface

---

## UART

The UART module sends and receives bytes through the UART interface.

Verification ensures:

* Correct transmission
* Correct reception
* Synchronization between transmitter and receiver

---

## Sysctrl

The System Controller verifies global coordination across the SoC.

This includes:

* Reset control
* Clock control
* Peripheral management
* Global configuration registers
* Boot configuration
* Interrupt coordination
* Power management

---

## sram_exec

This test verifies that the processor can successfully fetch instructions from SRAM and execute them.

It also validates:

* CPU-to-memory connectivity
* Memory controller functionality
* SRAM operation

---

## spi_master

The SPI Master module communicates with external devices through the SPI protocol.

It is called a master because it:

* Initiates communication
* Generates the SPI clock
* Controls transactions with slave devices

SPI remains widely used because it is lightweight, simple, and highly standardized.

---

## pullupdown

GPIO pins should not be left floating when idle.

Floating signals can cause:

* Unpredictable behavior
* Noise sensitivity
* Antenna-like effects

To prevent this, weak pull-up or pull-down resistors connect the signal to either:

* VDD
* GND

These resistors are intentionally weak so they do not interfere when an external driver actively drives the signal.

Most SoCs allow these pull configurations to be controlled through software.

---

## PLL

The basic idea behind a PLL (Phase-Locked Loop) is straightforward.

A PLL consists of:

* A reference clock
* A voltage-controlled oscillator (VCO)
* Input dividers
* Output dividers
* A feedback loop

The feedback mechanism continuously adjusts the oscillator until the desired output frequency is achieved while maintaining phase alignment.

By modifying divider ratios, different output frequencies can be generated.

However, each component has strict operating limits, meaning the PLL can only function correctly within specific frequency ranges.

Outside those ranges, behavior may become unpredictable.

---

## mem

This test verifies memory functionality.

The firmware:

1. Writes data to memory.
2. Reads the same memory location.
3. Compares the values.

This simultaneously validates:

* Memory cells
* Interconnects
* Memory controller operation

---

## hkspi_power

This module focuses on housekeeping and power management functionality.

Responsibilities include:

* Managing control subsystems
* Supporting boot operations
* Maintaining system operation
* Configuring PLL registers
* Managing power settings

### How Do You Verify Power from the Digital Domain?

Power behavior is inherently analog and cannot be measured directly through digital logic.

Instead, verification is performed indirectly by:

1. Triggering control signals.
2. Observing system behavior.
3. Inferring power state changes from the resulting responses.

---

## gpio_mgmt

This test verifies GPIO management functionality.

It checks:

* Input mode operation
* Output mode operation
* Pin ownership control

It also ensures that GPIOs remain under management SoC control rather than being directly driven by user logic, preventing contention issues.

---

## hkspi

This is the top-level housekeeping SPI verification module.

It is effectively a superset of `hkspi_power`.

Verification includes:

* Startup behavior
* Infrastructure access
* Debug functionality
* Housekeeping SPI operation
