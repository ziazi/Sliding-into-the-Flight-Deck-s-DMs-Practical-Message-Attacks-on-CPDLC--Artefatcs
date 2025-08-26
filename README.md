# Sliding into the Flight Deck's DMs: Practical Message Attacks on CPDLC --Artefatcs
This repository contains the artefacts associated with the attacks described in **Section 5 (Practical Attacks on CPDLC)** of the paper. The logs were captured using `dumpvdl2` under the experimental setup detailed in **Section 6.2 (Laboratory Configuration)**.  

---

## Methodology

- **Traffic Capture:** All logs were collected using `dumpvdl2`, ensuring a faithful representation of the CPDLC traffic during experiments.  
- **DoS Attack Procedure:** For all denial-of-service (DoS) attacks (except the *Malformed Frame Injection*, which is included only for illustrative purposes), the aircraft hardware was prompted to establish a CPDLC session with the legitimate test ATC.Upon timeout or disconnection, a new session was requested to maximize resilience attempts against our attacks, simulate a more realistic scenario, and capture as many teardown events as possible within the experiment duration. The aircraft hardware still failing to establish a CPDLC session demonstrated the potential severity of such attacks.
- **Rogue Ground Station Transmission:** In the *RGS* experiment, the aircraft hardware was induced to accept adversarial CPDLC messages, which were displayed as legitimate.  

---

## Log Files

### 1. `Legitimate_Connection_anonymized.log`  
Demonstrates a legitimate connection between an aircraft and a ground station, with CPDLC transmissions conducted without interference. Serves as a baseline reference.  

### 2. `RGS.log`  
Associated with **Section 5.2 (Rogue Ground Station)**.  
Shows that a Rogue Ground Station can transmit malicious CPDLC messages to the aircraft, which are accepted and displayed as legitimate.  

### 3. `FRMR_DOS_anonymized.log`  
Associated with **Section 5.1.1 (Malicious FRMR Injection DoS)**.  
- Demonstrates how targeting X.25 RR frames can sustain a DoS indefinitely without requiring extensive protocol knowledge.  
- Targeting key AVLC I-frames would follow an analogous process.  

> **Note:** Attacks were only launched immediately after an aircraft X.25 RR frame. Any other FRMR frames observed in the log are the result of legitimate link layer mismatches.  

### 4. `Broadcast_DOS_anonymized.log`  
Associated with **Section 5.1.2 (Broadcast DoS)**.  
Demonstrates how leveraging AVLC U DISC frames with the broadcast address allows indefinite disruption at negligible cost.  
The attack was launched across multiple protocol stages, including:  
- During IDRP updates  
- Immediately before CPDLC message processing (causing message loss)  
- During COTP connection setup  
- During ATN CM Logon  

### 5. `CFI_IDRP_anonymized.log`  
Associated with **Section 5.1.3 (Control Flow Injection on IDRP)**.  
- Demonstrates targeted disruption of the IDRP FSM by injecting an IDRP CEASE PDU, forcing the aircraft into the CLOSED state.  

> **Note:** The attack was launched only immediately after ground station IDRP updates. Any additional CEASE or ERROR PDUs in the log result from the IDRP state mismatches induced by the attack.  

### 6. `MalformedFrame_anonymized.log`  
Associated with **Section 5.1.4 (Malformed Frame Injection)**.  
- Demonstrates the injection of a malformed CPDLC message with an incorrect ATN checksum.  
- This results in link state resets.  
- While more illustrative than practical, the log highlights a design flaw: although COTP Protocol Class 4 natively supports error reporting, the ATN checksum was implemented as a user data field, leading to unintended consequences.  

---

## Summary

These logs serve as concrete artefacts validating the feasibility and impact of the attacks described in **Section 5 (Practical Attacks on CPDLC)**. Together, they demonstrate the vulnerability of PM CPDLC communications to protocol-level manipulations, ranging from DoS to RGS attacks, under realistic experimental conditions.  
