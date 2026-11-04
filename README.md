# DC-Fast-Charger-Controller-EV-Charger-Communication-Protocols-
Charger-Side Hardware, and Buck / Boost Converters A Complete Technical Research &amp; Interview-Preparation Report Beginner → Intermediate Engineering Level Prepared for: Technical interview preparation — automotive / EV engineering startup Topics as advised by the engineering manager: DC fast charger controller · EV↔charger communication
1. Executive Summary
1.1 What is a DC fast charger?
A DC fast charger (also called DCFC, or "Level 3" informally in the US) is an off-board charging station that converts grid AC power into DC power inside the charger itself and feeds that DC directly into the vehicle's high-voltage (HV) battery. This is possible only because the charger takes over the job that the vehicle's on-board charger (OBC) normally does during AC charging: rectification, power-factor correction, galvanically-safe voltage conversion and current regulation. Because the power electronics are off-board, they can be sized far larger — commercial DC fast chargers typically deliver roughly 50 kW to 350+ kW per dispensing unit (architecture-dependent), versus roughly 3–22 kW for typical AC charging [Source: NREL/DOE infrastructure reports, §17].



1.2 Why is a charger controller required?
The power hardware alone is "blind". A charger controller (an embedded MCU/SoC system) is required because the whole session is a negotiated, closed-loop, safety-supervised process. The controller must:

Communicate with the EV's battery management system (BMS) over the CCS digital protocol, and with the vehicle's analog Control Pilot (CP) signalling, to agree exactly what voltage and current may be delivered, at every instant.
Regulate the power stage in closed loop: sample output voltage/current via ADC, adjust the PWM of the DC/DC modules, and track the EV's request within tolerance.
Protect: supervise insulation resistance, connector temperature, DC-link voltage, coolant temperature, contactor feedback, and execute safe shutdowns.
Manage the session: authorise the user, meter energy, talk to the payment/back-end server (OCPP), and log everything for diagnostics.
IEC 61851-23 makes this supervision a certification requirement for DC EVSE, including insulation monitoring, pre-charge control and emergency shutdown paths [Source: Bender, Gongyuan IEC 61851-23 guides — §17].

1.3 AC charging vs DC fast charging
AC charging (Mode 3, Type 2)	DC fast charging (Mode 4, CCS2)
Converter location	On-board charger (OBC) inside the EV	Off-board, inside the charger
What flows in the cable	AC mains voltage	Already-rectified DC (up to ~1000 V systems — architecture-dependent)
Typical power	~3–22 kW (limited by OBC)	~50–350+ kW (architecture-dependent)
Who sets charge rate	OBC (charger only signals available current on CP duty cycle)	EV BMS requests exact V/I; charger follows in closed loop
Communication	CP PWM only (IEC 61851-1); optional ISO 15118 for smart functions	CP PWM (5% = digital signalling) plus PLC data link: DIN 70121 / ISO 15118-2/-20
Battery-side switching	OBC output connects after internal contactors	Charger output must be pre-charged to battery voltage before HV contactor closure
Key standards: IEC 61851-1 (conductive charging, CP/PP, basic signalling) [Source: eInfochips IEC 61851 guide; Geocadra IEC 61851 summary — §17]; IEC 61851-23 (DC EVSE safety); DIN SPEC 70121 and ISO 15118-2/-20 (high-level DC communication); CCS = Combined Charging System combining these [Source: CharIN HPC guide — §17].

1.4 Power stage vs control/communication stage
Every DC fast charger is really two machines in one box:

Power stage — the "muscle": breakers, EMI filter, PFC rectifier, DC-link capacitors, isolated DC/DC modules, contactors, cabling. It handles hundreds of amperes. It does not think; it obeys.
Control/communication stage — the "brain and nervous system": MCU(s), gate drivers (the nerve endings), sensors (the senses), PLC modem + SECC (the translator that talks to the EV), HMI/RFID, and backend connectivity. It senses, decides, and commands.
The interview one-liner: "The power stage converts and delivers energy; the controller converts information — sensing the EV's request and enforcing safety — into precise PWM commands for that power stage."

1.5 The complete system in simple language
A driver plugs in. The charger notices the plug (Proximity Pilot) and the car (Control Pilot voltage change), locks the connector, and switches CP to 5% PWM to say "I can speak digitally". A data modem conversation (SLAC over the same CP cable) builds an Ethernet-over-powerline link. Car and charger exchange capabilities, the charger checks insulation, matches its output voltage to the battery (pre-charge), closes its contactors, and then delivers exactly the current the BMS asks for, re-checked ~10 times per second, with both sides ready to stop within milliseconds if anything looks wrong. When the BMS says stop, the charger ramps down, opens contactors, unlocks, and sends the billing record. That is the whole story this report unpacks.

