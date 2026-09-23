# Lab 1: Analyzing UE–gNB Connectivity in an OAI 5G SA Network

## 5. Identify the Basic 5G SA Architecture

### Component table

| Component      | IP address         | Evidence from the capture                          |
| -------------- | ------------------ | -------------------------------------------------- |
| UE PDU address | **[from capture]** | PDU Session Establishment Accept / UE IPv4 address |
| gNB            | **[from capture]** | NGAP / GTP-U endpoints                             |
| AMF            | **[from capture]** | NGAP packets                                       |
| UPF            | **[from capture]** | GTP-U packets                                      |
| Data Network   | **[from capture]** | ICMP / user-plane traffic                          |

### Interface table

| Interface | Connected components | Main protocol     | Purpose                                                               |
| --------- | -------------------- | ----------------- | --------------------------------------------------------------------- |
| N1        | UE ↔ AMF             | NAS               | Control/signaling between UE and AMF. NAS is transported via the gNB. |
| N2        | gNB ↔ AMF            | NGAP over SCTP    | Control plane communication between the RAN and the 5G Core.          |
| N3        | gNB ↔ UPF            | GTP-U over UDP/IP | Transport of user-plane data between the gNB and UPF.                 |

---

## 6. Analyze the RRC Connection Establishment

### RRC message table

| Message          | Direction | Logical channel / SRB | Main purpose                                                                          |        Packet number |
| ---------------- | --------- | --------------------- | ------------------------------------------------------------------------------------- | -------------------: |
| RRCSetupRequest  | UE → gNB  | UL-CCCH / SRB0        | UE requests establishment of an RRC connection.                                       | **[from Wireshark]** |
| RRCSetup         | gNB → UE  | DL-CCCH / SRB0        | gNB accepts the request and provides the configuration for the RRC connection.        | **[from Wireshark]** |
| RRCSetupComplete | UE → gNB  | SRB1                  | UE confirms completion of RRC establishment and carries the NAS Registration Request. | **[from Wireshark]** |

### 1. What is the establishment cause in `RRCSetupRequest`?

The establishment cause indicates **why the UE wants to establish an RRC connection**.

For a normal data connection, the establishment cause is typically:

**`mo-Data` – Mobile Originated Data**

The exact value should be verified in the `RRCSetupRequest` packet in Wireshark.

### 2. What SRB does `RRCSetupRequest` use? Why?

`RRCSetupRequest` uses **SRB0**.

SRB0 is used for **initial RRC signaling before a dedicated signaling connection has been established**.

### 3. Which side sends `RRCSetup`?

The **gNB sends `RRCSetup` to the UE**.

The sequence is:

**UE → RRCSetupRequest → gNB → RRCSetup → UE**

### 4. Which signaling radio bearer is used after the RRC connection is established?

The main signaling radio bearer is **SRB1**.

* **SRB0** → initial RRC signaling
* **SRB1** → normal RRC signaling after the RRC connection is established

### 5. Which NAS message is carried inside `RRCSetupComplete`?

The NAS message is:

**`Registration Request`**

It is carried inside:

`RRCSetupComplete → dedicatedNAS-Message → Registration Request`

### 6. At the end of this procedure, is the UE only connected to the gNB, or is it already registered with the 5G Core?

At the end of the RRC connection establishment procedure, the UE has an **RRC connection with the gNB**, but it is **not yet fully registered with the 5G Core**.

The `Registration Request` has already been sent inside `RRCSetupComplete`, so the 5G Registration procedure has **started**, but it still needs to be completed.

The UE becomes successfully registered after the 5G Core sends **Registration Accept** and the UE responds with **Registration Complete**.

---

## 7. Connect RRC Signaling to NGAP and NAS

### Mapping table

| Stage      | Protocol message      | Sender → receiver | Encapsulated information   |
| ---------- | --------------------- | ----------------- | -------------------------- |
| Radio side | RRCSetupComplete      | UE → gNB          | NAS `Registration Request` |
| Core side  | NGAP InitialUEMessage | gNB → AMF         | NAS `Registration Request` |

### 1. What is the role of the gNB when it transports NAS messages?

The gNB acts as an **intermediate transport node** between the UE and the AMF.

The NAS message is not terminated at the gNB. The gNB transports the NAS message from the UE toward the AMF using **NGAP**.

### 2. What is the difference between RRC and NAS signaling?

**RRC (Radio Resource Control)** handles signaling between the **UE and gNB**. It controls the radio connection and radio resources.

**NAS (Non-Access Stratum)** handles signaling between the **UE and the 5G Core**, mainly the AMF. It is used for procedures such as registration, authentication, and mobility management.

### 3. Is the Registration Request delivered directly from the UE to the AMF?

No. The Registration Request follows this protocol path:

**UE → RRC → gNB → NGAP → AMF**

The NAS message is carried inside RRC on the radio side and then transported by the gNB inside NGAP toward the AMF.

### 4. Which message confirms that Registration has completed successfully?

The UE sends **`Registration Complete`** after receiving **`Registration Accept`**.

The relevant sequence is:

**Registration Request → Authentication → Security Mode → Registration Accept → Registration Complete**

---

## 8. Verify the UE IP Address and User-Plane Traffic

| Field           | Observed value                                          |
| --------------- | ------------------------------------------------------- |
| UE IPv4 address | **[from Wireshark – PDU Session Establishment Accept]** |

### Questions

**1. What IPv4 address was assigned to the UE?**

**[Enter the UE IPv4 address shown in the PDU Session Establishment Accept.]**

**2. How many ICMP Echo Request/Reply pairs are present?**

**[Count the ICMP Echo Request and Echo Reply packets in Wireshark.]**

**3. What does the successful Echo Reply prove about the UE connection?**

A successful ICMP Echo Reply shows that **user-plane connectivity is working**. The UE's IP packet is transported through the 5G user plane using **GTP-U between the gNB and UPF**, allowing communication with the Data Network.

---

## 9. Final UE Connection Sequence

The overall procedure can be summarized as:

```text
UE                  gNB                 AMF                 UPF          Data Network
 |                   |                   |                    |                |
 |-- RRCSetupRequest->|                  |                    |                |
 |<----- RRCSetup-----|                  |                    |                |
 |-- RRCSetupComplete>|                  |                    |                |
 |   + Registration Request              |                    |                |
 |                   |-- InitialUEMessage ->|                  |                |
 |                   |                   |                    |                |
 |                   |                   |<-- Authentication -->|                |
 |                   |                   |<-- Security Mode --->|                |
 |                   |                   |                    |                |
 |                   |<-- Registration Accept --|             |                |
 |<-- Registration Complete --------------|                    |                |
 |                   |                   |                    |                |
 |-------- PDU Session Establishment ------------------------>|                |
 |                   |                   |                    |                |
 |==================== GTP-U User Plane =====================>|--------------->|
 |<=================== ICMP Echo Reply ========================|<---------------|
```

### Important distinction

**Control plane:**

* RRC
* NAS
* NGAP

**User plane:**

* IP traffic
* GTP-U
* ICMP

The RRC connection establishes the signaling connection between **UE and gNB**. The NAS/NGAP procedures then establish registration with the **5G Core**. Finally, PDU Session Establishment provides the UE with an IP address and enables **user-plane connectivity through the UPF**.

