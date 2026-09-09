---
title: Remote Attestation Extensions for EST
category: info

docname: draft-novak-lamps-tacra-est
submissiontype: IETF
number:
date:
# consensus: true
v: 3
area: "Security"
workgroup: "Limited Additional Mechanisms for PKIX and SMIME"
keyword:
 - trustworthy workload identity
 - remote attestation
 - credential enrollment
 - credential retrieval
venue:
  group: "Limited Additional Mechanisms for PKIX and SMIME"
  type: "Working Group"
  mail: "spasm@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/spasm/"
  github: "TheBankster/lamps-tacra-est"
  latest: "https://TheBankster.github.io/lamps-tacra-est/draft-novak-lamps-tacra-est.html"

author:
 - ins: M. Novak
   name: Mark F. Novak
   org: J.P. Morgan Chase & Co.
   email: mark.f.novak@jpmchase.com

 - ins: M. Richardson
   name: Michael Richardson
   org: Sandelman Software Works
   email: mcr+ietf@sandelman.ca

 - ins: H. Birkholz
   name: Henk Birkholz
   org:  Fraunhofer Inst.
   email: Henk.Birkholz@ietf.contact

normative:
  RFC7030: EST
  RFC9334: RATS

informative:
  INTERACTION-MODELS: I-D.ietf-rats-reference-interaction-models
  ATTESTATION-FRESHNESS: I-D.ietf-lamps-attestation-freshness
  TACRA:
    target: https://TheBankster.github.io/rats-tacra/draft-novak-rats-tacra.html
    title: Trustworthy Acquisition of Credentials via Remote Attestation
    author:
      - ins: M. Novak
        name: Mark F. Novak
      - ins: M. Richardson
        name: Michael Richardson
      - ins: H. Birkholz
        name: Henk Birkholz
  TWISIGDef:
    -: TWISIGDef
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Definitions.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Definitions
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  TWISIGReq:
    -: TWISIGReq
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Requirements.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Requirements
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG

...

--- abstract

This document specifies extensions to Enrollment over Secure Transport (EST) that realize the Trustworthy Acquisition of Credentials via Remote Attestation (TACRA) architecture {{TACRA}} over EST.
Remote Attestation Procedures (RATS) are used as an authorization input for workload credential provisioning.
Two modes are defined:

1. Enrollment: the Attester submits a PKCS#10 CSR and Evidence. A Credential Authority, as RATS Relying Party, authorizes issuance from Attestation Results and binds the credential to the CSR key.
2. Retrieval: the Attester submits Evidence that includes a Credential Encryption Key (CEK). A Secret Vault, as RATS Relying Party, releases an existing credential bundle with secrets encrypted to CEKpub.

These extensions add EST resources for a two-leg exchange: `attest-initiate`, then either `attest-enroll` or `attest-retrieve`.

--- middle


# Introduction {#intro}

EST ({{!RFC7030}}) defines an HTTPS-based protocol for certificate enrollment and management, typically between an EST Client and an EST Server acting as an interface to a Credential Authority.
In modern environments (e.g., cloud, containers, confidential computing), workloads often lack pre-provisioned credentials and require issuance based on runtime properties.
Additionally, zero trust environments place additional restrictions limiting which parties have access to secrets and credentials.
This means that EST Clients and Servers may not be trusted to handle such restricted information in plaintext.

RATS ({{!RFC9334}}) defines an architecture and roles for Remote Attestation.
This document maps the TACRA {{TACRA}} architecture onto EST: the Attester generates keys, Evidence, and CSRs; the EST Client and EST Server are conduits that carry them to a Verifier and a RATS Relying Party (RRP) ({{INTERACTION-MODELS}}, Section 10 of {{RFC9334}}, {{ATTESTATION-FRESHNESS}}).
The RRP in this case is either a Secret Vault, where pre-provisioned credentials are located, or a Credential Authority capable of creating new ones.

Two modes are defined:

1. Enrollment: issuance of a fresh credential bound to a workload-generated Credential Signing Key (CSK)
2. Retrieval: release of a shared credential bundle, with secrets encrypted to an attester-provided Credential Encryption Key (CEK)


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## RATS Entities