2. Complete DC Fast Charger Architecture
Fig. 1 — DC Fast Charger: Power Path, Control Path, Communication Path
POWER PATH (thick energy flow)
AC Grid
400V 3-φ (typ.)
Circuit Breaker
+ Fuses + SPD
EMI Filter
CM/DM choke, X/Y caps
AC/DC Rectifier
with PFC (Vienna/two-level,
Si-IGBT or SiC) — topology-dep.
DC-Link Caps
e.g. 800VDC rail
DC/DC Power Stage
isolated LLC / DAB / buck-type,
often modular & interleaved
Output Contactor
+ pre-charge relay
CCS2 Connector
DC+, DC−, CP, PP, PE
→ EV battery
EV
CONTROL PATH (sense → decide → command)
Sensors: V_out, I_out (Hall/
shunt), DC-link V, temps
connector, coolant, modules
Charger MCU / Controller
ADC sampling, PWM gen, state
machine, safety supervision, logging
Power-Module Controller
+ Gate Drivers (isolated),
soft-start, module current sharing
Drives PFC + DC/DC
switching devices:
IGBT / Si-MOSFET / SiC
Contactors, IMD,
cooling fans/pumps,
aux PSU supervision
sensed on power path
actuates power path
COMMUNICATION PATH (talk with EV and cloud)
EV: BMS / EVCC
(in the vehicle — not the charger)
CCS2 Interface
Control Pilot wire carries both
PWM signalling and PLC data
PLC Modem (HomePlug Green PHY)
+ SECC — SLAC, DIN 70121 /
ISO 15118-2/-20 protocol stack
Charger Controller
executes negotiated setpoint:
V*/I* to power stage
Back-end / OCPP
auth., billing, remote diag.
via 4G/Ethernet (optional)
bidirectional
controller ⇄ SECC: setpoints, status
Legend: solid thick = energy · green = control/sensing · blue dashed = data/communication. Double-ended arrows are bidirectional.
Block-by-block explanation (power path)
Block
Purpose
Inputs → Outputs
Interaction with neighbours
Type
AC input / distribution
Safe entry of grid energy, phase distribution
Grid 3-φ AC → terminal busbars
Feeds breaker; SPG/earth stud reference
Essential
Breaker / fuses / SPD
Overcurrent & surge isolation; lightning/grid-transient clamp
AC in → protected AC out
Trips on fault; SPD status wired to MCU
Essential
EMI filter
Blocks switching noise from entering/leaving the grid (EMC)
Noisy AC → clean AC
Sits between grid and PFC; heatsink monitored
Essential (IEC 61851-23 EMC)
PFC rectifier (AC/DC)
Converts AC to DC while forcing input current sinusoidal (PF≈0.99)
3-φ AC → regulated DC-link (e.g. 800 V)
Gate-driven by controller; feeds DC-link caps
Essential
DC-link capacitors
Energy buffer; absorbs PFC ripple, supplies DC/DC stage
Charged rail → stiff DC bus
Voltage sensed by MCU; discharge circuit safety-faults it
Essential
DC/DC stage
Isolated voltage conversion, output V/I regulation (the "engine")
DC-link → regulated EV-side DC
PWM via gate drivers; parallel/interleaved modules
Essential (topology varies)
Contactors + pre-charge
Connect/disconnect EV safely; limit inrush into cable+caps
DC bus → CCS2 pins
Coil drivers + auxiliary feedback contacts to MCU
Essential
CCS2 outlet
Physical interface + CP/PP signalling + PLC coupling
DC+, DC−, CP, PP, PE
Cable temp sensed; lock solenoid driven
Essential
Figure 1 — Original overall architecture diagram. Power path (red), control path (green), communication path (blue). Topology names (Vienna, LLC, DAB) are typical examples, not universal — every such choice is architecture-dependent.
Architecture-dependent notes (Fig. 1). Many designs place the PFC and DC/DC stages in a separate power cabinet with their own module controllers; the "Charger MCU" may then be a hierarchy (site controller + power-module controllers + dispenser controller) connected by CAN or Ethernet. Some 400 V-class chargers use non-isolated buck-derived output stages; most modern high-power units use isolated stages (LLC, DAB, or CLLC). Liquid cooling is standard above ~150 kW; air-cooled below. Do not present any single topology as "the" topology in the interview.
3. Charger-Side Hardware and Components
Fig. 2 — Charger-Side Hardware Block Diagram (one plausible architecture)
AC Terminal + SPD
distribution, surge clamp
Breaker / Fuses
short-circuit protection
EMI / EMC Filter
CM choke + X/Y caps
PFC Rectifier
Si/SiC switches + gate drv.
current-loop + V-loop ctrl
DC-Link Caps
+ bleed resistors
DC/DC Modules
isolated, N parallel,
output relay + polarity
Contactor + Pre-chg
+ shunt/Hall sensor
→ CCS2 gun
Sensor group
V·I·T: output V/I, module T, coolant T, cable T, door/EM-stop
Main MCU / Controller
ADC · PWM · eCAP · CAN · Ethernet · state machine · logging
Gate drivers (isolated)
+ desat/OCP protection, soft-off
IMD + Ground check
insulation resistance monitor
Contactors / cooling
coil drv., pump/fan PWM
Aux PSU
12/24V, standby rail
CP/PP conditioning circuit
levels A–F, PWM 5%, diode monitor, lock
PLC modem (HomePlug Green PHY)
+ SECC stack (DIN 70121 / ISO 15118) — SLAC
Energy meter
class 0.5/1 billing-grade V·I·kWh
HMI / RFID / NFC
user interface, auth.
4G / Ethernet
OCPP to back-end
Red = energy chain · Green = control & safety chain · Blue = user & data chain. Arrows into the MCU are ADC/digital inputs; arrows out are PWM/GPIO/drive signals.


# DC Fast Charger Controller – Complete Technical Explanation

This topic covers four important areas of EV (Electric Vehicle) charging technology:

1. DC Fast Charger Controller
2. EV Communication Protocols
3. Charger-Side Hardware
4. Buck/Boost Converters

These are closely connected topics used in EV charging stations, power electronics, embedded systems and EV charger development.

Since you're interested in embedded systems and hardware testing, this is a useful technical domain to understand, especially for EV charger controller and hardware validation interviews.

## 1. DC Fast Charger Controller

