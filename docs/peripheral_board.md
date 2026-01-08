# Peripheral Board Design
## Summary of the evolution of the Peripheral Board
The peripheral board has gone through a few different iterations and changed as the project has moved forward. This board is optional which is why there were different descoping and iterations. Originally the peripheral board was going to be on an FPGA board that was modeled after The Aerospace Corporation called [SatCat5](https://github.com/the-aerospace-corporation/satcat5). That was changed due to a lack of documentation, the current open-source iteration was not working, and I am a team of 1 trying to recreate what they have in a brief period which was not seen as feasible. The idea was scrapped before the creation of custom hardware. 

The board was changed from the described above to a microcontroller that will give the end user a way to communicate to the flight processor and the machine learning computational unit along with lower-level peripherals like I2C, SPI and UART. This was done by using a Raspberry Pi for the lower-level signals, an SPI to Ethernet converter and then an unmanaged Ethernet switch to create a local area network for the satellite to give the end user access to higher speed peripherals over Ethernet. This brought up the problem of having three different system architectures on board all running F´. At the time F´ Arduino and F´ Zephyr was not stable. That is currently being worked on. The flight processor would still need sensors, and the flight processor could be used for those as well. Since the board is optional, it allows the board to simplify its status.

Currently the peripheral board is just a high-speed unmanaged Ethernet switch. This mirrors closely what our current SCALES demo is and could be changed out with a custom board or payload of the end user's choice. 


## Peripheral Board as an Unmanaged Ethernet Switch
This is an unmanaged Gigabit ethernet switch which allows the end user to communicate and connect to the SCALES system. It was modified from the original peripheral board. It uses the [MICROCHIP KSZ9896CTXI](https://www.lcsc.com/product-detail/C638297.html?s_z=n_KSZ9896). The board is currently under review but has been checked by Microchip support as ready for testing. Updates will be provided if the board is created and used in the first prototype of the SCALES system. This board is optional and is to help give the end user many different connections to communicate with.

### General Information of the board
Front of the board:
<img src="/docs/Images/periph_ethernet_switch_front.png" alt="Peripheral Unmanaged Ethernet Switch Front" width="600">

Back of the board:
<img src="/docs/Images/periph_ethernet_switch_back.png" alt="Peripheral Unmanaged Ethernet Switch Back" width="600">

If this board is created and tested then combining the old with the new might be the next iteration or there could be some alternate version of just having the SPI to Ethernet. The board files and everything can be found on [the Bronco Space Lab SCALES hardware page](https://github.com/BroncoSpace-Lab/scales-hardware).

## *OLD* Peripheral Board with the Microcontroller
This board serves two different purposes.

1. Inter communication between the Jetson and i.MX8X.
2. Gives the end user access to ports to attach the peripherals that they want. 
    The ports given to user:
    * At least 1 Ethernet port
    * At least 2 SPI/UART ports

### General Information of each major block of the board
<img src="/docs/Images/peripheral_board_main.png" alt="Peripheral Board General Info & Block Diagram" width="600">

The above image is of the root page of the schematic with basic information and a block diagram of the board and its use.

This is the front of the board in KiCad's 3D view.
<img src="/docs/Images/peripheral_board_front.png" alt="Peripheral Board Front" width="600">

This is the back of the board in KiCad's 3D view.

<img src="/docs/Images/peripheral_board_back.png" alt="[Peripheral Board Back" width="600">


### Core Components 
Each section is about individual chips, and it has basic information on the layout. 

- Raspberry Pi bought from 
    [JLC Link](https://jlcpcb.com/partdetail/RaspberryPi-RP2350A/C42411118) to the RP2350A.

    Layout and other information from the [PROVES dev kit](https://github.com/proveskit) and the [Raspberry Pi Hardware minimum configuration](extension://efaidnbmnnnibpcajpcglclefindmkaj/https://datasheets.raspberrypi.com/rp2350/hardware-design-with-rp2350.pdf)

- Wiznet bought from 
    [JLC Link](https://jlcpcb.com/partdetail/Wiznet-W5500/C32843) to the Wiznet W5500.

    Layout and other information from the [PROVES dev kit](https://github.com/proveskit) and [Wiznet W5500 Eval Board](https://github.com/Wiznet/W5500-EVB)

    This product can be purchased bought as [WIZ850io](https://wiznet.io/products/network-modules/wiz850io) which is a compact module of just the chip and an RJ45 or as [W5500-EVB-Pico](https://wiznet.io/products/evaluation-boards/w5500-evb-pico) which is the chip with a Raspberry Pi Pico with open holes for GPIO manipulation.

    There is [Coding with Circuit Python on W5100S-EVB-Pico2](https://maker.wiznet.io/viktor/projects/coding-with-circuitpython-on-w5100s-evb-pico2/) that is on the Wiznet Makers website that should be easy to follow for setting up the Wiznet RP combination.

- 5-port Ethernet Switch bought from 
    [JLC Link](https://jlcpcb.com/partdetail/MicrochipTech-KSZ8795CLXIC/C69416) to the Microchip 5-Port Ethernet Switch.

    The original eval board is no longer made but there is an update of the chip with a [newer Eval Board](https://www.microchip.com/en-us/development-tool/KSZ8795CLXD-EVAL). For this project we will be using the strap in mode of the chip to be just a 4 port Ethernet switch that will not require any software or modification.

    The first few boards were not working in their hardware configuration, and it was confirmed by Microchip that there is some aspect of software that should be used to get the switch working.



Last updated on 1/8/2026
By John Pollak