---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
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
   name: Mark Novak
   org: J.P. Morgan Chase & Co.
   email: mark.f.novak@jpmchase.com

 - ins: M. Richardson
   name: Michael Richardson
   org: Sandelman Software Works
   email: mcr+ietf@sandelman.ca

 - ins: H. Birkholz
   name: Henk Birkholz
   org:  Franhaufer Inst.
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

This document specifies extensions to Enrollment over Secure Transport (EST, {{!RFC7030}}) that realize the Trustworthy Acquisition of Credentials via Remote Attestation (TACRA) architecture {{TACRA}} over EST.
Remote Attestation Procedures (RATS, {{RFC9334}}) are used as an authorization input for workload credential provisioning.
Two modes are defined:

1. Attested Credential Enrollment Mode: a workload submits a PKCS#10 Certificate Signing Request (CSR) and Remote Attestation Evidence. The EST Server is a conduit to a Verifier and to a Credential Authority acting as RATS Relying Party. That Relying Party authorizes issuance based on Attestation Results. The issued credential is cryptographically bound to the CSR key through proof-of-possession and explicit key-binding.
2. Attested Credential Retrieval Mode: a workload submits Remote Attestation Evidence that includes an asymmetric Credential Encryption Key (CEK). Upon successful verification and authorization by the Relying Party (a Secret Vault), the EST Server returns an existing shared public credential bundle (e.g., an X.509 certificate, a WIMSE WIC), and a secret, such as an associated signing key, a bearer token, or a preshared key, encrypted to CEKpub.

These extensions add new EST resources, request/response envelopes, processing rules, and security requirements for replay protection, freshness, key-binding, confidentiality, auditability, and key distribution risk management.
Both modes use a two-leg EST exchange: `attest-initiate`, which returns a Freshness kind and an optional Handle, followed by `attest-enroll` or `attest-retrieve`.

--- middle


# Introduction {#intro}

EST ({{!RFC7030}}) defines an HTTPS-based protocol for certificate enrollment and management, typically between an EST Client and an EST Server acting as an interface to a Credential Authority.
In modern environments (e.g., cloud, containers, confidential computing), workloads often lack pre-provisioned credentials and require issuance based on runtime properties.
Additionally, zero trust environments place additional restrictions limiting which parties have access to secrets and credentials.
This means that EST Clients and servers may not be trusted to handle such restricted information in plaintext.

RATS ({{RFC9334}}) defines an architecture and roles for Remote Attestation.
This document integrates RATS with EST by making the EST Client and EST Server conduits for credential issuance or credential release based on verified attestation, following TACRA {{TACRA}}.
The Attester generates keys, Evidence, and CSRs; the EST Client renders those as EST requests, and the EST Server forwards them to the Verifier and the Relying Party.
EST Server takes responses from the Verifier and the RATS Relying Party and renders them back as EST responses that it sends to the EST Client.

This document defines two complementary modes:

1. Enrollment: issuance of fresh credentials bound to a workload-generated Credential Signing Key or CSK (traditional EST semantics)
2. Retrieval: release of a shared credential bundle to replicated workloads that produce equivalent attestation results, using encryption of secrets to an attester-provided Credential Encryption Key or CEK (credential broker semantics).

In either case, RATS-style Remote Attestation authenticates the workload to the RATS Relying Party.
The EST Client and EST Server carry the TACRA two-phase sequence: `attest-initiate` followed by `attest-enroll` or `attest-retrieve`.
The first leg always returns a Freshness kind; it returns a Handle only when that kind requires one ({{INTERACTION-MODELS}}, Section 10 of {{RFC9334}}, {{ATTESTATION-FRESHNESS}}).


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## RATS Entities

* Attester: the workload instance producing Evidence
* Verifier: the component that appraises Evidence and produces Attestation Results
* RATS Relying Party (RRP): consumes Attestation Results to make an authorization decision; in the context of this document, the authorization decision pertains to issuing new or releasing existing credentials to the Attester
* RATS-Unaware Relying Party (RUP): authenticates the Attester using credentials issued by or retrieved from the RRP

## EST Entities

