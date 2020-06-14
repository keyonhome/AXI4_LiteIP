# AXI4_LiteIP
This is a user defined verilog IP core for FPGA transcation via AXI4_Lite from slave side, and the TB attached will simulate the process of write&read of a RAM.
# Folder structure
"AXI4demo" is the Ip core from XILINX and "myAXI4IP" is the top level design with instantiating the AXIdemo.
And the TB is the testbench for baisic transcation.

# Code 
 The signal of AXI4_Lite listed below:
![image](https://github.com/keyonhome/AXI4_LiteIP/blob/master/img/Liteport.png)

## CRS rigister( Configure & Status Report )
  In an SoC system, the processor needs to configure a large number of peripheral configuration registers, and read the status of the peripheral from the peripheral's status register to perform operations such as initialization completion judgment.
  The AXI interface module will latch the AXI data from the master, generally the processor, into the register, and convert the value into a logic level signal and output it to the peripheral module. The signal from the peripheral device is also latched into the register. When the master sends a read request, the value of the status register is sent to the host in the form of AXI read data channel. The AXI interface module plays two roles in the system: the configuration input of the configuration register and the status reporting of the status register.
''' verilog
reg [C_S_AXI_DATA_WIDTH-1:0]   slv_reg0;
reg [C_S_AXI_DATA_WIDTH-1:0]   slv_reg1;
'''
