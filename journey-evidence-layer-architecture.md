# Journey Evidence Layer: Architecture Design Document

**Version:** 1.0
**Date:** January 15, 2026
**Status:** Draft - Architectural Conceptualization
**Author:** Sridhar Dhulipala

---

## Executive Summary

The **Journey Evidence Layer** is a DPI-based architectural pattern that bridges the gap between payment initiation and ownership verification by creating a verifiable chain of transaction evidence. Unlike traditional payment systems that treat transactions as discrete events, this architecture treats each transaction as part of a user's **journey** that requires persistent, verifiable evidence accessible across multiple service providers.

**Core Innovation:** Leveraging Digilocker as a **Transaction Verifiable Credentials Vault** while using DPIx (Digital Dropbox) as an **orchestration agent** that coordinates intent flows across Aadhaar (identity), UPI (payment), and service providers.

---

## 1. Problem Statement

### 1.1 The Payment-Ownership Gap

Current digital transactions suffer from a fundamental disconnect:

1. **Payment happens** → User pays via UPI
2. **Confirmation exists** → Bank SMS, payment gateway email
3. **Service delivery fails** → Application doesn't receive callback
4. **Evidence is scattered** → User has proof, but system cannot ingest it

**Example Case:** Groww Investment Failure (Reference: vault1.html)
- User initiates ₹10,000 investment
- Deutsche Bank debits account ✓
- Razorpay confirms payment ✓
- Groww system fails to acknowledge ✗
- User has email/SMS proof, but no mechanism to inject it into resolution flow

### 1.2 Structural Issues

| Issue | Description | Current Impact |
|-------|-------------|----------------|
| **Context Loss** | Payment gateways and service providers operate in silos | Manual reconciliation required |
| **Silent Drops** | Async webhooks timeout without retry logic | Transactions fall through cracks |
| **Trust Gap** | User holds evidence, system cannot verify it | Multi-day resolution cycles |
| **Fragmented Evidence** | SMS, email, app notifications across multiple channels | No single source of truth |

---

## 2. Solution Architecture Overview

### 2.1 The Journey Evidence Layer

The Journey Evidence Layer introduces a **standardized, user-controlled repository** for transaction evidence that:

1. **Captures** payment confirmations, service acknowledgments, and journey milestones
2. **Stores** them as verifiable credentials in Digilocker
3. **Makes them accessible** to authorized services via consent-based protocols (DEPA)
4. **Enables** faster dispute resolution and automated reconciliation

### 2.2 Architectural Principles

```
┌─────────────────────────────────────────────────────────┐
│  Journey Evidence Layer (Conceptual)                     │
│                                                          │
│  User Journey: Intent → Payment → Evidence → Service    │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │  DPIx    │───▶│ Digilocker│───▶│ Service  │          │
│  │(Orchestr)│    │  (Vault)  │    │ Provider │          │
│  └──────────┘    └──────────┘    └──────────┘          │
│       ▲               ▲                                  │
│       │               │                                  │
│  ┌────┴───┐      ┌───┴────┐                            │
│  │ Aadhaar│      │  UPI   │                            │
│  │ (Auth) │      │(Payment)│                            │
│  └────────┘      └────────┘                            │
└─────────────────────────────────────────────────────────┘
```

**Key Insight:** DPIx does NOT store data. Digilocker is the vault. DPIx is the agent that orchestrates the flow.

---

## 3. Component Architecture

### 3.1 Core Components

#### 3.1.1 Digilocker (Transaction Vault)

**Role:** Secure, user-controlled repository for transaction verifiable credentials

**Enhancements Required:**
- **Transactional Storage Area:** New document type for dynamic transaction evidence (beyond static certificates)
- **Private Spaces:** Encrypted containers for sensitive transaction data
- **Credential Schemas:** Standardized formats for:
  - Payment confirmations (UPI transaction IDs, timestamps, amounts)
  - Service acknowledgments (order IDs, NAV allocation dates)
  - Journey milestones (KYC completion, payment gateway confirmations)