* Attester: the workload instance producing Evidence
* Verifier: the component that appraises Evidence and produces Attestation Results
* RATS Relying Party (RRP): consumes Attestation Results to make an authorization decision; in the context of this document, the authorization decision pertains to issuing new or releasing existing credentials to the Attester

## TACRA Entities

* CAS: Credential Acquisition System
* RATS-Unaware Relying Party (RUP): authenticates the Attester using credentials issued by or retrieved from the RRP

## EST Entities

* EST Client: TACRA CAS Client for EST. Renders Attester payloads as EST requests, and EST Server responses as payloads to the Attester.
* EST Server: TACRA CAS Server for EST. Forwards Handles, Evidence, Attestation Results, CSRs, and encrypted material between the EST Client, the Verifier, and a Relying Party (Credential Authority or Secret Vault).

## Artifacts

* Evidence: attestation evidence produced by the Attester
* Attestation Results: RATS Verifier output
* CSR: PKCS#10 certification request (DER-encoded)
* PoP: proof-of-possession of the private key corresponding to the CSR public key
* CEK: Credential Encryption Key; when used, CEKpub is carried in Evidence, the response is encrypted to CEKpub
* Credential Bundle: container that may include an X.509 chain, WIMSE WIC(s), and optionally a signing key and metadata
* Freshness kind: the recency method returned by `attest-initiate`; can be one of `absent-timestamp`, `absent-none`, `absent-epoch`, `present-nonce`, or `present-epoch` ({{INTERACTION-MODELS}}, Section 10 of {{RFC9334}})
* Handle: freshness element included in Evidence when the kind is `present-nonce` or `present-epoch` ({{INTERACTION-MODELS}})


# Architecture

The EST encoding is the same in Passport and Background Check modes; the modes differ in who originates a Freshness Handle and whether the EST Server contacts the Verifier and then the RATS Relying Party (Passport) or the RATS Relying Party directly (Background Check).
Only Passport mode is illustrated.

Both modes use two EST legs: `attest-initiate`, then `attest-enroll` or `attest-retrieve` ({{attest-initiate}}).

Neither EST Client nor EST Server has a RATS role.
Both are subject to the following restrictions:
* MUST NOT generate keys, Evidence, or CSRs
* MUST NOT mint `present-nonce` or `present-epoch` Handles
* MUST NOT appraise Evidence or Attestation Results
* MUST NOT authorize credential issuance or release
* MUST NOT hold plaintext secrets.

## Attested Enrollment Mode

