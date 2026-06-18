# 5G Core Location Management Function (LMF): Open5GS Gap Analysis

> **Status:** Documentation and gap analysis only. Implementation work tracked separately.  
> **Testbed:** https://github.com/joroch/open5gs-srsran-testbed 
> **Specs:** TS 38.305 · TS 23.273 · TS 29.572 · TS 29.518 · TS 38.455 · TS 37.355

---

## 1. The Problem

The Location Management Function (LMF) is the 5G Core Network Function responsible for all UE positioning. Every NR positioning method defined in TS 38.305—UL-TDOA, DL-TDOA, Multi-RTT, E-CID—routes through it. The gNB sends positioning measurements to the LMF over NRPPa (TS 38.455); the UE receives assistance data from the LMF over LPP (TS 37.355); the AMF relays both protocols transparently without decoding them. The LMF is not optional: without it, there is no positioning plane.

**Open5GS currently implements AMF, SMF, UPF, PCF, UDM, AUSF, NRF, NSSF, BSF, and SCP. It does not implement an LMF.**

The consequence is concrete. The srsRAN Project (OCUDU) CU-CP already implements E-CID and TRP Information NRPPa procedures (`nrppa_context`, initiated in `cu_cp_impl.cpp`). When that gNB sends `UplinkUEAssociatedNRPPaTransport` or `UplinkNonUEAssociatedNRPPaTransport` (TS 38.413 §8.14) to an Open5GS AMF, the AMF has no handler for those messages. There is no SBI client to forward the NRPPa payload. There is no LMF to receive it. The `Nlmf_Location` service interface (TS 29.572) has no server.

This repository maps that gap precisely: interface by interface, message by message, spec clause by spec clause. It is based on direct analysis of the Open5GS source, the srsRAN Project CU-CP NRPPa implementation, and the 3GPP specifications listed above.

The goal is not to explain what 5G positioning is, but to define exactly what needs to be built.

---

## 2. LMF Core Responsibilities

In a 5G Standalone (SA) architecture, the LMF acts as the central control-plane engine for localization. A fully realized LMF is responsible for:

* **Method Selection:** Evaluating UE and gNB capabilities to select the optimal localization method (e.g., E-CID, UL-TDOA).
* **RAN Coordination:** Managing gNB measurement requests and TRP data extraction via NRPPa.
* **UE Coordination:** Managing UE-assisted or UE-based measurement requests via LPP.
* **Coordinate Calculation:** Processing raw radio measurements into geolocation coordinates and exposing them to the core via SBI.
* **Network Registration:** Registering its profile (NF Type: `LMF`) with the NRF (TS 29.510) to allow AMF discovery and routing.

---

## 3. 3GPP Reference Architecture

End-to-end positioning relies on a control-plane architecture where the AMF acts as a transparent router. The AMF encapsulates and unwraps payloads between the RAN/UE and the LMF without decoding the underlying positioning protocols.

This gap analysis targets the following 3GPP Rel-16/Rel-17 specifications:
* **TS 23.273:** 5G System Location Services (LCS) architecture and routing flows.
* **TS 38.305:** NG-RAN positioning procedures (e.g., UL-TDOA in Rel-16 Sec 9.1.2).
* **TS 38.455 (NRPPa):** The control-plane protocol between gNB and LMF (transported over N2).
* **TS 37.355 (LPP):** The control-plane protocol between UE and LMF (transported over N1/N2).
* **TS 29.572 (Nlmf):** The LMF SBI service interface (e.g., `Nlmf_Location_DetermineLocation`).
* **TS 29.518 (Namf):** The AMF SBI service interface used by the LMF to push downward payloads (`Namf_Communication_N1N2MessageTransfer`).

---

## 4. Open5GS Gap Analysis

Based on a direct audit of the Open5GS `main` branch source code against 3GPP specifications, the ecosystem is structurally prepared for localization but lacks the behavioral routing logic and the LMF network function itself.

| Component / Interface | Required by (3GPP Ref) | Open5GS Status | Engineering Notes |
| :--- | :--- | :--- | :--- |
| **NRPPa / LPP ASN.1 Codecs** | TS 38.455 / TS 37.355 | **Supported** | `lib/asn1c/` contains the compiled schemas. Open5GS can natively decode positioning payloads without requiring new ASN.1 integration. |
| **NRF LMF Profiles** | TS 29.510 | **Supported** | OpenAPI structures (`lmf_info.c`, `nf_type.c`) are present. The NRF is ready to accept an LMF registration. |
| **NGAP Transport Routing** | TS 38.413 | **Missing** | AMF lacks handlers for Procedure Code 59 (`UplinkUEAssociatedNRPPaTransport`) and related messages in `src/amf/`. AMF currently drops the payload. |
| **AMF Nlmf Client** | TS 29.572 | **Missing** | AMF lacks the SBI client logic to encapsulate the NRPPa payload into JSON and POST it to the LMF. |
| **Namf_Communication** | TS 29.518 | **Partial** | AMF supports N1N2MessageTransfer but lacks specific bridging logic to map DL NRPPa payloads from the LMF back to the correct gNB/UE context over N2. |
| **Location Management Function** | TS 23.273 | **Missing** | No `src/lmf/` daemon exists to process requests, manage positioning state, or calculate coordinates. |

---

## 5. Minimal Implementation Requirements

To achieve a functional, end-to-end positioning pipeline supporting an AMF-initiated **E-CID** (Enhanced Cell-ID) procedure with an srsRAN gNB, the following minimal implementation is required:

**1. AMF Modifications (`src/amf/`)**
* **NGAP Handlers:** Implement handlers for the four NGAP NRPPa transport messages to extract the transparent NRPPa/LPP payloads.
* **SBI Routing:** Implement an `Nlmf` HTTP/2 client to forward uplink payloads to the LMF via the `Nlmf_Location_DetermineLocation` service.

**2. New LMF Daemon (`src/lmf/`)**
* **Initialization:** Create an `open5gs-lmfd` process that registers its NF profile (`OpenAPI_nf_type_LMF`) with the NRF upon boot.
* **SBI Connections:** Implement an HTTP/2 server to receive AMF triggers via `Nlmf`, and an HTTP/2 client to push downward NRPPa requests via `Namf_Communication`.
* **Positioning Logic:** Implement a basic positioning engine capable of parsing E-CID measurement reports and managing a local context dictionary to track per-UE positioning sessions.