**Storage Model:**
```
Digilocker Document Structure:
├── Issued Documents (Current)
│   ├── Aadhaar.pdf
│   ├── PAN.pdf
│   └── ...
└── Transaction Evidence (NEW)
    ├── Private Space (Encrypted)
    │   ├── payment_confirmations/
    │   │   ├── upi_txn_12345.json (signed)
    │   │   └── razorpay_conf_98765.json (signed)
    │   └── service_evidence/
    │       └── groww_investment_proof.json
    └── Shared Evidence (Consent-based access)
```

#### 3.1.2 DPIx (Digital Dropbox / Orchestration Agent)

**Role:** User-side agent that manages the journey flow and intent coordination

**NOT a central service** - behaves like a client-side orchestrator (similar to a wallet app)

**Capabilities:**
1. **Intent Weaving:** Chains together Aadhaar auth → UPI payment → service delivery
2. **Evidence Collection:** Captures responses from each step
3. **Credential Writing:** Stores evidence in Digilocker with user consent
4. **Dispute Initiation:** Constructs "context packets" from stored evidence for service providers

**Architectural Pattern:**
```
DPIx Agent Flow:
1. User initiates transaction via service provider (e.g., Groww)
2. DPIx intercepts intent and orchestrates:
   a. Aadhaar auth (if needed)
   b. UPI payment intent
   c. Captures payment confirmation
   d. Writes to Digilocker private space
   e. Returns control to service provider
3. On failure: DPIx can retrieve evidence and present to support
```

#### 3.1.3 Aadhaar App (Identity Layer)

**Current Limitation:** Intent calls for authentication are NOT currently available from the Aadhaar app itself

**Proposed Integration:**
- **eKYC Intent:** Standardized Android/iOS intent for authentication
- **Response Format:** Signed XML with demographic data + photo (similar to existing eKYC)
- **Privacy-Preserving:** Only requested attributes shared (not full Aadhaar details)

**Intent Flow Example:**
```kotlin
// Android Intent (Proposed)
val intent = Intent("in.gov.uidai.aadhaar.AUTH")
intent.putExtra("requestedAttributes", listOf("name", "dob"))
intent.putExtra("purpose", "Investment KYC")
intent.putExtra("consentDuration", "one-time")
startActivityForResult(intent, AADHAAR_AUTH_REQUEST)
```

#### 3.1.4 UPI (Payment Layer)

**Current State:** UPI already supports intent-based payments

**Journey Evidence Enhancement:**
- **Transaction Receipt as Verifiable Credential:** UPI app signs transaction confirmation
- **Automatic Digilocker Storage:** Option to auto-store receipts in Digilocker
- **Merchant Identifier Linking:** UPI transaction linked to service provider's GSTIN/PAN

**Intent Flow:**
```kotlin
// Existing UPI Intent Pattern
val uri = "upi://pay?pa=merchant@upi&pn=Groww&am=10000&tn=Investment"
val intent = Intent(Intent.ACTION_VIEW, Uri.parse(uri))
startActivityForResult(intent, UPI_PAYMENT_REQUEST)

// Enhanced with Journey Evidence
// UPI app returns signed receipt → DPIx stores in Digilocker
```

#### 3.1.5 DEPA (Consent Layer)

**Role:** Manages consent for service providers to access transaction evidence stored in Digilocker

**Consent Flow:**
1. Service provider requests access to specific transaction evidence
2. User receives consent request (via DPIx or Digilocker app)
3. User grants time-bound, purpose-limited access
4. Service provider retrieves evidence via DEPA AA (Account Aggregator) protocol

**Example Use Case:** Groww requests access to Razorpay payment confirmation for failed investment
```
Consent Request:
- Issuer: Groww (GSTIN: 12345)
- Purpose: Investment reconciliation
- Data Requested: UPI transaction ID ABC123, timestamp 2026-01-10 09:03
- Duration: 7 days
- User Action: Approve/Deny
```

---

## 4. Data Flow & Interaction Patterns

### 4.1 Normal Transaction Flow (Happy Path)

