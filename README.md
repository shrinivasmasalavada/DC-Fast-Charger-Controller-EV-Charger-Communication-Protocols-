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
