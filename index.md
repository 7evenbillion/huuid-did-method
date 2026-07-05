---
layout: default
title: did:huuid Method Specification
---

# did:huuid Method Specification

**Version:** 0.1.0-draft  
**Status:** Draft  
**Author:** HUUID Protocol Working Group  
**Contact:** josephtdnarnor@gmail.com  
**License:** Open. No intellectual property restrictions. Free to implement.

## Abstract

The `did:huuid` DID method is a health-domain decentralized identifier 
protocol. It maps a cryptographically-anchored patient identifier to health 
record endpoints across disparate clinical systems globally. HUUID functions 
as the "DNS of Health" — resolving patient identity without storing medical 
data.

## Method Syntax

The `did:huuid` method syntax conforms to the W3C DID Core specification.

    did:huuid:[country-code]:[base58-encoded-public-key-hash]

    Example: did:huuid:gh:7X29ALPHAxyz4Kf9mR2vNbQs

Components:
- `did` — W3C DID scheme prefix (fixed)
- `huuid` — Method name (fixed)
- `country-code` — ISO 3166-1 alpha-2 country code of issuing node
- `base58-encoded-public-key-hash` — Base58Check-encoded SHA-256 hash of patient public key

## DID Operations (CRUD)

### Create

A new `did:huuid` identifier is created by an authorized HUUID Node Operator 
(an accredited health facility or national health authority). Creation requires:

1. Patient biometric enrollment at an authorized facility
2. Generation of an Ed25519 keypair — private key stored on patient device only
3. Construction of a DID Document conforming to the schema below
4. Submission to the HUUID Resolution API for deduplication and activation

Endpoint: `POST https://resolver.huuid.health/1.0/identifiers`  
Authorization: Facility certificate (Ed25519 signed JWT)

### Read (Resolve)

A `did:huuid` identifier is resolved by querying the HUUID Resolution API.

    GET https://resolver.huuid.health/1.0/identifiers/{did}

Required headers:
- `Authorization: Bearer {facility-JWT}`
- `X-HUUID-Purpose: Treatment | Administrative | Emergency`
- `X-HUUID-Facility: {facility-DID}`
- `X-HUUID-Request-ID: {uuid-v4}`

The resolver returns a W3C DID Resolution Result containing the DID Document 
and resolution metadata. Every resolution is logged in an immutable audit trail 
before the response is returned.

### Update

A `did:huuid` DID Document may be updated by the issuing Node Operator to:
- Add or remove service endpoints (health record pointers)
- Update the verification method (key rotation)
- Modify consent scope on registered endpoints

Endpoint: `PATCH https://resolver.huuid.health/1.0/identifiers/{did}`  
Authorization: Patient private key signature + issuing facility certificate

Updates are versioned. All previous versions are retained in the audit log.

### Deactivate

A `did:huuid` identifier is deactivated (revoked) by setting its status field 
to `revoked`. Deactivation is immediate and broadcast to all registered 
facilities. A deactivated DID returns HTTP 410 on resolution attempts.

Deactivation is triggered by:
- Patient request (loss of card or device)
- Issuing facility request (confirmed identity fraud)
- Root Authority action (compromised facility certificate)

Endpoint: `DELETE https://resolver.huuid.health/1.0/identifiers/{did}`  
Authorization: Patient private key signature OR Root Authority certificate

Deactivated DIDs are never deleted. The record is permanently retained for 
audit purposes with status `revoked`.