```
┌──────┐       ┌──────┐       ┌──────┐       ┌──────────┐       ┌──────┐
│ User │       │ DPIx │       │ UPI  │       │Digilocker│       │Groww │
└──┬───┘       └──┬───┘       └──┬───┘       └────┬─────┘       └──┬───┘
   │              │              │                 │                │
   │ Initiate     │              │                 │                │
   │ Investment   │              │                 │                │
   ├─────────────▶│              │                 │                │
   │              │              │                 │                │
   │              │ UPI Payment  │                 │                │
   │              │ Intent       │                 │                │
   │              ├─────────────▶│                 │                │
   │              │              │                 │                │
   │              │    Payment   │                 │                │
   │              │    Success   │                 │                │
   │              │◀─────────────┤                 │                │
   │              │              │                 │                │
   │              │ Store Receipt│                 │                │
   │              │ (Signed)     │                 │                │
   │              ├──────────────┼────────────────▶│                │
   │              │              │                 │                │
   │              │              │    Notify Groww │                │
   │              │              │    (w/ txn ID)  │                │
   │              ├──────────────┼─────────────────┼───────────────▶│
   │              │              │                 │                │
   │              │              │                 │ Allocate Units │
   │              │◀─────────────┴─────────────────┴────────────────┤
   │              │              │                 │                │
   │ Success      │              │                 │                │
   │◀─────────────┤              │                 │                │
```

### 4.2 Failed Transaction Flow (with Evidence Recovery)

```
┌──────┐       ┌──────┐       ┌──────┐       ┌──────────┐       ┌──────┐
│ User │       │ DPIx │       │ UPI  │       │Digilocker│       │Groww │
└──┬───┘       └──┬───┘       └──┬───┘       └────┬─────┘       └──┬───┘
   │              │              │                 │                │
   │ Investment   │              │                 │                │
   ├─────────────▶│              │                 │                │
   │              │              │                 │                │
   │              │ UPI Payment  │                 │                │
   │              ├─────────────▶│                 │                │
   │              │              │                 │                │
   │              │ Success      │                 │                │
   │              │◀─────────────┤                 │                │
   │              │              │                 │                │
   │              │ Store Receipt│                 │                │
   │              ├──────────────┼────────────────▶│                │
   │              │              │                 │                │
   │              │ Notify Groww │                 │                │
   │              ├──────────────┼─────────────────┼───────────────▶│
   │              │              │                 │                │
   │              │              │                 │  ❌ Webhook    │
   │              │              │                 │     Timeout    │
   │              │              │                 │                │
   │ Payment      │              │                 │                │
   │ Failed       │              │                 │                │
   │◀─────────────┤              │                 │                │
   │ (Groww UI)   │              │                 │                │
   │              │              │                 │                │
   │ Query DPIx   │              │                 │                │
   ├─────────────▶│              │                 │                │
   │              │              │                 │                │
   │              │ Retrieve     │                 │                │
   │              │ Evidence     │                 │                │
   │              ├──────────────┼────────────────▶│                │
   │              │              │                 │                │
   │              │◀─────────────┼─────────────────┤                │
   │              │              │                 │                │
   │ Evidence     │              │                 │                │
   │ Displayed    │              │                 │                │
   │◀─────────────┤              │                 │                │
   │              │              │                 │                │
   │ Grant Consent│              │                 │                │
   │ to Groww     │              │                 │                │
   ├─────────────▶│              │                 │                │
   │              │              │                 │                │
   │              │ DEPA Consent │                 │                │
   │              ├──────────────┼────────────────▶│                │
   │              │              │                 │                │
   │              │              │       Groww requests evidence     │
   │              │              │                 │◀───────────────┤
   │              │              │                 │                │
   │              │              │ Evidence shared (w/ consent)     │
   │              │              │                 ├───────────────▶│
   │              │              │                 │                │
   │              │              │                 │ Manual         │
   │              │              │                 │ Reconciliation │
   │              │              │                 │ + Unit Alloc   │
```

---

## 5. Technical Design Details

### 5.1 Intent Call Architecture

**Challenge:** Aadhaar app does not currently support authentication intents

**Solution Approaches:**

#### Option A: Aadhaar OTP via Intent (Interim)
- Use Aadhaar OTP for authentication
- DPIx captures OTP response and stores as authentication proof
- **Limitation:** Not true eKYC, only authentication proof