~~~~ ascii-art
{::include attested_enrollment.txt}
~~~~
{: #fig-enroll title="Attested Enrollment Mode (Passport)"}

1. Attester initiates Remote Attestation
2. EST Client forwards to EST Server (`attest-initiate`)
3. EST Server obtains Freshness from the Verifier or Relying Party and returns it via the EST Client
4. Attester generates CSK, CSR, and Evidence bound to the CSR and Freshness
5. EST Client POSTs CSR and Evidence (`attest-enroll`)
6. EST Server forwards Evidence to the Verifier
7. EST Server forwards CSR and Attestation Results to the Credential Authority, which verifies PoP and authorizes issuance; the credential is returned to the Attester via the EST Client

## Attested Retrieval Mode

~~~~ ascii-art
{::include attested_retrieval.txt}
~~~~
{: #fig-retrieve title="Attested Retrieval Mode (Passport)"}

1. Attester initiates Remote Attestation
2. EST Client forwards to EST Server (`attest-initiate`)
3. EST Server obtains Freshness from the Verifier or Relying Party and returns it via the EST Client
4. Attester generates CEK and Evidence including CEKpub, bound to Freshness
5. EST Client POSTs Evidence (`attest-retrieve`)
6. EST Server forwards Evidence to the Verifier
7. EST Server forwards Attestation Results to the Secret Vault, which encrypts the matching credential to CEKpub; the EST Server, through EST Client, returns that encrypted blob to the Attester, which decrypts it with CEKpri


# Protocol Overview

Three new EST resources are added under the existing EST “/.well-known/” prefix:

1. `/.well-known/est/attest-initiate`
2. `/.well-known/est/attest-enroll`
3. `/.well-known/est/attest-retrieve`

These resources are used in addition to existing EST resources.
They support TACRA Initiate-Credential-Acquisition, Enroll-Credential, and Retrieve-Credential {{TACRA}} commands, encoded with EST semantics.

All exchanges MUST use HTTPS as required by {{RFC7030}}.
Server authentication via TLS is REQUIRED.


# Resources and Methods

## attest-initiate {#attest-initiate}

The first leg of both modes is where the Attester initiates Remote Attestation by invoking the EST Client.

* Method: GET
* Request: None
* Success: 200 OK
* Response: optional Freshness kind

TODO: add additional parameters to the Initiate Response (e.g., acceptable cipher suites)

* If the EST Client is already configured for an absent Freshness kind (`absent-timestamp`, `absent-none`, or `absent-epoch`), it MAY complete `attest-initiate` locally and, in that case, MUST NOT contact the EST Server. Otherwise, `attest-initiate` is a GET with no body and no query parameters.
* The EST Server, if contacted, obtains the Freshness kind, and a Handle if any, from the configured Verifier or Relying Party and returns that result. "Absent" Freshness kinds carry no Handle.
* `present-nonce` and `present-epoch` MUST include a Handle from the party chosen by the EST Server.
* The EST Client forwards the response to the Attester.
* The Attester produces Evidence as the Freshness kind requires.
* The EST Client then POSTs `attest-enroll` or `attest-retrieve` as the Attester indicates.
* If the second leg fails because a `present-epoch` moved or a `present-nonce` is no longer valid, the Attester retries `attest-initiate`.

* Method: GET
* Success: 200 OK
* Response: AttestationInitiateResponse

TODO: Define error code for invalid Freshness

## attest-enroll (POST)

* Method: POST
* Request: AttestedEnrollmentRequest (CSR and Evidence)
* Success: 200 OK
* Response: enrollment response as for simpleenroll in {{RFC7030}}

## attest-retrieve (POST)

* Method: POST
* Request: AttestedRetrievalRequest (Evidence including CEKpub)
* Success: 200 OK
* Response: EncryptedCredentialBundle


# Media Types and Encodings

TODO: discuss envelopes and media types in more detail; verify correctness
Implementations MUST support at least one of CBOR or JSON envelopes, using to-be-registered media types ({{iana}}).
Servers advertise supported types with Content-Type and Accept; clients MUST send a supported type.
Evidence blobs are opaque byte strings.


# Common Structures

## AttestationInitiationResponse

Fields:

TODO: validate everything below

* `freshness_kind` (string, REQUIRED): one of
    * `absent-timestamp`: stamp Evidence from a trusted clock ({{RFC9334}}, Section 10.1)
    * `absent-none`: no freshness claim
    * `absent-epoch`: embed an epoch marker already held locally
    * `present-nonce`: single-use Handle; embed it in Evidence
    * `present-epoch`: current epoch marker as Handle; embed it in Evidence; retry `attest-initiate` if the epoch moved
* `handle` (bytes): REQUIRED for `present-nonce` and `present-epoch`; MUST be absent otherwise ({{attest-initiate}})
* `expires_in` (integer, OPTIONAL): seconds until a `present-nonce` or `present-epoch` Handle is no longer valid
* `max_age` (integer, OPTIONAL): for `absent-timestamp`, the maximum Evidence age in seconds acceptable to the Verifier or Relying Party
* `acceptable_evidence` (array): identifiers for evidence formats
* `required_bindings` (array): required binding mechanisms for the indicated mode
* `acceptable_cek` (array, only when `mode` is `retrieve`): acceptable CEK algorithms/suites
* `mode` (string, REQUIRED): `enroll` or `retrieve`

The Handle originator MUST ensure `present-nonce` uniqueness and MUST correlate the second-leg request with the Handle from this `attest-initiate`.
The Verifier appraises whether Evidence is bound to a still-valid Handle.


# Attested Credential Acquisition Modes

## Credential Enrollment Mode

### Request: AttestedEnrollmentRequest

Fields:

TODO: validate everything below

* freshness_kind (string, REQUIRED): MUST match the preceding `attest-initiate` response
* handle (bytes): REQUIRED when `freshness_kind` is `present-nonce` or `present-epoch`; MUST be absent otherwise. MUST equal the Handle returned by `attest-initiate` when present.
* csr (bytes, REQUIRED): DER-encoded PKCS#10 CSR
* evidence (bytes, REQUIRED): MUST be bound to the Freshness returned by `attest-initiate`, if any
* endorsements (bytes, OPTIONAL)
* binding (object, REQUIRED): declares how the CSR key is bound to Evidence
* profile (string, OPTIONAL): requested issuance profile identifier

### Response (Success)

On success, the EST Server returns the enrollment response produced by the Credential Authority, as for simpleenroll in {{RFC7030}}.

### Key Binding and PoP Requirements

TODO: validate everything below

1. PoP: The Credential Authority MUST verify possession of the CSR private key.
2. Evidence-to-CSR: Evidence MUST bind the CSR so a different CSR cannot be substituted. The Verifier MUST attest to that binding.
3. Evidence-to-Freshness: Evidence MUST be bound to the Freshness from `attest-initiate` ({{attest-initiate}}). The Verifier MUST attest to that binding.

At least one Evidence-to-CSR mechanism MUST be produced by the Attester and verified by the Verifier:

* CSR Hash: Evidence contains H(csr_der)
* Public Key Thumbprint: Evidence contains a thumbprint of the CSR SubjectPublicKeyInfo
* Key Certification: Evidence states that the CSR key is resident in protected hardware/TEE and matches the CSR public key

The binding object MUST name the method and any identifiers (e.g., hash algorithm).

### Enrollment Server Processing

Upon receiving AttestedEnrollmentRequest, the EST Server MUST:

1. Validate syntax, media type, and size limits.
2. Correlate `handle` with the preceding `attest-initiate` for this session, if a Handle was returned.
3. Forward Evidence (and endorsements, if any) to the Verifier and obtain Attestation Results.
4. Forward the CSR and Attestation Results to the Credential Authority, which verifies PoP and authorizes issuance.
5. Return the Credential Authority's enrollment response.

The Credential Authority MUST NOT mint identities (e.g., DNS names) beyond policy for the attested identity context.

## Credential Retrieval Mode

### Request: AttestedRetrievalRequest

Fields:

TODO: validate everything below

* freshness_kind (string, REQUIRED): MUST match the preceding `attest-initiate` response
* handle (bytes): REQUIRED when `freshness_kind` is `present-nonce` or `present-epoch`; MUST be absent otherwise. MUST equal the Handle returned by `attest-initiate` when present.
* evidence (bytes, REQUIRED) -- MUST include CEKpub; MUST be bound to the Freshness returned by `attest-initiate`, if any
* endorsements (bytes, OPTIONAL)
* credential_type (string, OPTIONAL): e.g., x509, wimse-wit, bundle
* profile (string, OPTIONAL): profile identifier -- TODO: is it needed here? Why?

### Evidence-to-CEK Binding

TODO: validate everything below

Evidence MUST integrity-protect the Freshness from `attest-initiate` ({{attest-initiate}}) and a claim conveying CEKpub or a thumbprint of CEKpub.
The Verifier MUST reject Evidence that does not, and the Secret Vault MUST deny release when Attestation Results do not confirm these bindings.

### Credential Group ID Determination

TODO: validate everything below

The Secret Vault MUST map Attestation Results to a credential group ID that is stable for replica workloads and distinct across security domains, tenants, and profiles.

A typical construction is:

group_id = H(attestation_subject \|\| profile \|\| policy_version)

Where attestation_subject is derived from Attestation Results (not raw Evidence) to avoid nonce/freshness variability.

### Response: EncryptedCredentialBundle

TODO: validate everything below

The response MUST be an authenticated-encryption container encrypted to CEKpub. It contains:

* group_id
* `credential_items` (array):
    * X.509 chain (if requested/authorized)
    * WIMSE WIT(s) (if requested/authorized)
    * OPTIONAL: a shared signing key (high risk; see {{security}})
* metadata (optional): validity, refresh hints, rotation epoch, key identifiers
* (implicit or explicit): associated data binding at least {group_id, handle if any, profile, server_id}

Mandatory-to-implement encryption mechanism: The specification MUST choose one baseline.

* CMS EnvelopedData (aligned with EST’s CMS usage), OR
* HPKE (RFC 9180) with a specific required ciphersuite, or
* COSE_Encrypt0 with a required AEAD suite.

TODO: Select exactly one as MUST in the final draft; multiple MAY be supported.

### Retrieval Server Processing

Upon receiving AttestedRetrievalRequest, the EST Server MUST:

1. Validate syntax and size limits, and correlate `handle` with the preceding `attest-initiate` as in enrollment processing.
2. Forward Evidence to the Verifier and obtain Attestation Results.
3. Forward Attestation Results to the Secret Vault, which computes group_id, authorizes, fetches the bundle, and encrypts it to CEKpub
4. Return the EncryptedCredentialBundle produced by the Secret Vault.

A variant in which the EST Server receives a plaintext vault secret and re-encrypts to CEKpub is possible but discouraged. TODO: Elsewere there are "MUST NOT" directives against this.


# Error Handling

Servers SHOULD reuse HTTP status codes from {{RFC7030}} and a machine-readable error body:

* 400 Bad Request: malformed envelope, missing fields
* 401 Unauthorized: missing/invalid authentication required by deployment policy
* 403 Forbidden: attestation failed or policy denies enrollment/retrieval
* 409 Conflict: Handle replay detected, or `present-epoch` marker has moved
* 415 Unsupported Media Type: unsupported encoding
* 429 Too Many Requests: rate limiting
* 500/503: verifier unavailable or internal error

Error bodies MUST NOT leak sensitive attestation details. Servers MAY provide a correlation identifier for debugging.


# Security Considerations {#security}

TODO: Validate everything below

## Common to Both Modes

* Freshness: a nonce Handle MAY come from the Verifier or the Relying Party; a timestamp from the Attester's clock; an epoch marker from a local hold or a returned Handle (Section 10 of {{RFC9334}}). Evidence MUST be bound as required by the kind from `attest-initiate`. The originator MUST reject `present-nonce` reuse and a stale `present-epoch`.
* The channel from EST Server to Verifier MUST provide integrity, authenticity, and replay protection.
* Attestation Results SHOULD be audience-restricted to the Relying Party, not the EST Server.
* The EST Server SHOULD enforce size and rate limits on Evidence.
* If classic EST and attested resources both exist, the Relying Party MUST be able to require attestation. The EST Server MUST NOT substitute a classic EST operation for an attested request.

## Specific to Attested Enrollment Mode

* Key Substitution: attestation success is not sufficient without Evidence-to-CSR binding and PoP.
* Identity Over-Issuance: Credential Authority policy must constrain subject/SAN to the attested identity context.

## Specific to Attested Retrieval Mode

* Shared Signing Key Distribution: If the credential bundle includes a private signing key shared across replicas, compromise of one replica compromises the group. This mode SHOULD be restricted to environments where unwrap and key use are strongly protected.
* Non-Exportability Requirements: Deployments that transport a signing key SHOULD require Evidence to attest that CEKpri is non-exportable and that decryption/unwrapping occurs only within an approved protected environment (e.g., TEE/TPM-sealed key usage).
* Attribution: Shared keys eliminate per-instance attribution. If accountability is required, consider per-instance keys with identical identity claims, or a centralized signing service.

# IANA Considerations {#iana}

TODO: Treat as early draft, revisit later

This document requests registrations for:

* New EST well-known paths (if applicable under EST registries).
* Media types for:
    * AttestationInitiationResponse
    * AttestedEnrollmentRequest
    * AttestedRetrievalRequest
    * EncryptedCredentialBundle
* Registry of acceptable_evidence identifiers and credential_type identifiers (if not reused from existing registries).

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
