# SCALES IMX8 Carrier Board
By Luca Lanzillotta, John Pollak, and Kelly Williams 
Updated 9/2/2025

# Design Notes
The IMX8 Carrier board seeks to implement the Phytec IMX8QXP SOM into a massively stripped down version of the Phytec PCM-942 Dev board. This design drastically reduces the footprint of the dev board and implements a small portion of the available serial communication protocols. By reducing the available IO and streamlining what we want available for the end user, we are able to provide a smaller, more easily integratable PCB into the SCALES hardware stackup.

# System Requirements
- Follow the 10cm x 10cm structural design constraint
- Provide the end user with the following communication protocols
    * 10/100 Base-T Ethernet
    * FTDI
    * UART
    * I2C
    * SPI
    * QSPI
    * SD Card
- Utilize two GPIO outputs to control Ethernet and Jetson subsystems
- Watchdog Pet GPIO  

# Operational Requirements
The IMX8 Carrier board is responsible for petting its respective watchdog on the power board approximately every 30 seconds. This requires the system to boot in less than that and send a pet signal via the specific GPIO before that pet window expires. 
The IMX8 Carrier board is also responsible for reading power board I2C telemetry provided by the INA260 sensors and the MCP9808 sensors.
FTDI must always be accessible, as it serves as a live debugging port, which can be accessed via teraterm or tabby.
Ethernet must always be accessible, so that the system may interface directly to the network switch board along with the Jetson directly.

# Component Selection
The majority of these components are pulled straight from the Phytec PCM-942 Dev board schematic. Although alternative components with similar specifications work as well, we have tried to stay as close as possible to the reference design for the components we wanted to implement in this design. 

