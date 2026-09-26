---
title: Remote Attestation Extensions for EST
category: info

docname: draft-novak-lamps-tacra-est-latest
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
  CSR-ATTEST: I-D.ietf-lamps-csr-attestation
  RFC9266: Channel Binding for TLS 1.3
  RFC9180: HPKE
  RFC5652: CMS
  RFC9052: COSE
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
  AMD-SNP-ABI:
    target: https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/56860.pdf
    title: SEV Secure Nested Paging Firmware ABI Specification (Publication 56860)
    author:
      org: Advanced Micro Devices
  TDX-ABI:
    target: https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/documentation.html
    title: Intel Trust Domain Extensions (Intel TDX) Module Architecture Application Binary Interface (ABI) Reference Specification
    author:
      org: Intel Corporation
  TWISIGReq:
    -: TWISIGReq
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Requirements.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group — Requirements
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG

...

--- abstract

This document specifies extensions to Enrollment over Secure Transport (EST) that realize the Trustworthy Acquisition of Credentials via Remote Attestation (TACRA) architecture {{TACRA}}.
Remote Attestation Procedures (RATS) are used as an authorization input for workload credential provisioning.

Two modes are defined:

1. Enrollment: the Attester submits a PKCS#10 CSR and Evidence. A Credential Authority, as RATS Relying Party, authorizes issuance from Attestation Results and binds the credential to the CSR key.
2. Retrieval: the Attester submits Evidence that includes a Credential Encryption Key (CEK). A Secret Vault, as RATS Relying Party, releases an existing credential bundle with secrets encrypted to CEKpub.

These extensions add EST resources for a two-leg exchange: optional `attest-initiate`, followed by either `attest-enroll` or `attest-retrieve`.

This memo is informational and intended to support discussion and implementation experience.
It is not a completed interoperability specification.

--- middle


# Introduction {#intro}

EST ({{!RFC7030}}) defines an HTTPS-based protocol for certificate enrollment and management, typically between an EST Client and an EST Server acting as an interface to a Credential Authority.
In modern environments (e.g., cloud, containers, confidential computing), workloads often lack pre-provisioned credentials and require issuance based on runtime properties.
Zero trust environments place additional restrictions limiting which parties have access to secrets and credentials.
This means that EST Clients and Servers may not be trusted to handle such restricted information in plaintext.

RATS ({{!RFC9334}}) defines an architecture and roles for Remote Attestation.
This document extends EST to match the TACRA {{TACRA}} architecture: the Attester generates keys, Evidence, and CSRs; the EST Client and EST Server are conduits that carry them to a Verifier and a RATS Relying Party (RRP) ({{INTERACTION-MODELS}}, Section 10 of {{RFC9334}}, {{ATTESTATION-FRESHNESS}}).
The RRP in this case is either a Secret Vault, where pre-provisioned credentials are stored, or a Credential Authority capable of creating new ones.

Two modes are defined:

1. Enrollment: issuance of a fresh credential bound to a workload-generated Credential Signing Key (CSK)
2. Retrieval: release of a shared credential bundle, with secrets encrypted to an attester-supplied Credential Encryption Key (CEK)


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
* EST Server: TACRA CAS Server for EST. Forwards Freshness Handles, Evidence, Attestation Results, CSRs, and encrypted material between the EST Client, the Verifier, and a RATS Relying Party (Credential Authority or Secret Vault).

## Artifacts

* Evidence: attestation evidence produced by the Attester
* Attestation Results: RATS Verifier output
* CSR: PKCS#10 certification request
* PoP: proof-of-possession of the private key corresponding to the CSR public key
* CEK: Credential Encryption Key; when used, CEKpub is carried in Evidence, the response is encrypted to CEKpub
* Credential Bundle: container that may include an X.509 chain, WIMSE WIC(s), and optionally a signing key and metadata
* Credential Hint: optional, implementation-defined information supplied by the Attester about the credential it expects. The Credential Authority or Secret Vault MAY use the hint, ignore it, or reject the request. The hint does not authorize issuance or release.
* Freshness Kind: the recency method; can be one of `absent-timestamp`, `absent-none`, `absent-epoch`, `present-nonce`, or `present-epoch` ({{INTERACTION-MODELS}}, Section 10 of {{RFC9334}})
* Freshness Handle: freshness element included in Evidence when Freshness Kind is not `absent-none` ({{INTERACTION-MODELS}})