* EST Client: the protocol participant that renders Attester payloads as EST requests and EST responses as payloads back to the Attester; the TACRA Credential Acquisition System (CAS) Client for EST. It is a conduit in both directions. It does not generate keys, Evidence, or CSRs, and it does not decrypt retrieved secrets.
* EST Server: the server exposing EST resources; the TACRA CAS Server for EST. It is a conduit to the Verifier and to a RATS Relying Party (a Credential Authority or a Secret Vault). It forwards Handles, Evidence, Attestation Results, CSRs, and encrypted credential material. It does not originate Freshness, appraise Evidence or Attestation Results, authorize issuance or release, or hold plaintext secrets.

Both EST Client and EST Server have no RATS role. They MUST NOT appraise Evidence or Attestation Results, MUST NOT mint `present-nonce` or `present-epoch` Handle values, MUST NOT generate Evidence, keys, or CSRs, and MUST NOT substitute their own authorization decision for that of the Relying Party.

## Artifacts

* Evidence: attestation evidence produced by the Attester
* Attestation Results: RATS Verifier output
* CSR: PKCS#10 certification request (DER-encoded)
* PoP: proof-of-possession of the private key corresponding to the CSR public key
* CEK: Credential Encryption Key; when used, CEKpub is carried in Evidence, the response is encrypted to CEKpub
* Credential Bundle: container that may include an X.509 chain, WIMSE WIC(s), and optionally a signing key and metadata
* Freshness kind: the recency method returned by `attest-initiate`; can be one of `absent-timestamp`, `absent-none`, `absent-epoch`, `present-nonce`, or `present-epoch` ({{INTERACTION-MODELS}}, Section 10 of {{RFC9334}})
* Handle: a freshness information element included in Evidence when the Freshness kind is `present-nonce` or `present-epoch` ({{INTERACTION-MODELS}})

The EST Client and EST Server forward a Handle obtained from the Verifier or the RATS Relying Party; they do not generate `present-*` Handle values.


# Architecture

The mechanisms described below work equally well in both Passport and Background Check RATS modes.
Only the Passport mode is illustrated.
Differences between the two modes are immaterial for the EST encoding; they matter for who originates a Handle and who contacts the Verifier to obtain Attestation Results.

Both modes always use two EST legs.
The first leg is `attest-initiate`.
The EST Client carries that call; it does not choose the Freshness kind.
The Attester obtains the Freshness kind and, when the kind is `present-nonce` or `present-epoch`, a Handle, then generates Evidence accordingly.
It is a RATS challenge/response only for those present kinds.
For `absent-timestamp`, `absent-none`, and `absent-epoch`, `attest-initiate` still occurs; the response carries no Handle.

How that response is obtained through the EST Server is specified in {{attest-initiate}}.

## Attested Enrollment Mode (Credential Authority-mediated issuance)

* Attester, through the EST Client, performs `attest-initiate` and obtains a Freshness kind and, if any, a Handle
* Attester generates a Credential Signing Key (CSK) keypair and CSR, and produces Evidence bound to the CSR and to the returned Freshness
* Attester, through the EST Client, submits CSR + Evidence to `attest-enroll`
* EST Server forwards Evidence to the Verifier and obtains Attestation Results
* EST Server forwards the CSR and Attestation Results to the Relying Party (Credential Authority), which verifies PoP and authorizes issuance
* EST Server returns the issued credential in an EST enrollment response