* [SOM Connectors](https://www.samtec.com/products/bth-070-02-l-d-a-k-tr#cadmodels)
* [DF11 Connectors](https://lcsc.com/product-detail/Wire-To-Board-Wire-To-Wire-Connector_HRS-Hirose-HRS-DF11-16DP-2DSA-08_C530981.html)
* [Power Distribution Switch](https://lcsc.com/product-detail/Power-Distribution-Switches_ROHM-Semicon-BD2204GUL-E2_C314699.html?s_z=n_BD2204GUL-E2)
* [Micro SD Card Connector](https://lcsc.com/product-detail/SD-Card-Memory-Card-Connector_MOLEX-5027740891_C330255.html?s_z=n_TF-SMD_5027740891)
* [FTDI Linear Voltage Regulator](https://lcsc.com/product-detail/Linear-Voltage-Regulators_TI_LP38693MP-ADJ-NOPB_LP38693MP-ADJ-NOPB_C181420.html)
* [FTDI Controller](https://lcsc.com/product-detail/USB_FTDI_FT2232HL_FT2232HL_C27882.html)
* [Micro USB-B Connector](https://lcsc.com/product-detail/USB-Connectors_MOLEX_47346-0001_47346-0001_C132560.html)
* [SPI EEPROM](https://jlcpcb.com/parts/componentSearch?searchTxt=C890471)
* [FTDI Crystal Oscillator](https://www.lcsc.com/product-detail/C9002.html?s_z=n_C9002)
* [UART Level Shifters](https://lcsc.com/product-detail/Logic-ICs_TI_TXS0101DCKR_TXS0101DCKR_C132031.html)
* [ESD Surge Protector](https://lcsc.com/product-detail/TVS_SEMTECH_SRV05-4-TCT_SRV05-4-TCT_C13612.html)
* [Ethernet PHY to MAC 10/100 Base T Translator](https://www.ti.com/cn/lit/ds/symlink/dp83867ir.pdf?ts=1748027510532&ref_url=https%253A%252F%252Fjlcpcb.com%252F)
* [Ethernet Crystal Osciallator](https://www.lcsc.com/product-detail/C13740.html?s_z=n_C13740)
* [Low Voltage AND Gate](https://lcsc.com/product-detail/74-Series_TI_SN74AUP1G08DBVR_SN74AUP1G08DBVR_C139409.html)
* [Ethernet TVS Diodes](https://www.lcsc.com/product-detail/C13612.html)
* [RJ45 Connector with Integrated Magnetics](https://jlcpcb.com/partdetail/AmphenolIcc-RJHSE5384/C464587)
* [Linear Regulator](https://www.lcsc.com/product-detail/C2872754.html?s_z=n_C2872754)
* [Linear Regulator](https://www.lcsc.com/product-detail/C145717.html)
* [P-Channel MOSFET](https://www.infineon.com/dgdl/irlml6401pbf.pdf?fileId=5546d462533600a401535668b96d2634)
* [N-Channel MOSFET](https://www.onsemi.com/pub/Collateral/BSS138-D.PDF)
* [Switch](https://www.lcsc.com/product-detail/C7471372.html?s_z=n_C7471372)
* [Button](https://www.lcsc.com/product-detail/C72443.html)

### IMX8 Carrier V2 Block Diagram
![IMX8 Carrier V2](Images/IMX8CarrierV2Block.png)

# Schematic
Root:

![Root](Images/IMX8CarrierV2Root.png)

Power Input:

![Power Input](Images/IMX8CarrierV2PowerInput.png)

Boot Configuration:

![Boot Configuration](Images/IMX8CarrierV2BootConfig.png)

Debug USB:

![Debug USB](Images/IMX8CarrierV2DebugUSB.png)

Ethernet:

![Ethernet](Images/IMX8CarrierV2Ethernet.png)

QSPI-SPI:

![QSPI-SPI](Images/IMX8CarrierV2QSPISPI.png)

GPIO-I2C:

![GPIO-I2C](Images/IMX8CarrierV2GPIOI2C.png)

# PCB Layout

The IMX8 Carrier board is a 6 layer, impedance controlled PCB stackup. This is done to ensure proper Ethernet and USB differential pair routing, along with ease of trace routing. Impedance control was done using the JLCPCB impedance calculator and stackup configuration tool paired with the layout and datasheet recommendations for the specific components requiring particular impedance values.
In this design there are four track widths that were used to route specific traces, with the impedance controlled tracks, both USB and Ethernet having their own corresponding width. Additionally via sizes vary according to the type of connection necessary.

Signal1:

![Signal1](Images/IMX8CarrierV2Sig1.png)

GND1:

![GND1](Images/IMX8CarrierV2GND1.png)


Power:

![Power](Images/IMX8CarrierPWR.png)

Signal2:

![Signal2](Images/IMX8CarrierSignal2.png)

GND2:

![GND2](Images/IMX8CarrierGND2.png)

Signal3:

![SIGNAL3](Images/IMX8CarrierSignal3.png)

# PCB 3D Renders
Front:

![Front](Images/IMX8CarrierFront.png)

Back:

![Back](Images/IMX8CarrierBack.png)

# Resources Used for this Design

- [IMX8 Quickstart Guide](https://docs.phytec.com/projects/yocto-phycore-imx8x/en/latest/quickstart/index.html)
- [IMX8 Hardware Manual](https://livecsupomona.sharepoint.com/sites/broncospacelab/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20-%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X%2FL-864e%2EA4_phyCORE-i%2EMX8X_HW%20Manual%20%285%29%2Epdf&parent=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20-%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X)
- [IMX8 Hardware Development Guide](https://livecsupomona.sharepoint.com/sites/broncospacelab/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X%2Fi%2EMX8%20QXP%20Hardware%20Developer%27s%20Guide%2Epdf&parent=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X)
- [IMX8 Applications Processor Reference
Manual](https://livecsupomona.sharepoint.com/sites/broncospacelab/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X%2FIMX8DQXPRM%2Epdf&parent=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X)
- [IMX8 Dev Board Schematic](https://livecsupomona.sharepoint.com/sites/broncospacelab/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X%2Fi%2EMX%208X%20dev%20board%20schematics%2Epdf&parent=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X)
- [IMX8 SOM Schematic](https://livecsupomona.sharepoint.com/sites/broncospacelab/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X%2Fi%2EMX%208X%20SOM%20schematics%2Epdf&parent=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X)
- [PHYTEC Pinmux Tool](https://pinmux.phytec.com/)

### Access this design with the link below
https://github.com/BroncoSpace-Lab/scales-hardware/tree/main/imx8x_carrier_board

# Testing and Evaluation Notes
    
   - Board testing and evaluation is on the way!