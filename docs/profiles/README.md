
# Huawei OLT GPON Service Configuration Guide

This document provides a theoretical explanation and practical steps for configuring GPON services on a Huawei OLT, focusing on plan creation, T-CONTs, GEM Ports, and service provisioning.

## References

  * **Huawei Forum - Theoretical Explanation:** forum.huawei.com (Provides a theoretical explanation of what needs to be done.)
  * **Huawei Forum - General Commands:** forum.huawei.com (Contains general information about the commands to be used.)
  * **Huawei Support - Similar Commands & FreeRADIUS Scheme:** support.huawei.com (Information on a switch with similar commands and a Huawei-specific FreeRADIUS configuration scheme.)

-----

## GPON Structure Overview

### T-CONT (Transmission Container)

A T-CONT is a type of buffer that carries services, primarily used to transmit upstream data units. T-CONTs are introduced to enable dynamic bandwidth assignment (DBA) for upstream bandwidth, enhancing line utilization.

**Purpose:** Used for dynamic bandwidth assignment in the upstream direction.

#### T-CONT Types

| Type | Name      | Guarantee                                                                                             | Typical Use Cases                                                                   |
| :--- | :-------- | :---------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------- |
| 1    | **Fixed** | Fixed and fully reserved bandwidth. Not a single bit of this bandwidth is shared with anyone else.    | Critical services highly sensitive to latency (telephony, live video).              |
| 2    | **Assured** | Minimum guaranteed bandwidth. If not fully utilized, the remaining capacity becomes available for others. | Applications with strict SLAs that can still benefit from "leftovers" (e.g., corporate VPN). |
| 3    | **Non-assured** | No fixed guarantee, but can use any remaining capacity up to a predefined maximum.                      | "Important" data not requiring reservation but shouldn't exceed a certain cap (e.g., deferred backups). |
| 4    | **Best-effort** | No guarantee or limit: uses only what's currently available. If the network is busy, it might not transmit. | Lower-priority traffic (e.g., background updates, non-critical downloads).          |
| 5    | **Mixed** | Combines several of the above (Fixed/Assured/Non-assured/Best-effort) in a single queue.              | ONUs with diverse services (voice, data, IPTV) where you want to simplify profiles and group quotas. |

### GEM Port (GPON Encapsulation Method Port)

The minimum unit for service transport. The relationship with T-CONTs is one (T-CONT) to many (GEM Ports).

#### GEM Port Configuration Modifiers