#### Option B: Third-party eKYC Providers (Current Reality)
- Use existing licensed eKYC service providers
- DPIx captures eKYC response and stores in Digilocker
- **Limitation:** Adds third-party dependency, cost

#### Option C: Future Aadhaar Auth Intent (Proposed)
- UIDAI releases official authentication intent API
- Similar pattern to UPI intent calls
- Returns signed demographic data + auth timestamp
- **Advantage:** Direct, no intermediaries

**Recommended Approach:** Start with Option B, advocate for Option C with UIDAI

### 5.2 Digilocker Storage Schema

**Transaction Evidence Document Format:**

```json
{
  "documentType": "TransactionEvidence",
  "version": "1.0",
  "transactionId": "JOURNEY_2026011509030001",
  "userId": "aadhaar_hash_xxxxx",
  "timestamp": "2026-01-15T09:03:00Z",
  "evidence": {
    "payment": {
      "provider": "UPI",
      "transactionId": "UPI_ABC123",
      "amount": 10000,
      "currency": "INR",
      "payerVPA": "user@oksbi",
      "payeeVPA": "merchant@razorpay",
      "timestamp": "2026-01-15T09:03:00Z",
      "signature": "signed_by_upi_app"
    },
    "service": {
      "provider": "Groww",
      "serviceType": "Investment",
      "orderId": "GROWW_INV_88392",
      "status": "pending",
      "expectedCompletion": "2026-01-15T09:05:00Z"
    },
    "intermediary": {
      "provider": "Razorpay",
      "confirmationId": "RZP_9983",
      "timestamp": "2026-01-15T09:03:30Z",
      "merchantId": "next_billion_tech",
      "signature": "signed_by_razorpay"
    }
  },
  "metadata": {
    "createdBy": "DPIx_Agent_v1.0",
    "consentRequired": true,
    "retention": "7_years",
    "encryption": "AES256_user_key"
  }
}
```

### 5.3 On-Device AI for Intent Processing

**Use Case:** Process transaction confirmations and extract structured data

**Approach:**
- Small Language Model (SLM) running on-device (e.g., Gemma Nano)
- Extracts transaction details from SMS/email/app notifications
- Populates Transaction Evidence document
- No data leaves device until user explicitly stores in Digilocker

**Example:**
```
Input (SMS): "Your payment of Rs 10000 to Groww via Razorpay is successful. Ref: RZP_9983"

AI Processing:
- Amount: 10000
- Currency: INR
- Merchant: Groww
- Gateway: Razorpay
- Reference: RZP_9983
- Timestamp: [extracted from message metadata]

Output: Structured JSON for Digilocker
```

---

## 6. Security & Privacy Architecture

### 6.1 Private Spaces in Digilocker

**Threat Model:**
- **Risk:** Digilocker compromise exposes all transaction evidence
- **Mitigation:** Private Spaces with user-controlled encryption

**Implementation:**
```
Private Space Architecture:
1. User sets up Private Space with passphrase/biometric
2. Space encrypted with key derived from user secret (not server-known)
3. Transaction evidence stored in Private Space by default
4. Decryption only happens on-device when user accesses
5. DEPA consent flows require explicit Private Space unlock
```

**Trade-offs:**
| Approach | Security | Recovery | UX Complexity |
|----------|----------|----------|---------------|
| Server-side encryption | Medium | Easy | Low |
| User-key encryption | High | Hard | High |
| **Hybrid (Recommended)** | High | Medium | Medium |

**Hybrid Approach:**
- User-key encryption for Private Space
- Recovery via secure multi-party computation (MPC) with trusted contacts
- Similar to social recovery in crypto wallets

### 6.2 DEPA Consent Management

**Granular Consent Controls:**
```
Consent Request Parameters:
- Purpose: "Investment reconciliation"
- Data Fields: ["upi_transaction_id", "timestamp", "amount"]
- Duration: 7 days
- Frequency: One-time access
- Revocable: Yes

User Consent UI:
┌────────────────────────────────────┐
│  Groww wants to access:            │
│  • UPI Transaction ABC123          │
│  • Payment timestamp               │
│  • Amount: ₹10,000                 │
│                                    │
│  Purpose: Verify failed investment │
│  Access: One-time, 7 days          │
│                                    │
│  [ Approve ] [ Deny ]              │
└────────────────────────────────────┘
```

