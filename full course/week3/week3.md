
Phase 1 - Task 1 

Cloning the vsdsquadron-soc git.

![alt text](image.png)

Successful output. Do note that the riscv compiiler was not present on the machine and was manually downloaded for the following output to work.

SPI-MASTER :-

![alt text](image-1.png)

![alt text](image-2.png)

So let's break down what's happening

We see compilation commands. This is happening because the spi_master.c is technically a firmware running on a simulated risc-v core. It compiles the .c file for risc-v instructions and feeds it in binary to the simulated risc-v core. 

When the make command is called, it calls the risc core initially and then the firmware is woken up and the SPI interface rtl is initialized. This is represented by the .vvp file. 

ELF and HEX files exist because the simulation exists until the firmware layer. Therefore the bare metal risc-v core without an OS is initiated and a firmware is iniated on it.

Once the firmware is set up, it uses the SPI interface by manipulating the registers. 

The output from the simulated RTL of the SPI interface is then compared with the testbench to see if we get the requisite output, which we do!

![alt text](Screenshot_20260523-091709.png)

We can confirm that the rtl has passed the Design verification tests.

GPIO-Mgmt :-

![alt text](image-3.png)

This is the successful output of the GPIO-Mgmt

We can clearly see the PASS recorded on the ouput.

Just like SPI-Master, The first step is compiling the code for risc-v ISA, specifically for 32 bit.

![alt text](image-4.png)

the freestanding flag is used to signify use of bare metal IC without OS.

Obviously the registers should not drive the external pins directly. They are connected to an output driver which increases the current output of the signal and can drive larger traces with ease. These also decouple any external disruptions to the traces from the internal circuitry protecting it. 

Pad transistors are used, generally larger and have thicker oxide, they are more resilliant than the transistors in the core.

UART:-

![alt text](image-5.png)

Analysing make:-

The make file is ridiculously modularized.

![alt text](image-6.png)

The makefile uses the name of the folder to name its files and find its files, hence my following some file nomenclature, you can easily use the same make file for a batch of designs instead of having to write individual makes for each.

A single design root is fixed and it is used as the sole source of truth for all the paths. This is also elegant modularization as if I decide to move files around on my computer, I can change a single line in the make file and it would immediately adapt to the new folder location.

![alt text](image-7.png)


We can see that there is pdk paths used in the makefile. However the verification as far as I can tell is strictly logic. My guess is that the same make could be used for further verification which is out of the scope for this activity, or to integrate it into other EDA systems like openlane or ORFS. It is better to inherit all the paths and maybe not use some rather than going through them individually and vetting them to decide if you need it.

![alt text](image-8.png)

We set the cpu in one variable and use different files according to the cpu used. This can be used to verify different CPUs with the same file by simply adding more 'ifeq's to this part of the make.

![alt text](image-9.png)

This is the first part of the make that accomplishes a step in verification. We are generating the firmware files and verifying it immediately.

We also verify that the linker scripts, startup source exists before starting generation using prepare-generated. we have also chained both the processes to always check prepare-generated before running check-fw. 

We can see flags used to communicate bare metal silicon again in the check-fw process.

![alt text](image-10.png)

What is a linker script?

A linker script is used in order to create the final memory image of the code you have compiled. In a general purpose computer like our desktops, this is decidedly done by the Operating systems itself. It can handle virtual memory and dynamic allocation. But on an embedded system, this is not possible as there is no Operating system. The firmware is the operating system. hence, it needs to know exactly which memory corresponds to which part of the computing stack. This is how memory is distributed and handled. 

To a processor itself, all of these are simply x-bit addresses but the firmware takes educated decisions on which data goes where precisely. Hence the linker scripts defines the memory mapping.

How does a linker script define memory mapping?

An IC often has cold and hard hardware limitations on it's design. Maybe it's uart can only read addresses inside a particular range or it's sram physically only has access for a particular range. These are strict rules that if the software ignores, it might lead to incoherant data, garbled values, unpredictable behavior. To prevent this, we create a stern memory map of the whole range and hand it to the firmware which would use it as a source of truth on where to store which data and when/how to access it when it is needed.

VVP Engine:-

Elaborates the verilog behaviour and instantiates the modules ready for simulation. We also include the hex and the testbench as it is part of the simulation along with the hardware. Just like choosing the CPU, here as well, we use ifeq to seperate full flows. This is another modularization attempt. Since our simulation is RTL, the other ifeqs like GL or GL_SDF are irrelevant to this simulation.

Some relevant flags seem here are -Ttyp, the typical timing corner is used. However, since this is a pure rtl verification, this barely matters. 

![alt text](image-11.png)

Finally we are running the simulation here. This toggles clocks and records the inputs and outputs. It handles the dump into the .vcd file. finally, it renames the output files to rtl-design for easier organization or more automation. 


Timer :-

First run