# Architecture

The EST encoding is the same in Passport and Background Check modes.
The modes differ in 1) who originates a Freshness Handle and 2) whether the EST Server contacts the Verifier and then the RATS Relying Party (Passport) or the RATS Relying Party directly (Background Check).
Only Passport mode is illustrated.

Both modes use two EST legs: optional `attest-initiate`({{attest-initiate}}), followed by either `attest-enroll` ({{attest-enroll}}) or `attest-retrieve` ({{attest-retrieve}}).
Note that `attest-initiate` is optional: in cases where the Attester can know with certainty which Freshness Kind to use (and that the Freshness Kind is not either `present-nonce` or `present-epoch`), whether to ask for new or existing credentials, as well as which ciphers the RRP expects, it can be omitted.

Neither EST Client nor EST Server has a RATS role.
Both are subject to the following restrictions:
* MUST NOT generate keys, Evidence, or CSRs
* MUST NOT mint `present-nonce` or `present-epoch` Freshness Handles
* MUST NOT appraise Evidence or Attestation Results
* MUST NOT authorize credential issuance or release

## Attested Enrollment Mode

~~~~ ascii-art
{::include attested_enrollment.txt}
~~~~
{: #fig-enroll title="Attested Enrollment Mode (Passport)"}

1. Attester initiates Remote Attestation
2. EST Client forwards to EST Server (`attest-initiate`)
3. If Freshness is required: EST Server obtains Freshness from the Verifier or Relying Party and returns it via the EST Client
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
3. If Freshness is required: EST Server obtains Freshness from the Verifier or Relying Party and returns it via the EST Client
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
EST Server authentication via TLS is REQUIRED.


# Resources and Methods

## attest-initiate {#attest-initiate}

The first leg of both modes is where the Attester initiates Remote Attestation by invoking the EST Client.

* Method: GET
* Request: None
* Success: 200 OK
* Response: AttestationInitiationResponse ({{initiation-response}}): Freshness Kind and Handle, Credential Acquisition Mode (`enroll` or `retrieve`), acceptable CSK or CEK algorithms

* If the EST Client already knows all the information the Attester needs to proceed, i.e., it is already configured for an absent Freshness kind (`absent-timestamp`, `absent-none`, or `absent-epoch`), and it knows what Credential Acquisition Mode is expected, and which ciphers to use, it MAY complete `attest-initiate` locally and, in that case, MUST NOT contact the EST Server. Otherwise, `attest-initiate` is a GET with no body and no query parameters.
* The EST Server, if contacted, obtains the Freshness Kind and Handle, if any, from the configured Verifier or Relying Party and returns that result.
* `present-nonce` and `present-epoch` MUST include a Freshness Handle from the party chosen by the EST Server. `absent-*` Freshness Kinds carry no Freshness Handle.
* The EST Client forwards the response to the Attester.
* The Attester produces Evidence as the Freshness Kind requires.
* The EST Client then POSTs `attest-enroll` or `attest-retrieve` as the Attester indicates.
* If the second leg fails because a `present-epoch` moved or a `present-nonce` is no longer valid, the Attester retries `attest-initiate`.

## attest-enroll {#attest-enroll}

* Method: POST
* Request: AttestedEnrollmentRequest (CSR and Evidence)
* Success: 200 OK
* Response: enrollment response as for simpleenroll in {{RFC7030}}

## attest-retrieve {#attest-retrieve}

* Method: POST
* Request: AttestedRetrievalRequest (Evidence including CEKpub)
* Success: 200 OK
* Response: EncryptedCredentialBundle


# Relationship to Other LAMPS Work {#relationship}

Two LAMPS documents cover parts of the exchange defined here.

{{ATTESTATION-FRESHNESS}} defines a nonce request and response for certificate management protocols, including an EST resource at `/.well-known/est/nonce` (GET, or POST with `application/est-attestation-freshness+json`) whose response carries a `nonce` of 8 to 64 bytes and an OPTIONAL `expiry` in seconds (Section 5.1 of {{ATTESTATION-FRESHNESS}}).
`attest-initiate` ({{attest-initiate}}) covers the same first leg for the `present-nonce` Freshness Kind and, in addition, returns the Freshness Kind itself, the Credential Acquisition Mode, the identity of the server the Handle was issued for ({{initiation-response}}), and the acceptable CSK or CEK algorithms, none of which the nonce resource carries.
A `present-nonce` Handle is a nonce in the sense of {{ATTESTATION-FRESHNESS}}, and the requirement of Section 5.1 of that document that the server associate the nonce with the subsequent request applies to `attest-initiate` and its second leg alike.
An EST Server MAY offer both resources.

{{CSR-ATTEST}} conveys Evidence inside the CSR, in the `id-aa-attestation` attribute, and makes the CA/RA responsible for validating the binding between the attestations and the CSR's public key (Section 6.1 of {{CSR-ATTEST}}).
This document conveys Evidence beside the CSR, in the AttestedEnrollmentRequest envelope, and binds the two through the Evidence itself ({{key-binding}}): the Evidence is produced over a digest of the finished CSR, so it cannot travel inside it.
The two are not exclusive.
An Attester whose platform can also make claims about the CSR key MAY place such attestations in the CSR per {{CSR-ATTEST}}, and a Credential Authority MAY require either or both.
Carrying Evidence beside the CSR is required in any case for Retrieval mode, which has no CSR.


# Media Types and Encodings

Implementations MUST support at least one of CBOR or JSON envelopes, using to-be-registered media types ({{iana}}).
Servers advertise supported types with Content-Type and Accept; clients MUST send a supported type.
Evidence blobs are opaque byte strings.


# Common Structures

## AttestationInitiationResponse {#initiation-response}

Fields:

* `freshness_kind` (string, REQUIRED): one of
    * `absent-timestamp`: stamp Evidence from a trusted clock ({{RFC9334}}, Section 10.1)
    * `absent-none`: no freshness claim
    * `absent-epoch`: embed an epoch marker already held locally
    * `present-nonce`: single-use Freshness Handle; embed it in Evidence
    * `present-epoch`: current epoch marker as Freshness Handle; embed it in Evidence; retry `attest-initiate` if the epoch moved
* `handle` (bytes): Freshness Handle - REQUIRED for `present-nonce` and `present-epoch`; MUST be absent otherwise ({{attest-initiate}})
* `server_id` (string, REQUIRED): the identity of the EST Server on whose behalf the response is given, as the Attester binds it into Evidence ({{binding-input}}): the origin of the EST Server's URI (scheme, host and port), unless a deployment profile specifies another identifier. When `attest-initiate` is completed locally, `server_id` is the configured identity of the intended EST Server.
* `expires_in` (integer, OPTIONAL): seconds until a `present-nonce` or `present-epoch` Handle is no longer valid
* `max_age` (integer, OPTIONAL): for `absent-timestamp`, the maximum Evidence age in seconds acceptable to the Verifier or Relying Party
* `mode` (string, REQUIRED): `enroll` or `retrieve`
* `acceptable_cek` (array, only when `mode` is `retrieve`): acceptable CEK algorithms/suites
* `acceptable_csk` (array, only when `mode` is `enroll`): acceptable CSK algorithms/suites

The Freshness Handle originator MUST ensure `present-nonce` uniqueness and MUST correlate the second-leg request with the Handle from this `attest-initiate`.
The Verifier appraises whether Evidence is bound to a still-valid Freshness Handle.
The Attester MUST compare `server_id` with the Target it named in Initiate-Credential-Acquisition (Section 5.1 of {{TACRA}}) and MUST NOT produce Evidence if they differ.


# Attested Credential Acquisition Modes

## Credential Enrollment Mode

### Request: AttestedEnrollmentRequest

Fields:

* freshness_kind (string, REQUIRED): MUST match the preceding `attest-initiate` response
* handle (bytes): Freshness Handle - REQUIRED when `freshness_kind` is not `absent-none`; MUST be absent otherwise. MUST equal the Handle returned by `attest-initiate` when present.
* csr (bytes, REQUIRED): DER-encoded PKCS#10 CSR
* evidence (bytes, REQUIRED): MUST be bound to the Freshness returned by `attest-initiate`, if any
* binding (object, REQUIRED): declares how the CSR key is bound to Evidence
* credential_hint (string, OPTIONAL): Credential Hint supplied by the Attester; the Credential Authority MAY use it, ignore it, or reject the request

### Response (Success)

On success, the EST Server returns the enrollment response produced by the Credential Authority, as for simpleenroll in {{RFC7030}}.

### Key Binding and PoP Requirements {#key-binding}

1. PoP: The Credential Authority MUST verify possession of the CSR private key.
2. Evidence-to-CSR: Evidence MUST bind the CSR so a different CSR cannot be substituted. The Verifier MUST attest to that binding.
3. Evidence-to-Freshness: Evidence MUST be bound to the Freshness from `attest-initiate` ({{attest-initiate}}). The Verifier MUST attest to that binding.
4. Evidence-to-Target: Evidence MUST bind `server_id` ({{initiation-response}}), so that Evidence produced for one EST Server cannot be presented to another. The Verifier MUST attest to that binding, and the Credential Authority MUST recompute it with its own `server_id`.

At least one Evidence-to-CSR mechanism MUST be produced by the Attester and verified by the Verifier:

* CSR Hash: Evidence contains H(csr_der)
* Public Key Thumbprint: Evidence contains a thumbprint of the CSR SubjectPublicKeyInfo
* Key Certification: Evidence states that the CSR key is resident in protected hardware/TEE and matches the CSR public key. This mechanism is available only from an Attesting Environment that makes claims about keys, such as a TPM or an enclave key-attestation service. The hardware Evidence of a confidential VM does not: an AMD SEV-SNP attestation report {{AMD-SNP-ABI}} and an Intel TDX TDREPORT {{TDX-ABI}} carry no claim about keys the guest generates (the SEV-SNP report's key digests describe the launch identity keys, and its KEY_INFO field the report's own signing key), and the only guest-chosen content of each is a 64-octet field. On such platforms Key Certification can come only from a second Attesting Environment inside the guest, such as a vTPM or measured software, whose own measurement is then part of what the Verifier appraises.

The binding object MUST name the method and any identifiers (e.g., hash algorithm).

### Platform Forms of the Binding {#platform-forms}

How the binding value reaches the Evidence is platform-specific, and producing it is the responsibility of the Platform Plug-in of {{TACRA}}:

* Direct: the Attesting Environment writes the value into a guest-chosen field of the hardware Evidence, such as REPORT_DATA of an AMD SEV-SNP report, REPORTDATA of an Intel TDX quote, or the user data of an AWS Nitro attestation document.
* Nested: a lower layer owns that field, and the value travels through a nested attestation whose report data the guest controls, such as a vTPM quote. On Azure confidential VMs the paravisor fixes REPORT_DATA at boot, so the SEV-SNP report itself cannot carry a per-request value.
* Provider-scoped: the Evidence is signed by a key shared across a provider's fleet and identifies the provider's key domain rather than a machine. On AWS SEV-SNP instances in shared tenancy the report is signed by a VLEK and its CHIP_ID is zero.

An EST Server and a Credential Authority MUST NOT assume the direct form.
The Verifier reports in the Attestation Results which form it appraised, so that a Credential Authority whose policy requires a per-machine identity can refuse provider-scoped Evidence.

### Binding Input {#binding-input}

Where the Evidence carries the bindings of {{key-binding}} as one digest in a guest-chosen field (the direct form of {{platform-forms}}), the digest is computed over the following octet string, in which `len32(x)` is the length of `x` in octets as a 32-bit big-endian unsigned integer:

~~~
binding_input = len32(handle)    || handle
             || len32(server_id) || server_id
             || len32(subject)   || subject
~~~

`handle` is the Freshness Handle, or empty for the `absent-*` Freshness Kinds; `server_id` is the UTF-8 encoding of the `server_id` string; `subject` is the DER encoding of the CSR (Enrollment) or the DER-encoded SubjectPublicKeyInfo of CEKpub (Retrieval).
The digest is SHA-512 where the field is 64 octets, as REPORT_DATA of AMD SEV-SNP and REPORTDATA of Intel TDX are; a Platform Plug-in for a shorter field uses the hash the platform prescribes, and the Verifier reports which one was used.
The length prefixes make the input unambiguous: no choice of `server_id` and `subject` can produce the same octet string as another.
The Credential Authority (Enrollment) or the Secret Vault (Retrieval) recomputes `binding_input` from its own `server_id`, the Handle it issued and the CSR or CEKpub it received, and refuses the request unless the Attestation Results report that value from the Evidence.
{{test-vector}} gives a worked example.

### Enrollment Server Processing

Upon receiving AttestedEnrollmentRequest, the EST Server MUST:

1. Validate syntax, media type, and size limits.
2. Correlate `handle` with the preceding `attest-initiate` for this session, if a Freshness Handle was returned.
3. Forward Evidence (and endorsements, if any) to the Verifier and obtain Attestation Results.
4. Forward the CSR and Attestation Results to the Credential Authority, which recomputes the binding input ({{binding-input}}) from its own `server_id`, the Handle and the CSR, refuses issuance unless the Attestation Results report that value from the Evidence, verifies PoP and authorizes issuance.
5. Return the Credential Authority's enrollment response.

The Credential Authority MUST NOT mint identities (e.g., DNS names) beyond policy for the attested identity context.

## Credential Retrieval Mode

### Request: AttestedRetrievalRequest

Fields:

* freshness_kind (string, REQUIRED): MUST match the preceding `attest-initiate` response
* handle (bytes): Freshness Handle - REQUIRED when `freshness_kind` is not `absent-none`; MUST be absent otherwise. MUST equal the Handle returned by `attest-initiate` when present.
* evidence (bytes, REQUIRED) -- MUST include CEKpub; MUST be bound to the Freshness returned by `attest-initiate`, if any
* credential_type (string, OPTIONAL): e.g., x509, wimse-wit
* credential_hint (string, OPTIONAL): Credential Hint supplied by the Attester; the RATS Relying Party (Secret Vault or Credential Authority) MAY use it, ignore it, or reject the request

### Evidence-to-CEK Binding

Evidence MUST integrity-protect the Freshness from `attest-initiate` ({{attest-initiate}}) and a claim conveying CEKpub or a thumbprint of CEKpub.
The Verifier MUST reject Evidence that does not, and the Secret Vault MUST deny release when Attestation Results do not confirm these bindings.

### Credential Group ID Determination

The Secret Vault MUST map Attestation Results to a credential group ID that is stable for replica workloads and distinct across security domains, tenants, and credential selections.

A typical construction is:

group_id = H(attestation_subject \|\| credential_hint \|\| policy_version)

Where attestation_subject is derived from Attestation Results (not raw Evidence) to avoid nonce/freshness variability.

### Response: EncryptedCredentialBundle {#bundle}

The response MUST be an authenticated-encryption container encrypted to CEKpub. It contains:

* group_id
* `credential_items` (array):
    * X.509 chain (if requested/authorized)
    * WIMSE WIT(s) or WIC(s) (if requested/authorized)
    * a shared signing key (high risk; see {{security}})
* metadata (optional): validity, refresh hints, rotation epoch, key identifiers
* (implicit or explicit): associated data binding at least {group_id, handle if any, credential_hint, server_id}

Mandatory-to-implement encryption mechanism: The specification MUST choose one baseline.

* CMS EnvelopedData ({{RFC5652}}, aligned with EST’s CMS usage), OR
* HPKE ({{RFC9180}}) with a specific required ciphersuite, or
* COSE_Encrypt0 ({{RFC9052}}) with a required AEAD suite.

Whichever container is chosen, the response MUST authenticate its origin to the Attester.
CEKpub is not a secret: it travels in Evidence through the EST Client and the EST Server, which are untrusted conduits ({{TACRA}}), so any party that has seen it can produce a well-formed container encrypted to it, and an Attester that decrypts such a container would use whatever key it holds.
Origin authentication is provided by one of:

* HPKE in `mode_auth` (Section 5.1.3 of {{RFC9180}}), with the Secret Vault's static KEM key as the sender key and `server_id` and `handle` in the `info` parameter, which binds the ciphertext to the sender's identity as that section recommends; or
* a signature by the Secret Vault over the container and its associated data: for CMS, a SignedData that encloses the EnvelopedData ({{RFC5652}}); for COSE, a COSE_Sign1 over the COSE_Encrypt0 ({{RFC9052}}).

The Attester MUST hold a trust anchor for the Secret Vault's origin key, provisioned as its trust in the Credential Authority is provisioned, which is out of scope for this document.

TODO: Ensure that TACRA architecture can carry these and other encryption mechanisms to the Attester in a predictable format.

### Retrieval Server Processing

Upon receiving AttestedRetrievalRequest, the EST Server MUST:

1. Validate syntax and size limits, and correlate `handle` with the preceding `attest-initiate` as in enrollment processing
2. (Passport mode only, Background Check mode achieved by reversing the order) Forward Evidence to the Verifier and obtain Attestation Results
3. Forward Attestation Results to the Secret Vault, which recomputes the binding input ({{binding-input}}) from its own `server_id`, the Handle and CEKpub, refuses release unless the Attestation Results report that value from the Evidence, computes group_id, authorizes, fetches the bundle, and encrypts it to CEKpub
4. Return the EncryptedCredentialBundle produced by the Secret Vault

A variant in which the EST Server receives a plaintext secret from the Secret Vault and re-encrypts to CEKpub is possible but discouraged.

### Attester Processing {#retrieval-attester}

Upon receiving an EncryptedCredentialBundle, the Attester MUST, before using any secret it contains:

1. Verify the origin of the container under the Secret Vault's trust anchor ({{bundle}}).
2. Verify that `server_id` in the associated data equals the `server_id` it bound into Evidence ({{binding-input}}), and that `handle`, if present, equals the Handle it embedded.
3. Decrypt with CEKpri and verify that `group_id` and `credential_hint` are the ones it requested.

A container that fails any of these checks MUST be discarded.


# Error Handling

Servers SHOULD reuse HTTP status codes from {{RFC7030}} and a machine-readable error body:

* 400 Bad Request: malformed envelope, missing fields
* 401 Unauthorized: missing/invalid authentication required by deployment policy
* 403 Forbidden: attestation failed or policy denies enrollment/retrieval
* 409 Conflict: Freshness Handle replay detected, or `present-epoch` marker has moved
* 415 Unsupported Media Type: unsupported encoding
* 429 Too Many Requests: rate limiting
* 500/503: verifier unavailable or internal error

Error bodies MUST NOT leak sensitive attestation details. Servers MAY provide a correlation identifier for debugging.


# Security Considerations {#security}

## Specific to Attested Enrollment Mode

* Key Substitution: attestation success is not sufficient without Evidence-to-CSR binding and PoP
* Identity Over-Issuance: Credential Authority policy must constrain subject/SAN to the attested identity context
* Server substitution by the conduit: without the Evidence-to-Target binding, an EST Client acting as the Attester's conduit can carry a genuine CSR and Evidence to a different EST Server and Credential Authority, which would then certify the Attester's attested identity without the Attester having chosen it. The `server_id` binding of {{key-binding}} closes this; the same consideration applies to Retrieval ({{retrieval-attester}}).
* Linking identity and proof of possession: Section 3.5 of {{RFC7030}} links the two by placing the tls-unique value of the TLS session in the CSR's challenge-password. That mechanism is not available here: tls-unique is not defined for TLS 1.3 ({{RFC9266}}), and the TLS peer of the EST Server is the EST Client, which SHOULD be outside the Attester's trust boundary ({{TACRA}}). The bindings of {{key-binding}} take its place. Where the Attester is itself the EST Client, it MAY additionally place the tls-exporter value of {{RFC9266}} in the challenge-password, which is the TLS 1.3 form of Section 3.5 of {{RFC7030}}.

## Specific to Attested Retrieval Mode

* Shared Signing Key distribution: If the credential bundle includes a private signing key shared across replicas, compromise of one replica compromises the group. This mode SHOULD be restricted to environments where unwrap and key use are strongly protected.
* Non-Exportability Requirements: Deployments that transport a signing key SHOULD require Evidence to attest that CEKpri is non-exportable and that decryption/unwrapping occurs only within an approved protected environment (e.g., TEE/TPM-sealed key usage). On AMD SEV-SNP and Intel TDX such a claim can only be made by a second Attesting Environment inside the guest, as noted for Key Certification in {{key-binding}}.
* Attribution: Shared keys eliminate per-instance attribution. If accountability is required, consider per-instance keys with identical identity claims, or a centralized signing service.
* Bundle substitution: a conduit can return a container produced by a Secret Vault of its choosing, or by anyone who has seen CEKpub. Without origin authentication the Attester cannot tell, and would sign with a key the attacker chose. See {{bundle}} and {{retrieval-attester}}.

## Common to Both Modes

* Freshness: a nonce Handle MAY come from the Verifier or the Relying Party; a timestamp from the Attester's clock; an epoch marker from a local hold or a returned Handle (Section 10 of {{RFC9334}}). Evidence MUST be bound as required by the kind from `attest-initiate`. The originator MUST reject `present-nonce` reuse and a stale `present-epoch`.
* The channel from EST Server to Verifier MUST provide integrity, authenticity, and replay protection.
* The EST Server SHOULD enforce size and rate limits on Evidence.
* If classic EST and attested resources both exist, the Relying Party MUST be able to require attestation. The EST Server MUST NOT substitute a classic EST operation for an attested request.


# IANA Considerations {#iana}

This document requests registrations for:

* New EST well-known paths (if applicable under EST registries)
* Media types for:
    * AttestationInitiationResponse
    * AttestedEnrollmentRequest
    * AttestedRetrievalRequest
    * EncryptedCredentialBundle
* Registry of acceptable_evidence identifiers and credential_type identifiers (if not reused from existing registries)

--- back

# Test Vector for the Binding Input {#test-vector}
{:numbered="false"}

Enrollment, direct form, SHA-512. The Handle and the CSR are those of an enrollment run on an AMD SEV-SNP guest (Google Cloud, 12 September 2026); the CSR is ECDSA P-256 with subject CN=workload.tacra.example.

~~~
handle (32 octets) =
  c456ed0cc3b9379313a47beb9b0d6f7aff3bb5c9f6e4745cda9c3f6c6cb11b1f

server_id (23 octets) = "https://est.example.com"

subject = CSR DER (223 octets) =
  3081dc3081830201003021311f301d06035504030c16776f726b6c6f61642e
  74616372612e6578616d706c653059301306072a8648ce3d020106082a8648
  ce3d03010703420004d7a07787a429d424c288172b564acdbbd6e16872e4b1
  0dcd8ee7b2be07e42b9e368cf36caeb130b0aaf6d5c998b74dcb86d30b7db8
  6bda85d64a5f080d01cb64a000300a06082a8648ce3d040302034800304502
  206af6c73e02c80fef59d6ab75f5572a04537a294288d796070d69a47899d7
  cfff022100c054ae5070c5852487d4cb8f486a2462a507404d3ab68099b2e8
  c592e8e14327

binding_input = 00000020 || handle
             || 00000017 || server_id
             || 000000df || subject        (290 octets)

SHA-512(binding_input) =
  bb515f3f08c5d1d903e3a9098af603f93cc4c0e1db035f3ec394e511c44824
  9bb840ecd91342168ecfc5ee1ce2c7aa6b3323867927b4377ce4d3fe4683df
  4e5f
~~~

# Acknowledgments
{:numbered="false"}

The authors thank the Confidential Computing Consortium's Trustworthy Workload Identity (TWI) Special Interest Group for creating the TACRA architecture.
