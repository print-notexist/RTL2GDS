WEEK 4:-

Understanding the Wrapper Module:-

Let's start with the very top. The module is defined here at the very start. This is the module that would called in the Caravel ecosystem when building. We can also see that it has a parameter for the number of bits used. 

There is an immediate use of ifdef next. 

ifdef is used in order to define elements of a circuit as and when required. Now that these power pins are defined through ifdef, They would only get defined if there is a 'define USE_POWER_PINS' in the module. This allows us to 'pick and place' the definitions leveraging the preprocessor.

Another thing to note is defining the power pins itself. Ideologically, since RTL lives at the highest level of abstraction and PDN happens way below in the RTL2GDS process, it might seem odd to define power pins in an RTL. The reason for doing this is because this is not a standalone module. This is part of the larger caravel ecosystem. and that means that these power pins need to be exposed at this level of abstraction to properly define the properties

![alt text](image.png)

The logic analyser has an output enable for each output line. It is low active. 

Why use an output enable for EACH output wire instead of one globally for all output pins?
This is for granularity of control. Enabling or disabling 128 signals will get tiring to manage fast as you would have to deal with 128 output signals any time you want to check a tiny change even.

![alt text](image-1.png)

Another Interesting thing to note is how the IO pads have a variable that is not defined before in this verilog.

What is it then?

This is called a preprocessor macro. This allows us to set values for instantiations beyond the scope of the current verilog Module. Somewhere in the full ecosystem, there would be a definition for this Macro and that is how we are able to use it here without defining it. 

![alt text](image-3.png)

A quick grep search does validate this.

![alt text](image-2.png)

We can see definitions of an analog io even though RTL is entirely digital. This is because, if we want to fetch the exact analog voltages of our pins, then we can define the pins as analog so that we may add ADCs and DACs on them. It provides us the circuitry to interpret analog signals.

![alt text](image-4.png)

Here we can see the usage of ifndef. ifndef is the exact opposite of ifdef. This might sound redundant. But this is used specifically when we want exclusions. 

In this particular case, we are using the definition to route the debug output to the debug region and everything else to the user region. If GPIO_TESTING is defined, then this circuitry would not exist, and we can instead define a custom logic in its place.

![al text](image-5.png)

Both ifndef and ifdef tells us the importance of knowing the other modules you are interfacing with, especially in large integrations like these, to both have better compliance across modules and make the code writing easier. 

![alt text](image-6.png)

All of the used modules are used this module are defined in ifdefs and ifndefs, the nature of which has been discussed above for this reason.

Thus, these module instantiations might not happen if the definitions allign when this module (user_project_wrapper) itself is called.

Exporting User_Project_Wrapper to ORFS designs to run the flow along with all the needed dependencies: -

![alt text](image-7.png)

The flow claimed that there is simply not enough space for putting the required IO pins.
Hence I added a die area myself manually to the config.

![alt text](image-8.png)

ORIGINAL config

![alt text](image-12.png)

modified config

![alt text](image-13.png)

but this threw some errors too:-

![alt text](image-11.png)

This is essentially happening because I have mentioned both core utilization and the area itself. Which is mutually exclusive in this flow. If I define the core utilization, the flow would calculate the area, or vice versa, but I cannot ask for both simultanously.

I commented out the core utilization assignment

![alt text](image-14.png)

Next, I ended up with this error at CTS.

![alt text](image-15.png)

This happened because the Clock names were not modified from the copy pasted .sdc file. After modifying the names of the clock.

![alt text](image-16.png)

We move on to the next error in routing repair.

![alt text](image-17.png)

However, global routing shows that the nets are properly placed and the resizer is trying to fix about a thousand errors but... oddly enough, it is not changing anything in the iterations itself.

![alt text](image-18.png)

Due to the unusual behaviour observed only during the repair stage, while the remaining stages of the flow completed without reporting major issues, the error is more likely associated with the repair process itself rather than the design implementation.

Hence, We elect to skip over this phase to continue.

![alt text](image-19.png)

Final Output after all debugging

![alt text](image-20.png)

## Phase 5: Collection of RTL-to-GDS Implementation Outputs

| Output Category               | Generated File     | Description                                                                                                |
| ----------------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------- |
| Synthesized Design Database   | `1_synth.odb`      | OpenDB database generated after synthesis and technology mapping.                                          |
| Floorplan Database            | `2_floorplan.odb`  | Database containing floorplan information including die/core dimensions and initial placement constraints. |
| Placement Database            | `3_place.odb`      | Database after standard-cell placement optimization.                                                       |
| Clock Tree Synthesis Database | `4_cts.odb`        | Database after clock tree insertion and clock network optimization.                                        |
| Global Routing Database       | `5_1_grt.odb`      | Database after global routing stage.                                                                       |
| Detailed Routing Database     | `5_2_route.odb`    | Database after detailed routing completion.                                                                |
| Fill Cell Database            | `6_1_fill.odb`     | Database after fill-cell insertion and physical finishing operations.                                      |
| Final DEF                     | `6_final.def`      | Final Design Exchange Format file containing complete physical implementation information.                 |
| Final Verilog Netlist         | `6_final.v`        | Post-implementation gate-level netlist generated by ORFS.                                                  |
| Final SDC                     | `6_final.sdc`      | Final timing constraints file used for implementation and signoff.                                         |
| Final SPEF                    | `6_final.spef`     | Standard Parasitic Exchange Format file containing extracted RC parasitics.                                |
| Global Routing Guide          | `route.guide`      | Routing guide generated during the global routing stage.                                                   |
| Final GDSII Layout            | `6_final.gds`      | Final manufacturable GDSII layout generated after DEF-to-GDS conversion and library merge.                 |
| Merged GDSII Layout           | `6_1_merged.gds`   | Intermediate merged GDS generated during the final layout assembly stage.                                  |
| Timing Constraint Reference   | `updated_clks.sdc` | Updated clock constraint file generated during timing optimization.                                        |
| Memory Statistics             | `mem.json`         | Resource utilization and memory statistics generated during implementation.                                |

### Implementation Summary

The RTL-to-GDS flow for `user_project_wrapper` was successfully completed using OpenROAD Flow Scripts (ORFS). The flow progressed through synthesis, floorplanning, placement, clock tree synthesis, global routing, detailed routing, fill-cell insertion, parasitic extraction, and final GDSII generation. The final implementation artifacts include the gate-level netlist (`6_final.v`), extracted parasitics (`6_final.spef`), final DEF (`6_final.def`), and manufacturable GDSII layout (`6_final.gds`).

 
Final GDS Output:-

After getting the Original GDS output, It looked suspiciously uniform, with almost identical standard cells spanning across the design. I figured that the area was too big and the utilization has fallen to just one percent. Hence, I dramatically reduced the core and die area in order to improve the design's Area efficiency.

![alt text](image-22.png)

The layout did open on klayout, but the routing and cells were more visible here, hence openroad -gui for the screenshot.

The parameters of the design:-

Area : 6320 um2
Final Die Area :- 180 x 180
Utilization :- 33%
instances = 518
vias = 3278 with a maximum metal layer of 3
wirelength = 26344um
no slack violations