~~~~ ascii-art
{::include attested_enrollment.txt}
~~~~
{: #fig-enroll title="Attested Enrollment Mode (Passport)"}

1. Attester initiates credential acquisition (`attest-initiate`)
2. EST Client forwards `attest-initiate` to the EST Server
3. EST Server obtains the Freshness kind, and a Handle if any, from the Verifier or the Relying Party and forwards that result ({{attest-initiate}}). It MUST NOT generate `present-nonce` or `present-epoch` values. The figure shows the Verifier-originated case. The Freshness kind and Handle (if any) are returned to the Attester via the EST Client
4. Attester generates Evidence and CSR, bound to the returned Freshness, and sends them to the EST Client
5. EST Client forwards the Evidence and CSR to the EST Server (`attest-enroll`)
6. EST Server forwards the Evidence to the Verifier and obtains Attestation Results
7. EST Server forwards the CSR and Attestation Results to the Credential Authority and returns the newly issued Credential to the Attester via the EST Client

## Attested Retrieval Mode (Secret Vault Broker)

* Attester, through the EST Client, performs `attest-initiate` and obtains a Freshness kind and, if any, a Handle
* Attester generates an asymmetric CEK keypair and produces Evidence that includes CEKpub and is bound to the returned Freshness
* Attester, through the EST Client, submits Evidence to `attest-retrieve`
* EST Server forwards the Evidence to the Verifier and obtains Attestation Results, which include CEKpub from the Evidence
* EST Server forwards the Attestation Results to the Secret Vault (the Relying Party)
* The Secret Vault identifies the Attester from the Attestation Results, retrieves the matching key or credential, and encrypts it to CEKpub before returning the encrypted blob to the EST Server
* EST Server returns that result to the EST Client
* EST Client returns the encrypted credential to the Attester, which decrypts it with CEKpri

~~~~ ascii-art
{::include attested_retrieval.txt}
~~~~
{: #fig-retrieve title="Attested Retrieval Mode (Passport)"}

1. Attester initiates credential acquisition (`attest-initiate`)
2. EST Client forwards `attest-initiate` to the EST Server
3. EST Server obtains the Freshness kind, and a Handle if any, from the Verifier or the Relying Party and forwards that result ({{attest-initiate}}). It MUST NOT generate `present-nonce` or `present-epoch` values. The figure shows the Verifier-originated case. The Freshness kind and Handle (if any) are returned to the Attester via the EST Client
4. Attester generates Evidence (including CEKpub), bound to the returned Freshness, and sends it to the EST Client
5. EST Client forwards the Evidence to the EST Server (`attest-retrieve`)
6. EST Server forwards the Evidence to the Verifier and obtains Attestation Results, which also include CEKpub
7. EST Server forwards the Attestation Results to the Secret Vault and returns the credentials and associated secrets, encrypted to CEKpub, to the Attester via the EST Client


# Protocol Overview

These extensions define new resources under the existing EST “/.well-known/” prefix:

* `/.well-known/est/attest-initiate`
* `/.well-known/est/attest-enroll`
* `/.well-known/est/attest-retrieve`

These resources are used in addition to, not in place of, existing EST resources.
They carry TACRA's two-phase sequence over EST: `attest-initiate` corresponds to Initiate-Credential-Acquisition; `attest-enroll` and `attest-retrieve` correspond to Enroll-Credential and Retrieve-Credential {{TACRA}}.
EST need not be nonce-based; it MUST be able to carry `attest-initiate`.

All exchanges in this document MUST use HTTPS as required by RFC 7030.
Server authentication via TLS is REQUIRED.
Attester authentication for purposes of credential acquisition is performed through the remote attestation-driven mechanisms defined herein (and optionally additional mechanisms).


# Resources and Methods

## attest-initiate {#attest-initiate}

The first leg of both Attested Enrollment Mode and Attested Credential Retrieval Mode.

When the EST Client's configuration already determines that no Handle is required — Freshness kind `absent-timestamp`, `absent-none`, or `absent-epoch` — the EST Client MAY complete `attest-initiate` locally and MUST NOT send a request to the EST Server.
Otherwise `attest-initiate` is a GET with no request body and no query parameters.

The EST Client forwards the response to the Attester.
The Attester receives whatever Freshness kind, Handle (if any), and constraints the response contains, and acts accordingly: it produces Evidence as that kind requires, then the EST Client POSTs `attest-enroll` or `attest-retrieve` as indicated by the Attester.

The response to `attest-initiate` is a Freshness kind and parameters instructing the Attester on which ciphers, etc. to use.
It returns a Handle only when the Freshness kind is `present-nonce` or `present-epoch`.
For `absent-timestamp`, `absent-none`, and `absent-epoch`, the response carries no Handle.

* Method: GET
* Success: 200 OK
* Response: AttestationInitiateResponse

### Obtaining the Freshness kind and Handle

The EST Client does not choose the Freshness kind.
The EST Server does not originate it either.

The EST Server is a conduit: on GET `attest-initiate` it obtains the Freshness kind, and a Handle if any, from the Verifier or the Relying Party identified by deployment configuration, and returns that result.
That configuration identifies the Verifier and the Relying Party (Credential Authority or Secret Vault) whose freshness requirements apply.
The Freshness kind then determines whether a Handle is returned and who originates it:

* `absent-timestamp`, `absent-none`, or `absent-epoch`: the EST Server returns that kind and MUST NOT include a Handle. It need not contact the Verifier or the Relying Party on this leg. An EST Client MAY complete `attest-initiate` locally, without contacting the EST Server, when its own configuration already determines one of these absent kinds.
* `present-nonce` or `present-epoch`: the EST Server MUST obtain the Handle from the Verifier or the Relying Party, as indicated by that configuration, and MUST return that Handle with the kind. When the Handle is Verifier-originated, the EST Server fetches it from the Verifier and forwards it. When the Handle is Relying-Party-originated, the EST Server fetches it from the Credential Authority or Secret Vault and forwards it. The EST Server MUST NOT generate `present-nonce` or `present-epoch` values itself.

The EST Client always carries `attest-initiate` first — locally when an absent kind is already configured, otherwise as GET — then carries `attest-enroll` or `attest-retrieve` as indicated by the Attester.
If `attest-enroll` or `attest-retrieve` fails because a `present-epoch` marker has moved, or because a `present-nonce` is no longer valid, the Attester retries from `attest-initiate` through the EST Client.

## attest-enroll (POST)

Attested enrollment request (CSR + Evidence bound to the Freshness returned by the preceding `attest-initiate`).

* Method: POST
* Success: 200 OK with an EST enrollment response body, as for simpleenroll in {{RFC7030}}
* Request: AttestedEnrollmentRequest

## attest-retrieve (POST)

Attested credential retrieval request (Evidence includes CEKpub and is bound to the Freshness returned by the preceding `attest-initiate`).

* Method: POST
* Success: 200 OK
* Response: CredentialBundle


# Media Types and Encodings

This document defines abstract message structures. Implementations MUST support at least one interoperable encoding. Two encoding families are permitted:

* CBOR-based envelopes (recommended for compactness), using a to-be-registered media type ({{iana}}).
* JSON-based envelopes (for debugging and ecosystems standardized on JOSE), using a to-be-registered media type.

A server MUST indicate supported request and response media types via the HTTP Content-Type and Accept headers. Clients MUST send a supported media type.

Note: Evidence and Endorsements formats are intentionally opaque to this document; they are carried as byte strings.


# Common Structures

## AttestationInitiateResponse

Fields:

TODO: validate everything below

* `freshness_kind` (string, REQUIRED): one of
    * `absent-timestamp`: stamp Evidence from a trusted clock ({{RFC9334}}, Section 10.1)
    * `absent-none`: no freshness claim
    * `absent-epoch`: embed an epoch marker already held locally
    * `present-nonce`: single-use Handle; embed it in Evidence
    * `present-epoch`: current epoch marker as Handle; embed it in Evidence; retry `attest-initiate` if the epoch moved
* `handle` (bytes): REQUIRED when `freshness_kind` is `present-nonce` or `present-epoch`; MUST be absent otherwise. This is the Handle from {{INTERACTION-MODELS}}, obtained from the Verifier or the Relying Party as specified in {{attest-initiate}}.
* `expires_in` (integer, OPTIONAL): seconds until a `present-nonce` or `present-epoch` Handle is no longer valid
* `max_age` (integer, OPTIONAL): for `absent-timestamp`, the maximum Evidence age in seconds acceptable to the Verifier or Relying Party
* `acceptable_evidence` (array): identifiers for evidence formats
* `required_bindings` (array): required binding mechanisms for the indicated mode
* `acceptable_cek` (array, only when `mode` is `retrieve`): acceptable CEK algorithms/suites
* `mode` (string, OPTIONAL): `enroll` or `retrieve`; the Attester produces an enrollment or retrieval payload accordingly, which the EST Client POSTs as `attest-enroll` or `attest-retrieve`

When `freshness_kind` is `present-nonce`, the originator of the Handle MUST ensure uniqueness within a replay window.
The Handle originator MUST correlate a subsequent `attest-enroll` or `attest-retrieve` with the Handle it returned to `attest-initiate` for this session.
Appraisal of whether Evidence is bound to a still-valid Handle is performed by the Verifier.


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

On success, the EST Server returns credentials as in EST enrollment, consistent with RFC 7030 for simpleenroll.
This document does not change the EST enrollment response formats; it only defines new request paths and authorization inputs.

### Key Binding and PoP Requirements

Attested Enrollment Mode MUST provide:

TODO: validate everything below

1. PoP for CSR Key: The Credential Authority MUST verify that the requester possesses the private key corresponding to the CSR public key, using the CSR’s standard proof mechanisms or an equivalent channel-bound PoP defined by profile.
2. Evidence-to-CSR Binding: The request MUST include a binding that prevents substitution of a different CSR by an attacker reusing Evidence. The Verifier MUST attest to the integrity of that binding.
3. Evidence-to-Freshness Binding: Evidence MUST be bound to the Freshness returned by `attest-initiate`: the Handle when `freshness_kind` is `present-nonce` or `present-epoch`; a trusted-clock stamp when `absent-timestamp`; a locally held epoch marker when `absent-epoch`; nothing additional when `absent-none`. The Verifier MUST attest to the integrity of that binding.

At least one of the following binding mechanisms MUST be implemented by the Attester (to produce the binding) and verified by the Verifier:

* CSR Hash Claim Binding: Evidence (as appraised by the Verifier) contains a claim equal to H(csr_der) (hash of the CSR DER), and the verifier attests to its integrity.
* Public Key Thumbprint Binding: Evidence contains a claim equal to a thumbprint of the CSR SubjectPublicKeyInfo (SPKI).
* Key Certification Binding (Hardware-backed): Evidence contains a verifiable statement that the CSR key is resident in protected hardware/TEE and corresponds to the CSR public key.

The binding object MUST indicate which method is used and include any required identifiers (e.g., hash algorithm ID).

### Enrollment Server Processing

Upon receiving AttestedEnrollmentRequest, the EST Server MUST:

1. Validate message syntax, media type, and size limits.
2. Correlate the request with the preceding `attest-initiate` for this session: when a Handle was returned, `handle` MUST match it; when `freshness_kind` is `absent-timestamp`, `absent-none`, or `absent-epoch`, no Handle is present.
3. Forward evidence (+ endorsements if provided) to a Verifier and obtain Attestation Results. The Verifier appraises Evidence, including Freshness and Evidence-to-CSR binding. The EST Server MUST NOT appraise Evidence itself.
4. Forward the CSR and Attestation Results to the Relying Party (Credential Authority), which verifies PoP and applies authorization policy mapping Attestation Results to:
    * whether issuance is permitted
    * issuance profile (subject/SAN constraints, EKU, validity, etc.)
5. Return the enrollment response produced by the Credential Authority.

In the Background Check model, the EST Server forwards Evidence to the Credential Authority and need not obtain Attestation Results itself; the Relying Party contacts the Verifier.

The EST Server MUST NOT appraise, modify, or replace Attestation Results.
The Credential Authority MUST NOT mint identities (e.g., DNS names) beyond what policy explicitly allows for the attested identity context.

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

Evidence, as appraised by the Verifier, MUST integrity-protect:

* the Freshness returned by `attest-initiate`: the Handle when `freshness_kind` is `present-nonce` or `present-epoch`; a trusted-clock stamp when `absent-timestamp`; a locally held epoch marker when `absent-epoch`; nothing additional when `absent-none`, and
* a claim conveying CEKpub or a stable thumbprint identifier for CEKpub.

The EST Server MUST forward Evidence to the Verifier. The Verifier MUST reject Evidence that does not integrity-protect these bindings, and the Relying Party MUST deny release when Attestation Results do not confirm them.

### Credential Group ID Determination

TODO: validate everything below

The holder of the requested credential — the Secret Vault — MUST be able to map the Attestation Results from the Verifier to the credential "group ID" that the Attester is expecting. The mapping MUST be stable for “replica” workloads intended to receive identical credentials and MUST vary across security domains/tenants/profiles. The EST Client and EST Server MUST NOT compute this mapping from raw Evidence, and SHOULD NOT be the party that holds the plaintext secret.

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

1. Validate syntax and size limits, and correlate the request with the preceding `attest-initiate` for this session as in enrollment processing.
2. Forward Evidence to a Verifier and obtain Attestation Results. The EST Server MUST NOT appraise Evidence itself.
3. Forward the Attestation Results to the Secret Vault, which:
    1. Computes group_id using policy.
    2. Authorizes the requested credential type/profile for group_id.
    3. Fetches the credential bundle corresponding to (group_id, profile, credential_type).
    4. Encrypts the bundle to CEKpub with AAD binding including the Handle (if any) and group_id.
4. Return the EncryptedCredentialBundle produced by the Secret Vault.

In the Background Check model, the EST Server forwards Evidence to the Secret Vault and need not obtain Attestation Results itself; the Relying Party contacts the Verifier.

The EST Server MUST NOT appraise, modify, or replace Attestation Results.
It MUST NOT mint `present-nonce` or `present-epoch` Handles.
It SHOULD NOT see the plaintext secret. A deployment in which the EST Server unwraps a vault secret and re-encrypts to CEKpub is possible but discouraged; the Secret Vault remains the Relying Party that authorizes release.


# Error Handling

Servers SHOULD reuse HTTP status codes consistent with RFC 7030 and provide a machine-readable error body for these resources.

Recommended error conditions:

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

* Freshness and Replay Protection: `attest-initiate` MUST return a Freshness kind. For `present-nonce`, the Handle originator MUST provide an unpredictable Handle and MUST reject reuse within an enforcement window. For `present-epoch`, the originator MUST reject a Handle that no longer names the current epoch. Evidence MUST be bound to that Handle when one is present, or produced according to the absent kind (`absent-timestamp`, `absent-epoch`, or `absent-none`) when one is not. The EST Server MUST NOT generate `present-nonce` or `present-epoch` values; it forwards a Handle obtained from the Verifier or the Relying Party and correlates the second-leg request with that session.
* Freshness origin: RFC 9334 Section 10 does not assign a single issuer. A nonce Handle MAY come from the Verifier or from the Relying Party. A timestamp comes from the Attester's trusted clock. An epoch marker MAY already be held locally (`absent-epoch`) or be returned as a Handle (`present-epoch`). The EST Server only conveys the kind; it does not originate it.
* Verifier Trust: The EST Server does not appraise Evidence. The channel from the EST Server to the Verifier MUST provide integrity, authenticity, and replay protection.
* Attestation Results SHOULD be audience-restricted to the Relying Party (Credential Authority or Secret Vault), not to the EST Server.
* DoS Considerations: Evidence appraisal can be expensive; the EST Server SHOULD enforce size limits and rate limits, and MAY cache Verifier results where the Verifier permits and where doing so does not weaken freshness.
* Downgrade Resistance: If both classic EST and attested resources exist, the Relying Party MUST be able to require attestation and MUST NOT silently fall back to weaker modes. The EST Server MUST NOT substitute a classic EST enrollment or retrieval for an attested request.

## Specific to Attested Enrollment Mode

* Key Substitution: Prevented only if Evidence-to-CSR binding is mandatory and verified by the Verifier, and PoP is verified by the Credential Authority. Implementations MUST NOT treat “attestation success” as sufficient absent key-binding + PoP.
* Identity Over-Issuance: The Credential Authority's policies must constrain subject/SAN issuance to attested identity context.

## Specific to Attested Retrieval Mode

* Shared Signing Key Distribution: If the credential bundle includes a private signing key shared across replicas, compromise of one replica compromises the group. This mode SHOULD be restricted to environments where unwrap and key use are strongly protected.
* Non-Exportability Requirements: Deployments that transport a signing key SHOULD require Evidence to attest that CEKpri is non-exportable and that decryption/unwrapping occurs only within an approved protected environment (e.g., TEE/TPM-sealed key usage).
* Attribution: Shared keys eliminate per-instance attribution. If accountability is required, consider per-instance keys with identical identity claims, or a centralized signing service.

# IANA Considerations {#iana}

TODO: Treat as early draft, revisit later

This document requests registrations for:

* New EST well-known paths (if applicable under EST registries).
* Media types for:
    * AttestationInitiateResponse
    * AttestedEnrollmentRequest
    * AttestedRetrievalRequest
    * EncryptedCredentialBundle
* Registry of acceptable_evidence identifiers and credential_type identifiers (if not reused from existing registries).

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
