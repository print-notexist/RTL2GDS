
Phase 1 

We can see that the flow was completed entirely. 

![alt text](image.png)

This netlist was selected because it is the final gate-level implementation generated after completion of the RTL-to-GDS flow. It includes all synthesis, placement, clock tree synthesis, and routing optimizations and therefore represents the final design used for GLS verification.

![alt text](image-1.png)

By filtering with the technology library names, we can see that the netlist has library specific gates.


First We copy the generated netlist to __user_project_wrapper which previously held the raw rtl of the design.

![alt text](image-2.png)

The next part was dependency fixing. For GL, The make expects various files which are either in the wrong place or simply does not exist. 

These include 
mgmt_core which was in a different location which I simply copied

This is DFFRAM which never existed in the git repo so I commented out the requirement.

![alt text](image-3.png)

sky130_sram_2kbyte_1rw1r_32x512_8.v was another missing dependency which did exist so I simply copied it.

UART successful simulation

![alt text](image-4.png)

![alt text](image-7.png)

Timer successful simulation

![alt text](image-5.png)

![alt text](image-8.png)

Debug successful simulation

![alt text](image-6.png)

![alt text](image-9.png)

gpio_mgmt successful simulation

![alt text](image-10.png)

![alt text](image-11.png)

irq failed simulation

![](image-12.png)

Reason: The RTL itself failed back in week3. There were unsuccessful debugging efforts back in RTL. Hence this is to be expected. But this is still a substantially better output than originally observed before week 3 debugging, which was stuck in a single status for the whole run.

![alt text](image-13.png)

Mem failed simulation:-

Mem failed simulation. It managed to reach stage one which is A040 but never progressed after that

![alt text](image-15.png)

![alt text](image-14.png)

Spi-master successful simulation

![alt text](image-16.png)

![alt text](image-17.png)

| Test       | Status (Sky130) |
| ---------- | --------------- |
| GPIO Mgmt  | PASS            |
| mem        | FAIL            |
| uart       | PASS            |
| timer      | PASS            |
| irq        | FAIL            |
| debug      | PASS            |
| spi_master | PASS            |

Caravel Integrated simulations


Originally, I had issues that did not allow me to run the flow directly. This was due to the fact that the final verilog file does not have power pins.

![alt text](image-19.png)

The -DUSE_POWER_PINS flag determines if power pins are included in the design through ifdefs.

After adding the flag to the config.mk of the ORFS flow.

![alt text](image-20.png)

gpio-mgmt

![alt text](image-18.png)

![alt text](image-21.png)

hkspi

![alt text](image-22.png)

![alt text](image-23.png)

hkspi-power

![alt text](image-24.png)

![alt text](image-25.png)

mem

![alt text](image-26.png)

![alt text](image-27.png)



pass-thru

![alt text](image-28.png)

![alt text](image-29.png)

pass-thru fix

![alt text](image-30.png)

![alt text](image-31.png)

PLL

![alt text](image-32.png)

![alt text](image-33.png)

pullupdown

![alt text](image-34.png)

![alt text](image-35.png)

spi_master:

![alt text](image-36.png)

![alt text](image-37.png)

sram exec:-

![alt text](image-38.png)

![alt text](image-39.png)

uart:-

![alt text](image-40.png)

![alt text](image-41.png)

sysctrl

![alt text](image-42.png)

sysctrl failed back in rtl itself so this is not that surprising of a result.

![alt text](image-44.png)

user_pass_thru:-

![alt text](image-45.png)

![alt text](image-46.png)

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
| mem | FAIL |
| hkspi_power | PASS |
| gpio_mgmt | PASS |
| hkspi | PASS |