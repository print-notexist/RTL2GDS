Creating the lef file to attach to the picorv32a model

![alt text](image.png)


After making the custom layout, the layout  lef and .lib files have been copied to the src location where they will be used to be linked up to the design itself
![alt text](image-1.png)

The config has been modified to include the new lef files:-

![alt text](image-2.png)

The lefs have been linked by using the CLI interface.

![alt text](image-3.png)

We can see that the new Design has been incorporated into the flow successfully(LT130_inv)

![alt text](image-5.png)

There are no slack and timing violations of the design:-

![alt text](image-4.png)

Power Gating CTS:-

And or Or gates can be used to gate the clock signals

![alt text](image-6.png)

Each element was taken out of of the circuit, given a series of input slews and output capacitances and figured out the delay for that

![alt text](image-7.png)

By changing the Lef file for the inv, I did not notice a change in delay despite having conclusive proof for incorporating the custom inverter file.

However, there was a huge delay and slack violation seen in the tutorial. The following is not me following but rather just learning how to deal with such slack violations as this would be my first encounter of it.

Step 1) Go through the synthesis variables.

one variable we could tweak is the Synth stragtegy variable. 0,1 are delay oriented while 2,3 are area oriented.

Synth_buffering - adds buffers to drive high fanout pins with higher driving power.

Synth_sizing - up or down sizing buffers to optimize for area and compensate for weak driven signals.

Synth Driving Cell - cells driving the inputs.

After the current changes, we can see that the area has increased, likely as a result of the synth strategy variable.

![alt text](image-8.png)

The worst and total slack has fallen drastically, proving that taampering the variables has worked. It is still on the downside and needs more fixing.

![alt text](image-9.png)

Static timing analysis:-

![alt text](image-10.png)

In this case, there is a launch and capture flip flop, and there is a combinational logic in between. So any signal that comes into the circuit has to arrive at the D port of the launch flop, pass hold time, wait for the clock, propogate through the launch flop, exit through the Q port, go through the combinational logic and repeat the exact same process at the capture flop. All within the given time period T.

A D flip flop is basically two muxes. At time 0, when the signal and clock is introduced into the circuit, A small delay is added to the signal between D and Qm. When clock is shifted from logic 0 to logic 1,

![alt text](image-11.png)

Now for the OpenSTA run.

Originally the run was done with the pre_sta.conf file and a base.sdc replicated from the tutorial.

![alt text](image-12.png)

![alt text](image-13.png)

However, this run failed with a huge slack violation.

![alt text](image-17.png)

But the Openlane's STA reported a 520 slack margin

![alt text](image-15.png)

But after debugging, I figured out that both the implementations were using different corners. The STA look a more pessimistic approach and tried to use the slowest library for slack and the fastest for hold, the most pessimistic case possible.

After changing the .conf file in order to use the typical corners instead... 

![alt text](image-16.png)

I ended up with this.

![alt text](image-18.png)

This however was not convincing, the difference was still roughly 2.5 times, that is not normal. And then after more debugging, i figured out that openlane adds some non ideal effects on its own and OpenSTA being a fresh run lacked all of this. So I added these variables with the values seen from the openlane sta reports to make up for the difference.

![alt text](image-19.png)

finally, the difference dropped to something reasonable.

![alt text](image-20.png)

The remainind differeences originate from even more parameters mismatching but since the difference is a lot less significant, I ended my debugging here.

in the tutorial, we can see that the negative slack is controlled by reducing the fanout. However, in the tutorial, they did manage to get identical timing reports which makes this easier.

Here we are seeing the STA's CLI at work. These could be helpful to modify a rogue timing error.

![alt text](image-21.png)

In this flip flop clock provision, t2 will be greater than t1. This is due to very obvious reasons. The wire leading to t2 has higher capacitance and resistance due to the fact that it is longer.

![alt text](image-22.png)

The difference between t1 and t2 is called the skew and we usually try to minimize this. This is done by introducing intentional delays in shorter circuits.

To build clock networks that has less skews, we build something called the H-tree. This is done by splitting the clock signals many times in equal lengths to make sure that the recipient cells receive clock signals very close to each other. 

It is to be noted that clock skew and clock delay are usually inversely proportional

The clocks actually end up with a decent amount of slew loss as they pass through long wires inside the IC. This is why the clocks need to be buffered and repeated. 

Timing reports change dramatically as these non ideal clocks are taken into account.

Clock net shielding:-

In order to make sure that cross talk does not decimate vital clock signals, we need to shield it. 

Let's assume that a strong aggressor is transitioning its state. A glitch can cause this to couple with a victim wire which will record a temporary dip in voltage. If this happens in a crucial line like reset, it might end up disorienting the entire unit from the Chip. And in general, it would take each of these signals significantly longer to converge as they have to account for all these possible glitches before the next clock cycle begins.

![alt text](image-23.png)

Here we can see a representation of the aforementioned glitch resulting in cross talks when both the lines are trying to transition at roughly the same time. This negatively impacts slew and adds unnecessary jerks to the signal transition. We try to avoid this.

![alt text](image-24.png)

Successful run of CTS or clock tree synthesis:-

![alt text](image-25.png)

timing report of the same

![alt text](image-26.png)

We are now seeing behind each of these commands. run_cts for instance is enabled by a single command but has a lot of steps that it follows in order.

To see this, we head to OpenLane/scripts/tcl_commands where we see the commands native to each of the steps in the rtl2gds process.

![alt text](image-27.png)

If we open any of these files, lets say cts.tcl for instance, we can see that the command itself is a 'proc' or a procedure. This can be thought of as a function. It is called procedure instead of function as the language itself is extremely old and uses older notations. Note that tcl was created in 1988.

The tutorial asks us why there is no synthesis.tcl for instance in the openroad folder. But I am given a few moments to brainstorm. I believe it is because synthesis is not one of openroad's functionalities. I specifically remember using yosys for running the synthesis so it cannot possibly be under openroad.

![alt text](image-28.png)

I seem to be right!

![alt text](image-29.png)

Hold Analysis:-

Hold condition states that the combinational delay should atleast be of some value x for which the clocked elements require stability.

The time taken in the mux2 to send the data from Qm to Q is called the hold time. The flip flop as a whole needs to maintain the same data levels during this situation when the muxes are in analog limbo. If not, the flop could fall into metastability where we dont know what the state of the flop could be for certainty.

We continue doing the STA flow which is incorporated in openroad UNDER openlane

Why?
by using this configuration, we never leave the openroad flow and hence we can use all the environmental variables that are endemic to the openlane flow that we cannot otherwise.

![alt text](image-30.png)

![alt text](image-31.png)

The list of buffers has also been modified to remove the weakest one on the first run itself(because I read the word doc first)

![alt text](image-33.png)

The slack ended up becoming better after this. My understanding of this situation is that before CTS, the clock tree doesnt exist yet but to run the remaining logic, pessimistic estimations and assumptions are made by the earlier tools in order to do their respective functions. When the clock tree is finally built, the actual tree is usually less pessimistic than these assumptions and helps our timings

![alt text](image-32.png)

