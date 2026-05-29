# Caravel Integrated Verification Results

## user_pass_thru

![alt text](image-21.png)

We can see successful execution. However, I did have to comment out some includes to get this working, from the includes.rtl.caravel specifically. There were includes to paths and files that simply did not exist from the git pull, hence I had to completely comment them out. Luckily they did not affect execution, else I would have had to find the files on git and add them manually.

## UART

Since includes.rtl.caravel is global, fixing it once fixes the dependencies for all the modules to be tested for. Hence this executed first try.

![alt text](image-22.png)

## SRAM Execute

![alt text](image-23.png)

## Sysctrl

![alt text](image-24.png)

## SPI Master

![alt text](image-25.png)

## Pullupdown

![alt text](image-26.png)

## PLL

![alt text](image-28.png)

I did have to modify the .c file to add delays to get the proper outputs.

![alt text](image-27.png)

## pass_thru_fix

![alt text](image-29.png)

## Memory

![alt text](image-30.png)

## hkspi_power

![alt text](image-31.png)

## gpio_mgmt

![alt text](image-32.png)

## hkspi

![alt text](image-33.png)

| tests-caravel | status |
|---|---|
| user_pass_thru | PASS |
| uart | PASS |
| sysctrl | FAIL |
| sram_exec | PASS |
| spi_master | PASS |
| pullupdown | PASS |
| pll | PASS |
| pass_thru_fix | PASS |
| mem | PASS |
| hkspi_power | PASS |
| gpio_mgmt | PASS |
| hkspi | PASS |


# Module Overview

## user_pass_thru

this basically creates a direct pass through of signals on the IO interface. It can tell us if the IO connectivity and the padrame is working properly.

## UART

Sends and receives bytes on the uart interface and checks if the sender and the receiver are synchronous and verifies it.

### Sysctrl

Sysctrl checks global coordination among the different modules in the SOC. This includes:-
Reset control, clock control, peripherals, global config registers, boot config, interrupt coordination, power management.

sram-exec:-

Checks if the processor can retrieve bytes from the memory and execute it successfully. This also parallely verifies the connections between the cpu, memory controller and the sram itself. 

### spi_master

Usually connects to external pins for interface with foreign devices. It is called SPI-master because this is the MASTER and not the slave in the configuration and hence also generates the clocks. SPI is very standardized because it is lightweight and simple.

## Pullupdown

The gpio pins are usually not left hanging when not driven. This is inherently bad because this can lead to unnexpected behaviour and antenna behavior. Thus, we usually enable a weak resistor to 1 or 0, namely VDD or GND. This is done to base the signal when not driven. It is intentionally weak as when driven, the pull up or down should not counter power the driving signal itself. In SOCs this is usually configurable.

## PLL

The basic idea of a PLL or a phase locked loop is simple. There is a reference clock usually way smaller and a voltage controlled oscillator. There are dividers at the input and output. And a feedback loop keeps the phase intact AFTER the target frequency is achieved. The dividers are small and light FSMs that reduce the clock frequencies from and to the voltage controlled oscillator. By controlling the dividers, I can essentially control the output frequency of the system. But this is very hard as each of these components have very strict limitations in frequency range of operation. hence, the entire device also has ranges in which they are properly functional. Outside that, they could have unpredictable behaviour.

## Memory

Checking if all the memory connections work. We write and read from the same memory addresses to see if the bytes were actually stored in that memory location. And this also verifies that the wire connections and the memory controller is functional. 

## hkspi_power

This is for House keeping and power management. This involves checking and verifying the control and management subsystems. This helps boot and maintain the chip when the other modules perform specific tasks. As the name suggests, this particular module is for house keeping the SPI module, which includes managing the PLL registers and power settings. 

'how do you verify power from digital domain?'

Power is inherently analog and hence cannot be directly interpretted in digital. Instead we check power by triggering certain control signals and verifying the behavior of the systems and infer power from that rather than directly.