| Modifier              | Description                                                                                                                                              |
| :-------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cascade`             | Allows this GEM Port to forward traffic to another OLT in cascaded architectures, instead of processing it locally.                                    |
| `downstream-priority-queue` | Assigns this GEM Port's downstream traffic to a specific priority queue on the OLT (e.g., voice, video, data).                                          |
| `encrypt`             | Activates OMCI encryption on this GEM Port to protect the management signal between the OLT and the ONU.                                                  |
| `gem-car`             | Enables Committed Access Rate (CAR) control on the GEM Port, defining Committed Information Rate (CIR) and Peak Information Rate (PIR) to limit upstream traffic. |
| `priority-queue`      | Selects the upstream priority queue on the OLT for traffic sent by this GEM Port, ensuring service order according to configured priority.                |

### Dynamic Bandwidth Assignment (DBA)

DBA is based on bandwidth allocation in microsecond intervals. Here are the general steps for the assignment procedure:

  * **Service Priority:** "In general, voice service enjoys the highest, then video service and data service the lowest in terms of service priority."
  * The OLT includes the bandwidth assignment map (BW Map) in the header of downstream frames, based on prior calculations.
  * The ONU, following the information in that BW Map, sends a status report on how much data it has pending in its T-CONT during the indicated time slot.
  * When the OLT receives that status report, it executes the DBA algorithm, updates the BW Map, and inserts it into the next downstream frame.
  * Upon receiving the new BW Map from the OLT, the ONU transmits its data in the assigned time slots.

-----

## OLT Configuration Structure

Huawei OLTs use profiles to define how services are delivered to ONUs.

### DBA Profile

  * **What it is:** A set of rules that defines how the OLT dynamically allocates upstream bandwidth among ONUs.
  * **Purpose:** Establishes guaranteed quotas (fixed/assured) and "overflow usage" rules (non-assured/best-effort) for each T-CONT.

### Line Profile

  * **What it is:** A Layer 2 template that groups the configuration of T-CONTs and GEM Ports (logical queues and GEM ports) for an ONU.
  * **Purpose:** Defines how many logical queues (T-CONTs) the ONU will have, which DBA Profile each uses, and how traffic will be tagged (GEM Port → VLAN).

### Service Profile

  * **What it is:** A Layer 3/service template that configures the ONU's user ports (Ethernet, POTS, IGMP, TR-069, VLANs, QoS, NAT, etc.).
  * **Purpose:** Determines which interfaces the ONU exposes and with what service parameters (VLAN maps, QoS profiles, management, telephony, etc.).

### Service Port

  * **Purpose:** Used to establish 802.1Q tagging rules and define allowed protocols for transport.

-----

## Base Profile Creation Commands (Huawei Examples)

### DBA Profile Creation

```
EA5800-X2(config)#dba-profile add profile-id 10 profile-name "dba-profile_10" type1 fix 512000
EA5800-X2(config)#display dba-profile all
```

### Line Profile Creation

```
EA5800-X2(config)#ont-lineprofile gpon profile-id 10 profile-name "line-profile_10"
EA5800-X2(ont-lineprofile-10)#tcont 4 dba-profile-id 10
EA5800-X2(ont-lineprofile-10)#gem add 11 eth tcont 4 encrypt on
EA5800-X2(ont-lineprofile-10)#gem mapping 11 0 vlan 13
EA5800-X2(ont-lineprofile-10)#commit
EA5800-X2(ont-lineprofile-10)#quit
```

#### Explanation of Line Profile Commands:

This block of commands creates and configures a Line Profile (Layer 2 profile) on your OLT for the GPON ONUs that will use it.

  * **`EA5800-X2(config)# ont-lineprofile gpon profile-id 10 profile-name "line-profile_10"`**
      * Creates (or accesses) Line Profile number 10, named "line-profile\_10".
      * This profile will define the queues and GEM ports that each ONU provisioned with it will use.
  * **`EA5800-X2(ont-lineprofile-10)# tcont 4 dba-profile-id 10`**
      * Adds a T-CONT with ID 4.
      * Assigns DBA Profile 10 to it, which contains your bandwidth allocation rules (fixed/assured/...).
  * **`EA5800-X2(ont-lineprofile-10)# gem add 11 eth tcont 4 encrypt on`**
      * Generates a GEM Port with index 11 of type ETH (Ethernet traffic).
      * Links it to T-CONT 4, which you just created.
      * `encrypt on` activates OMCI encryption for that GEM Port.
  * **`EA5800-X2(ont-lineprofile-10)# gem mapping 11 0 vlan 13`**
      * Tells the system that GEM Port 11 should carry traffic for VLAN 13.
      * `0` is the mapping index (you can define multiple mappings per GEM).
  * **`EA5800-X2(ont-lineprofile-10)# commit`**
      * Applies and saves all the configuration you made within this profile.
  * **`EA5800-X2(ont-lineprofile-10)# quit`**
      * Returns to global configuration mode.

**What you've achieved:**
You have defined how Layer 2 traffic travels for ONUs:

  * A logical queue (T-CONT 4) with DBA rules from profile 10.
  * A GEM port (11) that encapsulates Ethernet traffic.
  * This ONU's ETH traffic will travel on VLAN 13 towards the OLT.

When you later run `ont add ... ont-lineprofile-id 10`, each associated ONU will adopt this queue and encapsulation configuration.

