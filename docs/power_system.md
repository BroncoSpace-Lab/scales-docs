# SCALES EPS Board
By Luca Lanzillotta
Updated 9/2/2025

# Design Notes
This design serves as the power supply for the entire SCALES system, it powers three subsystems.
The current design is for development use and is set to be utilized in TVAC, Vibration, and TID testing. The changes from this board to the flight ready version will be a change in connectors.

This entire board was designed from scratch given power requirements, operational constraints and design requirements.
The components selected were selected for significant overhead to ensure reliability.

Note:
This board took three PCB revisions to get working correctly, and in total seven schematic revisions to get a fully finalized and conceptually working design. It is because of this process that you may see PCB versions go from 1A, 1B, 1C, and schematic revisions go from REVA to REVF.

# System Requirements
Overall System - (+28v 8A Max)
- ML/Edge Computer - Nvidia Jetson (+20V 4A)
- OBC/Flight Computer - IMX8 (+3.3V 2A)
- Peripheral System - Ethernet Switch (+5V 2A)

Design Requirements:
- Load switch
- Switching regulator
- Clock generator
- I/V sensor
- Temp. sensor
- Watchdog

# Operational Requirements
- OBC is the primary subsystem, which is always enabled. The OBC controls the ML and Perif. via load switches. OBC must hold the respective subsystem EN pin high for it to be active.
- Each subsystem has a Watch dog that verifies its normal operation; in the case of a fault, the subsystem is disabled by its watch dog.
    - OBC: If the OBC hangs and fails to pet, it is power cycled, as the EN pin for its load switch is always held high with respect to the battery voltage
    - ML/Perif: If either of these subsystems hang and fail to pet, they are disabled until the OBC re-enables them.
- Monitoring is done by the OBC via I2C from the I/V sensors, and temp sensors, which allows the subsystem to get basic telemetry on the subsystem state of operation

## Concept of Operations Block Diagram
![Con. Ops Diagram](Images/EPSREVF_ConOps.png)

# Component Selection
Based on a few months of research and evaluating the most efficient options for what the overall design should do, I concluded on the following components after establishing a system topology through the following block diagram.
Note: The watchdog timer circuit was generously given to us for use by Andrew Greenberg from over at Portland State University.

### EPS Rev. F Block Diagram
![EPS REV F Block Diagram](Images/EPS_REVF_BlockDiagram.png)

  - Load Switch + Controller: TPS1HA08-Q1
  - Switching Regulator: LT8612
  - IV-Sensor: INA260AIPWR
  - Temperature Sensor: MCP9808
  - Clock Generator: LTC6902
  - I2C Buffer: TCA4307
  - Comparator (WD Circuit): TLV1704-SEP
  - Subsystem Connector: DF-11 (2x8)
  - Power Connector: XT-60PWF

# Schematic
The root contains the layout for the entire system in both block diagram form as well as the concept of operations format for clearer understanding of design practices. 
Each subsystem is nearly identical, likewise for the watchdogs other than a few pull ups that dictate which systems are normally operating.

Root:
![Root](Images/EPSREVF_Root.png)

OBC Subsystem:
![OBC Subsystem](Images/EPSREVF_OBCSubsystem.png)

Peripheral Subsystem:
![Perif Subsystem](Images/EPSREVF_PerifSubsystem.png)

Jetson Subsystem:
![Jetson Subsystem](Images/EPSREVF_JetsonSubsystem.png)

OBC Watchdog Circuit:
![OBC Watchdog Circuit](Images/EPSREVF_OBCWD.png)

Peripheral Watchdog Circuit:
![Perif. Watchdog Circuit](Images/EPSREVF_PerifWD.png)

Jetson Watchdog Circuit:
![Jetson Watchdog Circuit](Images/EPSREVF_JetsonWD.png)

# PCB Layout
Due to being a power board, having thicker traces was a necessary design constraint along with thicker copper pours on each layer. This board utses 2oz fill on the internal layers and a 1oz fill on the external layers. Traces up to 1mm were used for specific high current lines to ensure low impedance high surface area and volume conductivity.

Signal1 Layer:
![Signal1 Layer](Images/EPSREVF_Signal1.png)

GND Layer:
![GND Layer](Images/EPSREVF_GND.png)

Power Layer:
![Power Layer](Images/EPSREVF_Power.png)

Signal2 Layer:
![Signal2](Images/EPSREVF_Signal2.png)


