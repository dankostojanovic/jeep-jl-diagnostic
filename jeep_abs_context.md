# Car Diagnostic Context — 2018 Jeep Wrangler JL

> Paste this at the start of a new Claude session to continue debugging.

## Vehicle
- **Year/Make/Model:** 2018 Jeep Wrangler JL
- **Trim / Engine:** Sport (base trim), manual transmission
- **Mileage:** 71,500 miles

## Diagnostic Tools
- VSLinker MC+ (OBD2 adapter)
- JScan app (Jeep-specific diagnostics & module initialization)

---

## Symptom Timeline

### Pre-existing / Intermittent (before main failure)
These symptoms existed even when the car appeared "normal" — likely early signs of the underlying issue:
- Occasional loud pop through the speakers (no apparent trigger)
- Cruise control would randomly stop working, restored only by cycling ignition
- Infotainment system was janky/unreliable (may or may not be related)

### Main Failure — Initial Presentation
- ABS not working
- Power steering very hard to turn (EHPS — electric hydraulic power steering — affected)
- Dashboard lit up with ABS-related warning lights/errors
- Infotainment system stopped working entirely

### Repair Performed
- Replaced the ABS module (hardware)
- Used VSLinker MC+ + JScan to initialize/configure the new module
- All warning lights cleared; system appeared fully functional

### Recurrence (after ~2 days parked)
- Same ABS warnings returned
- Car would not fully shut down — long-pressing start button moved to ACC but never powered off
- Suggests a module is holding the CAN bus active and blocking normal shutdown sequencing

---

## Additional Repairs Performed (Post-Initial ABS Failure)
- Replaced brake rotors, pads, and brake hoses
- No visual damage found on ABS wiring or wheel speed sensors during this work
- ABS module is located high in the engine bay — was not disturbed during brake work, ruling out accidental connector disturbance

---

## Diagnostic Investigation So Far

### Fuse Pulls (no effect on symptoms)
- Pulled **F34** (ESC/EHPS/SBCM Wake Up) — symptoms remained
- Pulled **F100** — symptoms remained

### Relay Findings — K14 and K15 Tested, Mechanically Normal
- **K14 = Run/Acc #1** and **K15 = Run/Acc #2** found significantly hotter than all other relays
- BCM controls both Run/Acc relays; both being stuck on means BCM is holding the car in a powered state
- BCM may be a victim (responding to a module keeping CAN bus awake) rather than the root cause

**Bench test results — relays are mechanically normal, no further testing needed:**
- Coil resistance (pins 85/86): all 13748474 relays in PDC read 0–0.4 ohms — this is normal for this relay design, not a fault
- Contact test (pins 30/87 at rest): open circuit — contacts are not welded
- Heat on K14/K15 is caused by something commanding them to stay on, not a relay fault

**Open question:** What is commanding K14/K15 to stay energized? Focus shifts to BCM and CAN bus — JScan DTC scan is the most important next step.

### Physical Inspection
- Inspected CAN bus connectors behind glovebox — no discoloration, no loose connections, visually normal

### Key Module of Interest
- **Security Gateway Module (SGW)** — fuse F05 in PDC. Sits between OBD port and CAN bus. A fault here can cause cascading module communication failures and abnormal sleep/wake behavior across the whole network.

### Battery Info
- **Main battery:** A few months old — load tested GOOD
- **Auxiliary battery:** Factory rating 200 CCA — load tested at 300 CCA, exceeds spec. In good health. Low voltage was from cranking without main battery; resolved after charging.

### DTC Scan Results (full scan, 3/8/2026, odometer 71264)
Car would not start and could not be turned off during this scan.

**Key finding: CAN C1 Bus Off (U0002-00)** — the CAN bus shut itself down due to excessive errors. This is the root cause of the avalanche of "lost communication" codes across all modules. It is one failure cascading everywhere, not multiple independent failures.