### Service Profile Creation

```
EA5800-X2(config)#ont-srvprofile gpon profile-id 10
EA5800-X2(config-gpon-srvprofile-10)#ont-port eth adaptive pots adaptive
EA5800-X2(config-gpon-srvprofile-10)#port vlan eth 1 13
EA5800-X2(config-gpon-srvprofile-10)#commit
EA5800-X2(config-gpon-srvprofile-10)#quit
```

### ONU Provisioning

```
EA5800-X2(config-if-gpon-0/1)#ont add 0 0 sn-auth "xxxx" omci ont-lineprofile-id 10 ont-srvprofile-id 10
EA5800-X2(config)#vlan 13 smart    ------------------------------------------------------------------------------------->First of all
EA5800-X2(config)#port vlan 13 0/3 0
OLT02(config)#service-port vlan 13 gpon 0/1/0 ont 0 gemport 11 multi-service user-vlan 13 tag-transform translate
```

#### Explanation of ONU Provisioning Commands:

The command `port vlan 13 0/3 0` is used in global configuration mode to assign VLAN 13 as the native (untagged) VLAN to a physical port on the chassis. In this case:

  * **`13`**: Is the ID of the VLAN you just created (`vlan 13 smart`).
  * **`0/3/0`**: Indicates slot 0, sub-slot 3, and port 0 of the chassis (for example, an Ethernet uplink port on the front face of your OLT).

**What it does:**

  * Passes all untagged traffic entering or leaving that port to the VLAN 13 domain internally.
  * Sets the PVID (Port VLAN ID) of that port to 13, so if you receive tagged frames from another VLAN, they will be discarded, but untagged frames will be treated as VLAN 13.

Then, when you create your service-port:
`service-port 5 vlan 13 gpon 0/1/0 ont 0 gemport 11 multi-service user-vlan 13 tag-transform translate`

You tell the OLT: "all VLAN 13 traffic going through GPON port 0/1/0, ONU 0, GEM 11, must be translated (`translate`) between untagged on the uplink and encapsulated in GPON."

In summary, `port vlan 13 0/3 0` prepares your physical port 0/3/0 to communicate natively on VLAN 13, and the `service-port ... tag-transform translate` is responsible for mapping that traffic to the GPON fiber and vice versa.

The `tag-transform translate` in the `service-port` command tells the OLT that:

  * **Uplink direction (OLT → network):** When it receives frames on GPON via GEM Port 11, it unpacks them, re-inserts the 802.1Q VLAN 13 tag, and sends them to the Ethernet port as tagged traffic.
  * **Downstream direction (network → OLT → ONU):** When tagged frames on VLAN 13 arrive via the Ethernet port, the OLT removes that tag (treats them as native untagged) before encapsulating them in GEM Port 11 towards the ONU.

In other words, `translate` performs bidirectional translation between:

  * VLAN tag 13 on the Ethernet side, and
  * GEM Port 11 encapsulation on the GPON side,
    ensuring that traffic maintains the correct VLAN at both ends.

### Uplink Configuration

```
EA5800-X2(config)#port vlan 13 0/3 1
EA5800-X2(config)#interface mpu 0/3
EA5800-X2(interface-mpu-0/3)#native-vlan 1 vlan 13
EA5800-X2(interface-mpu-0/3)#quit
```

#### Explanation of Uplink Configuration:

Both command blocks configure the native VLAN (PVID) for the physical ports of the MPU (Multi-service Processing Unit) 0/3 module of your OLT, but from two different contexts:

**Global mode, with the `port vlan` command:**

  * `EA5800-X2(config)# port vlan 13 0/3 1`
      * Here you tell the system that physical port 1 of chassis slot/line 0/3 should pass all untagged traffic as part of VLAN 13. It's a quick way to assign PVID 13 to port 1.

