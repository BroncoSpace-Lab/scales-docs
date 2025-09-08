# Peripheral Board Design
## Summary of the evelution of the Peripheral Board
The peripheral board has gone through a few different iterations and changed as the project has moved forward. This board is optional which is why there were different descoping and iterations. Orignally the peripheral board was going to be the an FPGA board that was modeled after The Aerospace Corporation called [SatCat5](https://github.com/the-aerospace-corporation/satcat5). That was changed due to a lack of documentation, the current open source iteration was not working, and I am a team of 1 trying to recreate what they have in a short period of time which was not seen as feasible. The idea was scrapped before creation of custom hardware.

The board was changed from the described above to a microcontroller that will give the end user a way to communicate to the flight processor and the machine learning computational unit along with lower level peripherals like I2C, SPI and UART. This was done by using a Raspberry Pi for the lower level singals, an SPI to Ethernet converter and then an unmanaged Ethernet switch to create a local area network for the satellite to give the end user access to higher speed peripherals over Ethernet. This brought up the problem of having three different system archetecutres on board all running F´. At the time F´ Arduino and F´ Zephyr was not stabel. That is currently being worked on. The flight processor would still need sensors and the flight processor could be used for those as well. Since the board is optional it allows the simplifaction of the board to what it currently is.

Currently the peripheral board is just a high speed unmanaged Ethernet switch. This mirrors closely to what our current SCALES demo is and could be changed out with a custom board or payload of the end user's choice.


## Peripheral Board with the Microcontroller
This board is conprized for two different purposes.

1. Inter communication between the Jetson and i.MX8X.
2. Gives the end user access to ports to attach the peripherals that they want. 
    The ports given to user:
    * At least 1 Ethernet port
    * At least 2 SPI/UART ports

### General Information of each major block of the board

![Peripheral Board General Info & Block Diagram](/docs/Images/peripheral_board_main.png)

The above image is of the root page of the schematic with basic infomration and a block diagram of the board and its use.

This is the front of the board in KiCad's 3D view.
![Peripheral Board Front](/docs/Images/peripheral_board_front.png)

This is the back of the board in KiCad's 3D view.
![Peripheral Board Back](/docs/Images/peripheral_board_back.png)

### Core Components 
Each section is about the individual chips and it has basic information on the layout.

- Raspberry Pi bought from 
    [JLC Link](https://jlcpcb.com/partdetail/RaspberryPi-RP2350A/C42411118) to the RP2350A.

    Layout and other information from the [PROVES dev kit](https://github.com/proveskit) and the [Raspberry Pi Hardware minimum configuration](extension://efaidnbmnnnibpcajpcglclefindmkaj/https://datasheets.raspberrypi.com/rp2350/hardware-design-with-rp2350.pdf)

- Wiznet bought from 
    [JLC Link](https://jlcpcb.com/partdetail/Wiznet-W5500/C32843) to the Wiznet W5500.

    Layout and other informatuon from the [PROVES dev kit](https://github.com/proveskit) and [Wiznet W5500 Eval Board](https://github.com/Wiznet/W5500-EVB)

    This product can be purchased bought as [WIZ850io](https://wiznet.io/products/network-modules/wiz850io) which is a compact module of just the chip and an RJ45 or as [W5500-EVB-Pico](https://wiznet.io/products/evaluation-boards/w5500-evb-pico) which is the chip with a Raspberry Pi Pico with open holes for GPIO manipulation.

    There is [Coding with CircuitPython on W5100S-EVB-Pico2](https://maker.wiznet.io/viktor/projects/coding-with-circuitpython-on-w5100s-evb-pico2/) that is on the Wiznet Makers website that should be easy to follow for setting up the Wiznet RP combination.

- 5-port Ethernet Switch bought from 
    [JLC Link](https://jlcpcb.com/partdetail/MicrochipTech-KSZ8795CLXIC/C69416) to the Microchip 5-Port Ethernet Switch.

    The original eval board is no longer made but there is an update of the chip with a [newer Eval Board](https://www.microchip.com/en-us/development-tool/KSZ8795CLXD-EVAL). For this project we will be using the strap in mode of the chip to be just a 4 port etherenet switch that will not require any software or modification.

    The first few boards were not working in their hardware configuration and confirmed by Microchip that there is some aspect of software that should be used to get the switch working.

## Peripheral Board as Unmanaged Ethernet Switch
This board is still being developed and is a current work in progress. We have decided to go with the [MICROCHIP KSZ9896CTXI](https://www.lcsc.com/product-detail/C638297.html?s_z=n_KSZ9896) which is a 6 port gigabit unmanaged Ethernet switch.

Learning from the past I have designed a minimal development board that should be able to be used for all of the enviornmental testing.

*More on its way*

Last updated on 9/8/2025 
By John Pollak