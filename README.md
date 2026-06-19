# 5G Core Location Management Function (LMF): Open5GS Gap Analysis

> **Status:** Documentation and gap analysis only. Implementation work tracked separately.  
> **Testbed:** https://github.com/joroch/open5gs-srsran-testbed.  
> **Specs:** TS 38.305 · TS 23.273 · TS 29.572 · TS 29.518 · TS 29.510 · TS 38.455 · TS 37.355  
> **Verified against:** `open5gs/open5gs` @ commit `b89625d` (2026-06-18). Every source claim
> in this repo is grep-reproducible against that commit  
> see> [docs/open5gs-gap.md](docs/open5gs-gap.md) for exact file paths and line references.

---

## 1. The Problem

The Location Management Function (LMF) is the 5G Core Network Function responsible for all UE positioning. Every NR positioning method defined in TS 38.305—UL-TDOA, DL-TDOA, Multi-RTT, E-CID—routes through it. The gNB sends positioning measurements to the LMF over NRPPa (TS 38.455); the UE receives assistance data from the LMF over LPP (TS 37.355); the AMF relays both protocols transparently without decoding them. The LMF is not optional: without it, there is no positioning plane.

**Open5GS currently implements AMF, SMF, UPF, PCF, UDM, AUSF, NRF, NSSF, BSF, and SCP. It does not implement an LMF.**

The consequence is concrete. The srsRAN Project (OCUDU) CU-CP already implements E-CID and TRP Information NRPPa procedures (`nrppa_context`, initiated in `cu_cp_impl.cpp`). When that gNB sends `UplinkUEAssociatedNRPPaTransport` or `UplinkNonUEAssociatedNRPPaTransport` (TS 38.413 §8.10) to an Open5GS AMF, the AMF has no handler for those messages. There is no SBI client to forward the NRPPa payload. There is no LMF to receive it. The `Nlmf_Location` service interface (TS 29.572) has no implementation anywhere in the codebase.

This repository maps that gap precisely: interface by interface, message by message, spec clause by spec clause. It is based on direct analysis of the Open5GS source, the srsRAN Project CU-CP NRPPa implementation, and the 3GPP specifications listed above.

The goal is not to explain what 5G positioning is, but to define exactly what needs to be built.

### Related community work