### 6.3 Addressing Single Point of Failure (SPOF)

**Identified Risk:** Digilocker becomes SPOF for transaction evidence

**Mitigations:**

1. **Backup to User-Controlled Storage:**
   - Export evidence to user's cloud storage (encrypted)
   - Automatic backups with user consent

2. **Multi-Issuer Model:**
   - Evidence can also be stored by service providers
   - Cross-verification between Digilocker and provider records

3. **Blockchain Anchoring (Future):**
   - Hash of evidence stored on public blockchain
   - Digilocker holds full document, blockchain provides tamper-proof timestamp
   - **Trade-off:** Adds complexity, evaluate necessity

---

## 7. Strengths, Weaknesses & Mitigations

### 7.1 Strengths

| Strength | Impact |
|----------|--------|
| **Leverages Existing Trust** | Digilocker already has 320M+ users, government-backed |
| **Standardized Digital Receipts** | Reduces reconciliation time from days to minutes |
| **User-Controlled Evidence** | Empowers users in dispute resolution |
| **DPI Alignment** | Builds on Aadhaar, UPI, DEPA stack |
| **Privacy-Preserving** | Private Spaces, consent-based access |

### 7.2 Weaknesses & Mitigations

| Weakness | Mitigation |
|----------|------------|
| **Latency Concerns** | Async storage; DPIx doesn't block main flow. Evidence write happens in background |
| **SPOF (Digilocker)** | Private Space encryption + user backups + multi-issuer model |
| **Privacy Concerns** | End-to-end encryption, consent management, minimal data retention |
| **Adoption Friction** | Gradual rollout; start with high-value use cases (investments, insurance) |
| **Intent Call Unavailability** | Start with existing eKYC providers; advocate for native Aadhaar intent API |

### 7.3 Risk Matrix

```
┌─────────────────────────────────────────────┐
│              IMPACT                         │
│      Low           Medium          High     │
│  ┌──────────────────────────────────────┐   │
│H │            │ SPOF      │ Privacy     │   │
│I │            │ Risk      │ Breach      │   │
│G │────────────┼───────────┼─────────────┤   │
│H │            │ Latency   │ Adoption    │   │
│  │            │ Issues    │ Friction    │   │
│  ├────────────┼───────────┼─────────────┤   │
│M │            │ Intent    │             │   │
│E │            │ Compat.   │             │   │
│D │────────────┼───────────┼─────────────┤   │
│I │ Data       │ UX        │             │   │
│U │ Formatting │ Complexity│             │   │
│M │            │           │             │   │
│  ├────────────┼───────────┼─────────────┤   │
│L │ Minor      │           │             │   │
│O │ Bugs       │           │             │   │
│W │            │           │             │   │
│  └────────────┴───────────┴─────────────┘   │
│           LIKELIHOOD                         │
└─────────────────────────────────────────────┘

Legend:
• SPOF Risk → Mitigate with Private Spaces + backups
• Privacy Breach → Mitigate with E2E encryption + consent
• Adoption Friction → Mitigate with gradual rollout
• Latency → Mitigate with async architecture
```

---

## 8. Implementation Roadmap

### Phase 1: Foundation (Months 1-3)
- [ ] Define Transaction Evidence schema
- [ ] Implement Private Spaces in Digilocker (POC)
- [ ] Build DPIx agent prototype (Android)
- [ ] Integrate with existing eKYC providers (not Aadhaar direct)
- [ ] Test with UPI intent calls

### Phase 2: Integration (Months 4-6)
- [ ] DEPA consent flow implementation
- [ ] On-device AI for receipt parsing (SLM integration)
- [ ] Partner with one service provider (e.g., Groww) for pilot
- [ ] Implement backup and recovery mechanisms