## DID Document Schema

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://huuid.health/contexts/v1"
  ],
  "id": "did:huuid:gh:7X29ALPHAxyz4Kf9mR2vNbQs",
  "verificationMethod": [{
    "id": "did:huuid:gh:7X29ALPHAxyz4Kf9mR2vNbQs#key-1",
    "type": "Ed25519VerificationKey2020",
    "controller": "did:huuid:gh:7X29ALPHAxyz4Kf9mR2vNbQs",
    "publicKeyMultibase": "z6MkhaXgBZDvotDkL..."
  }],
  "authentication": [
    "did:huuid:gh:7X29ALPHAxyz4Kf9mR2vNbQs#key-1"
  ],
  "service": [{
    "id": "did:huuid:gh:7X29ALPHAxyz4Kf9mR2vNbQs#record-korlebu",
    "type": "HUUIDHealthRecord",
    "serviceEndpoint": "https://emr.kbu.gh/api/huuid/records",
    "facilityCode": "GH-KBU-001",
    "consentScope": ["summary", "allergies", "medications"]
  }],
  "huuid:status": "active",
  "huuid:biometricCommitment": {
    "algorithm": "SHA3-256",
    "salt": "a9f3c2...e8d1b4",
    "commitment": "7f2a91...c3e8b2"
  }
}
```

## Resolution Endpoint

    GET https://resolver.huuid.health/1.0/identifiers/{did}

## Security Considerations

**Key management.** The patient's Ed25519 private key never leaves their 
device. The HUUID resolver holds only the public key hash. Key compromise 
requires the patient to present their biometric at an authorized facility 
to revoke the old HUUID and issue a new one.

**Replay attacks.** Every resolver request requires a fresh UUID v4 in the 
`X-HUUID-Request-ID` header. Duplicate request IDs within 24 hours are 
rejected with HTTP 409.

**Biometric commitment security.** Biometric templates are hashed using 
SHA3-256 with a per-patient random salt (HKDF-derived). The raw biometric 
is never stored by the HUUID resolver. The salted commitment prevents 
precomputation attacks against known sensor formats.

**JWT expiry.** Facility JWTs have a maximum validity window of 300 seconds. 
Any token with `exp - iat > 300` is rejected. This prevents stolen tokens 
from being replayed.

**Break-Glass abuse prevention.** Emergency access is rate-limited to 10 
triggers per facility per 24-hour window. The 10th trigger suspends the 
facility's emergency access capability pending Root Authority review.

**Certificate transparency.** All Node Operator and Facility certificates 
are published to a public certificate transparency log within 24 hours of 
issuance. Certificates not present in the CT log are rejected by the resolver.

**Tamper-evident audit log.** Every resolver query writes an immutable, 
cryptographically signed audit record before the HTTP response is sent. 
The audit log cannot be modified or deleted by any party, including the 
Root Authority.

**Resolver availability.** The resolver operates as a stateless, horizontally 
scaled service. Clinic-level offline resilience is provided by signed offline 
tokens embedded in patient QR cards, verifiable without internet connectivity 
using the resolver's cached public key.

## Privacy Considerations

**No medical data at rest.** The HUUID resolver stores zero medical records, 
diagnoses, prescriptions, or clinical data. It stores identity pointers only. 
An attacker who compromises the resolver gains access to a directory of 
pointers — not to any patient's medical history.

**Data localization.** Medical records remain at the facility where they were 
created. No record leaves its origin facility without the patient's explicit 
consent, expressed through physical presentation of their HUUID.

**Minimum disclosure.** The resolver returns only the data scopes requested 
and consented to. Emergency access (Break-Glass) is explicitly limited to 
blood type, critical allergies, and current medications — never full records.

**Patient notification.** Every Break-Glass (emergency) access triggers a 
patient notification within 60 seconds via SMS or WhatsApp. Patients are 
never unaware that their data was accessed without their active consent.

**Right to erasure.** Deactivation (revocation) of a HUUID immediately 
prevents all future access. Historical audit records are retained for legal 
compliance but the DID Document and service endpoints are cleared.

**Sensitive health categories.** Mental health, reproductive health, and 
genetic data categories are explicitly excluded from all Break-Glass access 
scopes. Access to these categories requires active patient consent under 
Track A (Physical-Present-Consent) only.

**GDPR and HIPAA posture.** Because the HUUID resolver holds no medical data, 
it does not constitute a covered entity under HIPAA or a data processor of 
special category data under GDPR Article 9. The facilities holding medical 
records bear those compliance obligations independently.

## Status

Full technical specification: HUUID-RESOLUTION-SPEC-v0.1.1  
Active development by the HUUID Protocol Working Group.  
W3C DID Method Registration: github.com/w3c/did-extensions/pull/722
