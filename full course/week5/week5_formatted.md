# Phase 1

The flow completed successfully from RTL-to-GDS.

![alt text](image.png)

This netlist was selected because it is the final gate-level implementation generated after completion of the RTL-to-GDS flow. It includes all synthesis, placement, clock tree synthesis, and routing optimizations and therefore represents the final design used for GLS verification.

![alt text](image-1.png)

By filtering with the technology library names, we can see that the netlist contains technology-specific standard cell instances, confirming that synthesis and physical implementation have been completed.

## Netlist Integration

First, the generated netlist was copied to `__user_project_wrapper`, which previously contained the raw RTL implementation of the design.

![alt text](image-2.png)

## Dependency Fixing

The next step involved resolving dependencies required by the GLS flow. The Makefiles expected several files which were either located elsewhere or missing entirely.

These included:

* `mgmt_core`, which existed in a different location and was copied into the expected directory.
* `DFFRAM`, which was referenced but did not exist in the repository, so the requirement was commented out.
* `sky130_sram_2kbyte_1rw1r_32x512_8.v`, which existed elsewhere and was copied into the required location.

![alt text](image-3.png)

## Standalone GLS Results

### UART Successful Simulation

![alt text](image-4.png)

![alt text](image-7.png)

### Timer Successful Simulation

![alt text](image-5.png)

![alt text](image-8.png)

### Debug Successful Simulation

![alt text](image-6.png)

![alt text](image-9.png)

### GPIO Management Successful Simulation

![alt text](image-10.png)

![alt text](image-11.png)

### IRQ Failed Simulation

![](image-12.png)

**Reason:** The RTL version of this test already failed during Week-3. Multiple debugging attempts were made at the RTL level without success, so this result was expected.

However, this output is still significantly better than the original behavior observed during Week-3 debugging, where the simulation remained stuck in a single status throughout execution.

![alt text](image-13.png)

### MEM Failed Simulation

The MEM test managed to reach stage one (`A040`) but did not progress beyond that point.

![alt text](image-15.png)

![alt text](image-14.png)

### SPI Master Successful Simulation

![alt text](image-16.png)

![alt text](image-17.png)

## Standalone GLS Results Summary

| Test       | Status (Sky130) |
| ---------- | --------------- |
| GPIO Mgmt  | PASS            |
| mem        | FAIL            |
| uart       | PASS            |
| timer      | PASS            |
| irq        | FAIL            |
| debug      | PASS            |
| spi_master | PASS            |

# Caravel Integrated Simulations

Initially, I encountered issues that prevented the flow from running correctly. This was caused by the final generated Verilog netlist not containing power pins.

![alt text](image-19.png)

The `-DUSE_POWER_PINS` flag determines whether power pins are included through conditional compilation directives (`ifdefs`).

After adding this flag to the `config.mk` used by the ORFS flow:

![alt text](image-20.png)

the Caravel GLS simulations were able to compile and execute correctly.

## GPIO Management

![alt text](image-18.png)

![alt text](image-21.png)

## HKSPI

![alt text](image-22.png)

![alt text](image-23.png)

## HKSPI Power

![alt text](image-24.png)

![alt text](image-25.png)

## MEM

![alt text](image-26.png)

![alt text](image-27.png)

## Pass Thru

![alt text](image-28.png)

![alt text](image-29.png)

## Pass Thru Fix

![alt text](image-30.png)

![alt text](image-31.png)

## PLL

![alt text](image-32.png)

![alt text](image-33.png)

## Pullupdown

![alt text](image-34.png)

![alt text](image-35.png)

## SPI Master

![alt text](image-36.png)

![alt text](image-37.png)

## SRAM Exec

![alt text](image-38.png)

![alt text](image-39.png)

## UART

![alt text](image-40.png)

![alt text](image-41.png)

## SYSCTRL

![alt text](image-42.png)

The SYSCTRL test already failed during RTL verification in Week-3, so this result is not particularly surprising.

![alt text](image-44.png)

## User Pass Thru

![alt text](image-45.png)

![alt text](image-46.png)

## Caravel GLS Results Summary

| tests-caravel  | status |
| -------------- | ------ |
| user_pass_thru | PASS   |
| uart           | PASS   |
| sysctrl        | FAIL   |
| sram_exec      | PASS   |
| spi_master     | PASS   |
| pullupdown     | PASS   |
| pll            | PASS   |
| pass_thru_fix  | PASS   |
| mem            | FAIL   |
| hkspi_power    | PASS   |
| gpio_mgmt      | PASS   |
| hkspi          | PASS   |

# RTL vs GLS Comparison

The GLS results closely matched the RTL verification results obtained during Week-3.

The blocks that passed during RTL simulation also passed during gate-level simulation, while the blocks that failed during RTL simulation (`irq`, `mem`, and `sysctrl`) continued to fail during GLS. This indicates that the observed failures are likely functional issues already present in the RTL rather than issues introduced during synthesis or physical implementation.

Overall, the GLS results demonstrate functional equivalence between the RTL design and the final gate-level implementation.
