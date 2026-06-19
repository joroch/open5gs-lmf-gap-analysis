# LMF Control Plane Architecture & Message Flows

This document details the target architecture for introducing a Location Management Function
(LMF) into the Open5GS ecosystem: triggering mechanisms, SBI message sequencing, and state
management requirements.

Every Open5GS-specific claim is checked against `open5gs/open5gs` @ commit `b89625d`
(2026-06-18) and cited with exact file paths. Every 3GPP API path is verified against the
OpenAPI yaml files Open5GS bundles under `lib/sbi/support/r17-20230301-openapitools-6.4.0/`.

---

## 1. Triggering Mechanisms (TS 23.273)

The LMF is a reactive positioning engine — it acts only when triggered externally. 3GPP defines
three primary flows:

* **MT-LR (Mobile Terminated Location Request):** An external AF or GMLC requests a UE's
  location. The GMLC calls `Namf_Location` on the AMF, which calls `Nlmf_Location` on the LMF.
* **MO-LR (Mobile Originated Location Request):** The UE requests its own location via uplink
  NAS transport.
* **NI-LR (Network Induced Location Request):** The 5GC internally initiates the request, e.g.
  for emergency routing.

### Architectural scope and the GMLC problem

**Open5GS has no GMLC.** A standard MT-LR flow would require building a second new NF before
the LMF is even reachable from the outside. This document defines two trigger paths:

**Path A — Standards-compliant (full MT-LR chain):** GMLC (or test script acting as GMLC) →
`Namf_Location` on AMF → `Nlmf_Location` on LMF. This is the correct production path but
requires implementing the `Namf_Location` server route in the AMF, which does not currently
exist. Verified: an exhaustive search for `namf-loc`, `namf_location`, and `gmlc` across
`src/amf/` returns nothing.