## gpio_mgmt

This involves changing the meta states of the gpio pins and verifying if it works appropriately. This involves checking if the pin can toggle between input and outputs. It also verifies if the management SOC controls the pins and not the user modules directly. This would lead to contention issues. 

## hkspi

Is the global module for SPI housekeeping. This is a super-set of the hkspi_power module. It verifies startup, debug path and infrastructure access.

# Understanding the Code and Verification Workflows

## GPIO_MGMT

gpio_mgmt.c

The gpio interface works with a MMIO or memory mapped IO. The processor simply writes to a register which is attached to more analog hardware. We can see some configuration registers too.

![alt text](image-34.png)

here, the input and output has both been set. The .c file sets the pins to high and low. The test bench tests if there is a high and low one after each other and uses that to determine pass/fail.

Do note that since there is a firmware involved, the testbenches merely check the system and validate them instead of giving stimulus to the DUTs for all the modules.

## user_pass_thru

Usually the housekeeping SPI, just interacts with the internal management registers. But here, they are decidedly routed out the external pins to observe its output and the connectivity between the wires are checked logically. The output pins are configured to the management output pins. 

![alt text](image-35.png)

## UART

Just like user pass through, we immediately set the GPIO pins to the management pins. 

![alt text](image-36.png)

We have set the tx pin as the pin 6 and we are monitoring pins 16-31.

![alt text](image-37.png)

we set the data input and verify if the checkbits reciprocate. This would indicate that the firmware started.

![alt text](image-38.png)

The print statement is called and it eventually uses the UART interface for each character.

![alt text](image-39.png)

We use delay loops because UART transmission is not instant. without this, the CPU exits and the simulation will end abruptly without finishing the full transmission.

![alt text](image-40.png)

### Sysctrl

This checks if the clocks of the chip is properly configured and can be sent to the GPIO output pins. The caravel SOC has multiple clock domains and verifying if both of them are accessible in any parts of the chip, especially at the outputs is necessary.

![alt text](image-41.png)

The first test does not enable either of the clocks, and monitors the outputs. We are expecting no pulses here.

In the second test, we check if the first system clock works.

In the third test, we check if the second user clock works and is functional. 

The delays wait for the edges to accumulate because the testbenches count these clocks at their edges to verify.

## SRAM Execute

A processor usually boots and executes from a flash through an SPI. This verifies if it can do the same from SRAM. This code copies code into sram and execute from sram. The gpio pins are configured to interact with the TB and we can verify if it works from the TB.

![alt text](image-42.png)

the source and destination of the memory byte transfer is defined as the flash and the SRAM respectively. Then It is recursively copied. The CPU's pointer is then moved to the SRAM. 

The testbench has taken a secondary role in this module again. The firmware does most of the heavy-lifting.

The testbench observes the outputs through the GPIO pins and records the phase of execution. Note how we have also simultaneously checked the functioning of the pointers.

## SPI Master

Just like before, we set the data bits to something specific and check the Testbench can recognize that. We can use this to verify the execution stage by stage. 

We asset chip select because we intend to begin transmission.

The firmware uses wrapper functions for write and read simultaneously. Do note that SPI is full duplex, therefore SPI can receive and send bytes at the same time. However, since SPI is a master-slave relation, to receive some data, the master must send some dummy bits to receive data in return.

![alt text](image-43.png)

We instantiate two SPI flashes to interface them both. Now this firmware is talking to a Simulated external peripheral from the caravel SOC.

![alt text](image-44.png)

### Pullupdown

This is a particularly interesting test because we are trying to analyse the analog behavior (pull up and pull down) in digital design in pure binary. 

Besides the few pins needed by the firmware itself, the rest is disconnected from all digital drivers

![alt text](image-45.png)

We are not done yet, Some of the GPIO pins are usually connected to the SPI interface. Hence we disable the SPI housekeeper to have full GPIO access.

