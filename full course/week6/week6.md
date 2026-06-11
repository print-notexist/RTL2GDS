The caravel/verilog/rtl folder:-

![alt text](image.png)

I choose the housekeeping_spi.v Block. I plan on reusing the makefile from the caravel tests and create a test bench to use the .v file.

It has no submodule instantiation, a blatent coincidence.

The module is quite compact for the amount of features it provides.

The states are defined at the very first. The SPI interface works with the implication that the first transmission is the command, and the next byte is address and every consequent byte induced is a data byte.

![alt text](image-1.png)

This is particularly clever, the last address bit already exists on the SDI line, so why wait for the bit to be shifted into the address register to shift it out? We can directly send the data one clock faster.

![alt text](image-2.png)

Architectural optimizations like this is what makes our daily machines faster every day.


![alt text](image-3.png)

We can also see the serial output transmission line wired in combinational logic. It is directly linked to the MSB of the ldata register which holds the byte to be transmitted.

Another key thing to not is the clock(SCK), notice how it is an input, hence this is the slave of the communication. This also makes sense intuitively as the master in the housekeeping spi would want to send and receive data and the SoC as a whole simply responds to such impulse tests.

![alt text](image-4.png)

Standard reset of the key registers at the triggers.

![alt text](image-5.png)


SDOenb is another important aspect of this design. It controls when and how the housekeeping SPI drives the SDO pin. This results in predictable output even when the slave has nothing meaningful to transact back to the master. Notice how it is pulled DOWN during readmode. This suggests that this could be a low enabled pin.

![alt text](image-6.png)

This whole negative edge triggered logic also help us understand and realize the importance of decoupling capacitors. Without them, with all this load spiking within picoseconds, the power distribution will not be able to handle all of it without losing some timing efficiency.

![alt text](image-7.png)

All these if statements would get synthesized as muxes in hardware logic. Hence this would build a mux tree for this logic. 

A testbench for this module was designed after understanding the functions.

The testbench starts with a standard definitions and instantiation of the Unit/Design under test.

![alt text](image-8.png)

A simple clock counter so that during runtime, we will know if the implementation is running or has hung entirely.

![alt text](image-10.png)

![alt text](image-9.png)

This is to capture addresses exactly when the strobe is raised

By modularizing the repetitive parts of the test, we can manage to create more standardized testbenches.

![alt text](image-11.png)

All the tests have been bundled into tasks that can be called in a single line during execution.

This is how the final initial is extremely small

![alt text](image-13.png)

Test 1 - command decode:-

We send a command over the SPI interface and observe if the commands actually affect the state of the housekeeping SPI slave.

![alt text](image-14.png)

Test 2 - Write transaction:-

Write transaction needs to fetch data from the SPI receiver and send it upstream. 

![alt text](image-15.png)

Test 3 - Read transaction -

Read transaction expects input from external logic for the address and the data to be transacted back to the SPI master. We use the captured address block from earlier to update this.

![alt text](image-16.png)

Test 4 - Address Auto-increment - 

This SPI interface has an auto-incrememnt set up for its addresses. This is to verify its functioning. We send a byte, wait for 8 clocks to send 8 bits individually and check the address again to see if it has automatically increased by 1.

![alt text](image-17.png)

ORFS Flow run:-

The config file:-

![alt text](image-19.png)

The constraint file:-

![alt text](image-20.png)

Final design on Openroad -gui:-

synthesis:-

![](image-21.png)

Floorplan:-

![alt text](image-26.png)

Placement:-

![alt text](image-27.png)

CTS:-

![alt text](image-28.png)

routing:-

![alt text](image-29.png)

final:-

![alt text](image-30.png)

final GDS file:-

![alt text](image-18.png)

GLS verification:-

The testbenches were modified to accomodate for the changes in the names between RTL and GDS.

![alt text](image-31.png)

The state names that were previously used with the help of defines, has been removed as the GLS changes the nomenclature. 

![alt text](image-32.png)

The final run passed all tests first try.

