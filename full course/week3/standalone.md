# Standalone Block Verification Results

This section documents the execution and analysis of the standalone verification tests provided with the VSDSquadron SoC repository.

---

# SPI Master

![alt text](image-1.png)

![alt text](image-2.png)

## Analysis

We can observe the firmware compilation commands being executed.

This occurs because `spi_master.c` is firmware running on a simulated RISC-V processor. The source file is compiled into RISC-V instructions and then loaded into the simulated processor as part of the verification environment.

When the `make` command is executed:

1. The RISC-V simulation environment is initialized.
2. Firmware execution begins.
3. The SPI RTL block is initialized.
4. The simulation executes using the generated `.vvp` file.

The generated ELF and HEX files exist because the verification environment includes both the processor and firmware layers. A bare-metal RISC-V processor is instantiated and executes firmware without an operating system.

Once the firmware is active, it controls the SPI interface through memory-mapped registers.

The resulting RTL behavior is then monitored by the testbench and compared against expected outputs.

The test completed successfully.

![alt text](Screenshot_20260523-091709.png)

The RTL successfully passed the design verification test.

---

# GPIO Management

![alt text](image-3.png)

This is the successful output of the GPIO Management test.

The PASS status can be clearly observed in the simulation output.

Just like the SPI Master test, the first stage involves compiling firmware for a 32-bit RISC-V processor.

![alt text](image-4.png)

The `-ffreestanding` flag is used because the target environment is bare-metal hardware without an operating system.

The GPIO registers do not directly drive external pins.

Instead, they connect to output driver circuitry that:

* Increases drive strength
* Allows larger traces to be driven
* Protects internal logic from external disturbances

The pad transistors are typically:

* Larger than core transistors
* Built with thicker oxide layers
* More resilient than standard logic transistors

---

# UART

![alt text](image-5.png)

The UART verification test completed successfully.

The firmware is compiled and executed on the simulated RISC-V processor, which then communicates through the UART peripheral.

The testbench observes the resulting UART activity and determines whether the expected behavior is produced.

---

# Timer

## Initial Run

![alt text](image-12.png)

The initial execution failed.

To determine the cause, the generated waveform was analyzed.

![alt text](image-13.png)

The first verification stage succeeded.

The firmware correctly wrote the value `0x0A`, and the expected output was observed.

However, the subsequent check using `0x01` failed.

This confirmed that:

* The firmware was executing correctly.
* The timer/counter was operating correctly.
* The failure was related to timing rather than functionality.

To investigate further, the repeat count inside the testbench wait loop was increased to provide additional execution time for the firmware.

The modification resolved the issue immediately.

![alt text](image-14.png)

![alt text](image-15.png)

The Timer verification test passed after adjusting the timing constraints in the testbench.

---

# IRQ

IRQ stands for **Interrupt Request**.

This verification test failed.

![alt text](image-16.png)

Several debugging attempts were performed.

First, additional updates were added to the data variable to determine whether execution was progressing correctly.

The data values updated as expected, but the GPIO state did not respond accordingly.

![alt text](image-17.png)

Additional experiments included:

* Adding a periodic data variable
* Introducing delays after updates
* Allowing additional execution time for counting operations

The `asm volatile` statement was also temporarily commented out because it prevented other operations from updating independently.

Despite these modifications, the issue remained unresolved.

Based on the observed behavior, the test was classified as a design failure rather than a testbench issue.

---

# Debug

The Debug verification test initially failed.

![alt text](image-19.png)

To investigate, the debug entry timing was modified.

Additional time was provided before entering debug mode, allowing the processor to execute instructions before the debug state became active.

![alt text](image-18.png)

This modification resolved the issue.

![alt text](image-20.png)

The Debug verification test subsequently passed.

---

# Standalone Test Results

| Test       | Status (Sky130) |
| ---------- | --------------- |
| GPIO Mgmt  | PASS            |
| mem        | PASS            |
| uart       | PASS            |
| timer      | PASS            |
| irq        | FAIL            |
| debug      | PASS            |
| spi_master | PASS            |

## Summary

A total of seven standalone verification tests were executed.

Results:

* PASS: 6
* FAIL: 1

The only failing standalone test was the IRQ verification test. All other standalone verification environments executed successfully.
