# Ford Special Codes on ELM327

The 95 Ford Mustang shop manual (and others of that era as well I understand) give various generic scan
tool codes for triggering things like the Key On Engine Off (KOEO) test.  There is some [incomplete
discussion](https://www.explorerforum.com/forums/index.php?threads/running-koeo-koer-using-a-generic-scantool-elm327.183165/)
from 2007 on the exporerformum on how to send these using an ELM327.  This is my notes based on the forum, the shop
manual, and my experience messing around with it.

# Breaking Down the Code

The codes follow the short lived expanded diagnostic protocol (EDP) given in [SAE
J2205](https://www.sae.org/standards/content/j2205_199512/).  In essense, they are small programs input into the
scan tool to extend its ability by telling it what to send and how to interpret what it gets back.  The input string
is broken down as follows according to the shop manual

1. string id number (any two hex digits)
2. total digit of string count (including the commas)
3. transmit type
4. transmit field (the actual OBED command)
5. receive filter field (may be blank)
6. receive data processing field (commands the scan tool interprets to clear the screen, position the cursor,
   ASCII code to display in text, provide spacing, data conversion, byte number containing information, and code
   for altering execution)
7. data security verification (total hex ASCII value of characters in the string)

The key field is (4).  It is what needs to be transmitted over the bus.  It can also be useful to examine (6).
First by converting it to ASCII to see what text it may contain that a EDP compliant scantool would display on its
screen.  Second by getting to recognize the codes that specify how to interpret the result code.  The following is
my guess as to some of the last two bytes return code interepretations based on looking at many of these codes
(presumably this is documented EDP standard)

| LAST BYTES |  VALUE  |    UNITS   |
| ---------- | ------- | ---------- |
| 06xx00     | Mask xx | ON/OFF     |
| 06xx03     | Mask xx | OPEN/CLOSE |
| 0F         | Raw     | Count      |
| 08         | Raw     | Precent    |

The transmit code contains the first two bytes of the header.  The third byte is not included as it is the
identifier of the transmitting device.  Generally F1 is used as it is reserved for scan tools and so won't cause a
conflict with another device on the bus.  The reamaining bytes then are the payload data.

# Example

The shop manual gives the following sequence for the KOEO self test

* `04,31,21,C4103381,,9E 00 445443287329 20 8042 20 8062 20 8082 A851 FF, 2E`
* `03,32,22FF,C410220202,,9E 00 434E54 20 8061 A961 00 04, EA`
* `02,32,21,C410328100,,9E 00 574149 54 20 8081 A181 61 03, 5E`
* `01,32,21,C4103181,,9E 00 5354415254 20 8081 A181 00 02, 54`

The 4th field gives the transmitted fields.  Breaking this out for the first command we have

* priority/type - `C4`
* target address - `10`
* source address - `F1` (scanner tool ID added by device)
* data bytes - `33 81`

Running the 6th field through a hex to ASCII convert gives the following bits of text for all the commands

* DTC(s)
* CNT
* WAIT
* START

This is what we would see on the display of an EDP based tool, possibly followed by an extracted return value, so
they give a reasonable hint as to what the command is doing.

## Initializing the ELM327

The ELM327 is connected to over a serial port.  It is a good idea to issue a reset followed by enabling echoing and
line feeds to give a decent interactive text experience
```
> ATZ
> ATL1
> ATE1
```
For the first few times, it is also a good idea to enable displaying of full headers
```
> ATH1
```

The first two bytes of the 4th field are the first two bytes of the header.  The default header for the ELM327
device when talking to a 95 Mustang is `61 6A F1`.  It gives access to the legislated OBD2 emissions functions.
The above commands require using the header `C4 10 F1` (`F1` being the added scanner tool ID again)
```
> ATSHC410F1
```

## Transmitting the codes

Now the codes are transmitted by just typing their bytes, minus the header, into the ELM327
```
> 3381
> 220202
> 328100
> 3181
```

# Ford Codes

## Extended PIDs

Accessing the extended Ford PIDs is done with header `C4 10 F1`
```
> ATSHC410F1
```
and command `22` followed by the two byte PID code.  For example, the extended PID for status of the air
conditioning cycling switch is 1104.  The command to get this after settings the header would be
```
> 221101
```

The following table gives the Ford extend PIDs documented in the 95 Mustang shop manual.  The
`-bx` following some PIDs mean the status is returned it the xth bit of the result.

| PID#-b* | ACRONYM    | DESCRIPTION                                                 | UNITS                |
| ------- | ---------- | ----------------------------------------------------------- | -------------------- |
| 1104-b1 | ARC        | Automatic Ride Control                                      | ON/OFF               |
| 1101-b0 | ACCS       | Air Conditioning Cycling Switch                             | ON/OFF               |
| 1102-b0 | ACP        | A/C Head Pressure Sensor (Fan Ctrl                          | OPEN/CLOSED          |
| 1101-b1 | BOO        | Brake On/Off Switch                                         | ON/OFF               |
|  1128   | CAT TST1   | Catalyst #1 Test Frequency                                  | Hz                   |
|  1129   | CAT TST2   | Catalyst #2 Test Frequency                                  | Hz                   |
|  0200   | DTC CNT    | Continuous Code Counter                                     | Unitless             |
|  114E   | DPFEGR     | Differential Pressure Feedback EGR                          | VOLTS                |
|  114D   | ECTV       | Engine Coolant Temperature                                  | VOLTS                |
|  1139   | ECT        | Engine Coolant Temperature                                  | DEG                  |
|  11C0   | EPC        | Electronic Pressure Control                                 | PSI                  |
|  113C   | EGRVR      | EGR Vacuum Regulator                                        | Percent ON           |
|  110E   | FPA        | Fuel Pump Control                                           | ON/OFF               |
| 110C-b0 | FPM        | Fuel Pump Monitor                                           | ON/OFF/%             |
|  1141   | FUEL PW1   | Fuel Injector Pulse Width for Bank #1                       | mS                   |
|  1142   | FUEL PW2   | Fuel Injector Pulse Width for Bank #2                       | mS                   |
|  11B3   | GEAR       | Transmission Gear                                           | GEAR                 |
| 1103-b3 | HFC        | High Speed Fan                                              | ON/OFF               |
| 110E-b3 | HFCA       | High Speed Fan Output State Actual                          | ON/OFF               |
| 110C-b4 | HTR11A     | Oxygen Sensor Heater Actual                                 | ON/OFF               |
| 110C-b5 | HTR12A     | Oxygen Sensor Heater Actual                                 | ON/OFF               |
| 110C-b6 | HTR21A     | Oxygen Sensor Heater Actual                                 | ON/OFF               |
| 110C-b7 | HTR22A     | Oxygen Sensor Heater Actual                                 | ON/OFF               |
| 1102-b1 | HTRX1      | O2S Upstream Heater Control                                 | ON/OFF               |
| 1102-b2 | HTRX2      | O2S Downstream Heater Control                               | ON/OFF               |
|  114A   | IATV       | Intake Air Temperature                                      | VOLTS                |
|  1123   | IAT        | Intake Air Temperature                                      | DEG                  |
|  1153   | IAC        | Idle Air Control                                            | Percent OPEN         |
| 1103-b2 | LFC        | Low Speed Fan Control                                       | ON/OFF               |
| 110E-b2 | LFCA       | Low Speed Fan Control Actual                                | ON/OFF               |
|  1156   | LONGFT1    | Long Term, Fuel (KAMRF) Trim Bank #1                        | Percent OF RANGE     |
|  1157   | LONGFT2    | Long Term, Fuel (KAMRF) Trim Bank #2                        | Percent OF RANGE     |
|  115A   | LOAD(1)    | Calculated Engine Load                                      | Percent OF LOAD      |
| 1103-b6 | LOOP       | Fuel Control Type                                           | OPEN/CLOSED          |
|  1177   | MAFHV      | Mass Air Flow Rate                                          | VOLTS                |
| 1102-b5 | MISF       | Engine Misfire Status                                       | YES/NO               |
|  11B6   | TRV MODE   | Transmission Range Sensor                                   | GEAR                 |
|  1151   | TR-V       | Transmission Range Sensor                                   | VOLTS                |
|  1173   | O2S11(1/)  | Oxygen Sensor #11                                           | VOLTS                |
|  1174   | O2S12 (1/) | Oxygen Sensor #12                                           | VOLTS                |
|  1175   | O2S21 (1/) | Oxygen Sensor #21                                           | VOLTS                |
|  1176   | O2S22 (1/) | Oxygen Sensor #22                                           | VOLTS                |
| 1102-b3 | OCT ADJ    | Octane Adjust Control                                       | OPEN/CLOSED          |
| 1101-b3 | PNP        | Park Neutral Position Switch                                | DRIVE/NEUT           |
|  1165   | RPM        | Engine Revolutions Per Minute                               | RPM                  |
|  1158   | SHRTFT1    | Short Term Fuel (LAMBSE) Trim Bank #1                       | Percent of FULL TRIM |
|  1159   | SHRTFT2    | Short Term Fuel (LAMBSE) Trim Bank #2                       | Percent of FULL TRIM |
|  116B   | SPARK ADV  | Desired Spark Timing                                        | DEGREES              |
|  11B0   | TCC        | Torque Converter Clutch                                     | Percent              |
|  11BD   | TFTV       | Transmission Fluid Temperature                              | VOLTS                |
|  11B4   | TSS        | Transmission Speed Sensor                                   | RPM                  |
| 1101-b4 | TCS        | Transmission Control Switch                                 | ON/OFF               |
|  1154   | TPV        | Throttle Position                                           | VOLTS                |
|  1125   | TP MODE    | Throttle Position Mode                                      | CT/PT/WOT            |
|  1169   | TPCT       | Lowest TP Reading During Drive                              | VOLTS                |
| 1103-b7 | TRIP       | OBD II Trip Complete Except Catalyst Mntr                   | YES/NO               |
|  1172   | VPWR       | Vehicle Power                                               | VOLTS                |
|  1155   | VREF       | Internal PCM Reference Volts                                | VOLTS                |
|  11C1   | VSS        | Vehicle Speed Sensor                                        | MPH                  |
| 1104-b0 | WAC        | Wide Open Throttle Relay Status                             | ON/OFF               |
| 110E-b6 | WACA       | WAC Relay Status                                            | ON/OFF               |
| 1101-b2 | 4X4L       | 4 Wheel Drive Switch                                        | ON/OFF               |
| 1103-b4 | IMRC       | Intake Manifold Runner Control                              | ON/OFF               |
| 1103-b5 | MIL        | Malfunction Indicator Lamp                                  | ON/OFF               |
| 1104-b2 | TCIL       | Transmission Control Indicator Lamp                         | ON/OFF               |
| 1104-b4 | AIR        | Electronic Air Management                                   | ON/OFF               |
| 1105-b4 | SS1        | Shift Solenoid #1                                           | ON/OFF               |
| 1105-b5 | SS2        | Shift Solenoid #2                                           | ON/OFF               |
| 1105-b6 | SS3        | Shift Solenoid #3                                           | ON/OFF               |
| 110C-b1 | AIRM       | Electronic Air Management Monitor                           | ON/OFF               |
| 110C-b2 | FAN M      | Low Speed Fan Monitor                                       | ON/OFF               |
|  1166   | EVAPCP     | Canister Purge Duty Cycle                                   | Percent ON           |
|  1170   | FTP        | Fuel Tank Pressure Transducer                               | VOLTS                |
|  1627   | PF         | Purge Flow Sensor                                           | VOLTS                |
|  1629   | TPB        | Secondary Throttle Position Sensor                          | VOLTS                |
|  162A   | TATC       | Traction Assist Duty Cycle                                  | Percent ON           |
|  1634   | IMRCM      | Intake Manifold Runner Control Monitor                      | VOLTS                |
|  1636   | EVAPCVA    | Vapor Management Valve                                      | ON/OFF               |
|  1672   | FPD        | Fuel Pump Duty Cycle                                        | Percent ON           |
|  1673   | FPM        | Fuel Pump Monitor Duty Cycle                                | Percent ON           |
| 162F-b3 | AIRF       | Secondary Air Injection (AIR) output fault indicated        | ON/OFF               |
|  1127   | BARO       | Barometric Pressure                                         | Hz                   |
| 1101-b1 | BOO        | Brake applied                                               | ON/OFF               |
|  1609   | CATCAL1    | Bank 1 Catalyst ave. calibrated frequency                   | Hz                   |
|  160A   | CATCAL2    | Bank 2 Catalyst ave. calibrated frequency                   | Hz                   |
|  0101   | DRIVECT    | Number of OBD-II drive cycles completed                     |                      |
| 162E-b2 | EGRVRF     | EGR output fault (open circuit) detected                    | SHORT/OPEN           |
| 162E-b3 | EGRVRF     | EGR output fault (shorted to ground) detected               | SHORT/OPEN           |
|  11B2   | EPC V      | EPC Overcurrent Monitor                                     | VOLTS                |
| 162F-b2 | EVAPCPF    | Canister PURGE output fault detected                        | ON/OFF               |
|  1636   | EVAPCVA    | Canister VENT (VMV) output fault detected                   | ON/OFF               |
|  1672   | FP         | Duty cycle for modulated fuel pump control output           | Percent              |
| 162E-b6 | FPF        | Fuel Pump output fault detected                             | ON/OFF               |
| 162F-b1 | HFCF       | High Speed Fan output fault detected                        | ON/OFF               |
| 1631-b0 | HTR11      | Bank 1 Upstream O2S Heater ON                               | ON/OFF               |
| 1631-b4 | HTR11F     | Bank 1 Upstream O2S Heater output fault detected            | ON/OFF               |
| 1631-b1 | HTR12      | Bank 1 Downstream O2S Heater ON                             | ON/OFF               |
| 1631-b5 | HTR12F     | Bank 1 Downstream O2S Heater output fault detected          | ON/OFF               |
| 1631-b2 | HTR21      | Bank 2 Upstream O2S Heater ON                               | ON/OFF               |
| 1631-b6 | HTR21F     | Bank 2 Upstream O2S Heater output fault detected            | ON/OFF               |
| 1631-b3 | HTR22      | Bank 2 Downstream O2S Heater ON                             | ON/OFF               |
| 1631-b7 | HTR22F     | Bank 2 Downstream O2S Heater output fault detected          | ON/OFF               |
| 162E-b0 | IACF       | IAC Output fault detected (open circuit or short to ground) | ON/OFF               |
| 162D-b0 | INJ1F      | Fuel Injector #1 output fault detected                      | ON/OFF               |
| 162D-b1 | INJ2F      | Fuel Injector #2 output fault detected                      | ON/OFF               |
| 162D-b2 | INJ3F      | Fuel Injector #3 output fault detected                      | ON/OFF               |
| 162D-b3 | INJ4F      | Fuel Injector #4 output fault detected                      | ON/OFF               |
| 162D-b4 | INJ5F      | Fuel Injector #5 output fault detected                      | ON/OFF               |
| 162D-b5 | INJ6F      | Fuel Injector #6 output fault detected                      | ON/OFF               |
| 162D-b6 | INJ7F      | Fuel Injector #7 output fault detected                      | ON/OFF               |
| 162D-b7 | INJ8F      | Fuel Injector #8 output fault detected                      | ON/OFF               |
| 162F-b0 | LFCF       | Low Speed Fan output fault detected                         | ON/OFF               |
| 162E-b4 | MILF       | Service Engine Soon light (MIL) output fault detected       | ON/OFF               |
|  11B5   | OSS        | Transmission Output Shaft Speed                             | RPM                  |
|  1625   | PSP        | Power Steering Pressure Input                               | VOLTS                |
| 1632-b0 | SS1F       | Shift Solenoid #1 output fault detected                     | ON/OFF               |
| 1632-b1 | SS2F       | Shift Solenoid #2 output fault detected                     | ON/OFF               |
| 1632-b2 | SS3F       | Shift Solenoid #3 output fault detected                     | ON/OFF               |
| 162F-b7 | TATCF      | Secondary Throttle (Traction Control) output fault detected | ON/OFF               |
| 1104-b2 | TCIL       | Trans control indicator light on                            | ON/OFF               |
| 1101-b4 | TCS        | Trans Control button depressed                              | ON/OFF               |
| 1105-b2 | TOFMEM     | Transmission is over temperature                            | ON/OFF               |
| 1105-b1 | TOLOCK     | Transmission overtemp lockup mode                           | ON/OFF               |
|  1169   | TPCT       | Closed Throttle Position                                    | VOLTS                |
|  0100   | TRIP CNT   | Number of completed OBD-II trips                            |                      |

* (1) - Percent load NOT adjusted for atmospheric pressure.
* (1/) - Oxygen sensors are heated.
* *bx - Bitmap location

## Extended Commands

The following is a list of the extended commands given in the 95 Ford Mustang shop manual.  They are ran be setting
the header to `C4 10 F1` and the entering the codes sequentially.  Success is frequently indicated by observable
effects (e.g., engine fan motor cycling, etc.).

### Key On Engine Off (KOEO) Self-Test

| CODE   | TEXT   | EDP SCAN TOOL COMMAND                                                     |
| ------ | ------ | ------------------------------------------------------------------------- |
| 3381   | DTC(s) | 04,31,21,C4103381,,9E 00 445443287329 20 8042 20 8062 20 8082 A851 FF, 2E |
| 220202 | CNT    | 03,32,22FF,C410220202,,9E 00 434E54 20 8061 A961 00 04, EA                |
| 328100 | WAIT   | 02,32,21,C410328100,,9E 00 574149 54 20 8081 A181 61 03, 5E               |
| 3181   | START  | 01,32,21,C4103181,,9E 00 5354415254 20 8081 A181 00 02, 54                |

### Key On Engine Running (KOER) Self-Test

The BOO (brake on/off), 4x4L, and TCS (traction control switch) switches should be cycled if present after the test
begins.

| CODE   | TEXT   | EDP SCAN TOOL COMMAND                                                     |
| ------ | ------ | ------------------------------------------------------------------------- |
| 3382   | DTC(s) | 08,31,21,C4103382,,9E 00 445443287329 20 8042 20 8062 20 8082 A851 FF, 33 |
| 220202 | CNT    | 07,32,22FF,C410220202,,9E 00 434E54 20 8061 A961 00 08, F2                |
| 328200 | WAIT   | 06,32,21,C410328200,,9E 00 574149 54 20 8081 A181 61 07, 67               |
| 3182   | START  | 05,32,21,C4103182,,9E 00 5354415254 20 8081 A181 00 06, 5D                |

## Continuous DTCs

The standard OBD II mode 03 request only retrieves stored emissions related DTCs (diagnostic trouble codes).  The
following retrieves both stored emissions and non-emissions DTCs.  It should be entered with the key on and the
engine off.

| CODE   | TEXT    | EDP SCAN TOOL COMMAND                                           |
| ------ | ------- | --------------------------------------------------------------- |
| 13     | DTC CNT | 09, 2C, 21, C4 10 13, , 9E 00 44 54 43 20 43 4E 54 20 8B 44, B2 |


## Oxygen Sensor Monitor Test Results

The 95 Ford Mustang shop manual gives two Ford specific oxygen sensor test results accessible via the standard
mandated `05` oxygen sensor test results service in addition to the standard `01`, `02`, `07`, and `08` tests.

| TEST ID | DESCRIPTION                            | UNITS |
| ------- | -------------------------------------- | ----- |
| 41      | Oxygen sensor amplitude for test cycle | VOLTS |
| 61      | Oxygen sensor frequency for test cycle | Hz    |

## Ouput Tests

A variety of outputs can be toggled on/off.

### Normal On Outputs Off

| CODE     | TEXT    | EDP SCAN TOOL COMMAND                                         |
| -------- | ------- | ------------------------------------------------------------- |
| 25       |         | 02, 20, 20, C4 10 25,,9E 00 A1 91 00 03, 16                   |
| 3184     |         | 03, 22, 20, C4 10 31 84,,9E 00 A1 91 00 04, AA                |
| B1002501 | O/P OFF | 04, 2A, 20, C4 10 B1 00 25 01,,9E 00 4F 2F 50 20 4F 46 46, 6C |

### All Outputs On

| CODE     | TEXT   | EDP SCAN TOOL COMMAND                                      |
| -------- | ------ | ---------------------------------------------------------- |
| 25       |        | 02, 20, 20, C4 10 25,,9E 00 A1 91 00 03, 16                |
| 3184     |        | 03, 22, 20, C4 10 31 84,,9E 00 A1 91 00 04, AA             |
| B1002502 | ALL ON | 04, 2A, 20, C4 10 B1 00 25 02,,9E 00 41 4C 4C 20 4F 4E, 36 |

### Low Speed Fan On

| CODE     | TEXT   | EDP SCAN TOOL COMMAND                                      |
| -------- | ------ | ---------------------------------------------------------- |
| 25       |        | 02, 20, 20, C4 10 25,,9E 00 A1 91 00 03, 16                |
| 3184     |        | 03, 22, 20, C4 10 31 84,,9E 00 A1 91 00 04, AA             |
| B1002503 | LFC ON | 04, 2A, 20, C4 10 B1 00 25 03,,9E 00 4C 46 43 20 4F 4E, 33 |

### High Speed Fan On

| CODE     | TEXT   | EDP SCAN TOOL COMMAND                                   |
| -------- | ------ | ------------------------------------------------------- |
| 25       |        | 02, 20, 20, C4 10 25,,9E 00 A1 91 00 03, 16
| 3184     |        | 03, 22, 20, C4 10 31 84,,9E 00 A1 91 00 04, AA
| V1002504 | HFC ON | 04, 2A, 20, C4 10 B1 00 25 04,,9E 00 48 46 43 20 4F 4E, 2F

### All Outputs Off/Normal State

| CODE     | TEXT    | EDP SCAN TOOL COMMAND                                   |
| -------- | ------- | ------------------------------------------------------- |
| 3284     | ALL OFF | 05, 28, 20, C4 10 32 84,,9E 00 41 4C 4C 20 4F 46 46, 51 |

# References

The following references are mentioned in various documents.

* J1979 - E/E Daignostic Test Modes (legislated emissions standards)
* J2190 - Enhanced E/E Diagnostic Test Modes (extensions to standard that got deprecated but used by Ford)
* J2178 - Class B Communication Network  Messages - Detailed Header Formats and Physical Address Assignmnets