**`interface mpu` mode, with `native-vlan`:**

  * `EA5800-X2(config)# interface mpu 0/3`
  * `EA5800-X2(interface-mpu-0/3)# native-vlan 1 vlan 13`
  * `EA5800-X2(interface-mpu-0/3)# quit`
      * Enters the context of MPU 0/3 and, for its logical port 1, sets VLAN 13 as native. This is the explicit way to do exactly the same as the previous `port vlan` command, but at the internal service processor level instead of the global command.

**Why two ways?**

  * `port vlan` is a global shortcut that works for all line modules without entering a specific context.
  * `interface mpu` + `native-vlan` gives you more control if you later want to map other parameters (like different PVIDs for Q-in-Q, filters, etc.) on that same MPU.

In both cases, any untagged frame entering or leaving port 1 of module 0/3 will be treated as VLAN 13 on your OLT.

-----

## Service Plan Configuration

First, we identify the following fundamental services that will be mapped to different VLANs to ensure clear service classification across the network:

  * Streaming (Priority 1 & 2)
  * Video
  * Web Data

### Service Classification by VLAN

| VLAN | Service             |
| :--- | :------------------ |
| 100  | Streaming Priority 1 |
| 101  | Streaming Priority 2 |
| 200  | Video               |
| 300  | Web/Data            |

### Service Plans

| Plan    | Line Profile Name/ID | VLAN | Service             | GEM Port | T-CONT | DBA Profile Name/ID |
| :------ | :------------------- | :--- | :------------------ | :------- | :----- | :------------------ |
| **Standard** | `line-std_100` (100)   | 100  | Streaming Priority 1 | 10       | 1      | `dba-std-stream` (100) |
|         |                      | 101  | Streaming Priority 2 | 11       | 1      | `dba-std-stream` (100) |
|         |                      | 200  | Video               | 12       | 2      | `dba-std-video` (101) |
|         |                      | 300  | Web/Data            | 13       | 3      | `dba-std-web` (102)   |
| **Premium** | `line-prem_200` (200)  | 100  | Streaming Priority 1 | 20       | 1      | `dba-prem-stream` (200) |
|         |                      | 101  | Streaming Priority 2 | 21       | 1      | `dba-prem-stream` (200) |
|         |                      | 200  | Video               | 22       | 2      | `dba-prem-video` (201) |
|         |                      | 300  | Web/Data            | 23       | 3      | `dba-prem-web` (202)  |


✅ It's highlighted that for streaming service, two priorities are handled, aiming to create a hierarchy for video calls and meetings (higher priority) and another for live streams like those on platforms such as Twitch and Kick (lower priority). The latter services are very demanding in bandwidth consumption, and a delay or data loss in a stream is less critical than losing data in a meeting or video call.

Here is a graphical representation of how megas was used depending on the service:

![image](https://github.com/user-attachments/assets/31f4fdab-3411-438d-b9f4-fce10d6e31d1)

Each ONT eth connectios has an specific vlan for a specific service, also, each service has a wifi SSID to identify which service is the final user using.

![image](https://github.com/user-attachments/assets/415ae210-3f89-4c91-be18-da25c26e395c)

-----

## Implementation

### 1\. VLAN Creation

```
[Huawei] vlan 100 smart
[Huawei] interface vlan 100
[Huawei] description StreamingVlanForPriority1

[Huawei] vlan 101 smart
[Huawei] interface vlan 101
[Huawei] description StreamingVlanForPriority2

[Huawei] vlan 200 smart
[Huawei] interface vlan 200
[Huawei] description VlanForVideoServices

[Huawei] vlan 300 smart
[Huawei] interface vlan 300
[Huawei] description VlanForWebData
```

✅ Using `smart` allows you to flexibly assign the VLAN to interfaces (trunks, access ports) or GPON service-ports independently.


### 2\. DBA and Line Profiles

#### Explanation of a Sample DBA Command (Type 5 for Streaming)

```
dba-profile add profile-id 100 profile-name "dba-std-stream" type5 fix 20000 assure 10000 max 50000
```

| Field                       | Value     | Description  
**`type5`:** Mixed profile combining fixed, assured, non-assured, and best-effort.

  * **`fix 20000` (kbps):** The first portion of bandwidth, always reserved in each cycle. Up to 20 Mbps guaranteed before any other service.
  * **`assure 10000` (kbps):** A second, additional reserved portion. After covering fixed, up to an extra 10 Mbps are guaranteed.
  * **`max 50000` (kbps):** The combined limit for fixed + assured + non-assured before transitioning to best-effort. In this case, non-assured = Max - (Fix + Assure) = 50,000 - (20,000 + 10,000) = 20,000 kbps.
  * **`additional-bandwidth best-effort`:** After reaching the `max` limit, the remaining upstream bandwidth is served as best-effort.
  * **`best-effort-priority 0`:** Priority queue assigned to best-effort (0 = lowest).
  * **`best-effort-weight 128`:** Relative weight in the best-effort queue if multiple BE queues exist.
  * **`bandwidth compensation No`:** If "Yes," it would allow dynamically adjusting grants to compensate for previous unused frames.
  * **`fix-delay No`:** If "Yes," it would add a fixed delay to the fixed grant to space out transmission slots.

-----

### Implementation Commands

#### DBA Profiles

```
dba-profile add profile-id 100 profile-name "dba-std-stream" type5 fix 20000 assure 10000 max 50000
dba-profile add profile-id 101 profile-name "dba-std-video" type3 assure 15000 max 30000
dba-profile add profile-id 102 profile-name "dba-std-web" type2 assure 10000

dba-profile add profile-id 200 profile-name "dba-prem-stream" type5 fix 40000 assure 20000 max 100000
dba-profile add profile-id 201 profile-name "dba-prem-video" type3 assure 30000 max 60000
dba-profile add profile-id 202 profile-name "dba-prem-web" type2 assure 20000
```

#### Line Profile "Standard" (ID 100)

```
ont-lineprofile gpon profile-id 100 profile-name "line-std_100"

tcont 1 dba-profile-id 100
gem add 10 eth tcont 1 priority-queue 7 encrypt on
gem mapping 10 0 vlan 100
gem add 11 eth tcont 1 priority-queue 6 encrypt on
gem mapping 11 0 vlan 101

tcont 2 dba-profile-id 101
gem add 12 eth tcont 2 encrypt on
gem mapping 12 0 vlan 200

tcont 3 dba-profile-id 102
gem add 13 eth tcont 3 encrypt on
gem mapping 13 0 vlan 300
commit
quit
```

#### Line Profile "Premium" (ID 200)

```
ont-lineprofile gpon profile-id 200 profile-name "line-prem_200"

tcont 1 dba-profile-id 200
gem add 20 eth tcont 1 priority-queue 7 encrypt on
gem mapping 20 0 vlan 100
gem add 21 eth tcont 1 priority-queue 6 encrypt on
gem mapping 21 0 vlan 101

tcont 2 dba-profile-id 201
gem add 22 eth tcont 2 encrypt on
gem mapping 22 0 vlan 200

tcont 3 dba-profile-id 202
gem add 23 eth tcont 3 encrypt on
gem mapping 23 0 vlan 300
commit
quit
```

### 3\. Service Profile Creation

#### Service Profile for Standard Plan (ID 100)

```
ont-srvprofile gpon profile-id 100
ont-port eth adaptive pots adaptive # Active eth and voice ports in automatic
port vlan eth 1 100
port vlan eth 2 101
port vlan eth 3 200
port vlan eth 4 300
commit
quit
```

#### Service Profile for Premium Plan (ID 200)

```
ont-srvprofile gpon profile-id 200
ont-port eth adaptive pots adaptive # Active eth and voice ports in automatic
port vlan eth 1 100
port vlan eth 2 101
port vlan eth 3 200
port vlan eth 4 300
commit
quit
```

### 4\. ONU Provisioning

| ONT Vendor | Serial Number (Actual) | Serial Number (OLT-Interpreted) |
| :--------- | :--------------------- | :------------------------------ |
| Huawei     | `485754436B886FA5`     | `485754436B886FA5`              |
| Fiberhome  | `FHTT983DEFE8`         | `46485454983DEFE8`              |

The `gpon 0/1` interface contains all the ONU connections. Within this interface, there are ports directly connecting to the ONUs. The MPU interface, on the other hand, is the link to the Routerboard. Therefore, a trunk mode is necessary when handling multiple VLANs.

After adding the ONT, we proceed to create service ports for each VLAN so that the OLT knows which protocols to allow and how to handle TAG translations.

\<aside\>
⚠️ **Important:** The translation must be from VLAN 100 to VLAN 13, as VLAN 13 is the native VLAN for the MPU 0/3/1 uplink.
\</aside\>

#### Provisioning ONUs (Example for existing ONUs)

```
#(config-if-gpon-0/1)
ont add 0 0 sn-auth "485754436B886FA5" omci ont-lineprofile-id 100 ont-srvprofile-id 100 desc "HUAWEI ONT"
ont add 1 1 sn-auth "485754438CC094A2" omci ont-lineprofile-id 200 ont-srvprofile-id 200 desc "HUAWEI-PREMIUM"
```

#### Service Ports for Standard Plan (ONT 0, Port 0)

```
service-port 101 vlan 13 gpon 0/1/0 ont 0 gemport 10 multi-service user-vlan 100 tag-transform translate
service-port 102 vlan 13 gpon 0/1/0 ont 0 gemport 11 multi-service user-vlan 101 tag-transform translate
service-port 103 vlan 13 gpon 0/1/0 ont 0 gemport 12 multi-service user-vlan 200 tag-transform translate
service-port 104 vlan 13 gpon 0/1/0 ont 0 gemport 13 multi-service user-vlan 300 tag-transform translate
```

#### Service Ports for Premium Plan (ONT 1, Port 0)

```
service-port 201 vlan 13 gpon 0/1/1 ont 0 gemport 20 multi-service user-vlan 100 tag-transform translate
service-port 202 vlan 13 gpon 0/1/1 ont 0 gemport 21 multi-service user-vlan 101 tag-transform translate
service-port 203 vlan 13 gpon 0/1/1 ont 0 gemport 22 multi-service user-vlan 200 tag-transform translate
service-port 204 vlan 13 gpon 0/1/1 ont 0 gemport 23 multi-service user-vlan 300 tag-transform translate
```

On a Huawei OLT (like your EA5800-X2), MPU stands for Multi-service Processing Unit. It's the physical module within the chassis that aggregates traffic. The command `port vlan 100,101,200,300 0/3 0` might be used for trunking multiple VLANs on a physical port.

### Key Commands for ONT Management

  * `display ont info summary 0/1`: View the status of connected ONTs.
  * `interface gpon 0/1`: Enter the GPON interface.
  * `display ont info by-sn FHTT983DEFE8`: Display ONT information by serial number.

#### Detailed `ont add` Command Breakdown:

```
ont add 0 0 sn-auth "FHTT983DEFE8" omci ont-lineprofile-id 10 ont-srvprofile-id 10
```

Each part of this command does the following:

  * **`ont add 0 0`**: Adds a new ONU on the active PON port, assigning it `ONT-ID=0` and `sub-ID=0`.
  * **`sn-auth "FHTT983DEFE8"`**: Selects the SN-auth authentication method (by serial number) using the string `FHTT983DEFE8`.
  * **`omci`**: Indicates that subsequent provisioning (profiles) will be performed via OMCI (ONU management channel).
  * **`ont-lineprofile-id 10`**: Links the ONU to Line Profile 10, which defines its logical queues (T-CONTs/GEM Ports) and Layer 2 VLAN/QoS mappings.
  * **`ont-srvprofile-id 10`**: Applies Service Profile 10, which configures the ONU's user ports (Ethernet/POTS, VLANs, IGMP, DHCP, TR-069, etc.).

-----

## Removing Previous Installations

### Error when deleting an ONT subscription:

```
EA5800-X2(config-if-gpon-0/1)#ont delete 0 0
  Failure: This configured object has some service virtual
  ports
```

This error occurs because the ONT has associated service ports.

### Removing Service Ports:

```
EA5800-X2(config)#undo service-port vlan 13 gpon 0/1/0 ont 0 gemport 11
  Warning: The operation will delete multiple or all service ports and cause interruptions of many user services.
  It will take several minutes, and console may timeout,
  please use command idle-timeout to set time limit
  Are you sure to release service virtual port(s)? (y/n)[n]:y
  The number of total service virtual port in this
  operation:       1
  Deleting start...

  Deleting end:
  The number of total service virtual port which need be
  deleted:   1
  The number of total service virtual port which have been
  deleted: 1
```

Then, to delete the registered ONTs:

```
EA5800-X2(config-if-gpon-0/1)#ont delete 0 0
  Number of ONTs that can be deleted: 1, success: 1
```

### Enabling ONT Auto-Discovery

To enable auto-find for connected ONTs, refer to the guide: "Huawei - Learn how to configure your OLT + ONU from scratch | Knowledge Base".

**Required Command:**

```
EA5800-X2(config-if-gpon-0/1)#port 0 ont-auto-find enable
  Failure: Make configuration repeatedly
  EA5800-X2(config-if-gpon-0/1)#display ont autofind 0
{ <cr>||<K> }:

  Command:
          display ont autofind 0
   ----------------------------------------------------------------------------
   Number              : 1
   F/S/P               : 0/1/0
   Ont SN              : 485754436B886FA5 (HWTC-6B886FA5)
   Password            : 0x00000000000000000000
   Loid                :
   Checkcode           :
   VendorID            : HWTC
   Ont Version         : 159D.A
   Ont SoftwareVersion : V5R020C10S213
   Ont EquipmentID     : EG8145V5
   Ont Customized Info : COMMON
   Ont MAC             : 68E2-091F-5161
   Ont Equipment SN    : 2150083877EGM2010498
   Ont autofind time   : 2022-09-18 12:50:52+08:00
   Multi channel       : -
   ----------------------------------------------------------------------------
```

### Testing the Standard Plan

Now, we will test the Standard plan with this ONT. In this case, it is located on the GPON interface 0/1, port 0:

```
ont add 0 0 sn-auth "485754436B886FA5" omci ont-lineprofile-id 100 ont-srvprofile-id 100 desc "HUAWEI-STANDARD"
```

Now, the service ports corresponding to port 0 with ONT ID 0 are added:

```
service-port 101 vlan 100 gpon 0/1/0 ont 0 gemport 10 multi-service user-vlan 100 tag-transform translate

service-port 102 vlan 101 gpon 0/1/0 ont 0 gemport 11 multi-service user-vlan 101 tag-transform translate

service-port 103 vlan 200 gpon 0/1/0 ont 0 gemport 12 multi-service user-vlan 200 tag-transform translate

service-port 104 vlan 300 gpon 0/1/0 ont 0 gemport 13 multi-service user-vlan 300 tag-transform translate
```

Currently, the ONT is connected to `eth 1`, so traffic should exit on VLAN 100, as configured in the service profile:

```
ont-srvprofile gpon profile-id 100
ont-port eth adaptive pots adaptive # Active eth and voice ports in automatic
port vlan eth 1 100
port vlan eth 2 101
port vlan eth 3 200
port vlan eth 4 300
commit
quit
```
