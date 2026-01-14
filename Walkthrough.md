
# Overview

The logs shared document the two classes of attacks described in **Section 5 (Practical Attacks on CPDLC)**.

As previously mentioned, all the logs are first captured using [dumpvdl2](https://github.com/szpajder/dumpvdl2.git) (an open source software), which are then treated to filter out external traffic, ie. we simply remove frames associated with traffic originating from another aircraft or ground station and therefore completely unrelated with our attacks.

We treat below each attack type separately and try to provide more insight on the logs.
# Frame structure

Each frame within the log should look roughly like this:

[Time stamp && metadata (frequency, gain,....)]  
**Sender address** (Type, Location) -> **Receiver address** (Type): Command/Response flag  
      &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; *Body of the log frame*


The Type is either Ground station or Aircraft and Location is either Airborne or On Ground. Of course, all of this is encoded as flags and fields within the frame header.  
Frames and packets within the exchange will be (in majority)  nested, as we can see in ***Table 1: PM-CPLDC ATN-B1 protocol stack***; dumpvdl2 will therefore display them this way starting from the Layer 2 (Datalink), ie. AVLC layer.  

We can see some nesting already in this example (from Legitimate_Connection_anonymized.log):

[2025-08-25 17:37:42.467 CEST] [136.975] [-28.4/-45.8 dBFS] [17.5 dB] [1.0 ppm] **<-- Metadata**  
GROUND_Y (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  **<-- Participants**  
AVLC type: I sseq: 3 rseq: 3 poll: 0 **<-- This is the outer frame AVLC encapsulating all**  
 X.25 Data: grp: 11 chan: 255 sseq: 1 rseq: 1 more: 0 **<--First nested frame X.25**  
 &nbsp; X.25 reasm status: skipped  
  &ensp; X.233 CLNP Data (compressed header): **<--Second nested frame X.233**  
  &emsp; LRef: 0x40 Prio: 14 Flags: 0xf6  
  &emsp;Lifetime: 20.0 sec  
  &emsp;PDU Id: 0xd6c5  
  &emsp;IDRP Keepalive: seq: 3 ack: 1 credit_offered: 4 credit_avail: 3  

## Side note 

***Section 2.3 CPDLC Protocol Stack over ATN B1***  describes all the layer with more details; there is a painfully high number of parameters and flags within the stack, and quite a lot of them are displayed by dumpvdl2. However, we believe that comprehending this artifact would NOT require you to understand/track them (except maybe sequence numbers) .


# Rogue Ground Station / Evil Twin
As described in **Section5.2 (Rogue Ground Station / Evil Twin)**, we implement a Rogue Ground Station
(RGS) capable of issuing valid CPDLC messages to certified avionics hardware. 
The attack can be found in RGS_anonymized.log and the exchange mostly follows the flow of **Figure 2**.  
The most important part would be:  

[2025-08-25 19:43:47.969 CEST] [136.975] [1.8/-49.0 dBFS] [50.8 dB] [2.4 ppm]  
RGS (Ground station, On ground) -> AIRCRAFT (Aircraft): Command **<-- From the RGS to the Aircraft**  
AVLC type: I sseq: 2 rseq: 2 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 6 rseq: 5 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 8 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x9f28  
   X.224 COTP Data:  
    dst_ref: 0x0033  
    sseq: 2 req_of_ack: 0 EoT: 1  
    COTP reasm status: skipped  
    ATN checksum: 42 f5 1c 91  
    CPDLC Uplink Message:  
     Header:  
      Msg ID: 2  
      Timestamp: 2025-08-25 17:43:16  
      Logical ACK: required  
     Message data:  
      FREE TEXT    **<-- Adversarial transmission**  
       ATTENTION ALL AIRCRAFT, CPDLC NO LONGER IN USE. DO NOT DOWNLINK  ANY MESSAGES UNTIL FURTHER ADVISED    **<-- Rogue valid CPDLC message, you can find the proof of this in Figure 9**  
       
[2025-08-25 19:43:48.579 CEST] [136.975] [1.7/-49.0 dBFS] [50.8 dB] [2.6 ppm]
RGS (Ground station, On ground) -> AIRCRAFT (Aircraft): Response
AVLC type: S (Receive Ready) P/F: 0 rseq: 2   **<-- Some link ADM management (not relevant)**

[2025-08-25 19:43:48.883 CEST] [136.975] [1.1/-48.9 dBFS] [50.1 dB] [0.4 ppm]
AIRCRAFT (Aircraft, On ground) -> RGS (Ground station): Response
AVLC type: S (Receive Ready) P/F: 0 rseq: 3  **<-- Some link ADM management (not relevant)**

[2025-08-25 19:43:49.188 CEST] [136.975] [1.8/-49.0 dBFS] [50.8 dB] [2.3 ppm]
RGS (Ground station, On ground) -> AIRCRAFT (Aircraft): Response
AVLC type: S (Receive Ready) P/F: 0 rseq: 2  **<-- Some link ADM management (not relevant)**

[2025-08-25 19:43:49.797 CEST] [136.975] [1.8/-49.1 dBFS] [50.8 dB] [2.5 ppm]
RGS (Ground station, On ground) -> AIRCRAFT (Aircraft): Response
AVLC type: S (Receive Ready) P/F: 0 rseq: 3 **<-- Some link ADM management (not relevant)**

[2025-08-25 19:43:49.951 CEST] [136.975] [1.2/-49.1 dBFS] [50.2 dB] [0.3 ppm]  
AIRCRAFT (Aircraft, On ground) -> RGS (Ground station): Command  
AVLC type: I sseq: 3 rseq: 3 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 5 rseq: 7 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 11 Flags: 0xf6  
   Lifetime: 32.0 sec  
   PDU Id: 0xd  
   X.224 COTP Data Ack:  **<-- Aircraft acknowledges the reception of the data frame**  
    dst_ref: 0x0087  
    rseq: 3 credit: 2  
    ATN checksum: a9 41 e9 80  

[2025-08-25 19:43:50.407 CEST] [136.975] [1.8/-49.0 dBFS] [50.9 dB] [2.8 ppm]  
RGS (Ground station, On ground) -> AIRCRAFT (Aircraft): Response  
AVLC type: S (Receive Ready) P/F: 0 rseq: 3 **<-- Some link ADM management (not relevant)**

[2025-08-25 19:43:51.015 CEST] [136.975] [1.8/-49.1 dBFS] [50.9 dB] [2.8 ppm]  
RGS (Ground station, On ground) -> AIRCRAFT (Aircraft): Response  
AVLC type: S (Receive Ready) P/F: 0 rseq: 4 **<-- Some link ADM management (not relevant)**

[2025-08-25 19:43:51.626 CEST] [136.975] [1.1/-49.3 dBFS] [50.5 dB] [0.2 ppm]  
AIRCRAFT (Aircraft, On ground) -> RGS (Ground station): Command  
AVLC type: I sseq: 4 rseq: 3 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 6 rseq: 7 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 11 Flags: 0xf6  
   Lifetime: 32.0 sec  
   PDU Id: 0xe  
   X.224 COTP Data:  
    dst_ref: 0x0087  
    sseq: 2 req_of_ack: 0 EoT: 1  
    COTP reasm status: skipped  
    ATN checksum: 19 8e d5 24  
    CPDLC Downlink Message:  
     Header:  
      Msg ID: 1  
      Msg Ref: 2  
      Timestamp: 2025-08-25 17:43:39  
      Logical ACK: notRequired  
     Message data:  
      LOGICAL ACKNOWLEDGMENT **<-- Aircraft validates the CPDLC message and displays it!**  

As stated in the README.md, Legitimate_Connection_anonymized.log provides a baseline for a real, legitimate CPDLC session between the Aircraft, a ground station and  EUROCONTROL’s automated ATC.  
RGS_anonymized.log is the rogue RGS-Aircraft CPDLC session and one can see that the two logs both roughly follow the flow described in **Figure 2**.

# Denial of Service

The remaining logs would be demonstrations of **Section 5.1 (Denial of Service)**.  As Reviewer A  correctly noted, they  indeed validate ***Table 3***. As stated in the README.md, each log maps directly to an attack in  **Section 5.1 (Denial of Service)**.  
For all denial-of-service (DoS) attacks (except the _Malformed Frame Injection_, which is included only for illustrative purposes), the aircraft hardware was prompted to establish a CPDLC session with the legitimate test ATC. Upon timeout or disconnection, a new session was requested to maximize resilience attempts against our attacks, simulate a more realistic scenario, and capture as many teardown events as possible within the experiment duration **(around 30 mins EACH!)**. The aircraft hardware still failing to establish a CPDLC session demonstrated the potential severity of such attacks.  

### Important note !

Before diving in, we clarify that two legitimate ground stations are present in our experiments (due to the ground network topology), we refer to them as GROUND_X and GROUND_Y.  We keep the same EUROCONTROL’s automated ATC. The adversary spoofs the identity of a legitimate ground station.
A common (symmetrical) pattern that happens is:  
There is a GROUND_X-Aircraft link. Adversary spoofs GROUND_X and tears down the link. Recovery fails, upon retry, Aircraft switches to GROUND_Y-Aircraft link. This is fine as the attack "*follows*" this switch, the attacker spoofs GROUND_Y and tears down the link.  
There is a lot switching back and forth but the attack just dynamically spoofs the right identity for DoS.  

## Broadcast DoS

As described in **Section 5.1.2 Broadcast DoS** and shown in ***Table 3***, Broadcast DoS is a low-complexity high-impact frame. The attack is available in Broadcast_DOS_anonymized.log and can be easily recognized as it has the following pattern (It can also be found in Figure 8).  

[2025-08-25 20:28:22.381 CEST] [136.975] [-9.9/-49.4 dBFS] [39.5 dB] [2.8 ppm]  
GROUND_X (Ground station, On ground) -> FFFFFF (Aircraft): Command **<-- Spoofed ground station**  
AVLC type: U (DISC) P/F: 0 **<-- Disconnect frame with FFFFFF receiver address**  

As one can see in the logs, after processing  this frame the link from the Aircraft side is immediately torn-down! The ground station will attempts to resume the connection but the Aircraft will not respond because it does not even process messages from that session (no data-link entities exist any more, requires full, fresh new link establishment).  
Let's have a look at a full fledged example:  

[2025-08-25 20:28:21.011 CEST] [136.975] [1.8/-49.0 dBFS] [50.8 dB] [0.2 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_X (Ground station): Command  
AVLC type: I sseq: 7 rseq: 0 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 5 rseq: 5 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 8 Flags: 0xf6  
   Lifetime: 32.0 sec  
   PDU Id: 0x6  
   X.224 COTP Data:  
    dst_ref: 0x01ef  
    sseq: 0 req_of_ack: 0 EoT: 1  
    COTP reasm status: skipped  
    ATN checksum: 95 c4 57 97  
    X.225 Session SPDU: Short Connect  
     X.227 ACSE Associate Request:  
      Application context name: { 1.3.27.3.1 }  
      AP title: { 1.3.27.1.4947967.0 }  
      AE qualifier: 1  
      ATN Context Management - Logon Request: **<-- Normal Logon, ALL GOOD**  
       Flight ID: xxxxxxx  
       Long TSAP: c1 53 55 49 00 xx xx xx 01 01 55 4c 38 30 58 31 01 63 6d	".SUI.K....UL80X1.cm"  
       Ground-initiated applications:  
        Application Entity Qualifier: 22  
        Version number: 1  
        AP Address:  
         Long TSAP: c1 53 55 49 00 xx xx xx 01 01 55 4c 38 30 58 31 01 63 70	".SUI.K....UL80X1.cp"  
       Departure airport: xxxx  
       Destination airport: xxxx  

[2025-08-25 20:28:21.163 CEST] [136.975] [-9.2/-49.0 dBFS] [39.8 dB] [1.0 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 0 rseq: 0 poll: 0 **<-- ALL GOOD, Ground station ready for next**  
 X.25 Receive Ready: grp: 11 chan: 255 rseq: 6  

[2025-08-25 20:28:21.317 CEST] [136.975] [-12.8/-48.9 dBFS] [36.1 dB] [1.3 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 1 rseq: 0 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 5 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 8 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x114d  
   X.224 COTP Data Ack: **<-- Request received, ALL GOOD**  
    dst_ref: 0x0032  
    rseq: 1 credit: 2  
    ATN checksum: fa de af 23  

[2025-08-25 20:28:21.468 CEST] [136.975] [-12.8/-49.1 dBFS] [36.3 dB] [1.0 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 2 rseq: 0 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 6 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 8 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x114e  
   X.224 COTP Data:  
    dst_ref: 0x0032  
    sseq: 0 req_of_ack: 0 EoT: 1  
    COTP reasm status: skipped  
    ATN checksum: 9e eb f7 be  
    X.225 Session SPDU: Short Refuse  
     Refusal: persistent  
     Transport connection: release  
     X.227 ACSE Associate Response:  
      Application context name: { 1.3.27.3.0 }  
      Associate result: reject (permanent)  
      ATN Context Management - Logon Response:   
       Ground-only-initiated applications: **<-- Normal flow, ALL GOOD**  
        Application Entity Qualifier: 22  
        Version number: 1  

[2025-08-25 20:28:21.771 CEST] [136.975] [1.7/-49.5 dBFS] [51.1 dB] [0.3 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_X (Ground station): Response  
AVLC type: S (Receive Ready) P/F: 0 rseq: 3  **<-- Normal flow, ALL GOOD**  

[2025-08-25 20:28:22.381 CEST] [136.975] [-9.9/-49.4 dBFS] [39.5 dB] [2.8 ppm]  
GROUND_X (Ground station, On ground) -> FFFFFF (Aircraft): Command  
AVLC type: U (DISC) P/F: 0  **<--Broadcast DoS Attack!  Aircraft session doesn't exist anymore!**  

[2025-08-25 20:28:22.534 CEST] [136.975] [-9.2/-49.3 dBFS] [40.1 dB] [1.2 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Response  
AVLC type: S (Receive Ready) P/F: 0 rseq: 1 **<-- Normal flow**  

[2025-08-25 20:28:26.344 CEST] [136.975] [-9.3/-49.1 dBFS] [39.8 dB] [1.1 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 3 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 7 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 11 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x114f  
   X.224 COTP Connect Request: **<--Ground station ready to connect, needs the Aircraft to connect**  
    src_ref: 0x01f0 dst_ref: 0x0000  
    Initial Credit: 1  
    Protocol class: 4  
    Options: 00 (use normal PDU formats)  
    TPDU size (bytes): 1024  
    Called/responding transport selector: 63 70  
    Calling transport selector: 04 01  
    Priority: 3  
    Additional options: 0x00  
    Ack time (ms): 1000  
    Inactivity timer (ms): 360000  
    Checksum: 34 73  
    ATN checksum: 1d 9e 55 85  

[2025-08-25 20:28:30.307 CEST] [136.975] [-9.3/-48.8 dBFS] [39.5 dB] [1.0 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 3 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 7 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 11 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x114f  
   X.224 COTP Connect Request: **<--But it never happens..., so it tries again**  
    src_ref: 0x01f0 dst_ref: 0x0000  
    Initial Credit: 1  
    Protocol class: 4  
    Options: 00 (use normal PDU formats)  
    TPDU size (bytes): 1024  
    Called/responding transport selector: 63 70  
    Calling transport selector: 04 01  
    Priority: 3  
    Additional options: 0x00  
    Ack time (ms): 1000  
    Inactivity timer (ms): 360000  
    Checksum: 34 73  
    ATN checksum: 1d 9e 55 85  

[2025-08-25 20:28:33.353 CEST] [136.975] [-9.4/-48.9 dBFS] [39.5 dB] [1.0 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 4 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 0 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 8 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x1150  
   X.224 COTP Data:  
    dst_ref: 0x0032  
    sseq: 0 req_of_ack: 0 EoT: 1  
    COTP reasm status: skipped  
    ATN checksum: 9e eb f7 be  
    X.225 Session SPDU: Short Refuse   
     Refusal: persistent  
     Transport connection: release  
     X.227 ACSE Associate Response:  
      Application context name: { 1.3.27.3.0 }  
      Associate result: reject (permanent)  
      ATN Context Management - Logon Response:  
       Ground-only-initiated applications: **<-Tries retransmitting the previous message, maybe it was lost..**  
        Application Entity Qualifier: 22  
        Version number: 1  

[2025-08-25 20:28:34.573 CEST] [136.975] [-9.4/-49.1 dBFS] [39.7 dB] [1.1 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 3 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 7 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 11 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x114f  
   X.224 COTP Connect Request: **<-- No answer, tries to connect again..**  
    src_ref: 0x01f0 dst_ref: 0x0000  
    Initial Credit: 1  
    Protocol class: 4  
    Options: 00 (use normal PDU formats)  
    TPDU size (bytes): 1024  
    Called/responding transport selector: 63 70  
    Calling transport selector: 04 01  
    Priority: 3  
    Additional options: 0x00  
    Ack time (ms): 1000  
    Inactivity timer (ms): 360000  
    Checksum: 34 73  
    ATN checksum: 1d 9e 55 85  

[2025-08-25 20:28:34.573 CEST] [136.975] [-9.4/-49.1 dBFS] [39.7 dB] [1.1 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 4 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 0 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 8 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x1150  
   X.224 COTP Data:  
    dst_ref: 0x0032  
    sseq: 0 req_of_ack: 0 EoT: 1  
    COTP reasm status: skipped  
    ATN checksum: 9e eb f7 be  
    X.225 Session SPDU: Short Refuse  
     Refusal: persistent  
     Transport connection: release  
     X.227 ACSE Associate Response:  
      Application context name: { 1.3.27.3.0 }  
      Associate result: reject (permanent)  
      ATN Context Management - Logon Response:  
       Ground-only-initiated applications: **<-Tries retransmitting the previous message again, maybe everything was lost..**  
        Application Entity Qualifier: 22  
        Version number: 1  

[2025-08-25 20:28:38.380 CEST] [136.975] [-9.3/-49.4 dBFS] [40.1 dB] [1.3 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 5 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 1 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 11 Flags: 0xd6  
   Lifetime: 18.5 sec    
   PDU Id: 0x1152  
   X.224 COTP Connect Request:**<-Tries again...**  
    src_ref: 0x01f0 dst_ref: 0x0000  
    Initial Credit: 1  
    Protocol class: 4  
    Options: 00 (use normal PDU formats)  
    TPDU size (bytes): 1024  
    Called/responding transport selector: 63 70  
    Calling transport selector: 04 01  
    Priority: 3  
    Additional options: 0x00  
    Ack time (ms): 1000  
    Inactivity timer (ms): 360000  
    Checksum: 34 73  
    ATN checksum: 1d 9e 55 85  

[2025-08-25 20:28:39.449 CEST] [136.975] [-9.3/-49.5 dBFS] [40.2 dB] [1.1 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 3 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 7 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 11 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x114f  
   X.224 COTP Connect Request: **<-And again...**  
    src_ref: 0x01f0 dst_ref: 0x0000  
    Initial Credit: 1  
    Protocol class: 4  
    Options: 00 (use normal PDU formats)  
    TPDU size (bytes): 1024  
    Called/responding transport selector: 63 70  
    Calling transport selector: 04 01  
    Priority: 3  
    Additional options: 0x00  
    Ack time (ms): 1000  
    Inactivity timer (ms): 360000  
    Checksum: 34 73  
    ATN checksum: 1d 9e 55 85  

[2025-08-25 20:28:39.449 CEST] [136.975] [-9.3/-49.5 dBFS] [40.2 dB] [1.1 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 4 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 0 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 8 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x1150  
   X.224 COTP Data:  
    dst_ref: 0x0032  
    sseq: 0 req_of_ack: 0 EoT: 1  
    COTP reasm status: skipped  
    ATN checksum: 9e eb f7 be  
    X.225 Session SPDU: Short Refuse  
     Refusal: persistent  
     Transport connection: release  
     X.227 ACSE Associate Response:  
      Application context name: { 1.3.27.3.0 }  
      Associate result: reject (permanent)  
      ATN Context Management - Logon Response:  
       Ground-only-initiated applications: **<-And again...**  
        Application Entity Qualifier: 22  
        Version number: 1  

[2025-08-25 20:28:39.449 CEST] [136.975] [-9.3/-49.5 dBFS] [40.2 dB] [1.1 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 5 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 1 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 11 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x1152  
   X.224 COTP Connect Request: **<-And again...**  
    src_ref: 0x01f0 dst_ref: 0x0000  
    Initial Credit: 1  
    Protocol class: 4  
    Options: 00 (use normal PDU formats)  
    TPDU size (bytes): 1024  
    Called/responding transport selector: 63 70  
    Calling transport selector: 04 01  
    Priority: 3  
    Additional options: 0x00  
    Ack time (ms): 1000  
    Inactivity timer (ms): 360000  
    Checksum: 34 73  
    ATN checksum: 1d 9e 55 85  

[2025-08-25 20:28:47.371 CEST] [136.975] [-9.4/-49.0 dBFS] [39.6 dB] [1.1 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 3 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 7 rseq: 6 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 11 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x114f  
   X.224 COTP Connect Request: **<-And again...**  
    src_ref: 0x01f0 dst_ref: 0x0000  
    Initial Credit: 1  
    Protocol class: 4  
    Options: 00 (use normal PDU formats)  
    TPDU size (bytes): 1024  
    Called/responding transport selector: 63 70  
    Calling transport selector: 04 01  
    Priority: 3  
    Additional options: 0x00  
    Ack time (ms): 1000  
    Inactivity timer (ms): 360000  
    Checksum: 34 73  
    ATN checksum: 1d 9e 55 85  


**And it keeps going!**  
At this point, the Aircraft will never respond because no link entities exist, it will probably initiate a session with GROUND_Y, which will be torn-down ... At some point, **errors  cascade** because of link state mismatch between the Aircraft and Ground Stations. This is normal and is one reason that makes Broadcast DoS rank the best in **Table 3**.  



## Other DoS logs

The other logs map to the remaining techniques presented **Section 5.1 (Denial of Service)** and evaluated in ***Table 3***. These are more advanced due to state dependency.   

All of the other attacks follow the scheme in **Figure 3**:  
A Race Condition essentially; we must synchronise with the session and predict the parameters before the state update.  
MalformedFrame_anonymized.log documents the attack in **Section 5.1.4 (Malformed Frame Injection)**  and should be pretty straightforward to check as we hint at it with the following text *"THIS MEESAGE IS MALFORMED"*  

FRMR_DOS_anonymized.log documents the attack in **Section 5.1.1 Malicious AVLC Frame Reject (FRMR) Injection**.  

CFI_IDRP_anonymized.log is associated with **Section 5.1.3 (Control Flow Injection on IDRP)**.  

Since both have a similar pattern but the CFI_IDRP_anonymized.log we provide operates on IDRP layer (making it harder to understand), we will have a look at the harder CFI_IDRP_anonymized.log first.  
Essentially, the IDRP engine is specified as a finite-state machine (FSM); we demonstrates targeted disruption of the IDRP FSM by injecting an IDRP CEASE PDU, forcing the aircraft into the CLOSED state and creating a state mismatch.  

Let's have a look at an example:  

[2025-08-25 14:22:13.762 CEST] [136.975] [-10.0/-49.5 dBFS] [39.6 dB] [0.9 ppm]  
GROUND_Y (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 4 rseq: 3 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 2 rseq: 1 more: 0    
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x40 Prio: 14 Flags: 0xf6  
   Lifetime: 20.0 sec  
   PDU Id: 0x2b26  
   IDRP Update: seq: 3 ack: 1 credit_offered: 4 credit_avail: 2  **<-ALL GOOD, normal flow**  
    Route:  
     ID: 3769  
     Local preference: 0  
    Security:  
     Reg ID: 06 04 2b 1b 00 00  
     Info:  
      Supported ATSC classes: A  
      Subnetwork type:  
       Subnet: VDL  
       Permitted traffic: all  
    RD path:  
     RD_SEQ:  
      47 00 27 01 53 49 54 00 00 00 01 0b 47 00 27 01 53 49 54 00 59 55 4c	"G.'.SIT.....G.'.SIT.YUL"  
      47 00 27 01 53 49 54 00 59 55 4c	"G.'.SIT.YUL"  
    RD hop count: 2  
    Capacity: 12  
    Reachability info:  
     Protocol: CLNP  
     Prefix length: 32  
     Dest. address prefix: 47 00 27 01 20 47 00 27 81	"G.'. G.'."  

[2025-08-25 14:22:14.221 CEST] [136.975] [-12.9/-49.5 dBFS] [36.5 dB] [2.6 ppm]  
GROUND_Y (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 5 rseq: 3 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 3 rseq: 1 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x40 Prio: 11 Flags: 0xd6  
   Lifetime: 18.5 sec  
   PDU Id: 0x2b36  
   IDRP Cease: seq: 3 ack: 1 credit_offered: 0 credit_avail: 0  **<-CFI DoS Attack!!!**  

[2025-08-25 14:22:14.677 CEST] [136.975] [-10.2/-49.5 dBFS] [39.3 dB] [1.0 ppm]  
GROUND_Y (Ground station, On ground) -> AIRCRAFT (Aircraft): Response  
AVLC type: S (Receive Ready) P/F: 0 rseq: 4  

[2025-08-25 14:22:14.830 CEST] [136.975] [1.9/-49.4 dBFS] [51.3 dB] [0.0 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_Y (Ground station): Response  
AVLC type: S (Receive Ready) P/F: 0 rseq: 6  

[2025-08-25 14:22:14.831 CEST] [136.975] [-10.3/-49.3 dBFS] [39.1 dB] [1.0 ppm]  
GROUND_Y (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: U (FRMR) P/F: 1  
 Data (3 bytes):  
  c1 9a 08  **<-First state mismatch: lower layers**  

[2025-08-25 14:22:14.831 CEST] [136.975] [2.0/-49.3 dBFS] [51.3 dB] [0.1 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_Y (Ground station): Response  
AVLC type: U (UA) P/F: 1 **<-First state mismatch: lower layers**  

[2025-08-25 14:22:14.982 CEST] [136.975] [1.7/-49.0 dBFS] [50.7 dB] [0.3 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_Y (Ground station): Command  
AVLC type: I sseq: 0 rseq: 0 poll: 0  
 X.25 Receive Ready: grp: 11 chan: 255 rseq: 4  

[2025-08-25 14:22:15.134 CEST] [136.975] [-10.4/-49.1 dBFS] [38.7 dB] [1.1 ppm]  
GROUND_Y (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 0 rseq: 1 poll: 0  
 X.25 Reset Request: grp: 11 chan: 255  
  Cause: 0x05 (Local procedure error)  
  Diagnostic code: 0x02 (Invalid P(R))  **<-First state mismatch: lower layers**  

[2025-08-25 14:22:15.439 CEST] [136.975] [1.9/-49.4 dBFS] [51.3 dB] [0.0 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_Y (Ground station): Command  
AVLC type: I sseq: 1 rseq: 1 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 2 rseq: 4 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x40 Prio: 14 Flags: 0xf6  
   Lifetime: 32.0 sec  
   PDU Id: 0x3  
   IDRP Cease: seq: 3 ack: 3 credit_offered: 0 credit_avail: 3 **<-IDRP state mismatch! Aircraft aborts!!**  


Similar to Broadcast DoS, errors will cascade very badly at some point, please note that the attack was launched only immediately after ground station IDRP updates. Any additional CEASE or ERROR PDUs in the log result from the IDRP state mismatches induced by the attack.  
FRMR_DOS_anonymized.log is similar but much simpler, here is an example:

[2024-08-30 20:35:30 CEST] [136.975] [1.6/-46.3 dBFS] [47.9 dB] [0.8 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_X (Ground station): Command  
AVLC type: I sseq: 0 rseq: 7 poll: 0  
 X.25 Receive Ready: grp: 11 chan: 255 rseq: 3 **<-- Target frame, as in Figure 7**  
 
[2024-08-30 20:35:30 CEST] [136.975] [-21.1/-46.4 dBFS] [25.2 dB] [3.3 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: U (FRMR) P/F: 1 **<--AVLC FRMR injection, as in Figure 7**  
 Data (3 bytes):  
  e0 0c 08  

[2024-08-30 20:35:30 CEST] [136.975] [1.7/-46.7 dBFS] [48.4 dB] [0.6 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_X (Ground station): Response  
AVLC type: U (UA) P/F: 1 **<--Aircraft responds with an AVLC UA , as in Figure 7**  

[2024-08-30 20:35:30 CEST] [136.975] [1.2/-46.4 dBFS] [47.6 dB] [1.7 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: U (FRMR) P/F: 1  
 Data (3 bytes):  
  73 3e 00 **<--This triggers an AVLC FRMR from the ATC system , as in Figure 7**  

[2024-08-30 20:35:30 CEST] [136.975] [1.8/-46.4 dBFS] [48.2 dB] [0.8 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_X (Ground station): Command  
AVLC type: I sseq: 0 rseq: 0 poll: 0  
 X.25 Data: grp: 11 chan: 255 sseq: 3 rseq: 3 more: 0  
  X.25 reasm status: skipped  
  X.233 CLNP Data (compressed header):  
   LRef: 0x0 Prio: 14 Flags: 0xf6  
   Lifetime: 32.0 sec  
   PDU Id: 0x30    
   IDRP Keepalive: seq: 3 ack: 2 credit_offered: 3 credit_avail: 3  **<--Legitimate frame, this is due to the asynchronous nature of the DLE in ADM state**  

[2024-08-30 20:35:30 CEST] [136.975] [1.8/-46.4 dBFS] [48.2 dB] [0.9 ppm]  
AIRCRAFT (Aircraft, On ground) -> GROUND_X (Ground station): Response  
AVLC type: U (UA) P/F: 1 **<--Aircraft responds with an AVLC UA to the legitimate AVLC FRMR**   

[2024-08-30 20:35:31 CEST] [136.975] [1.2/-46.5 dBFS] [47.7 dB] [1.6 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 0 rseq: 1 poll: 0  
 X.25 Reset Request: grp: 11 chan: 255 **<--Error triggered in the link layer**   
  Cause: 0x05 (Local procedure error)  
  Diagnostic code: 0x01 (Invalid P(S))  

[2024-08-30 20:35:31 CEST] [136.975] [1.2/-46.5 dBFS] [47.6 dB] [1.9 ppm]  
GROUND_X (Ground station, On ground) -> AIRCRAFT (Aircraft): Command  
AVLC type: I sseq: 1 rseq: 1 poll: 0  
 X.25 Clear Request: grp: 11 chan: 255 **<--Link reset needed**  
  Cause: 0x00 (DTE originated)  
  Diagnostic code: 0x90 (Idle timer expired)  

And that’s a wrap for this quick walkthrough!  
The README.md has some more information on the logs and setup, and the paper goes way more in depth if you want additional details.