**Path B — Direct LMF call (testing shortcut):** Test script → `Nlmf_Location` on LMF directly,
bypassing the AMF trigger path entirely. This is not 3GPP-compliant but is a valid and practical
approach for validating the positioning pipeline in a testbed before the AMF route exists. This
is the approach used in [PR #4208](https://github.com/open5gs/open5gs/pull/4208), which calls
`http://127.0.0.16:7777/nlmf-loc/v1/determine-location` directly and has been verified to work
against real Ericsson hardware.

For the initial implementation, **Path B is recommended** — it decouples the positioning
pipeline test from the AMF trigger implementation, allowing both to be developed and validated
independently.

### The trigger script (Path B — direct to LMF)

Python HTTP/2 client (e.g. `httpx`) calling the LMF directly:

**Target endpoint** (verified against `TS29572_Nlmf_Location.yaml`):
```
POST {apiRoot}/nlmf-loc/v1/determine-location
operationId: DetermineLocation
```

**Example payload:**
```json
{
  "supi": "imsi-999700000000001",
  "locationQoS": {
    "hAccuracy": 10.0,
    "verticalRequested": false
  },
  "supportedFeatures": "0"
}
```

### The trigger script (Path A — via AMF, once implemented)

**Target endpoint** (verified against `TS29518_Namf_Location.yaml`):
```
POST {apiRoot}/namf-loc/v1/{ueContextId}/provide-pos-info
operationId: ProvidePositioningInfo
```
This route must be implemented in `src/amf/` before it is usable.

---

## 2. Protocol Stack and the Transparent-Router Pattern

Before detailing message sequencing, the three positioning protocols and their transport
relationships are important to state precisely. A common confusion is conflating LPPa with LPP:

| Protocol | Spec | Interface | Transported over | AMF role |
|:---|:---|:---|:---|:---|
| **NRPPa** | TS 38.455 | gNB ↔ LMF | NGAP (N2), inside NGAP envelope | Transparent relay |
| **LPP** | TS 37.355 | UE ↔ LMF | NAS (N1), inside NAS envelope | Transparent relay |
| **LPPa** | TS 36.455 | eNB ↔ E-SMLC | S1AP (S1), inside S1AP envelope | **4G only** — MME role |

LPPa is the 4G predecessor to NRPPa. It appears in Open5GS's S1AP module
(`lib/asn1c/s1ap/S1AP_LPPa-PDU.h`) as an opaque `OCTET_STRING_t` — the same transparent-routing
pattern — but it has no handler in `src/mme/` and is irrelevant to the 5G SA positioning
architecture described here.

The transparent-relay behavior is confirmed at the ASN.1 level for NRPPa:

```c
// lib/asn1c/ngap/NGAP_NRPPa-PDU.h
typedef OCTET_STRING_t  NGAP_NRPPa_PDU_t;
```

The AMF can locate, extract, and forward this byte blob. It cannot parse the NRPPa messages
inside it. That is the LMF's job, and it requires the full TS 38.455 Annex A ASN.1 codec.

---

## 3. AMF–LMF SBI Message Sequence

The full positioning sequence for E-CID via Path A (standards-compliant) is:

**Step 1 — External trigger (AMF `Namf_Location`):**
Test script POSTs `ProvidePositioningInfo` to the AMF.
**Status: missing** — `namf-loc` route does not exist in `src/amf/`.

**Step 2 — AMF → LMF (`Nlmf_Location` trigger):**
AMF calls `POST {apiRoot}/nlmf-loc/v1/determine-location` on the LMF.
**Status: missing** — no `Nlmf` client exists in `src/amf/`.

**Step 3 — LMF → AMF (DL NRPPa via `Namf_Communication`):**
LMF generates an ASN.1-encoded NRPPa `E-CID Measurement Initiation Request` and calls
`Namf_Communication_N1N2MessageTransfer`, placing the NRPPa bytes in the N2 container.
**Status: needs modification** — `amf_namf_comm_handle_n1_n2_message_transfer()` in
`src/amf/namf-handler.c` is fully implemented, but needs to be adapted to work without a pdu session. The LMF becomes a new consumer of this API. The dedicated SBI envelope for the NRPPa payload is `OpenAPI_nrppa_information_t` (`lib/sbi/openapi/model/nrppa_information.h`), which wraps an `N2InfoContent` and an NF identifier — this struct exists but is currently not wired to anything.

**Step 4 — AMF → gNB (N2, `DownlinkUEAssociatedNRPPaTransport`):**
AMF extracts the NRPPa bytes from the N2 container and wraps them in NGAP procedure code 8
(`DownlinkUEAssociatedNRPPaTransport`) for UE-specific procedures, or code 5
(`DownlinkNonUEAssociatedNRPPaTransport`) for TRP-level procedures (e.g. TRP Information).
**Status: missing** — the NGAP struct is compiled (`lib/asn1c/ngap/
NGAP_DownlinkUEAssociatedNRPPaTransport.c` exists) but no AMF code builds or sends one.

**Step 5 — gNB → AMF (N2, `UplinkUEAssociatedNRPPaTransport`):**
srsRAN CU-CP takes radio measurements and replies over SCTP with NGAP procedure code 50
(`UplinkUEAssociatedNRPPaTransport`), or code 47 (`UplinkNonUEAssociatedNRPPaTransport`) for
Non-UE-Associated procedures.
**Status: missing on AMF side** — the NGAP struct is compiled but procedure codes 47 and 50 are
absent from the 26 procedure codes dispatched in `src/amf/ngap-handler.c`. The AMF currently
drops these messages silently.

**Step 6 — AMF → LMF (UL NRPPa forwarding):**
AMF extracts the NRPPa bytes and forwards them to the LMF, either completing the pending
`DetermineLocation` response or invoking a callback URI the LMF registered.
**Status: missing** — depends on the Nlmf client from Step 2.

**Step 7 — LMF computes and responds:**
LMF decodes the NRPPa response using the TS 38.455 ASN.1 codec, extracts E-CID IEs (Cell-ID,
RSRP, Rx-Tx time difference), computes a position estimate, and returns a `LocationData` JSON
body as the `200 OK` response to the original `DetermineLocation` request.

---

## 4. End-to-End E-CID Procedure (Annotated)

For the minimal srsRAN testbed target:

1. **Trigger:** Script POSTs to LMF `nlmf-loc/v1/determine-location` (Path B) with SUPI.
2. **Method selection:** LMF queries srsRAN CU-CP capabilities and selects E-CID.
3. **DL NRPPa:** LMF calls `Namf_Communication_N1N2MessageTransfer` with an ASN.1-encoded
   `E-CID Measurement Initiation Request` in the N2 container.
4. **N2 routing:** AMF's new handler wraps the bytes in `DownlinkUEAssociatedNRPPaTransport`
   (procedure code 8) and sends to srsRAN over SCTP.
5. **RAN side:** srsRAN `cu_cp_impl.cpp` processes the NRPPa message via `nrppa_context`,
   extracts E-CID metrics, and encapsulates the response.
6. **UL NRPPa:** srsRAN sends `UplinkUEAssociatedNRPPaTransport` (procedure code 50) to AMF.
7. **AMF forwarding:** AMF's new handler extracts the NRPPa bytes and forwards to LMF.
8. **LMF decode:** LMF decodes the `E-CID Measurement Initiation Response` ASN.1 structure,
   reads Cell-ID, RSRP, and timing advance values.
9. **Position estimate:** LMF computes coordinates from the cell database and measurement data.
10. **Response:** LMF returns `LocationData` JSON to the caller.

Steps 3 (transport side only, not codec) and — after implementation — steps 4 and 7 reuse
existing Open5GS infrastructure. Steps 2, 5, 6, 8, 9, 10 are new.

---

## 5. State Management

Open5GS NFs use `ogs_hash_t` and `ogs_list_t` for per-entity in-memory state. This is the live
convention in `src/amf/context.h`, which tracks `guti_ue_hash`, `suci_hash`, `supi_hash`, and
`gnb_addr_hash` as hash tables alongside `ogs_list_t` for GNB and UE context lists.

The LMF needs the same pattern for transaction tracking:

```c
// Proposed LMF context (following Open5GS convention)
typedef struct lmf_context_s {
    ogs_hash_t  *supi_hash;     /* SUPI → lmf_ue_t* */
    ogs_hash_t  *transaction_hash; /* LMF-transaction-ID → lmf_transaction_t* */
} lmf_context_t;
```

Because the SBI leg (HTTP/2) and the RAN leg (SCTP, relayed through AMF) are asynchronous:
- When the LMF sends an NRPPa request, it generates a transaction ID and records it in
  `transaction_hash` mapped to the relevant UE SUPI.
- The LMF suspends that transaction's state machine.
- When the AMF relays the NRPPa response (~50–100ms for E-CID), the LMF extracts the
  transaction ID from the NRPPa payload, looks it up in `transaction_hash`, and resumes.

Persistent storage (MongoDB, which Open5GS already uses for subscriber data) is not required for
single-shot E-CID. It would be needed for Rel-17 periodic tracking (`Nlmf_DataExposure`).

---

## 6. Open Questions for Implementation Phase

These were not resolved during the gap analysis and need answers before or during implementation:

**Q1 — NRPPa payload delivery to AMF via `N1N2MessageTransfer`:** The existing
`amf_namf_comm_handle_n1_n2_message_transfer()` currently requires a `pdu_session_id`
(`N1N2MessageTransferReqData->is_pdu_session_id` is checked). LMF-initiated NRPPa messages have
no PDU session context. This function may need a new code path or a wrapper, rather than being
usable as-is. This is the most important open question for Step 3.

**Q2 — Non-UE-Associated transport and context anchoring:** For TRP Information procedures
(Non-UE-Associated NRPPa, procedure codes 5/47), there is no UE NGAP ID to anchor the
transaction. What identifies the target gNB in the NGAP routing layer? Needs analysis of how
the AMF routes non-UE-associated NGAP messages to the correct gNB in other procedures.

**Q3 — srsRAN E-CID measurement IEs:** What does srsRAN's CU-CP actually populate in the E-CID
Measurement Initiation Response today (Cell-ID, RSRP, TA, AoA)? The gap between "IE is defined
in TS 38.455" and "srsRAN sends it" determines what the LMF's positioning logic can actually
use. Needs a Wireshark capture from the testbed with NRPPa dissector.

**Q4 — hug0lin's fork compatibility with srsRAN:** PR #4208 was tested against Ericsson
commercial hardware (BBU6655). srsRAN's NRPPa implementation (`nrppa_context`) may produce
different IE sets or message ordering. Testing the PR branch against the srsRAN testbed is the
fastest way to surface srsRAN-specific issues and is the natural next contribution to make.