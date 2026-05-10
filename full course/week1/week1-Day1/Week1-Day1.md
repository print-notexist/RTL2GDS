Results for WEEK1:-

#1)Setup and Verification that the Codespace is working and Openlane is setup.

![alt text](image.png)

#2)Running a custom design(picorv32)

![alt text](image-1.png)

succesful output - Full layout render of the picorv32a core(zoomed in):-

![alt text](image-2.png)

Understanding the directory of an Openlane project

A higher version package already exists & finding the verilog .v and .sdc files of the project. 

![alt text](image-3.png)

Priority of the configurations:-
SCL_130A._config.tcl > config.tcl > Openlane config

Setting up the file system using 'prep'

![alt text](image-4.png)

Exploring the structure of the each run folder and its contents

![alt text](image-5.png)

The config folder inside the runs folder with ALL the defaults

![alt text](image-6.png)


Synthesis successful run

![alt text](image-8.png)

Synthesis.errors 

![alt text](image-7.png)

Synthesis.log

![alt text](image-9.png)

Screenshot of the final area

![alt text](image-10.png)


Flop Ratio 

number of cells = 15762

![alt text](image-11.png)

Number of D flip flops = 1613

Flop ratio = 10.23%

Mappings inside the results folder/Synthesis/picorv32a

![alt text](image-12.png)

Statistics inside the reports folder - filename : 1-synthesis.AREA_0.chk.rpt

![alt text](image-13.png)

Timing statistics logs/2-sta.log

![alt text](image-14.png)

