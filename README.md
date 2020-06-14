# AXI4_LiteIP
 This is an 'user defined' verilog IP core for FPGA transcation via 'AXI4_Lite' from 'slave side', and the TB attached will simulate the process of write&read of a RAM.
# Folder structure
 "AXI4demo" is the Ip core from XILINX and "myAXI4IP" is the top level design with instantiating the AXIdemo.
And the TB is the testbench for baisic transcation.

# Code 
   The signal of AXI4_Lite listed below:[DataSheet from Xilinx](https://forums.xilinx.com/t5/Design-and-Debug-Techniques-Blog/AXI-Basics-5-Create-an-AXI4-Lite-Sniffer-IP-to-use-in-Xilinx/ba-p/1064306)
 
![image](https://github.com/keyonhome/AXI4_LiteIP/blob/master/img/Liteport.png)

## CRS rigisters( Configure & Status Report )
   In an SoC system, the processor needs to configure a large number of peripheral configuration registers, and read the status of the peripheral from the peripheral's status register to perform operations such as initialization completion judgment.<br>  
    The AXI interface module will latch the AXI data from the master, generally the processor, into the register, and convert the value into a logic level signal and output it to the peripheral module. The signal from the peripheral device is also latched into the register. When the master sends a read request, the value of the status register is sent to the host in the form of AXI read data channel. The AXI interface module plays two roles in the system: the configuration input of the configuration register and the status reporting of the status register.  
``` Verilog
//-- Number of Slave Registers 4
	reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg0;
	reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg1;
	reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg2;
	reg [C_S_AXI_DATA_WIDTH-1:0]	slv_reg3;
```
    For this module We predefine the dataBUS wide and Address Wide as below:
```Verilog
		parameter integer C_S_AXI_DATA_WIDTH	= 32,
		// Width of S_AXI address bus
		parameter integer C_S_AXI_ADDR_WIDTH	= 4
```
   >Here the address data units in byte and it is decided by the number of the registors and the width of each register.(4 registers and each gets 4 bytes)
   >>Waddr = log2(16) = 4 bit
    
In the bus operation, the chip select read & write registers are in units of registers rather than bytes, so when judging the register to be read and written by the address, only the upper 2 bits of the address need to be judged to select 4 registers. <br>
After selecting the register, can also use the 'WSTRB' field to control the writing enable of the specified byte in the register. 

## Interface timing implementation
 By analyse the process of writting ,to learn the protocol
 ![imge](https://github.com/keyonhome/AXI4_LiteIP/blob/master/img/AXIwrite.png)
 Check the level of 'awvalid' and 'wvalid' signals. When the master is ready on the write address and write data channel, the slave sets the 'awready' signal high to complete an address transmission. The same as the above '~awready' because the transmission only needs to be valid on the rising edge of the clock once(brust = 1 for AXI4_lite), and the ready signal should be set high at the same time. In the next cycle, the slave needs to set the 'awready' signal low, and the master generally also sets the 'awvalid' signal low.
 ```Verilog
 always @( posedge S_AXI_ACLK )
	begin
	  if ( S_AXI_ARESETN == 1'b0 )
	    begin
	      axi_awready <= 1'b0;
	      aw_en <= 1'b1;
	    end 
	  else
	    begin    
	      if (~axi_awready && S_AXI_AWVALID && S_AXI_WVALID && aw_en)
	        begin
	          // slave is ready to accept write address when 
	          // there is a valid write address and write data
	          // on the write address and data bus. This design 
	          // expects no outstanding transactions. 
	          axi_awready <= 1'b1;
	          aw_en <= 1'b0;
	        end
	        else if (S_AXI_BREADY && axi_bvalid)
	            begin
	              aw_en <= 1'b1;
	              axi_awready <= 1'b0;
	            end
	      else           
	        begin
	          axi_awready <= 1'b0;
	        end
	    end 
	end       
 ```
 
 When the slave sets up the 'awready' signal to complete the address channel transmission, the slave will also latch the current write address from the write address. The latched write address will be used to determine the register to be written in the following clock cycle.
 ```Verilog
 // Implement axi_awaddr latching
	// This process is used to latch the address when both 
	// S_AXI_AWVALID and S_AXI_WVALID are valid. 

	always @( posedge S_AXI_ACLK )
	begin
	  if ( S_AXI_ARESETN == 1'b0 )
	    begin
	      axi_awaddr <= 0;
	    end 
	  else
	    begin    
	      if (~axi_awready && S_AXI_AWVALID && S_AXI_WVALID && aw_en)
	        begin
	          // Write Address latching 
	          axi_awaddr <= S_AXI_AWADDR;
	        end
	    end 
	end       
 ```

 ```
 Next step is the most important for the overall operation which is wirte data into the registers
 ```Verilog
 assign slv_reg_wren = axi_wready && S_AXI_WVALID && axi_awready && S_AXI_AWVALID;

	always @( posedge S_AXI_ACLK )
	begin
	  if ( S_AXI_ARESETN == 1'b0 )
	    begin
	      slv_reg0 <= 0;
	      slv_reg1 <= 0;
	      slv_reg2 <= 0;
	      slv_reg3 <= 0;
	    end 
	  else begin
	    if (slv_reg_wren)
	      begin
	        case ( axi_awaddr[ADDR_LSB+OPT_MEM_ADDR_BITS:ADDR_LSB] )
	          2'h0:
	            for ( byte_index = 0; byte_index <= (C_S_AXI_DATA_WIDTH/8)-1; byte_index = byte_index+1 )
	              if ( S_AXI_WSTRB[byte_index] == 1 ) begin
	                // Respective byte enables are asserted as per write strobes 
	                // Slave register 0
	                slv_reg0[(byte_index*8) +: 8] <= S_AXI_WDATA[(byte_index*8) +: 8];
	              end  
	          2'h1:
	            for ( byte_index = 0; byte_index <= (C_S_AXI_DATA_WIDTH/8)-1; byte_index = byte_index+1 )
	              if ( S_AXI_WSTRB[byte_index] == 1 ) begin
	                // Respective byte enables are asserted as per write strobes 
	                // Slave register 1
	                slv_reg1[(byte_index*8) +: 8] <= S_AXI_WDATA[(byte_index*8) +: 8];
	              end  
	          2'h2:
	            for ( byte_index = 0; byte_index <= (C_S_AXI_DATA_WIDTH/8)-1; byte_index = byte_index+1 )
	              if ( S_AXI_WSTRB[byte_index] == 1 ) begin
	                // Respective byte enables are asserted as per write strobes 
	                // Slave register 2
	                slv_reg2[(byte_index*8) +: 8] <= S_AXI_WDATA[(byte_index*8) +: 8];
	              end  
	          2'h3:
	            for ( byte_index = 0; byte_index <= (C_S_AXI_DATA_WIDTH/8)-1; byte_index = byte_index+1 )
	              if ( S_AXI_WSTRB[byte_index] == 1 ) begin
	                // Respective byte enables are asserted as per write strobes 
	                // Slave register 3
	                slv_reg3[(byte_index*8) +: 8] <= S_AXI_WDATA[(byte_index*8) +: 8];
	              end  
	          default : begin
	                      slv_reg0 <= slv_reg0;
	                      slv_reg1 <= slv_reg1;
	                      slv_reg2 <= slv_reg2;
	                      slv_reg3 <= slv_reg3;
	                    end
	        endcase
	      end
	  end
	end    
 ```
 
 >axi_awaddr[ADDR_LSB+OPT_MEM_ADDR_BITS:ADDR_LSB]
 The slice operation is to manipulate which register it gonna write. For this case, we hvae 4 registers so the 'OPT_MEM_ADDR_BITS' is 2 and it will be used as a chip select signal to select 4 registers.
 
 The last setp is to respond for this transcation to end this transcation and ready for next.
 ```verilog
 always @( posedge S_AXI_ACLK )
	begin
	  if ( S_AXI_ARESETN == 1'b0 )
	    begin
	      axi_bvalid  <= 0;
	      axi_bresp   <= 2'b0;
	    end 
	  else
	    begin    
	      if (axi_awready && S_AXI_AWVALID && ~axi_bvalid && axi_wready && S_AXI_WVALID)
	        begin
	          // indicates a valid write response is available
	          axi_bvalid <= 1'b1;
	          axi_bresp  <= 2'b0; // 'OKAY' response 
	        end                   // work error responses in future
	      else
	        begin
	          if (S_AXI_BREADY && axi_bvalid) 
	            //check if bready is asserted while bvalid is high) 
	            //(there is a possibility that bready is always asserted high)   
	            begin
	              axi_bvalid <= 1'b0; 
	            end  
	        end
	    end
	end   
 ```
 
 After completing the transfer,the next cycle after the write data/address channel completes the handshake, the slave sets the 'bvalid' signal and gives an'OK' signal on the 'bresp' signal. After the response channel completes the handshake, set 'bvalid' low to complete the response operation.
 