**Critical BCM codes (active):**
- **U0121-00** — lost communication with ABS module
- **U0100-00** — lost communication with ECM/PCM
- **U0161-00** — lost communication with compass module
- **P1276-14** — starter control 2 circuit short to ground or open (explains no-start)
- **B2103-15** — ignition run/start 1 control circuit short to battery or open
- **B2121-15** — ignition run control 1 circuit short to battery or open (explains can't turn off)
- **B212E-15** — ignition run/acc control 1 circuit short to battery or open
- **B2119-15** — ignition run/acc/spad control circuit short to battery or open
- **B2335-15** — horn control circuit short to battery or open
- **B2312-15 / B2316-15 / B231A-15** — wiper controls circuit short to battery or open
- **U1514-87** — engine controller secret code missing message
- **U0002-00** — CAN C1 bus off (stored)
- **U0184-00** — lost communication with radio (stored)

**Other modules — all showing CAN bus / communication faults consistent with bus-off:**
- Instrument Panel Cluster: U0121, U0131, U0002, U1464
- Radio Frequency Hub: U0002, U0121, U0140, U0001, B2199 (low battery voltage)
- Steering Column Control Module: U0001, U0422
- Tire Pressure Monitor: U0100, U0121, U0140, U0422
- HVAC: U0422, U0184
- Integrated Center Stack: U0184
- Radio: B222C (vehicle configuration not programmed), U0422

**Most likely culprit: ABS module or its wiring** — appears as lost communication in every single module, was recently replaced, and a faulty CAN node can cause bus-off errors. 

**Immediate test: unplug the ABS module connector** and rescan — if CAN bus recovers, ABS module or its wiring is confirmed as the cause.

---

## Working Theory
The pre-existing symptoms (speaker pops, cruise control dropout) suggest **CAN bus communication instability that predates the main failure**. The combination of ABS faults, EHPS loss, infotainment failure, no-shutdown behavior, and both Run/Acc relays having shorted coils points to a **systemic electrical/network issue** rather than a simple bad ABS module.

Likely root cause candidates (all hypotheses except relay failure — need further diagnosis):
- K14 and K15 are mechanically normal — relays are being commanded to stay on by something upstream
- A module stuck awake or actively faulting on the CAN bus (ABS, BCM, or SGW) preventing system sleep
- A wiring or ground fault causing one or more modules to misbehave
- The ABS module replacement may have been masking a deeper underlying problem

---

## Open Questions
- What is commanding K14/K15 to stay on? BCM or a module keeping CAN bus awake?
- Is the ABS module detected by JScan at all when reconnected?
- Is the SGW (F05) involved — causing cascading CAN bus faults?
- What do full DTCs show across BCM and SGW specifically?
- Did something upstream cause the relay coils to short, or did they fail on their own?

---

## Suggested Next Steps (Prioritized)

**Step 1 — Unplug ABS module connector and rescan with JScan**
- If CAN bus recovers (U0002 clears, communication faults drop) → ABS module or its wiring is confirmed cause
- If CAN bus remains offline → fault is elsewhere on the network

**Step 2 — Based on Step 1 result**
- ABS confirmed → inspect ABS module wiring harness and connector for damage, corrosion, bent pins; confirm whether replacement module is OEM, remanufactured, or used; consider replacing module again or trying a known-good unit
- ABS not confirmed → systematically unplug other CAN nodes one at a time to find which one is pulling the bus offline

**Step 3 — Address no-start / no-shutdown**
- Likely resolves once CAN bus is restored; BCM ignition control codes (B2103, B2121) are probably a consequence of bus-off, not independent failures
- If they persist after CAN bus recovery, BCM may need further investigation

---

## Session Notes

### Session 1 — 3/8/2026
- **Actions taken:**
  - Load tested both batteries — main good, aux good (300 CCA vs 200 CCA factory rating)
  - Bench tested K14 and K15 relays — coils and contacts normal, relays are not the problem
  - Reinstalled both batteries
  - Ran full JScan DTC scan — see DTC Scan Results above
  - Unplugged ABS module connector as part of testing
- **Current state:** ABS module connector is unplugged. Car would not start and could not be turned off during scan. This is expected with ABS unplugged and CAN bus offline.
- **Immediate next step:** Plug ABS module back in, clear all DTCs, rescan to see what remains without testing artifacts

### Session 2
- **Date:**
- **DTCs found:**
- **Actions taken:**
- **Outcome / next steps:**

---

## ⚠️ Handoff Note — Pick Up Here
**Date:** 3/8/2026

The ABS module connector was unplugged during diagnostic testing. The current no-start and no-shutdown condition is a direct result of this — not a new problem.

**First thing to do in this session:**
1. Plug the ABS module connector back in (firmly, locking tab must click)
2. Clear all DTCs via JScan
3. Rescan all modules
4. Report remaining codes — those will represent the true underlying issue without testing artifacts
