# Block Selection and Analysis

The available RTL blocks can be found in the `caravel/verilog/rtl` folder.

![alt text](image.png)

I chose the `housekeeping_spi.v` block. The plan was to reuse the Makefile structure from the existing Caravel tests and develop a dedicated testbench for the module.

The module contains no submodule instantiations, which was a fortunate coincidence.

The design is relatively compact considering the amount of functionality it provides.

## RTL Analysis

The states are defined at the beginning of the module. The SPI interface operates with the assumption that the first transmission is the command, the second byte contains the address, and all subsequent bytes are treated as data bytes.

![alt text](image-1.png)

This implementation contains an interesting optimization. The final address bit already exists on the SDI line, so there is no need to wait for it to be fully shifted into the address register before beginning data transmission. The data can therefore be returned one clock cycle earlier.

![alt text](image-2.png)

Architectural optimizations such as this contribute to improving performance and reducing latency in digital systems.

![alt text](image-3.png)

The serial output transmission path is implemented using combinational logic. The output is directly connected to the MSB of the `ldata` register, which contains the byte currently being transmitted.

Another important observation is that `SCK` is defined as an input. This indicates that the module operates as an SPI slave. This is also consistent with the intended functionality, as the housekeeping SPI interface responds to transactions initiated by an external SPI master.

![alt text](image-4.png)

The design includes standard reset behavior for the key registers.

![alt text](image-5.png)

`SDOenb` is another important signal within the design. It controls when the housekeeping SPI drives the `SDO` pin. This ensures predictable output behavior even when no meaningful transaction data is available. During read mode the signal is driven low, suggesting that it is an active-low enable signal.

![alt text](image-6.png)

The design also makes extensive use of negative-edge-triggered logic. In a physical implementation, switching activity from sequential elements contributes to transient current demand, highlighting the importance of a robust power distribution network and adequate decoupling.

![alt text](image-7.png)

The conditional logic within the module would be synthesized into multiplexers during hardware implementation. As a result, this section of RTL ultimately forms a mux tree in the synthesized design.

# Testbench Development

A dedicated testbench was designed after understanding the functionality of the module.

The testbench begins with standard signal declarations and instantiation of the Design Under Test (DUT).

![alt text](image-8.png)

A simple clock counter was included to provide runtime visibility and determine whether execution had stalled.

![alt text](image-10.png)

![alt text](image-9.png)

This logic captures addresses whenever the strobe signal is asserted.

By modularizing repetitive operations, the testbench becomes easier to maintain and extend.

![alt text](image-11.png)

All tests were grouped into reusable tasks that can be invoked using a single line during execution.

As a result, the final `initial` block remains compact and easy to read.

![alt text](image-13.png)

# Verification Tests

## Test 1 - Command Decode

A command is transmitted through the SPI interface and the resulting state transitions are observed to verify correct command decoding.

![alt text](image-14.png)

## Test 2 - Write Transaction

The write transaction verifies that received SPI data is correctly captured and forwarded to the upstream interface.

![alt text](image-15.png)

## Test 3 - Read Transaction

The read transaction requires external logic to provide both the address and the corresponding data to be returned to the SPI master. The previously implemented address-capture logic is used to support this operation.

![alt text](image-16.png)

## Test 4 - Address Auto-Increment

The housekeeping SPI interface supports automatic address incrementing.

To verify this functionality, a byte is transmitted, eight clock cycles are allowed to complete, and the address is checked to confirm that it has incremented automatically.

![alt text](image-17.png)

# RTL-to-GDS Implementation Using ORFS

## Configuration Files

### ORFS Configuration

![alt text](image-19.png)

### Timing Constraints

![alt text](image-20.png)

## OpenROAD Flow Results

### Synthesis

![](image-21.png)

### Floorplan

![alt text](image-26.png)

### Placement

![alt text](image-27.png)

### Clock Tree Synthesis (CTS)

![alt text](image-28.png)

### Routing

![alt text](image-29.png)

### Final Database

![alt text](image-30.png)

### Final GDS

![alt text](image-18.png)

# Gate-Level Simulation (GLS) Verification

The testbench was modified to accommodate naming differences between the RTL implementation and the generated gate-level netlist.

![alt text](image-31.png)

State names previously referenced through RTL defines were removed because the GLS netlist uses different naming conventions after synthesis.

![alt text](image-32.png)

After updating the testbench, the generated gate-level netlist was simulated using the same verification environment.

All tests passed successfully on the first GLS execution.

# Conclusion

The `housekeeping_spi` block was successfully taken through the complete RTL-to-GDS flow using ORFS. A custom verification environment was developed, multiple functional tests were implemented, and the generated gate-level netlist was validated through GLS.

The successful execution of both RTL and GLS verification confirms that the synthesized implementation preserves the intended functionality of the original design.