![alt text](image-46.png)

Then we pull down the pins weakly for the first test.

![](image-47.png)

While this is happening, The Testbench checks on the other side if the final output reads 1'b0 as nothing is driving this output other than the weak pulldown. This is done individually for all 32 pins with a simple for loop.

![alt text](image-49.png)

Next, we pull up the pins weakly and repeat the process.

Finally we simply set it to std_analog which disconnects it from all digital drivers.

![alt text](image-48.png) 

Here, we should expect to see all high impedance. 

## PLL

Phase locked loop - We are checking the core and user clocks again. 

![alt text](image-50.png)

This line exposes both the clocks to the GPIOs. As we have seen before, the clocks are wired to pins 14 and 15. 

The testbench counts the user clocks and the core clocks at the edges.

For loop delays were added here. They were not there in the original code. The reason for this was that there was little to no time between the setting and the disabling of the clocks and hence, the clocks returned values that were either too close to or exactly 0.

![alt text](image-51.png)

I)In the first test, the PLL is entirely bypassed. 

This confirms that the system is able to use both clocks' external paths.

we expect to see the ucount and ccount to be identical. 

II)In the second test, we have the PLL enabled, but it is still bypass. We still expect ucount = ccount.

III)The Pll bypass is finally lifted on the third test and we expect to see some difference between ucount and ccount. Here particularly, the ratio between ucount:ccount is approximately 3:1

IV)In the next test, the feedback divider is changed and the testbench observes if there is a change in the ratio.

V)Then, the ratio is changed by changing output divider. 

pass_thru_fix

this checks if the spi pins behave when the GPIO pins itself are idle. If SPI behaves erratically when GPIO pins as a whole are expected to be idle, we might receive garbage data transmissions.

The .c file almost does anything substantially. Just the initial setup we saw in the other files, just that.

![alt text](image-52.png)

a while loop at the end to just hold the program in idle, which is the state we are trying to test here.

Apparently, the previous caravel versions switched the data and the clock at the same time. This is objectively bad as the receivers want to sample the data at the clocks and if the data starts transitioning then, it leads to meta state analog issues.

## Memory

The firmware loads full 32 bit words into the memory and read them to check if they have been stored properly. 
The testbench checks both the data transmissions and verifies. 

It then does this for 8 and 16 bit data pieces as well. This is crucial because the processor does all mixed width accesses all the time.

## hkspi_power

The test here is to verify if the housekeeping SPI can function when the user region is entirely powered off. The smoking gun is this line where the vdd is also gounded.

![alt text](image-53.png)

This is why this is one of the outliers in these modules where the testbenches are doing the grunt work. The testbench probes the housekeeping SPI. 

## HKSPI

Again, the testbench is heavy lifting functionality here by manipulating the pins and spi by itself. However, the firmware keeps the uart channel open and transmits keeping the CPU enabled while the HKSPI works externally.

![alt text](image-55.png)

This is where the SPI pins are established in the testbench.

![alt text](image-54.png)

The hkspi then resets externally. then the reset is deasserted.

# Verification Flow Diagrams

## user_pass_thru

![alt text](Screenshot_20260528-201149.png)

## UART

![alt text](Screenshot_20260529-082928.png)

## Sysctrl

![alt text](Screenshot_20260529-083942.png)

### sram exec

![alt text](Screenshot_20260529-084347.png)

### SPI_Master

![alt text](Screenshot_20260529-084910.png)

## Pullupdown

![alt text](Screenshot_20260529-085439.png)

## PLL

![alt text](Screenshot_20260529-090316.png)

### pass through fix

![alt text](Screenshot_20260529-091047.png)

## Memory

![alt text](Screenshot_20260529-091326.png)

## HKSPI_POWER

![alt text](Screenshot_20260529-091737.png)

## HKSPI

![alt text](Screenshot_20260529-092153.png)