[Eaton Green Motion DC EV Charger 3-phase 50kW 480V](https://images.openai.com/static-rsc-4/WmbNvAsjBklGcoeiCA0QHXUBgjGpQlJVbWaeMuq04DnQ12rd7_v-gjGyX1P0-JCmHr85671Z1tkIk1qqaza75g0_3el0LjRj0pJBO79hF3YFk4gxbm0nL7Oxh39Q7jd01rqmxJZObpRQF1J-rzZSEiHWjOPot__LehCDKZqtHR4?purpose=inline)

[Factory Supply DC EV Charger Controller of DC Fast Charging Station Control System - DC EV Charger Controller and DC Fast EV Charger Controller](https://images.openai.com/static-rsc-4/runPgEriFozzN9pszsroFSPmE-_7AN2idDF0uEBD8tagkRlGEZ36N5mbNEP2s9WGBfzyfXfM2MB7w01KmBFK7XtSqCoTyoJjwyVUr-FLoq9bjJ6s7rYDbk4rtIe-OKLeC6M8hjPRF2sKBnj-vdNp-064Ve3Eu7u1Pm5tiprJE7I?purpose=inline)

[UR100020-SW充电模块_优优绿能](https://images.openai.com/static-rsc-4/q-13R687or33o5vqkCuS7djp1YOX2SgxYr6MuNRttVnuYGMtWDEczrxotClL91scE6uG5FTbne3l1EYQ3UnXHQm3P60fI5ohwrk4bYQpN8HhESD4Oqqqe8Gi8Ida3-N-_kt4A_dvUHghDFXz4Q69ezYfbyNor5g5_tWdtLLg5qI?purpose=inline)

11

A DC Fast Charger (DCFC) supplies DC electrical power directly to an EV battery.

Unlike an AC home charger, which supplies AC power and relies on the vehicle's onboard charger to convert it into DC, a DC fast charger performs the main AC-to-DC conversion inside the charging station.

Basic working architecture:

[Webinar: Fast EV Charging with onsemi: Explore the 25kW EliteSiC Module Based Reference Design - Discrete Semiconductor Products - DigiKey TechForum - An Electronic Component and Engineering Solution Forum](https://images.openai.com/static-rsc-4/SDI3X40avXLE_kefachEJT2M9F_kuHnFCHKX1v_QQdgHvnLEQ9kA3kXqQ8bzM7PyIIvNQqm0iA6TBEM2D6ToUS2x2r5kvIMt_jyZW99yqTTzK42JSvNGDB3f2FnN_PJQt4JF112KpzJH09LWoyqYMo4XzPhJsHv24CgIYLHY9yo?purpose=inline)

[forum.digikey.com](https://forum.digikey.com/t/webinar-fast-ev-charging-with-onsemi-explore-the-25kw-elitesic-module-based-reference-design/31106?utm_source=chatgpt.com)

1. Grid supplies AC electricity.
2. AC-DC rectifier converts AC into DC.
3. Power factor correction (PFC) improves input power quality.
4. DC-DC converter adjusts voltage and current.
5. Controller manages the charging process.
6. DC power reaches the EV battery through the charging connector.

The controller monitors voltage, current, temperature, safety conditions and communication with the vehicle.

## 2. EV Communication Protocols

[European EV Charging Standards Explained](https://images.openai.com/static-rsc-4/V-w-jR2iLzw52Quz8qBMwlCFjBcDQbV8PygGzuuw_OXHamr_4z-kIPjDmYnrmHGzodCoPFK_-mMs1h7jpfI_XKUTeECwrvGqhJMwC4wMoQJVL-Xcecbs6bMvezY7mdANx8V_r3C9HbXafoNN-IlWzKa4M4BYQVjdjHo3QH7qf7w?purpose=inline)

[Building an Interoperability Test System for Electric Vehicle Chargers Based on ISO/IEC 15118 and IEC 61850 Standards](https://images.openai.com/static-rsc-4/Zn6d_els9kpI-AiMEP6FYN1eO7s23VfEhqIMqF955j-TX6efPHFo8d_VM6-jRt3mQYPTrakQ_flwG2fClZ9Dohu5xfTi8OvqeDS4TcBL62-RChYKjLr3xrIWZlqSi-yXuYiiW32U6Z_m4GPa_NawRiXmn0HXGv93Jb84FG9scj8?purpose=inline)

[Building an Interoperability Test System for Electric Vehicle Chargers Based on ISO/IEC 15118 and IEC 61850 Standards](https://images.openai.com/static-rsc-4/y1iG5v91kL8FcVrrhEsgS2kwPjD8mowJ7ja-OghkNzfTGwCyx2XSzkebsvdLrKuq2XdBmK_sW5SVwogeo0GPZ01CIlYqztlIJc40tVjd9ImMH1hpqz1hwElI-NQFeATh56hJs0aYCc-25vEpt2aBUvAst2JR7dVrYKTrwBlUM-4?purpose=inline)

7

The charger and EV must communicate before and during charging.

| Protocol   | Purpose                                                       |
| ---------- | ------------------------------------------------------------- |
| CCS / CCS2 | DC charging connector and charging interface                  |
| CAN        | Communication between electronic controllers                  |
| DIN 70121  | Digital communication for DC charging                         |
| ISO 15118  | Advanced EV-to-charger communication                          |
| CHAdeMO    | DC charging standard                                          |
| OCPP       | Communication between charger and charging management backend |

Important interview question: How does a charger communicate with an EV?

Example using CCS2:

- Vehicle connects to the charger.
- Charger detects the connection.
- Safety checks and connector locking take place.
- Communication is established using the applicable charging protocol.
- Vehicle BMS communicates battery voltage and charging requirements.
- Charger adjusts its output voltage and current.
- Charging stops safely when requested or when a fault occurs.

Remember: OCPP is primarily charger-to-backend communication, not the direct EV-to-charger protocol.

## 3. Charger-Side Hardware

[VerdeMobility](https://images.openai.com/static-rsc-4/HwUs7lJDmnejv5KTcUcXk6xTzh5n0Un7v0eTsfmNHujVp6WfhkxqwcWrJIvEQYtFvQYNazgw4tiq7DJGKCRYRtg7y47LMN6LLV14nIC7wzs6hKEHU_-GoGf0tFWlg5hdVKd9sKcvPl2gnswM-S9OdHZcBlFkMSg8pf7WV6p0-5k?purpose=inline)

[เอาชนะความท้าทายด้านการออกแบบสำหรับการชาร์จ EV | DigiKey](https://images.openai.com/static-rsc-4/f0In51G6hHmRuHGzQbc-ZaNdGLEqja8WXpSZSeQNniJvLrIrwKTiFfvAcN73VJa_OwbP1QB-qTnoaksa5aTAHjVK-odYz5lvS8H1XLIUFd4aQTiHfGl3fQoQls4YKRNteayO7P--ehdTWtP9Ivu8oFv-w7caauPc_yhRnd-QziU?purpose=inline)

[CXT-PLA3SA12550A - SiC MOSFET Intelligent Power Module 1200V/550A, 3-phase, AlSiC PinFin heatsink](https://images.openai.com/static-rsc-4/t1ORgutZcmHC_zTaOxgNml_k04N5L4LfJNOFKUx4mcU2a80Y7634VPwXbU5v8r9sv-TFMJflmu4LPoPkRsA_M-fGgJVCSepRdAYIKUJl2BxSEHMEwNa3jJcdn8fZcZIkdC6uD9BDqOLyWRmwtkwBPYoLMyvA3Zlb36CouuUJZBA?purpose=inline)

12

These are the major hardware components you should understand.

| Component                | Function                                   |
| ------------------------ | ------------------------------------------ |
| AC input circuit breaker | Protects against input faults              |
| EMI filter               | Reduces electromagnetic interference       |
| Rectifier                | Converts AC into DC                        |
| PFC circuit              | Improves power factor                      |
| DC-link capacitor        | Stabilizes DC voltage                      |
| SiC MOSFET / IGBT        | High-speed power switching                 |
| Gate driver              | Controls power semiconductor switching     |
| DC-DC converter          | Controls charging voltage/current          |
| Contactor                | Connects or disconnects high-voltage power |
| Current sensor           | Measures charging current                  |
| Voltage sensor           | Monitors voltage                           |
| Temperature sensor       | Monitors overheating                       |
| MCU / DSP                | Controls power electronics                 |
| PLC communication module | Communicates with the EV using CCS         |
| Cooling system           | Removes heat from power electronics        |

What would you do in a charger hardware testing role?

- Check PCB components and soldering quality.
- Perform visual inspection of connectors and wiring.
- Verify sensor readings.
- Test contactor operation and feedback.
- Monitor DC output voltage and current.
- Check temperature sensors and cooling fans.
- Inspect fault logs and communication errors.
- Validate emergency-stop and protection responses.

These are typical validation activities; actual procedures depend on the manufacturer's test plan and safety standards.

## 4. Buck and Boost Converters

[What is Buck Converter ? Working and Circuit - Nerds Do Stuff](https://images.openai.com/static-rsc-4/yTYZYOaz0OHwzd_5qO3O6ELL0YXz41_dmIYVTGMMVwNHjKcvMtm7BF854luql1dmPLdzEaQfzlPUjBYMaPQl9CXzxp_kQjICWJUokFhP1Uv9Q_WmTji0McjEylcbT7inJSw1agcKfcXwWAAbMldVf-WJfNt9G4q-netu0lWWNS0?purpose=inline)

[DC/DC Converters: Devices for Converting to a Higher Voltage - Technical Articles](https://images.openai.com/static-rsc-4/Urvgb_ZSypQTWILlJQi7fIYG_AvVXnWCh7eYPz4lttuZsIRdvSBdZS59mS5WrY2ylJlL3Oi1iU5YBqZpUV_5FmO3soEN-gDGsMXN4neVkhTKjN2Gcz5EsxAdORhMMxMagbw2LAmVzXi6LpxEn4JVyx-QATUwxa50R--qQ7hjrx0?purpose=inline)

[Selecting a MOSFET for Buck-Boost DC-DC Converters in Car USB PD Designs - Industry Articles](https://images.openai.com/static-rsc-4/5J-e78j-ZnjKuOmLBDvveLWjPz6cREAyLDNjC4i3bgAgfDhjwDzVFsJgeSiziT0mcRCq_B3Gbmq0XtAnOYysqp0H4RnLU8cRtnQIiKKnTy6V76oKmSUA813edMVexMhSbgy16pHhaIwNp8uQYvG6oryV78ewuoI-GvnIiVQEzLY?purpose=inline)

6

This is an especially important topic in EV charger interviews.

A. Buck converter – Step-down

A buck converter reduces DC voltage.

Example:

- Input: 400V DC
- Output: 300V DC
- Application: Reducing DC voltage to match a battery's charging requirements.

B. Boost converter – Step-up

A boost converter increases DC voltage.

Example:

- Input: 300V DC
- Output: 600V DC
- Application: Increasing DC-link voltage in suitable power-conversion architectures.

C. Buck-Boost converter

A buck-boost converter can increase or decrease DC voltage.

[A Review of Control Strategies for Four-Switch Buck–Boost Converters](https://images.openai.com/static-rsc-4/SbD-s_yjmMYAMYjtN_ZLv4QR5cftMYF5sSwEZ6nSLVGS6q9XcolyxS6ZSXgY-TRUk6cpSaeZtCZtw4FjcSUA6hgG6MoqPi_US1moSUrwbNPR7lMjaCs6ejF7evCNVd7DKW2bgPhhB6lDyCOPH7NiCo_ZKinLtWMjsHdKlzrknJw?purpose=inline)

[mdpi.com](https://www.mdpi.com/2032-6653/16/6/315?utm_source=chatgpt.com)

For example:

- Input voltage: 400V
- Required output: 350V → Buck operation.
- Required output: 600V → Boost operation.

The actual converter topology used in a commercial fast charger may be an isolated or non-isolated design, depending on the system architecture.

### Important formula

For an ideal basic buck converter operating in continuous conduction mode:

\\[ V\_{out}=D\times V\_{in} \\]

For an ideal basic boost converter:

\\[ V\_{out}=\frac{V\_{in}}{1-D} \\]

Where \\(D\\) is the switching duty cycle (between 0 and 1).

For example, a buck converter with 400V input and 75% duty cycle ideally produces:

\\[ 400\times0.75=300V \\]

Real circuits have switching losses, conduction losses and control limitations.

## 5. Complete architecture – How everything works together

[PowerFlex Doc Hub |](https://images.openai.com/static-rsc-4/PnTQEx6pyZnFmDi-SBeKzjvNGtZMrqNOeApZIRdOzYSPqbTP5e8mQzXtMWfIVKXdqv93RhLk8tUmhqT2z_2nLIQSU5ZZ159A-u__pjuq_avJSf-_UPQiM8c8K0ssvn_BkFF7euAgjKVl6vc5zA73rxcq3_zRmtVeicba6Va3UwE?purpose=inline)

[docs.powerflex.com](https://docs.powerflex.com/reference/how-dcfc-charger-works/?utm_source=chatgpt.com)

| Section            | Main responsibility                        |
| ------------------ | ------------------------------------------ |
| AC-DC stage        | Converts grid AC into DC                   |
| DC-link            | Provides stable intermediate DC            |
| DC-DC stage        | Regulates battery charging voltage/current |
| Charger controller | Coordinates operation and protection       |
| PLC controller     | Handles vehicle communication              |
| Sensors            | Provide real-time feedback                 |
| Contactors         | Control high-voltage connection            |
| BMS                | Reports battery charging requirements      |

A typical DC fast charger uses separate control functions for the power converter, vehicle communication and supervisory management.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.niti.gov.in\&sz=32)

Report1_EV-Update.docx+2



## 6. Important Interview Questions

Here are 15 questions to prepare for an EV charger controller or hardware engineering interview.

| No. | Interview question                                                 |
| --- | ------------------------------------------------------------------ |
| 1   | What is a DC Fast Charger?                                         |
| 2   | What is the difference between AC and DC charging?                 |
| 3   | What is the function of a DC charger controller?                   |
| 4   | What is the difference between CAN and PLC communication?          |
| 5   | What are CCS1 and CCS2 connectors?                                 |
| 6   | What is ISO 15118?                                                 |
| 7   | What is the purpose of OCPP?                                       |
| 8   | What is the function of a contactor?                               |
| 9   | What is a DC-link capacitor?                                       |
| 10  | What is the difference between MOSFET and IGBT?                    |
| 11  | What is a gate driver?                                             |
| 12  | What is the difference between buck and boost converters?          |
| 13  | How do you detect overcurrent and overvoltage?                     |
| 14  | What happens when a charger overheats?                             |
| 15  | How would you troubleshoot a DC fast charger that is not charging? |

### Three questions you should answer confidently

Q1. What is the difference between CAN and PLC?

- CAN: A wired communication bus commonly used between internal electronic controllers.
- PLC: Power Line Communication, which carries digital communication over the charging cable's control-pilot path in CCS.

Q2. What is the difference between ISO 15118 and OCPP?

- ISO 15118: Communication between EV and charging station.
- OCPP: Communication between charging station and central management system.

Q3. What happens if the charger output current becomes too high?

The controller detects the abnormal current using sensors and protection circuits, reduces or disables power switching, and disconnects the output using the appropriate protection sequence.

## 7. Your Practical Learning Roadmap

I recommend learning these topics in this order:

1. Electronics fundamentals

   Voltage, current, resistance, power, diodes, MOSFETs, capacitors and inductors.
2. Power electronics

   Rectifiers, PFC, buck, boost, buck-boost and DC-DC converters.
3. Embedded systems

   STM32, ESP32, ADC, PWM, UART, SPI and CAN.
4. EV communication

   CCS2, DIN 70121, ISO 15118, PLC and OCPP.
5. Hardware testing

   PCB inspection, sensor calibration, fault detection, contactor testing and troubleshooting.
6. EV charger architecture

   Understand how all these components work together.

## 8. Recommended Technical Resources

These are useful references for deeper learning:

1. NITI Aayog – [Fundamentals of Electric Vehicle Charging Technology](https://www.niti.gov.in/sites/default/files/2021-09/Report1-Fundamentals-ofElectricVehicleChargingTechnology-and-its-Grid-Integration_GIZ-IITB.pdf?utm_source=chatgpt.com). Particularly useful for understanding charger architecture.
2. Texas Instruments – [EV charging controllers and communication architecture](https://www.ti.com/lit/wp/slaaer3a/slaaer3a.pdf?ts=1779226122448\&utm_source=chatgpt.com).
3. onsemi – [DC Fast EV Charging: Common System Topologies and Power Devices](https://www.onsemi.com/company/newsroom/blog/automotive/dc-fast-ev-charging-common-system-topologies-and-power-devices?utm_source=chatgpt.com).
4. Microchip Technology – [Level 3 DC Fast Charging](https://www.microchip.com/en-us/solutions/automotive-and-transportation/charging-station/level-3-dc-fast-charging?utm_source=chatgpt.com).

My recommendation for your career preparation

Focus particularly on these four skills:

- Embedded C and STM32
- Power electronics and converter fundamentals
- CAN communication and CCS2/PLC architecture
- PCB hardware testing and troubleshooting

One important safety point: commercial DC fast chargers operate at dangerous high voltages and currents. Learn their architecture through simulations and low-voltage training circuits; never probe a live high-voltage charger without formal training and authorized safety procedures.

One question: Are you studying this topic for an EV charger controller job interview, or do you want to learn how to actually design and develop a DC fast charger controller?

That distinction will help me prepare the right practical training path.

# Advanced Level: DC Fast Charger Controller & EV Power Electronics

Let's move beyond basic definitions and study this at an EV Embedded Systems Engineer / Power Electronics Engineer level.

The objective is to understand how a commercial DC fast charger is designed, how its controller operates, how it communicates with an EV, and how engineers test and troubleshoot the system.

## 1. Advanced DC Fast Charger Architecture

[An Introduction to EV Charger Design: The Power Stages | EBV Elektronik](https://images.openai.com/static-rsc-4/aoeRlTZyHGeJz8r2r_-xc4R31it51rP-ZlWyTpayG1bvr9tsOa1e3l646L4tANhkf26EWfuIUqt088xfubHnjWgJuOb9UyUiXxurtf_egDn9yP1X_R7otRqFAp1TbDPhKSDROpp7LMpXayDEWWiMhbE0jj2eGQUGgA-WBtsnGMo?purpose=inline)

[my.avnet.com](https://my.avnet.com/ebv/resources/article/an-introduction-to-ev-charger-design-the-power-stages/?utm_source=chatgpt.com)

A typical high-power DC fast charger has the following architecture:

```
         THREE-PHASE AC GRID
                 |
          Circuit Breaker
                 |
              EMI Filter
                 |
          AC-DC PFC Stage
                 |
             DC LINK
                 |
       High Voltage Capacitor
                 |
        Isolated DC-DC Stage
                 |
        Output Filter / Sensors
                 |
          DC Contactors
                 |
             CCS2 Cable
                 |
          ELECTRIC VEHICLE
                 |
             Battery
```

The controller architecture is separate from the power path:

```
       MAIN CHARGER CONTROLLER
                 |
       +---------+----------+
       |         |          |
    MCU/DSP   PLC Module   HMI
       |         |          |
    Gate      CCS2 EV    Display
   Drivers   Communication
       |
  MOSFET / IGBT / SiC
       |
  Voltage & Current Feedback
       |
   Protection System
```

Important: Not every commercial charger uses the same topology. Some use modular power modules, interleaved converters, LLC resonant converters or dual-active-bridge converters.

## 2. Advanced Power Conversion Topologies

A. Three-phase Vienna Rectifier

Converts three-phase AC into regulated DC with high power factor and relatively low harmonic distortion. Commonly studied in high-power PFC applications.

[Control of interleaved boost converter - Electrical Engineering Stack Exchange](https://images.openai.com/static-rsc-4/h9zy2V4goQfX0POhIQwl5CkMNRaqD1qN8O3M08q7JChN8wnV0pkdnomEXOlf0BvuobIYdfwobYCrPXfd27QOeNNt3jUwA9MnRr8mD1flyxCLxIjXD-wpXZT9pe5CRX_7_ua6QrMROtFQAtDYrPtGSWp9dzqFjtOB49VMlGjBbpU?purpose=inline)

B. Interleaved Boost Converter

Uses multiple switching phases with staggered timing to reduce input-current ripple and distribute thermal stress.

[The Dual Active Bridge isolated DC/DC converter (a) the circuit diagram... | Download Scientific Diagram](https://images.openai.com/static-rsc-4/ppA1RWek23TPxXenZC98gJk_goQqxQ06OxTHh004yJ0JlSPquXBkBQ6uoAPyhgRGhaPBK7ZWjpIE3GuqlvKbvUd1XHM5byK8_K4mEdnhLPui4WBsJf1qV9xWYh9kkmsndMM3BaRkmBNkRCtWnj3EuIFEbCClb6EAHXX_4M0Myn8?purpose=inline)

C. Dual Active Bridge (DAB)

Uses two active switching bridges and a high-frequency transformer. Power flow is controlled through phase shift between the bridges. It supports bidirectional power conversion.

[Closed-form design optimization for LLC converters with wide output voltage range based on FHA | Scientific Reports](https://images.openai.com/static-rsc-4/j74E-rUi655s1nfkS0jAQ3rsWF5G6NOXF17wHNXwXmPOBxvF2e7HyMLwji5KLPEprnke37RyTN1l4qyQk0e3DCqBEc2LSmmYRn1FEkQqJsULJLy4lejma6udqZAeOj72EEMIk5SW4flJsvmmwOa2RNx7h1TG22-c1Pf1v3I5Dk8?purpose=inline)

D. LLC Resonant Converter

Uses resonant inductance, resonant capacitance and transformer magnetizing inductance. It can achieve soft switching and high efficiency.

## 3. Advanced Buck/Boost Converter Design

Let's understand the engineering calculations.

### A. Synchronous buck converter

[�X�C�b�\`���O���M�����\[�^�\[�œd�͂����艻����ɂ́FDC-DC�R���o�\[�^�\[���p�u���i2�j �d�͈��艻�i2�j�i2/2 �y�\[�W�j - EDN Japan](https://images.openai.com/static-rsc-4/yvRBhJpu7Bh07pev8eHmfMWHVGCcPeZdbSwT-IhP5kf-Fqz-3Dh4eh_Z5zJA6W1A2k72a81vhXl4XB8egoeY8mxXr7HuoQy7TzvLPfImL1aCtPvVzvG0nPkigO4Nm7HjtsIqYgTg3olNv9_BFInMYtdQFYH1FeQGRgbAQMxjCfo?purpose=inline)

[edn.itmedia.co.jp](https://edn.itmedia.co.jp/edn/articles/1708/14/news029_2.html?utm_source=chatgpt.com)

In a synchronous buck converter, two MOSFETs replace the conventional MOSFET-and-diode arrangement.

Ideal output equation:

\\[ V\_{out}=D V\_{in} \\]

Where:

- \\(V\_{in}\\) = input voltage
- \\(V\_{out}\\) = output voltage
- \\(D\\) = duty cycle

Example:

\\[ V\_{in}=800V,\quad V\_{out}=400V \\]

\\[ D=\frac{400}{800}=0.5 \\]

The ideal duty cycle is 50%.

### B. Boost converter

[Working Principles for Selecting a Boost Converter | Article | MPS](https://images.openai.com/static-rsc-4/oHd881y_hLLwcjJ4eeNRsqA0-1Wop2IFWfOcPO4HYysunWG_TlmwS3fyyMMrASDTL-g1proCtYdIYSnXzMEbUwtPgysnPvELk6bnKQSnuFwTMGoNjzOLu9K_YTznr_TPfBiOm240pWRLdGfU4nyWExf4dVmSr5Sp_iXpMtLz5ho?purpose=inline)

[monolithicpower.com](https://www.monolithicpower.com/en/learning/resources/working-principles-for-selecting-a-boost-converter?utm_source=chatgpt.com)

Ideal equation:

\\[ V\_{out}=\frac{V\_{in}}{1-D} \\]

Suppose:

\\[ V\_{in}=400V,\quad V\_{out}=800V \\]

\\[ D=1-\frac{400}{800}=0.5 \\]

Ideal duty cycle = 50%.

### C. Advanced converter parameters

Engineers must calculate:

- Switching frequency.
- Inductor ripple current.
- Capacitor ripple voltage.
- MOSFET voltage and current ratings.
- Switching and conduction losses.
- Thermal resistance and heatsink requirements.
- Efficiency and electromagnetic interference.

A simplified inductor ripple equation for a buck converter is:

\\[ \Delta I_L = \frac{(V\_{in}-V\_{out})D}{L f_s} \\]

Here, \\(L\\) is inductance and \\(f_s\\) is switching frequency.

## 4. Advanced Controller Design – MCU / DSP

[LaunchPad Development Kits – Texas Instruments | DigiKey](https://images.openai.com/static-rsc-4/oSaurx1t-8n8MKODajf1WDthPVNaZTN1ClfELrH6A8-4R43_cPx1DR0A-ctrd5VMiaeSTbdL1KZSdPUlAv-Fn1qPOkaw5GdXlV4LNjpBzdRyre5giUKmLx9zvDrjjexkYFOQqYPfQEuzHsSKqAaADvesfH18dN8FHmmK87vaEYc?purpose=inline)

[Waveshare STM32H7 Development Board for India | Ubuy](https://images.openai.com/static-rsc-4/7mt7sUPRRDOjcuopAm0D585q5X4Uz8t2QDrcbg5eGA2QkvnkOzycyxauk3kUkTm2hZrXbZwA53BhPmhdd8tN22tD5x9n84TRd5fpPkIyFedpukaKh_nMqmS2AUxkhRSD3xuTCeSGAYGIPhpkw-HA_FE02Mt6GzwsbGcwwjEUkQk?purpose=inline)

[ISO5852SDWEVM-017 | Buy TI Parts | TI.com](https://images.openai.com/static-rsc-4/n-gL77gIzz5LRwruki01wfvd6WHDuVED_qRcREIl3lICmsfH2vmKAbNuGdKBK2nJMO4wGZzyJZmVIYI0fErPDzXpglZOv5zeD7HKL_qHVsRfAXyLbzho7IZkW5OGk147AEyupwaWXNR-TopBlhvogvyyjODcJFYZBwHE38X60Bk?purpose=inline)

7

The charger controller is the brain of the system.

Common controller platforms include:

- Texas Instruments C2000 – Digital power conversion and motor control.
- STM32 – Embedded control, monitoring and communication.
- Microchip dsPIC – Power conversion and motor control.

### What does the controller actually do?

| Function               | Engineering implementation      |
| ---------------------- | ------------------------------- |
| Voltage measurement    | ADC                             |
| Current measurement    | ADC / isolated sensors          |
| PWM generation         | Advanced timer / PWM peripheral |
| MOSFET control         | Isolated gate driver            |
| Communication          | CAN, UART, SPI, Ethernet        |
| Fault detection        | Hardware comparators + firmware |
| Temperature monitoring | NTC / RTD / digital sensors     |
| Charging regulation    | PI control loops                |
| System supervision     | State machine / RTOS            |

### Advanced control loops

A DC fast charger generally has multiple nested control loops.

[Cascade-Type Pole-Zero Cancellation Output Voltage Regulator for DC/DC Boost Converters | MDPI](https://images.openai.com/static-rsc-4/EheCkTwuG26f1SOnxYrWSuy21hU1CGyT7EyBKZoa6QXHVDuIGYs_1Zz8mev1AI_IB2AN_bJEEyHhdQ5dXpograRWXFoAdKMAboAGP4LiJ5FQoNOoCjU7zE-Br12sMb5_KW_udVMZhwHuRqXUE8hOag2f7i7CAhL0zF0Ol58UKFM?purpose=inline)

[mdpi.com](https://www.mdpi.com/1996-1073/14/13/3824?utm_source=chatgpt.com)

1\. DC-link voltage control

Maintains a stable intermediate DC voltage.

2\. Converter current control

Regulates the current flowing through the converter.

3\. Battery output voltage control

Keeps the output within the requested voltage limits.

4\. Power management

Coordinates current limits, power limits, temperature derating and protection.

A PI controller is frequently used:

\\[ u(t)=K_p e(t)+K_i\int e(t)dt \\]

Where:

- \\(e(t)\\) = reference minus measured value.
- \\(K_p\\) = proportional gain.
- \\(K_i\\) = integral gain.

In a real implementation, engineers also consider sampling delays, PWM timing, saturation, anti-windup and control-loop stability.

## 5. Advanced EV Communication – CCS2 and PLC

[High-Power DC Charging Solutions for CCS Type 2 by Beihai](https://images.openai.com/static-rsc-4/QcwtDFlL6D3qMKCPZJypwhejcY39fsKlFuRxqkoUn_bPI_V-E49n-_osNhhqpITtG2cvMUEp2Qhb7S2WRTh2w_-GRv-pXKLCi1f4V27aAUfd1YCi79a6ZSOkiypY3Uq-68rrNTaQ2WmM9sRiyZCAIqrMwBfQUUYiUGcERGAHtak?purpose=inline)

[beihaipowers.com](https://www.beihaipowers.com/news/research-on-dc-charging-system-for-high-power-dc-charging-piles-ccs-type-2/?utm_source=chatgpt.com)

This is one of the most important areas for an EV charger embedded engineer.

CCS2 charging involves multiple communication and safety layers.

Communication sequence:

EV connects to charging station

Control Pilot / Proximity Pilot detection

PLC communication and SLAC matching

DIN 70121 / ISO 15118 session negotiation

Charging parameters exchanged

Insulation and safety checks

Controlled DC charging begins

### Important communication technologies

| Technology         | Advanced purpose                                                   |
| ------------------ | ------------------------------------------------------------------ |
| HomePlug Green PHY | PLC physical/data-link communication                               |
| SLAC               | Signal-level attenuation characterization                          |
| DIN 70121          | Digital DC charging communication                                  |
| ISO 15118-2        | EV-to-EVSE application communication                               |
| ISO 15118-20       | Advanced charging communication, including bidirectional functions |
| OCPP 2.0.1         | Charger-to-backend management communication                        |
| CAN                | Internal controller communication and other vehicle architectures  |

Advanced interview question: What is SLAC?

SLAC stands for Signal Level Attenuation Characterization. It helps the EV and charging station establish the appropriate PLC communication association over the charging cable.

## 6. Advanced Protection and Fault Detection

A professional DC charger must detect faults before they become dangerous.

| Fault              | Detection mechanism                  | Typical response         |
| ------------------ | ------------------------------------ | ------------------------ |
| Overcurrent        | Current sensor / hardware comparator | Disable power stage      |
| Overvoltage        | Voltage sensing                      | Stop power conversion    |
| Undervoltage       | Voltage monitoring                   | Inhibit or stop charging |
| Overtemperature    | NTC / RTD                            | Derate or shut down      |
| Insulation fault   | Insulation monitoring device         | Prevent or stop charging |
| Welded contactor   | Auxiliary contact / voltage checks   | Lock out operation       |
| Ground fault       | Ground-fault monitoring              | Protective shutdown      |
| Communication loss | Protocol timeout                     | Controlled charging stop |

An important concept is fault hierarchy:

- Hardware protection: Fast response to dangerous electrical faults.
- Firmware protection: Monitoring, fault handling and state transitions.
- Supervisory protection: System-level coordination and reporting.

Hardware protection must not depend solely on software execution.

## 7. Advanced Embedded Firmware Architecture

A charger controller needs reliable firmware, not just basic Arduino-style programming.

A possible firmware architecture:

```
DC FAST CHARGER FIRMWARE
│
├── Hardware Abstraction Layer
│   ├── ADC Driver
│   ├── PWM Driver
│   ├── GPIO Driver
│   └── Temperature Sensor Driver
│
├── Power Control
│   ├── PFC Control
│   ├── DC-DC Control
│   ├── Voltage Regulation
│   └── Current Regulation
│
├── Communication
│   ├── CAN
│   ├── UART
│   ├── SPI
│   ├── PLC Interface
│   └── Ethernet
│
├── Safety Management
│   ├── Overcurrent Protection
│   ├── Overvoltage Protection
│   ├── Thermal Protection
│   └── Insulation Monitoring
│
├── Charging State Machine
│
└── Diagnostics and Fault Logging
```

### Charging state machine

| State         | Operation                        |
| ------------- | -------------------------------- |
| IDLE          | Wait for vehicle                 |
| CONNECTED     | Detect connector                 |
| COMMUNICATION | Establish PLC communication      |
| AUTHORIZATION | Verify charging permission       |
| PRECHECK      | Perform electrical safety checks |
| PRECHARGE     | Prepare output voltage           |
| CHARGING      | Regulate DC output               |
| STOPPING      | Ramp down power and stop         |
| FAULT         | Enter safe shutdown              |

This state-machine design is particularly important for embedded firmware interviews.

## 8. Advanced Hardware Testing and Validation

[NH Research 4760 Programmable DC Electronic Load - 600 Volts](https://images.openai.com/static-rsc-4/nKG-lRmx1_8Lg2D66osAhWqHHYXDcUqrEKMqAVAQuzA7C8zsWtiTGKtDkbZP2CkatuwWoxue5dK0jU35Jz2k81lM1IcTmUSJtIdgrcxJbVuF9kwtmTdrwIG3-1hS7vspFNSdj_NARq39qLvq8IAygd4GsIFaCF1rGYf13rdo7cs?purpose=inline)

[UCC21222-Q1: Switching frequency - Power management forum - Power management - TI E2E support forums](https://images.openai.com/static-rsc-4/Vakb4t0E1OHWprk2YOpOEx4KnzisBX_Awbb2S8ke0lOtqysL7MCitqSXNuYbeg1CFOvE99i7vtU4cIKhUTidZ_eVo_-_1p7U6H3eOiMOyrcj79UrOe7jMy8cIv3Ii5yVG56mNWk-x2Pyyhc5uTR2B1acd3eifyb8nVtWeOpkoCs?purpose=inline)

[Thermal Design Best Practices for EV Chargers | Ohmframe Blog | Ohmframe](https://images.openai.com/static-rsc-4/JfJjXADH655QKMWtntmwPLlaw5cKWj3Mseb7ZxPwVSgiMrJ_eyu5bf0YlBMa2wkFU0a1QiRQRbR9D1oxddiIt1iTK0402xe64HuOEUuncBjS0fhcK-v8vyNcfQPYRBN9xLRc8D5-MbgX9JrX6d4PJQmA3t_DOxu3IqSu5OfCHyw?purpose=inline)

9

An EV charger validation engineer should understand these tests:

| Test                   | Equipment                               |
| ---------------------- | --------------------------------------- |
| DC output voltage test | High-voltage differential probe / meter |
| Current measurement    | Current probe / calibrated sensor       |
| PWM signal test        | Oscilloscope                            |
| Gate driver test       | Oscilloscope and differential probe     |
| CAN communication test | CAN analyzer                            |
| Thermal test           | Thermal camera                          |
| Insulation test        | Appropriate insulation test equipment   |
| Power efficiency test  | Power analyzer                          |
| Converter load test    | Programmable DC load                    |

Other advanced validation areas include:

- Ripple and transient response testing.
- EMI/EMC testing.
- Short-circuit protection validation.
- Converter efficiency mapping.
- Thermal derating validation.
- Communication timeout and recovery testing.
- Contactor welding detection.

High-voltage testing requires properly rated instruments, interlocks, barriers, discharge procedures and trained personnel.

## 9. Build an Advanced Practical Project

I recommend building a small-scale DC-DC converter controller before attempting a full EV charger.

Recommended learning project

Project: Smart DC-DC Converter Controller with CAN Monitoring

[GitHub - zeruns/Synchronous-Rectification-Buck-Boost-Digital-Power-Supply-Based-on-STM32: 一款基于STM32G474的四开关Buck-Boost数字电源，支持TypeC接口PD诱骗输入和DC5.5接口输入，输入/输出最高48V10A，这是我的毕业设计，现在开源出来，含原理图、PCB、程序源码、外壳3D模型等资料。](https://images.openai.com/static-rsc-4/jVyAIEqp3jsCJY4yW59WlmKgkwZPGv0f9aqGeqPFKOEmLsNtJdy1VBZxmYD5D0jwOjI7ldt9RvZw3AhVXCMcSgA8Ozd7gCKfKgizdi9sjOLTuXmQ3G_y8k7WEJdrordF4CEhGvAHqa4GT0K19KsEdJNP03yvTxH3jgf5ZIM8goA?purpose=inline)

[github.com](https://github.com/zeruns/Synchronous-Rectification-Buck-Boost-Digital-Power-Supply-Based-on-STM32?utm_source=chatgpt.com)

Hardware

- STM32 development board
- Low-voltage buck-boost converter
- Voltage sensor
- Current sensor
- Temperature sensor
- CAN transceiver
- LCD / OLED display
- Electronic load

Software

- Embedded C
- STM32 HAL
- PWM control
- ADC measurements
- PI controller
- CAN communication
- Fault monitoring

Expected results

1. Measure input and output voltage.
2. Monitor converter current.
3. Control output voltage using PWM.
4. Implement current limiting.
5. Detect overheating.
6. Send measurements through CAN.
7. Display faults and operating conditions.

## 10. Advanced Technical Skills Checklist

Your advanced learning tracker

0/20 completed

Power electronics

Three-phase rectifiers

PFC and DC-link design

Buck/Boost converters

LLC and DAB topologies

MOSFET and SiC switching losses

Embedded systems

STM32 / TI C2000

Embedded C

ADC and PWM

PI control algorithms

Interrupts and RTOS

EV communication

CAN and CAN FD

CCS2 architecture

PLC and SLAC

DIN 70121

ISO 15118 and OCPP

Hardware validation

PCB debugging

Oscilloscope measurements

Sensor calibration

Fault injection and protection

Thermal and efficiency testing

## 11. Choose Your Next Advanced Training

I can take you further into actual engineering-level learning.

Which area do you want to master first?

1\. DC Fast Charger Controller – complete hardware and firmware design

2\. Power Electronics – PFC, SiC MOSFET, LLC, DAB and converter calculations

3\. EV Communication – CCS2, PLC, SLAC, ISO 15118 and CAN

4\. EV Charger Testing – PCB debugging, calibration and fault analysis

5\. Complete EV Charger Simulation – MATLAB/Simulink and practical models

Preferred explanation

Advanced theory and engineering calculations

Practical implementation, circuits and coding

Both – complete industry-level training

Start advanced training&#x20;