![alt text](image-12.png)

We can see that it failed, clearly.

I opened the waveform and realized that the first check, essentially setting the output to a particular value (0x0A) and seeing if it reflects succeeds...

![alt text](image-13.png)

However, the following 0x01 does not succeed. This means that the firmware is active and the timer/counter clearly is as it can be seen updating regularly. Thus I tried increasing the repeat time in the wait loop in the testbench to ease the timings on the firmware, and it worked immediately.

![alt text](image-14.png)

![alt text](image-15.png)


IRQ:-

IRQ stands for Interrupt request. This failed the DV. 

![alt text](image-16.png)

I tried to add more updates to the data variable to see if it was updating. And it was, except the GPIO State was not affected by it.

![alt text](image-17.png)

Also tried to add the data periodic variable and a delay after it as it loads and counts. 

I commented the asm Volatile line as it simply did not allow anything else to update independently. 

All these still did not fix it and hence this is just a design fail.

Debug:-

The first test failed.

![alt text](image-19.png)

I tried to extend the debug entry point, and that gave the Processor enough time to execute before entering debug state. 

![alt text](image-18.png)

That fixed it.

![alt text](image-20.png)

| tests-standalone | status (sky130) |
|------------------|-----------------|
| GPIO Mgmt        | PASS            |
| mem              | PASS            |
| uart             | PASS            |
| timer            | PASS            |
| irq              | FAIL            |
| debug            | PASS            |
| spi_master       | PASS            |

The CARAVEL tests:-

user_pass_thru:-

![alt text](image-21.png)

We can see successful execution. However, I did have to comment out some includes to get this working, from the includes.rtl.caravel specifically. There were includes to paths and files that simply did not exist from the git pull, hence I had to completely comment them out. Luckily they did not affect execution, else I would have had to find the files on git and add them manually.

uart:-

Since includes.rtl.caravel is global, fixing it once fixes the dependencies for all the modules to be tested for. Hence this executed first try.

![alt text](image-22.png)

sram_exec:-

![alt text](image-23.png)

sysctrl:-

![alt text](image-24.png)

spi-master:-

![alt text](image-25.png)

pullupdown:-

![alt text](image-26.png)

pll:-

![alt text](image-28.png)

I did have to modify the .c file to add delays to get the proper outputs.

![alt text](image-27.png)

pass_thru_fix:-

![alt text](image-29.png)

mem:-

![alt text](image-30.png)

hkspi_power:-

![alt text](image-31.png)

gpio_mgmt:-

![alt text](image-32.png)

hkspi:-

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


What do each of these modules mean?

user_pass_thru:-

this basically creates a direct pass through of signals on the IO interface. It can tell us if the IO connectivity and the padrame is working properly.

UART:-

Sends and receives bytes on the uart interface and checks if the sender and the receiver are synchronous and verifies it.

Sysctrl:-

Sysctrl checks global coordination among the different modules in the SOC. This includes:-
Reset control, clock control, peripherals, global config registers, boot config, interrupt coordination, power management.

sram-exec:-

Checks if the processor can retrieve bytes from the memory and execute it successfully. This also parallely verifies the connections between the cpu, memory controller and the sram itself. 

spi_master:-

Usually connects to external pins for interface with foreign devices. It is called SPI-master because this is the MASTER and not the slave in the configuration and hence also generates the clocks. SPI is very standardized because it is lightweight and simple.

pullupdown:-

The gpio pins are usually not left hanging when not driven. This is inherently bad because this can lead to unnexpected behaviour and antenna behavior. Thus, we usually enable a weak resistor to 1 or 0, namely VDD or GND. This is done to base the signal when not driven. It is intentionally weak as when driven, the pull up or down should not counter power the driving signal itself. In SOCs this is usually configurable.

pll:-

The basic idea of a PLL or a phase locked loop is simple. There is a reference clock usually way smaller and a voltage controlled oscillator. There are dividers at the input and output. And a feedback loop keeps the phase intact AFTER the target frequency is achieved. The dividers are small and light FSMs that reduce the clock frequencies from and to the voltage controlled oscillator. By controlling the dividers, I can essentially control the output frequency of the system. But this is very hard as each of these components have very strict limitations in frequency range of operation. hence, the entire device also has ranges in which they are properly functional. Outside that, they could have unpredictable behaviour.

mem:-

Checking if all the memory connections work. We write and read from the same memory addresses to see if the bytes were actually stored in that memory location. And this also verifies that the wire connections and the memory controller is functional. 

hkspi_power:-

This is for House keeping and power management. This involves checking and verifying the control and management subsystems. This helps boot and maintain the chip when the other modules perform specific tasks. As the name suggests, this particular module is for house keeping the SPI module, which includes managing the PLL registers and power settings. 

'how do you verify power from digital domain?'