# PCB 3D Renders
Front:
![EPSREVF Front](Images/EPSREVF_Front.png)

Back:
![EPSREVF Back](Images/EPSREVF_Back.png)

### Access this design with the link below
https://github.com/BroncoSpace-Lab/scales-hardware/tree/main/power_system

# Testing and Evaluation Notes
    
Using the on board test points and the expected probe locations based on the schematic, we can check if all the voltages are as required, if the watch dogs are behaving as expected, and if there are any components that may not be performing as required.
    
> Testing Notes
- Output voltage @28v    
- Current Limit @4.0A (Designed for up to 8A)

  
> Overall Board

- TP 31, 39, 25 → 28v (Vbatt)
   - [x]  TP31 = 28v
   - [x]  TP39 = 28v
   - [x]  TP25 = 28v
   - [x]  Mounting Holes → GND
    
> OBC Subsystem

   - [x] WD Pad Soldered:
   - [x] TP 36 → EN
         - 4.2v
         - Working as intended
         - 28v * (10k/(56.2k + 10k)) = 4.22v
   - [x]  TP 10 → ~28v to Switching Regulator
         - 28v works as intended
   - [x]  TP 35 → SYNCPHASE1 (400Khz 0 deg out of phase)
        - ~3v works as intended  
   - [x]  TP 33 → Output of Switching regulator (~3.3V)
        - 3.3v works as intended
   - [x]  TP 34 → Output of INA260 (~3.3V)
        - 3.3v works as intended
    
> WD_OBC

   - [x] TP 47 → Ensures WD is supplied with power
      - ~2.7v works as intended
   - [x] TP 59 → WD Input from OBC GPIO for pet
      - Works as intended, after no pet, output is driven low of the WD which holds the switch low.
      - Input from function generator
    
> Jetson Subsystem:

   - [x]  TP 38 → Tests the GPIO input from the OBC/WD EN Pin (Should be held high when System is active, low on PET failure)
      - Pulled up by either a GPIO or Power supply for testing
      - When the GPIO is driven above ~2.3v, the Load Switch allows the Vbatt to pass
   - [x]  TP 25 → ~28v (Vbatt)
      - 28v works as intended
   - [x]  TP 26 → ~28v output from load switch
      - 28v works as intended
   - [x]  TP 27 → SYNCPHASE3 (400khz 180 deg out of phase from PHASE1)
      - ~3v works as intended      
   - [x]  TP28 → ~20v (Output from Switching regulator)
      - 20v works as intended
   - [x]  TP 29 → ~20v (Output from INA290 to Jetson Subsystem)
      - 20v works as intended
        
> WD_Jetson:

   - [x]  TP 30 → Check WD Supply power
      - ~5.7v to power supply rails, using voltage divider, 20v(43k/143010) = 6.1v, works as intended
   - [x]  TP 54 → WD Interface input line, held high when there is a pet
      - Takes in input from function generator or GPIO
   - [x]  TP 58 Open collector output from the Watchdog, should be half of whatever the OBC GPIO pin voltage is when high, and then ground any other time
      - Is pulled up to GPIO/power supply, pulls low after pet window isnt met.
    
> Peripheral Subsystem:

   - [x]  TP 37→ OBC GPIO input for EN line and WD En line, should be ~1.4v when high for OBC holding it high, and low when not held high.
      - Works as intended
        
   - [x]  TP 39 → ~28v (Vbatt)
      - ~28v works as intended
   - [x]  TP 40 → ~28v output of the load switch
      - ~28v works as intended
   - [x]  TP 41 → SYNCPHASE2 Output of LTC6902 (400khz 90 deg out of phase)
   - [x]  TP 42 → Input regulated power to the INA260, should be roughly +5v
      - ~5v works as intended
   - [x]  TP 43 → output for the Peripheral subsystem, should be roughly +5v
      - ~5v works as intended
    
> WD_Perif:

   - [x]  TP 5 → ~4.6v supply to power the watch dog
      - 4.6v works as intended
   - [x]  TP 15 → WD Perif output always pulled low, should be low when the system is off, and held high when the OBC enables it, when the pet fails it will hold low until the subsystem turns off.

> I2C Addresses (3.3v Logic)

   - Temperature Sensor I2C Addresses:
      - 0x19
      - 0x1A
      - 0x1B
   - I/V Sensors
      - 0x41
      - 0x45
      - 0x40