### Phase 3: Scale (Months 7-12)
- [ ] Multi-service provider integration
- [ ] Advanced analytics on journey completion rates
- [ ] Advocate for native Aadhaar auth intent with UIDAI
- [ ] Public launch with marketing campaign

### Phase 4: Ecosystem (Year 2+)
- [ ] Open-source DPIx agent protocol
- [ ] Interoperability with other DPI systems (ONDC, etc.)
- [ ] Blockchain anchoring for evidence tamper-proofing
- [ ] International expansion (adapt to other DPI ecosystems)

---

## 9. Open Questions & Research Areas

### 9.1 Technical
- **Q1:** Should evidence be stored as structured JSON or signed PDFs?
  - **Consideration:** JSON for machine readability, PDFs for legal validity
  - **Proposed:** Dual format - JSON for processing, PDF for archival

- **Q2:** How to handle offline scenarios where Digilocker is unreachable?
  - **Consideration:** DPIx local cache, sync when online
  - **Risk:** Sync conflicts, stale data

- **Q3:** What is the optimal consent duration for different transaction types?
  - **Investment reconciliation:** 7-30 days
  - **Insurance claims:** 90 days - 1 year
  - **E-commerce disputes:** 7-14 days

### 9.2 Policy & Governance
- **Q1:** Who governs the Transaction Evidence schema standards?
  - **Proposed:** NPCI or UIDAI-led working group with industry participation

- **Q2:** What are the legal implications of user-controlled evidence?
  - **Consideration:** Evidence tampering risk, digital signature requirements

- **Q3:** How to ensure cross-border evidence portability?
  - **Future:** Standards alignment with other DPI systems (Singapore, EU)

### 9.3 User Experience
- **Q1:** How to reduce user friction in consent flows?
  - **Research:** Behavioral studies on trust and consent fatigue

- **Q2:** What is the right balance between automatic evidence capture and user control?
  - **Proposed:** Default opt-in with granular opt-out controls

---

## 10. Conclusion

The **Journey Evidence Layer** represents a fundamental shift from treating transactions as discrete events to viewing them as part of a continuous, verifiable user journey. By positioning **Digilocker as the Transaction Vault** and **DPIx as the orchestration agent**, this architecture:

1. **Solves the payment-ownership gap** with verifiable, user-controlled evidence
2. **Builds on existing DPI rails** (Aadhaar, UPI, DEPA) for trust and scale
3. **Preserves user privacy** through Private Spaces and consent management
4. **Enables faster dispute resolution** by making evidence machine-readable and accessible

**Next Steps:**
1. Validate schema design with service providers (Groww, PhonePe, etc.)
2. Prototype DPIx agent with UPI + Digilocker integration
3. Conduct user research on consent UX patterns
4. Build coalition for Aadhaar intent API advocacy

**Strategic Alignment:**
This architecture positions India's DPI stack to solve a global problem: the disconnect between payment and ownership in digital economies. By standardizing transaction evidence as a public good (like identity and payments), we can reduce friction for billions of users.

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| **DPIx** | Digital Dropbox - User-side orchestration agent for intent flows |
| **Journey Evidence Layer** | Architectural pattern for storing verifiable transaction proofs |
| **Transaction Verifiable Credentials** | Signed, structured evidence of payment/service milestones |
| **Private Space** | Encrypted container in Digilocker for sensitive transaction data |
| **Intent Weave** | Chaining of Aadhaar, UPI, and service provider intents via DPIx |
| **DEPA** | Data Empowerment and Protection Architecture - consent layer |

---

## Appendix B: References

1. **vault1.html** - Context Tracing in Digital Finance (Groww case study)
2. **bharatux.html** - Billion Users, Million Journeys framework
3. **dpwr.html** - Power Laws and Platform Economics in DPI context
4. NPCI UPI Specifications 2.0
5. UIDAI eKYC API Documentation
6. DEPA Technical Standards v1.1

---

**Document Control:**
- **Created:** 2026-01-15
- **Last Updated:** 2026-01-15
- **Next Review:** 2026-02-15
- **Stakeholders:** UIDAI Design Team, NPCI, Service Providers
- **Classification:** Internal - Draft for Review