Power is inherently analog and hence cannot be directly interpretted in digital. Instead we check power by triggering certain control signals and verifying the behavior of the systems and infer power from that rather than directly.

gpio_mgmt:-

This involves changing the meta states of the gpio pins and verifying if it works appropriately. This involves checking if the pin can toggle between input and outputs. It also verifies if the management SOC controls the pins and not the user modules directly. This would lead to contention issues. 

hkspi:-

Is the global module for SPI housekeeping. This is a super-set of the hkspi_power module. It verifies startup, debug path and infrastructure access.

Understanding the codes and the workflows

GPIO_MGMT:-

gpio_mgmt.c

The gpio interface works with a MMIO or memory mapped IO. The processor simply writes to a register which is attached to more analog hardware. We can see some configuration registers too.

![alt text](image-34.png)

here, the input and output has both been set. The .c file sets the pins to high and low. The test bench tests if there is a high and low one after each other and uses that to determine pass/fail.

Do note that since there is a firmware involved, the testbenches merely check the system and validate them instead of giving stimulus to the DUTs for all the modules.

user_pass_thru:-

Usually the housekeeping SPI, just interacts with the internal management registers. But here, they are decidedly routed out the external pins to observe its output and the connectivity between the wires are checked logically. The output pins are configured to the management output pins. 

![alt text](image-35.png)

UART:-

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

Sysctrl:-

This checks if the clocks of the chip is properly configured and can be sent to the GPIO output pins. The caravel SOC has multiple clock domains and verifying if both of them are accessible in any parts of the chip, especially at the outputs is necessary.

![alt text](image-41.png)

The first test does not enable either of the clocks, and monitors the outputs. We are expecting no pulses here.

In the second test, we check if the first system clock works.

In the third test, we check if the second user clock works and is functional. 

The delays wait for the edges to accumulate because the testbenches count these clocks at their edges to verify.

sram_exec:-

A processor usually boots and executes from a flash through an SPI. This verifies if it can do the same from SRAM. This code copies code into sram and execute from sram. The gpio pins are configured to interact with the TB and we can verify if it works from the TB.

![alt text](image-42.png)

the source and destination of the memory byte transfer is defined as the flash and the SRAM respectively. Then It is recursively copied. The CPU's pointer is then moved to the SRAM. 

The testbench has taken a secondary role in this module again. The firmware does most of the heavy-lifting.

The testbench observes the outputs through the GPIO pins and records the phase of execution. Note how we have also simultaneously checked the functioning of the pointers.

spi-master:-

Just like before, we set the data bits to something specific and check the Testbench can recognize that. We can use this to verify the execution stage by stage. 

We asset chip select because we intend to begin transmission.

The firmware uses wrapper functions for write and read simultaneously. Do note that SPI is full duplex, therefore SPI can receive and send bytes at the same time. However, since SPI is a master-slave relation, to receive some data, the master must send some dummy bits to receive data in return.

![alt text](image-43.png)

We instantiate two SPI flashes to interface them both. Now this firmware is talking to a Simulated external peripheral from the caravel SOC.

![alt text](image-44.png)

Pullupdown:-

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

PLL:-

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

mem:-

The firmware loads full 32 bit words into the memory and read them to check if they have been stored properly. 
The testbench checks both the data transmissions and verifies. 

It then does this for 8 and 16 bit data pieces as well. This is crucial because the processor does all mixed width accesses all the time.

hkspi_power:-

The test here is to verify if the housekeeping SPI can function when the user region is entirely powered off. The smoking gun is this line where the vdd is also gounded.

![alt text](image-53.png)

This is why this is one of the outliers in these modules where the testbenches are doing the grunt work. The testbench probes the housekeeping SPI. 

HKSPI:-

Again, the testbench is heavy lifting functionality here by manipulating the pins and spi by itself. However, the firmware keeps the uart channel open and transmits keeping the CPU enabled while the HKSPI works externally.

![alt text](image-55.png)

This is where the SPI pins are established in the testbench.

![alt text](image-54.png)

The hkspi then resets externally. then the reset is deasserted.

Flow chart representations of the modules:-

user_pass_thru:-

![alt text](Screenshot_20260528-201149.png)

uart:-

![alt text](Screenshot_20260529-082928.png)

sysctrl:-

![alt text](Screenshot_20260529-083942.png)

sram exec:-

![alt text](Screenshot_20260529-084347.png)

SPI_Master:-

![alt text](Screenshot_20260529-084910.png)

pullupdown:-

![alt text](Screenshot_20260529-085439.png)

pll:-

![alt text](Screenshot_20260529-090316.png)

pass through fix:-

![alt text](Screenshot_20260529-091047.png)

mem:-

![alt text](Screenshot_20260529-091326.png)

HKSPI_POWER:-

![alt text](Screenshot_20260529-091737.png)

HKSPI:-

![alt text](Screenshot_20260529-092153.png)