An open pull request — [open5gs/open5gs#4208](https://github.com/open5gs/open5gs/pull/4208)
(hug0lin, Dec 2025) — implements an LMF with E-CID and Cell-ID positioning, tested against
commercial Ericsson hardware. As of this writing it has not been merged into `main` and the
author has noted the branch needs rebasing before re-submission. The official codebase therefore
remains without an LMF, and this gap analysis of `main` reflects that state. The PR independently
corroborates every gap identified in Section 4: AMF NGAP handlers, Nlmf SBI client, NRPPa ASN.1
codec, and NRF registration were all absent from `main` and had to be built from scratch.

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

End-to-end positioning relies on a control-plane architecture where the AMF acts as a transparent router. The AMF encapsulates and unwraps NRPPa/LPP payloads between the RAN and the LMF without decoding them. This is confirmed at the ASN.1 level in Open5GS itself: `NGAP_NRPPa_PDU_t` is defined as `typedef OCTET_STRING_t` — a raw byte buffer, not a decoded message structure (`lib/asn1c/ngap/NGAP_NRPPa-PDU.h`).

A note on nomenclature that causes frequent confusion in the positioning stack:

| Protocol | Spec | Connects | Network generation |
|:---|:---|:---|:---|
| **LPPa** | TS 36.455 | eNB ↔ E-SMLC | 4G / EPC only |
| **NRPPa** | TS 38.455 | gNB ↔ LMF | 5G SA |
| **LPP** | TS 37.355 | UE ↔ LMF (or E-SMLC) | 4G and 5G |

LPPa is the 4G predecessor to NRPPa. It appears in Open5GS's S1AP module
(`lib/asn1c/s1ap/S1AP_LPPa-PDU.h`) as a compiled but opaque `OCTET_STRING` — the same
transparent-routing pattern as NGAP/NRPPa. It is not LPP, and neither LPPa nor LPP has any
handler in `src/mme/` or `src/amf/`.

This gap analysis targets the following 3GPP Rel-16/Rel-17 specifications:
* **TS 23.273:** 5G System Location Services (LCS) architecture and routing flows.
* **TS 38.305:** NG-RAN positioning procedures (e.g., UL-TDOA in Rel-16 Sec 9.1.2).
* **TS 38.455 (NRPPa):** The control-plane protocol between gNB and LMF (transported over N2).
* **TS 38.413 (NGAP):** Defines the 4 NRPPa transport message types carried between gNB and AMF as opaque payloads (§8.10). Procedure codes: DL non-UE=5, DL UE=8, UL non-UE=47, UL UE=50.
* **TS 37.355 (LPP):** The control-plane protocol between UE and LMF (transported over N1(NAS)).
* **TS 29.572 (Nlmf):** The LMF SBI service interface (e.g., `Nlmf_Location_DetermineLocation`).
* **TS 29.518 (Namf):** The AMF SBI service interface used by the LMF to push downward payloads (`Namf_Communication_N1N2MessageTransfer`).
* **TS 29.510 (Nnrf):** NF registration and discovery, including the `LMF` NF type.

---

## 4. Open5GS Gap Analysis

Based on a direct audit of the Open5GS `main` branch source code against 3GPP specifications. The picture is not binary — some scaffolding exists and is reusable; some exists but is dead/unwired; some is absent. The distinction matters for sizing the implementation effort.

| Component / Interface | Required by (3GPP Ref) | Open5GS Status | Engineering Notes |
| :--- | :--- | :--- | :--- |
| **NGAP NRPPa transport envelope** | TS 38.413 §8.10 | **Compiled — not wired** | All 4 transport message structs (proc. codes 5, 8, 47, 50) are compiled in `lib/asn1c/ngap/`. The NRPPa payload is typed as `OCTET_STRING_t`, opaque by design. The envelope can be decoded; the inner NRPPa PDU cannot. |
| **NRPPa application-layer ASN.1 codec** | TS 38.455 Annex A | **Missing** | `lib/asn1c/` contains only `common`, `ngap`, `s1ap`, `support`, `util`. No NRPPa protocol module (E-CID IEs, measurement request/response structures) exists. The LMF and gNB need this; the AMF does not. |
| **LPP ASN.1 codec** | TS 37.355 Annex A | **Missing** | No TS 37.355 LPP module exists anywhere. Note: `S1AP_LPPa-PDU.h` in `lib/asn1c/s1ap/` is the 4G **LPPa** protocol (TS 36.455) — a different spec, different interface. |
| **AMF NGAP handlers** | TS 38.413 §8.10 | **Missing** | `src/amf/ngap-handler.c` dispatches on 26 procedure codes. None of the 4 NRPPa transport codes (5, 8, 47, 50) are present. The AMF currently has no path to handle or forward any NRPPa payload. |
| **AMF → LMF (`Nlmf_Location` client)** | TS 29.572 | **Missing** | Zero references to `Nlmf` or `determine-location` anywhere in `src/`. The OpenAPI yaml is bundled for code generation but no client is implemented. |
| **SBI NRPPa envelope** | TS 29.572 / TS 29.518 | **Scaffold exists — not wired** | `lib/sbi/openapi/model/nrppa_information.h` defines a dedicated `OpenAPI_nrppa_information_t` struct (`nf_id`, `nrppa_pdu`, `service_instance_id`) for carrying NRPPa inside SBI messages. `nrppa_trans_failure.c` defines the 3-value failure enum. Neither is referenced outside the generated model directory. |
| **LMF → AMF (`Namf_Communication`)** | TS 29.518 | **Needs modification** | `amf_namf_comm_handle_n1_n2_message_transfer()` in `src/amf/namf-handler.c` is fully implemented for SM/NAS signaling with both N1 and N2 containers. However this function will fail if pdu_session_id is not set. An LMF delivering NRPPa has no PDU session. This has to be modified and a new caller in the LMF is needed. |
| **AMF GMLC interface (`Namf_Location`)** | TS 29.518 | **Missing** | No `namf-loc` server route exists in `src/amf/`. Required for any standards-compliant external trigger. A test harness can bypass this and call the LMF's `Nlmf_Location` directly (as shown in [PR #4208](https://github.com/open5gs/open5gs/pull/4208)), but that is a testing shortcut, not a production-compliant path. |
| **NRF LMF NF profile** | TS 29.510 | **Dead scaffold** | `OpenAPI_nf_type_LMF` and `lmf_info.c/.h` exist in `lib/sbi/openapi/model/`, auto-generated from the full 3GPP NF-type catalogue. They are never referenced outside that directory — no NRF registration or discovery logic for an LMF exists. |
| **Location Management Function(daemon)** | TS 23.273 | **Missing** | No `src/lmf/` directory, process requests, manage positioning state, or positioning logic. |

---

## 5. Minimal Implementation Requirements

To build a functional end-to-end E-CID positioning pipeline against an srsRAN gNB, the following
is required. Note that `Namf_Communication` is already implemented and reusable, which meaningfully
narrows the scope.

**1. AMF modifications (`src/amf/`)**
* **NGAP handlers:** Implement dispatch for procedure codes 5, 8, 47, 50 to extract the opaque
  NRPPa payload from the NGAP envelope.
* **`Nlmf_Location` client:** Implement an HTTP/2 client to forward extracted payloads to the LMF
  via `POST {apiRoot}/nlmf-loc/v1/determine-location`.
* **`Namf_Location` server (for standards-compliant trigger only):** Implement
  `POST {apiRoot}/namf-loc/v1/{ueContextId}/provide-pos-info`. Not required if testing via direct
  API calls to the LMF — but required for any GMLC-facing deployment.

**2. New LMF daemon (`src/lmf/`)**
* **NRF registration:** Register an `LMF` NF profile on boot following the pattern in
  `src/pcf/nrf-handler.c`. The OpenAPI data model (`lmf_info.c`) exists; the registration logic
  does not.
* **`Nlmf_Location` server:** HTTP/2 server implementing `DetermineLocation`.
* **`Namf_Communication` client:** HTTP/2 client calling the existing AMF
  `N1N2MessageTransfer` endpoint to push DL NRPPa. This reuses a working AMF-side API.
* **NRPPa ASN.1 codec:** Compile TS 38.455 9.3 using an ASN1 generator.
* **E-CID positioning logic:** Parse `E-CID Measurement Initiation Response` IEs
  (Cell-ID, RSRP, Rx-Tx time difference) and compute a coordinate estimate.
* **Transaction state:** Track in-flight requests in an `ogs_hash_t` keyed by SUPI, mirroring
  the `supi_hash` / `guti_ue_hash` pattern in `src/amf/context.h`.

**Out of scope for the minimal E-CID target:** LPP (requires TS 37.355 ASN.1 module, NAS
transport integration); UL-TDOA / DL-TDOA / Multi-RTT (multi-TRP synchronization and RSTD
measurements); GMLC based location trigger; persistent location storage; periodic/event-triggered reporting (Rel-17).

---

## 6. Documentation

* [docs/lmf-architecture.md](docs/lmf-architecture.md) — LMF internals, trigger flows, SBI
  message sequencing, and state management.
* [docs/open5gs-gap.md](docs/open5gs-gap.md) — Detailed gap breakdown with exact file paths
  and line-level evidence.
* [docs/interfaces.md](docs/interfaces.md) — NRPPa, LPP, LPPa, and SBI interface reference.

---

## 7. References

| Spec | Title | Release |
|:---|:---|:---|
| **TS 23.273** | 5G System Location Services, Stage 2 | Rel-17 |
| **TS 38.305** | Stage 2 functional specification of UE positioning in NG-RAN | Rel-16 |
| **TS 38.413** | NG Application Protocol (NGAP) | Rel-17 |
| **TS 38.455** | NR Positioning Protocol A (NRPPa) | Rel-17 |
| **TS 37.355** | LTE Positioning Protocol (LPP) | Rel-17 |
| **TS 36.455** | LTE Positioning Protocol A (LPPa) — 4G predecessor to NRPPa | Rel-17 |
| **TS 29.572** | Location Management Services (`Nlmf`) | Rel-17 |
| **TS 29.518** | Access and Mobility Management Services (`Namf`) | Rel-17 |
| **TS 29.510** | Network Function Repository Services (`Nnrf`) | Rel-17 |

* **Open5GS source** — https://github.com/open5gs/open5gs, commit `b89625d` (2026-06-18)
* **srsRAN Project** — CU-CP NRPPa implementation (`nrppa_context`, `cu_cp_impl.cpp`)
* **Related PR** — open5gs/open5gs#4208 (unmerged, Dec 2025): working E-CID LMF implementation
  by hug0lin, tested against Ericsson commercial hardware


