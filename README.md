# Mapping HPS Peripherals, like I²C or CAN, over the FPGA fabric to FPGA I/O and using embedded Linux to control them

[Head Image]


Development Board’s for Intel SoC-FPGA allow often only a simple access to FPGA I/O-Pins. 
The only way to use HPS-Hard-IP Interfaces with these board’s are the routing over the FPGA fabric.

For example, has the Terasic DE10-Nano development Board an Arduino UNO compatible socket. 
This is a nice feature, because it is possible to use nearly any Arduino Shield from the huge Arduino community with SoC-FPGA’s. 
Here are also all Digital Pins connected to FPGA I/O Pins. 
With the Intel NIOSII-Softcore processor and some Soft-IP it is not too complicated to interact with these shields. 
But if the target applications require the connection of the capabilities of an embedded Linux system, running on the HPS, 
it will be. One reason is, that I only could find an official guide (AN 706), 
with a golden reference design a part of the required development steps.

With this following step by step guide I will give a complete description and I will also show as an example, 
how to transmit CAN-Bus Packages with a simple Linux Python script by using an Arduino Shield with the Terasic DE10-Nano board.

This project is a part of my rsYocto embedded Linux system, 
that I developed during my master study and I will it also use it here.

The build flow of rsYocto show the interaction of all for this project required components.

[Build Flow Image]

# 1. part: Creating the HPS FPGA-configuration 

* Use Terasic’s System Builder to generate an Intel Quartus Prime project with right I/O-Pin mapping
* Open this Quartus Prime Project and create a new Platform Designer model
* Insert the component “Arria V/Cyclone V Hard Processor System” to this model
* It should look like this:
	[Image 1]
* Note: Do not change the name “hps_0”!  With a different name could occur some errors with in the device tree building process of the Linux system with Intel EDS

### HPS component configuration 

*  Open the **HPS component Editor** and select the “**FPGA Interfaces**”-Tab
* Here you can configure the system like your wishes
* It makes sense to enable all AXI Bridges, they are not required for this project, but if a Linux application tries to access a disabled bridge address a Linux Kernel panic will be occur.
* I chose following configuration:
	[Image 2]
* Special Resets-, Interrupt- or DMA-Settings are not required
* Continue with the “**Peripheral Pins**”-Tab
* For the Usage of DE10-Nano`s HPS component’s choose following settings:

	| **Peripheral Name** | **Startus** | **Addional Settings**
	|:--|:--|:--|
	| Ethernet Media Access Controller 1 (**EMAC1**) | *HPS I/O Set 0* | *EMAC1 mode: "RGMII"*
	| SD/MMC Controller (**SDIO**) | *HPS I/O Set 0* | *SDIO mode ="4-bit Data"*
	|USB Controller 1 (**USB1**) | *HPS I/O Set 0* | *USB1 PHY interface mode="SDR with PHY clock output mode"*
	| UART Controller 0 (**UART0**) | *HPS I/O Set 0* | *UART0 mode="No Flow Control"*
	| I2C Controller 0 (**I2C0 **) | *HPS I/O Set 0* | *I2C0 mode="I2C"*

* To map the I²C-, SPI- , UART- and CAN-Bus select these points addionaly:
	| **Peripheral Name** | **Startus** | **Addional Settings**
	|:--|:--|:--|
	| UART Controller 1 (**UART1**) | *FPGA* | *UART0 mode="Full"*
	| I2C Controller 1 (**I2C1**) | *FPGA* | *I2C0 mode="Full"*
	| SPI Controller 0 (**SPI0**) | *FPGA* | *I2C0 mode="Full"*
	| CAN Controller 0 (**CAN0**) | *FPGA* | *I2C0 mode="Full"*
* Screenshots of my configuration are available here [Image 3]  and here [Image 4]
* Open the “**SDRAM**”-Tab 
  ( Do not chnage the settings of the "HPS-Cock"-Tab)
* In my Github repository is the **.qprs-memory configuration file** "D10STDNANO_DDR3.qprs" insired
* Copy this file to your Quartus Prime project folder and click inside the HPS comenent editor and **presents**-bar the "**new**"-buttor
     [Image 5]
* A **new pressend window** appers selcted select the .qprs-file and give it a name
     [Image 6]
* Again, close the pressend window, select in the list your pressend and click aply to execute the configration
* At this point is the HPS configuration done (Press **Finish** to close the window)

### HPS interconnection
 * The only nessary connection to use the HPS is: **clk_0:clk_in_reset --> hps_0:h2f_cold_reset**
 * Export the **SPI-, UART-,I2C- CAN-, memory** and **hps_IO** by douple clicking on the *export text filed*
	 [Image 7]
* Addionally connect  **PIO (Parallel I/O) Intel FPGA IP**-computents to a **HPS-to-FPGA interface** to LEDs or key-switches
	 [Image 8]
	 
* Generate with "**System/Assign Base Addresses**" memory address of the HPS Bridge interfaces
* Finally press the "**Generate HDL...**"-Button to build the final plattform

### Import the HPS Platform to Quartus Prime
 * Add the "*.qip*"- and the "*.sip*" to the project files as normal
 * Use following verilog code inisde the top level file to import the HPS-model:
	 ````verilog
	 base_hps u0 (
	/////////////////////////////////////////////////// CLOCKS //////////////////////////////////////////////////////
			.clk_clk                            	 	(CLOCK_50),       
			                   
	/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	  
	//////////////////////////////////////////// 	HPS    //////////////////////////////////////////////////////////  
	////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

	///////////////////////  Onboard DDR3 1GB Memmory  ///////////////////////////////////
	      .hps_0_ddr_mem_a                          ( HPS_DDR3_ADDR     ),                     
	      .hps_0_ddr_mem_ba                         ( HPS_DDR3_BA       ),                        
	      .hps_0_ddr_mem_ck                         ( HPS_DDR3_CK_P     ),                       
	      .hps_0_ddr_mem_ck_n                       ( HPS_DDR3_CK_N     ),                       
	      .hps_0_ddr_mem_cke                        ( HPS_DDR3_CKE      ),                        
	      .hps_0_ddr_mem_cs_n                       ( HPS_DDR3_CS_N     ),                    
	      .hps_0_ddr_mem_ras_n                      ( HPS_DDR3_RAS_N    ),                      
	      .hps_0_ddr_mem_cas_n                      ( HPS_DDR3_CAS_N    ),                      
	      .hps_0_ddr_mem_we_n                       ( HPS_DDR3_WE_N     ),                      
	      .hps_0_ddr_mem_reset_n                    ( HPS_DDR3_RESET_N  ),                    
	      .hps_0_ddr_mem_dq                         ( HPS_DDR3_DQ       ),                        
	      .hps_0_ddr_mem_dqs                        ( HPS_DDR3_DQS_P    ),                      
	      .hps_0_ddr_mem_dqs_n                      ( HPS_DDR3_DQS_N    ),                      
	      .hps_0_ddr_mem_odt                        ( HPS_DDR3_ODT      ),                        
	      .hps_0_ddr_mem_dm                         ( HPS_DDR3_DM       ),                         
	      .hps_0_ddr_oct_rzqin                      ( HPS_DDR3_RZQ      ),                         
	      
	 /////////////////////////////////////////// HPS Ethernet 1  //////////////////////////////////////////////    
	      .hps_0_io_hps_io_emac1_inst_TX_CLK ( HPS_ENET_GTX_CLK    ),     
	      .hps_0_io_hps_io_emac1_inst_TXD0   ( HPS_ENET_TX_DATA[0] ),
	      .hps_0_io_hps_io_emac1_inst_TXD1   ( HPS_ENET_TX_DATA[1] ),   
	      .hps_0_io_hps_io_emac1_inst_TXD2   ( HPS_ENET_TX_DATA[2] ),   
	      .hps_0_io_hps_io_emac1_inst_TXD3   ( HPS_ENET_TX_DATA[3] ),  
	      .hps_0_io_hps_io_emac1_inst_RXD0   ( HPS_ENET_RX_DATA[0] ),  
	      .hps_0_io_hps_io_emac1_inst_MDIO   ( HPS_ENET_MDIO       ),  
	      .hps_0_io_hps_io_emac1_inst_MDC    ( HPS_ENET_MDC        ),        
	      .hps_0_io_hps_io_emac1_inst_RX_CTL ( HPS_ENET_RX_DV      ),        
	      .hps_0_io_hps_io_emac1_inst_TX_CTL ( HPS_ENET_TX_EN      ),       
	      .hps_0_io_hps_io_emac1_inst_RX_CLK ( HPS_ENET_RX_CLK     ),       
	      .hps_0_io_hps_io_emac1_inst_RXD1   ( HPS_ENET_RX_DATA[1] ),  
	      .hps_0_io_hps_io_emac1_inst_RXD2   ( HPS_ENET_RX_DATA[2] ),   
	      .hps_0_io_hps_io_emac1_inst_RXD3   ( HPS_ENET_RX_DATA[3] ),  

	/////////////////////////////////////// SD Card (boot drive)  ////////////////////////////////////////// 
	      .hps_0_io_hps_io_sdio_inst_CMD     ( HPS_SD_CMD         ),          
	      .hps_0_io_hps_io_sdio_inst_D0      ( HPS_SD_DATA[0]     ),     
	      .hps_0_io_hps_io_sdio_inst_D1      ( HPS_SD_DATA[1]     ),     
	      .hps_0_io_hps_io_sdio_inst_CLK     ( HPS_SD_CLK   	     ),            
	      .hps_0_io_hps_io_sdio_inst_D2      ( HPS_SD_DATA[2]     ),      
	      .hps_0_io_hps_io_sdio_inst_D3      ( HPS_SD_DATA[3]     ),    
	        
	////////////////////////////////////////// 	USB HOST 	////////////////////////////////////////////  
	      .hps_0_io_hps_io_usb1_inst_D0      ( HPS_USB_DATA[0]    ),      
	      .hps_0_io_hps_io_usb1_inst_D1      ( HPS_USB_DATA[1]    ),      
	      .hps_0_io_hps_io_usb1_inst_D2      ( HPS_USB_DATA[2]    ),      
	      .hps_0_io_hps_io_usb1_inst_D3      ( HPS_USB_DATA[3]    ),     
	      .hps_0_io_hps_io_usb1_inst_D4      ( HPS_USB_DATA[4]    ),      
	      .hps_0_io_hps_io_usb1_inst_D5      ( HPS_USB_DATA[5]    ),     
	      .hps_0_io_hps_io_usb1_inst_D6      ( HPS_USB_DATA[6]    ),      
	      .hps_0_io_hps_io_usb1_inst_D7      ( HPS_USB_DATA[7]    ),     
	      .hps_0_io_hps_io_usb1_inst_CLK     ( HPS_USB_CLKOUT     ),     
	      .hps_0_io_hps_io_usb1_inst_STP     ( HPS_USB_STP	     ),         
	      .hps_0_io_hps_io_usb1_inst_DIR     ( HPS_USB_DIR        ),         
	      .hps_0_io_hps_io_usb1_inst_NXT     ( HPS_USB_NXT        ),         

	//////////////////////////////////////// UART 0 (Console)  /////////////////////////////////////////////
	      .hps_0_io_hps_io_uart0_inst_RX     ( HPS_UART_RX        ),          
	      .hps_0_io_hps_io_uart0_inst_TX     ( HPS_UART_TX        ), 
			
	////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
	/////////////////////////////// 	   On Board Compunents     ///////////////////////////////////  ////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

	///////////////////////////////////////////  HPS LED & KEY  ///////////////////////////////////////////
	      .hps_0_io_hps_io_gpio_inst_GPIO53  ( HPS_LED            ),                
	      .hps_0_io_hps_io_gpio_inst_GPIO54  ( HPS_KEY            ),   

	//////////////////////////////////  G-Sensor: I2C0 (Terasic Docu I2C1) /////////////////
		.hps_0_io_hps_io_i2c0_inst_SDA      (HPS_I2C1_SDAT        ),      		
		.hps_0_io_hps_io_i2c0_inst_SCL      (HPS_I2C1_SCLK        ),      		
			
	//////////////////////////// Onboard LEDs, Switches and Keys ////////////////////////
		  .led_pio_external_connection_export (LEDR               ), 
		  .pb_pio_external_connection_export  (Switch             ), 
		  .sw_pio_external_connection_export  (KEY_N              )
	);
	````
### Connect the HPS compunets to FPGA I/O Pins

*  Insiert following sinipett inside "**u0**":
	* **UART1:**
		````verilog
		.hps_0_uart1_cts                    (),                    
		.hps_0_uart1_dsr                    (),                    
		.hps_0_uart1_dcd                    (),                   
		.hps_0_uart1_ri                     (),                    
		.hps_0_uart1_dtr                    (),                    
		.hps_0_uart1_rts                    (),                  	
		.hps_0_uart1_out1_n                 (),                 	 
		.hps_0_uart1_out2_n                 (),                 	 
		.hps_0_uart1_rxd                    (uart1_rx),          
		.hps_0_uart1_txd                    (uart1_tx),````
	* **I2C1:**
		````verilog
		.hps_0_i2c1_clk_clk            	    (scl1_o_e),              	
		.hps_0_i2c1_scl_in_clk              (scl1_o),         
		.hps_0_i2c1_out_data                (sda1_o_e),                	
		.hps_0_i2c1_sda                     (sda1_o),  
	   ````
	* **CAN0:**
	   	````verilog
	 	 .hps_0_can0_rxd                     (can0_rx),           
	     .hps_0_can0_txd                     (can0_tx),		
	   ````
	  * **SPI:**
	   ````verilog
		.hps_0_spim0_sclk_out_clk           (spi0_clk),          
		.hps_0_spim0_txd                    (spi0_mosi),                    
		.hps_0_spim0_rxd                    (spi0_miso),                  
		.hps_0_spim0_ss_in_n                (1'b1),              
		.hps_0_spim0_ssi_oe_n               (spim0_ssi_oe_n),             
		.hps_0_spim0_ss_0_n                 (spi0_ss_0_n),                
		.hps_0_spim0_ss_1_n                 (),               
		.hps_0_spim0_ss_2_n                 (),                 
		.hps_0_spim0_ss_3_n                 (),
	    ````
	* At above the file these wires as temperary in-/output-values
		````verilog
		//// IO Buffer Temp I2c 1 & 3 
		wire scl1_o_e, sda1_o_e, scl1_o, sda1_o, 
		     scl3_o_e, sda3_o_e, scl3_o, sda3_o;
		//// IO Buffer Temp SPI 0 	  
		wire spi0_clk, spi0_mosi, spi0_miso,spi0_ss_0_n;
		//// IO Buffer Temp UART 1 	
		wire uart1_rx,uart1_tx;
		//// IO Buffer Temp CAN 0
		wire can0_rx, can0_tx; 
      ````
	*  To connect in-/output-values with pythical I/O Pins are I/O-Buffers requierd
	*  The outputs of Buffers are connected to following Arduino shild Pins:
	[Screenshot Arduino Shild Pinout] 
	* Add at the end of the file following lines of verilog code
		*  **UART1:**
		````verilog
		// UART1 -> RX
		ALT_IOBUF uart1_rx_iobuf (.i(1'b0), .oe(1'b0), .o(uart1_rx), .io(ARDUINO_IO[1]));
		// UART1 -> TX
		ALT_IOBUF uart1_tx_iobuf (.i(uart1_tx), .oe(1'b1), .o(), .io(ARDUINO_IO[0]));
      ````
		*  **I2C1:**
		````verilog
	    // I2C1 -> SCL 
	    ALT_IOBUF i2c1_scl_iobuf   (.i(1'b0),.oe(scl1_o_e),.o(scl1_o),.io(ARDUINO_IO[15]));
	  // I2C1 -> SDA 
	  ALT_IOBUF i2c1_sda_iobuf   (.i(1'b0),.oe(sda1_o_e),.o(sda1_o),.io(ARDUINO_IO[14]));
	  ````
		* **CAN0:**
	   	````verilog
		// CANO -> RX
		ALT_IOBUF can0_rx_iobuf (.i(1'b0), .oe(1'b0), .o(can0_rx), .io(ARDUINO_IO[9]));
		// CAN-> TX
		ALT_IOBUF can0_tx_iobuf (.i(can0_tx), .oe(1'b1), .o(), .io(ARDUINO_IO[8]));
	  ````
	* This Top-level Verilog file is also avalibil inisde my Github repository
	* Save the file
	
	### Generating the Bitstream for configuration with u-boot and Linux
	* **Start the Compalation process** of the previosly builded Quartus Prime project
	*  I n case of an **HPS I/O Fitting error** run flowing TCL Script manuely
	   --> *Tools/TCL Script .../.../hps_sdram_p0:pin_assignments.tcl*
	*  In case of **Error 14566**: "*The Fitter cannot place 1 periphery component(s) due to conflicts with existing constraints*"
		  * Run following TCL Command: 
			    ````shell
		set_global_assignment -name AUTO_RESERVE_CLKUSR_FOR_CALIBRATION OF````
	 *   To use FPGA I/O-Pins with the HPS is a FPGA configuration nessary
	 * Both posibile ways requiere diffrent  "*.raw*"-files
		 * For **configuration during the Boot with u-boot** use these export settings:
			-   Type: `Raw Binary File (.rbf)`
			-   Mode: `Passive Paralle x8       
		 * For **configuration of the FPGA with _Linux_** use these export settings:
				-   Type: `Raw Binary File (.rbf)`
				-   Mode: `Passive Paralle x16`       
	*  Use the export function of Quartus prime to generate these two files
	*  For more informations see [here](https://github.com/robseb/rsyocto/blob/master/doc/guides/6_newFPGAconf.md)

Now is the configuration of the FPGA fabric done. But to load this configuration and interact with them must bee two diffent boodlader and a embedded Linux system build. To help to simplyfy this huge development effort I dessigned for my own open *rsYocto* - Linux system a building system. All compunets are pre-build and can by chnaged randomly later on.  Internaly is then the Altera `make_sdimage.py` scriped called to put all compunets together and to build the final image. 

You can use the *rsYocto-Building script* with all compunets included. Then you need only to replace the `socfpga_nano.rbf`file ( FPGA configuration file written by the secoundry bootloader) and optinaly the `socfpga_nano_linux.rbf`file (FPGA configuration file can by written with Linux) with your files. The usige of this Building system is desicred and avibibil on the [*rsYocto*-Github repository](https://github.com/robseb/rsyocto) in [**Step by step guide 5**](https://github.com/robseb/rsyocto/blob/master/doc/guides/6_newFPGAconf.md).

In the next parts I show the building process in details.  If you want to do building parts at your own you only need to change this SD-folder files with yours. Then can you build your Linux image with rsyocto building script.


# 2. part: Building of the primary bootloader with Intel EDS

The job of the primary botloader is the configuration of the HPS and the Enabling of all choosen Hard-IP compunets. During the boot of this bootloader is the SDRAM-memory system started and the secoundery Bootloader (u-boot) loaded. 
For the Building of this bootloader provides Intel within EDS a tool called "**BSP-Editor**".  
The following illustration shows Intel EDS BSP Generator Flow for Cyclone V FPGAs:
[ Iustration Guide p.32] 

### Instalation of Intel EDS
The Intel Embedded Development Sutite (EDS) works with Windows and Linux. My prefered Verison is **Ubuntu** Linux running as a Virtual Machine

*  Download Intel FPGA SoC EDS for Linux
	*  [Download Link](http://fpgasoftware.intel.com/](http://fpgasoftware.intel.com/soceds/18.1/?edition=standard&platform=linux&download_manager=direct)
	* For this Tutorial is EDS Version 18 used
* Install Intel EDS
	* Run following Linux shell command to start the instaler
		````bash
		chmod +x SoCEDSSetup-19.1.0.670-linux.run
		sudo ./SoCEDSSetup-19.1.0.670-linux.run
		````
	* At the end of the instalation process uncheck: “*Launch DS-5 installation*”, 
	[ Iustration Guide p.16] 

### Generating of of the primary bootloader with Intel EDS
*	Start with this Command the Intel EDS shell:
	 ````bash
		 cd ~/intelFPGA/18.1/embedded
		 sudo ./embedded_comand_shell.sh
		 ````
* Use the next EDS Shell command to start Intel EDS BSP-Editor
	````bash
	 (EDS shell)# bsp-editor
	 ````
* Create a new BSP by naviagting inside the BSP Editor to: "**File / New HPS BSP...**"
* Select inside the appered window the Quartus Prime Handoff Folder "**.xml**"-file 
* Choose as “Preloader setting directory” the Platform Designer "**hps_isw_handoff**"-Folder inside of the Quartus Prime Folde
* Pres "**OK**" to close the "*new BSP*"-Window
* Modify the prealoder settings
	* The BSP Tool automatic imports all configurations from the Quartus Prime IDE
	* The GUI inside the BSP make it possible to customize and extend these configurations
	* All possible settings are documented inside [Intel’s EDS Development Guide](https://www.intel.com/content/dam/www/programmable/us/en/pdfs/literature/ug/ug_soc_eds.pdf)
	* choose SDMMC for the SD-Card as boot source by checking BOOT_FROM_SDMMC at the selection three position
		* spl --> boot --> BOOT_FROM_SDMMC
	* Enable the FAT supported:
		* spl --> boot --> FAT_SUPPORT
	* Press inside the main BSP-Editor window the "Generate" button 
	* The application generates all necessary code files for the primary bootloader
	* Close the BSP Editor
	
### Compiling the primery bootloader with Intel EDS 
The BSP-Editor puts the generated primery bootloader code to the folder "software" insde Quartus Prime Project folder.
* Navigate with the EDS shell to this folder:
	````bash
	cd <Qurtus Prime project> /software/spl_bsp
	````
* Build the code:
	````bash
	make
	````
	* **Note:** Be sure that this command was called inside the Intel EDS command shell!
* The compiler should output after a succsesfull build a file called "**preloader-mkimage.bin**"
* This file contains the executibil bootloader and must be later on pre installed on the boot SD-card
	
			 
# 3. part:  Development of the secoundry bootloader (u-boot)
The primary bootloader starts the secondary bootloader u-boot. His job is to enable the HPS to FPGA bridges, can configre the FPGA fabric and can the load Linux. 
A final working u-boot executibil is avalibil inside the *rsYocto*- SD folder on the  [*rsYocto*-Github repository](https://github.com/robseb/rsyocto). 
The imported part of u-boot bootloader is the executution of a bootloader script. This feature is inside this u-boot version pre-enbabled. 

### **Building of a secoundary bootloader script:**
* Create a new file: "**uboot_cy5.script**"
* Insert following lines to the script

 [Insert here the u.boot script p.40]



 * Run with the EDS shell following command to build the script:
 ````bash
 mkimage –A arm – O linux –T script –C none –a 0 –e 0 \
-n u-boot –d uboot_cy5.script \
uboot_cy5.script.scr
````
* Mkimage should greate the compiled script file: "**uboot_cy5.script.scr**"
* Replace with the file the file inside the **rsYocto SD folder** 

# 4. part:  Using the Yocto project to build a custum embedded Linux

Inside my forked [`meta-altera` BSP layer](https://github.com/robseb/meta-altera) I descript in detail how to get started with the Yocto project for Intel SoC FPGAs . 

Also I published an [Yocto project meta layer](https://github.com/robseb/meta-rstools) to bring tools to update the FPGA confiuguration with the running Linux and to interact with simple command the FPGA fabric. 

Please follow these guides to build your own embedded Linux or over jump this point by using the finished *rsYocto*. 

# 5. part:  Building the Device Tree for the embedded Linux

During the boot of Linux loads is the Devices Tree loaded. It contains system speczific information about witch driver is to load. 
It is posibile to use the Yocto project to build the device tree as well. But to get later on a higher felxibility is here the usige of the Intel EDS Device Tree generator shown. 
This tool can decoted the "*.sopcinfo*"-file of the Quartus Prime project and generateds an compitibil device tree. 

### Building the Device Tree with Device Tree Builder
*	Navigate with the Intel EDS Command shell to your Quartus Prime project
*	Start the `sopc2dts`tool:
	````bash
	 sopc2dts --input base_hps.sopcinfo \
		 --output socfpga_cy5.dts --type dts \
		 --board soc_system_board_info.xml \
		 --board hps_common_board_info.xml \
		 --bridge-removal all \
		  --clocks
	````
	* Note: Choose your "*.sopcinfo*"-file  for the atribut *input*   

*  Open the outputed "*socfpga_cy5.dts*" device tree file with a text editor
*  The Device Tree generator enabels also the Adrunio Shild componet with following modules:
	*  **UART1:**
	````dts
	hps_0_uart1: serial@0xffc03000 {
	compatible = "snps,dw-apb-uart-18.1", "snps,dw-apb-uart";
	reg = <0xffc03000  0x00000100>;
	interrupt-parent = <&hps_0_arm_gic_0>;
	interrupts = <0  163  4>;
	clocks = <&l4_sp_clk>;
	reg-io-width = <4>; /* embeddedsw.dts.params.reg-io-width type NUMBER */
	reg-shift = <2>; /* embeddedsw.dts.params.reg-shift type NUMBER */
	status = "okay"; /* embeddedsw.dts.params.status type STRING */
	}; //end serial@0xffc03000 (hps_0_uart1)
	````
	* **I2C1:**
	````dts 
		hps_0_i2c1: i2c@0xffc05000 {
		compatible = "snps,designware-i2c-18.1", "snps,designware-i2c";
		reg = <0xffc05000  0x00000100>;
		interrupt-parent = <&hps_0_arm_gic_0>;
		interrupts = <0  159  4>;
		clocks = <&l4_sp_clk>;
		emptyfifo_hold_master = <1>; /* embeddedsw.dts.params.emptyfifo_hold_master type NUMBER */
		status = "okay"; /* embeddedsw.dts.params.status type STRING */
	}; //end i2c@0xffc05000 (hps_0_i2c1)
	````
	*  **SPI1:**
	````dts
		hps_0_spim0: spi@0xfff00000 {
		compatible = "snps,dw-spi-mmio-18.1", "snps,dw-spi-mmio", "snps,dw-apb-ssi";
		reg = <0xfff00000  0x00000100>;
		interrupt-parent = <&hps_0_arm_gic_0>;
		interrupts = <0  154  4>;
		clocks = <&spi_m_clk>;
		#address-cells = <1>; /* embeddedsw.dts.params.#address-cells type NUMBER */
		#size-cells = <0>; /* embeddedsw.dts.params.#size-cells type NUMBER */
		bus-num = <0>; /* embeddedsw.dts.params.bus-num type NUMBER */
		num-chipselect = <4>; /* embeddedsw.dts.params.num-chipselect type NUMBER */
		status = "okay"; /* embeddedsw.dts.params.status type STRING */
	}; //end spi@0xfff00000 (hps_0_spim0)
	````
	*  **CAN0:**
	````dts
	hps_0_dcan0: can@0xffc00000 {
	compatible = "bosch,dcan-18.1", "bosch,d_can";
	reg = <0xffc00000  0x00001000>;
	interrupt-parent = <&hps_0_arm_gic_0>;
	interrupts = <0  131  4  0  132  4>;
	interrupt-names = "interrupt_sender0", "interrupt_sender1";
	clocks = <&can0_clk>;
	status = "okay"; /* embeddedsw.dts.params.status type STRING */
	}; //end can@0xffc00000 (hps_0_dcan0)
	````

# 6. part:  Generating a bootabil Linux image

At this point are all requiered files togthers. Now must be a bootbil image file createn with three diffrent partions embedded.
The *rsYocto*-Building can generate this system with a singe command.
Please follow [this instructions](https://github.com/robseb/rsyocto/blob/master/doc/guides/6_newFPGAconf.md) how to download 
and use the *rsYocto*-SD Folder.  

The next table shows the requiered pations with there content for this project.

| **File name** | **Part of Creation in this guide** | **Description** | **Disk Partition**
|:--|:--|:--|:--|
|\"makersYoctoSDImage.py\"| **-** | the automatic *rsYocto* script | **-** |
|\"make_sdimage.py\"| **-** | Altera SD image making script | **-**  | 
|\"infoRSyocto.txt\"| **-**  | rsYocto splech screen Infos | **-**  | 
| \"rootfs_cy5.tar.gz\"|**4**|compresed rootFs file for Cyclone V  | **2.Partition: ext3** |       
|\"preloader_cy5.bin\"|**2**| prestage bootloader | **1.Partition: RAW** | 
|\"uboot_cy5.img\"|**-**| Uboot bootloader | **2.Partition: vfat** |       
|\"uboot_cy5.scr\"|**3** | Uboot bootloader script | **3.Partition: vfat** |   
|\"zImage_cy5\"|**4** | compressed Linux Kernel  | **3.Partition: vfat** |   
|\"socfpga_cy5.dts\"|**5** | Linux Device Tree  | **3.Partition: vfat** |   
|\"socfpga_nano.rbf\"|**1**| FPGA configuration file writen by u-boot| **2.Partition: ext3** |  
|\"socfpga_nano_linux.rbf\"|**1**| FPGA Configuration that can be written by Linux   | **2.Partition: ext3** |          

. Files from other boards and platform can be delated. Replace self creted files with yours.
The Folder should now look like this: 
[Image 9] 

Use folllowing command on a CentOS-Linux Computer to start the building script:
````bash
sudo python makersYoctoSDImage.py   
````
The script will be geneate a "**.img**"-file. It can be deployed to any SD-Card.

# 7. part:  Flashing the image to a SD Card and booting Linux 
To install the embedded Linux image file to a bootabil SD-card follwoing [this instruction](https://github.com/robseb/rsyocto/blob/master/doc/guides/1_Booting.md).
This guide shows how to setup a Terasic DE10-Nano Board.


# 8. Part: Unsing the `i2c-tools` and `minicom` to interact with Adrunio shilds
The Adrunio shiled connecter has a "SDA"- and "SCL"- pin for the I2C-Bus. In Part 1 was the Hard-IP I2C-Controller 1 
connected to this FPGA I/O-Pins. Also are here a Serial Port, a SPI master and CAN Channel connected. 
On *rsYocto* are the `i2c-tools` pre-installed follow [this instructions](https://github.com/robseb/rsyocto/blob/master/doc/guides/2_FPGA_HARDIP.md)

# 9. Part: Writing a python script to transmit CAN-packages

* To enable the CAN0 execute this command to enable the CAN network Port:
  ````bash 
  ip link set can0 type can bitrate 125000
  ip link set up can0
  ````
* Install `python-can` with python pip
  ````bash
  pip install python-can
  ````
 Use Visual Studio Code Insider to debug this python can application remodly (see [here](https://github.com/robseb/rsyocto/blob/master/doc/guides/4_Python.md)) 
 Or to the use onbaord `nano`editor to write the python code.
 
 * Create a new Python file, for example:
  ````bash
  nano sendCanPackage.py
  ````
  * Insiert following to this python file as shown in the [python-can documentation](https://python-can.readthedocs.io/en/master/)
  
  ````python
    #!/usr/bin/env python
    # coding: utf-8

    """
    This example shows how sending a single message works.
    """

    from __future__ import print_function

    import can

    def send_one():

        # this uses the default configuration (for example from the config file)
        # see https://python-can.readthedocs.io/en/stable/configuration.html

        # Using specific buses works similar:
        bus = can.interface.Bus(bustype='socketcan', channel='can0', bitrate=12500)
        # ...

        msg = can.Message(arbitration_id=0xc0ffee,
                          data=[0, 25, 0, 1, 3, 1, 4, 1],
                          is_extended_id=True)

        try:
            bus.send(msg)
            print("Message sent on {}".format(bus.channel_info))
        except can.CanError:
            print("Message NOT sent")

    if __name__ == '__main__':
        send_one()
    ````
  * Save and close the python file
  * Connect to the **Adrunio Pin D8** to **CAN TX** and the Pin **D9** to **CAN RX** on a 3.3V Can-Bus Transiver (Pin out)[https://raw.githubusercontent.com/robseb/rsyocto/master/doc/symbols/DE10Nano_pinout.png]
  * Execute the python script
    ```python 
    python3 sendCanPackage.py
    ````
  * Now sends the Cyclone V SoC-FPGA a CAN package through the adruno heder with the ID 0xC0FFE
  [Image 10 Osi]
  
  The embedded Bosch CAN-Controller can detect linkage errors. 
  I case of a missing connection to an CAN-Bus member a Kernel Message will be triggerd and the CAN Controller shuts down.
  Use then follwoing command to restart the CAN-Controller:
    ````bash 
    link set can0 down
    ip link set up can0
     ````
  
  In the same way it is also posibile comunicate the Audrio Shields via UART,SPI or I2C. 
  On *rsYocto* are python scripts for this usecases preinstalled. 
  
  

<br>

# Author
* **Robin Sebastian**

*rsYocto* a project, that I have fully developed on my own. It is a academic project.
Today I'm a Master Student of electronic engineering with the major embedded systems. 
I ‘m looking for an interesting job offer to share and deepen my shown skills starting summer 2020.

  
  