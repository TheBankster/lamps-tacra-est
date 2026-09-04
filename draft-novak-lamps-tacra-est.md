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

This document specifies extensions to Enrollment over Secure Transport (EST, {{!RFC7030}}) to support Remote Attestation Procedures (RATS, {{RFC9334}}) as an authorization input for workload credential provisioning. Two modes are defined:

1. Attested Credential Enrollment Mode: a workload submits a PKCS#10 Certificate Signing Request (CSR) and Remote Attestation Evidence. The EST server (acting as a RATS Relying Party) authorizes issuance based on attestation and cryptographically binds the issued credential to the CSR key through proof-of-possession and explicit key-binding.
2. Attested Credential Retrieval Mode: a workload submits Remote Attestation Evidence that includes an asymmetric Credential Encryption Key (CEK). Upon successful verification and authorization, the EST server returns an existing shared public credential bundle (e.g., an X.509 certificate, a WIMSE WIC), and a secret, such as an associated signing key, a bearer token, or a preshared key, encrypted to the CEK.

These extensions add new EST resources, request/response envelopes, processing rules, and security requirements for replay protection, freshness, key-binding, confidentiality, auditability, and key distribution risk management.

--- middle


# Introduction {#intro}

EST ({{!RFC7030}}) defines an HTTPS-based protocol for certificate enrollment and management, typically between an EST client and an EST server acting as an interface to a Certification Authority (CA). In modern environments (e.g., cloud, containers, confidential computing), workloads often lack pre-provisioned credentials and require issuance based on runtime properties. Additionally, zero trust environments place additional restrictions limiting which parties have access to secrets and credentials. This means that EST clients and servers may not be trusted to handle such restricted information in plaintext.

RATS ({{RFC9334}}) defines an architecture and roles for remote attestation. This document integrates RATS with EST by making an EST server a conduit for credential issuance or credential release decisions based on verified attestation.

This document defines two complementary modes:

1. Issuance of fresh credentials bound to a workload-generated key (traditional EST semantics).
2. Release of a shared credential bundle to replicated workloads that produce equivalent attestation results, using encryption of secrets to an attester-provided key (credential broker semantics).

In either case, RATS style remote attestation serves to authenticate the workload to the EST server.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## RATS Entities
* Attester: the workload instance producing Evidence.
* Verifier: the component that appraises Evidence and produces Attestation Results.
* RATS Relying Party (RRP): consumes Attestation Results to make an authorization decision: in the context of this document, the authorization decision pertains to issuing new or releasing existing credentials to the Attester.
* RATS-Unaware Relying Party (RUP): authenticates the Attester using credentials issued by or retrieved from the RRP

## EST Entities
* EST Client: the protocol participant acting on behalf of the workload
* EST Server: the server exposing EST resources; in these modes it also acts as a conduit to the Verifier, as well as to RATS RP (RRP) such as a Certificate Authority or a Key/Credential Store

## Artifacts
* Evidence: attestation evidence produced by the Attester (opaque to this specification).
* Endorsements: auxiliary inputs for verification (optional; opaque to this specification).
* Attestation Results: RATS Verifier output.
* CSR: PKCS#10 certification request (DER-encoded).
* PoP: proof-of-possession of the private key corresponding to the CSR public key.
* CEK: Credential Encryption Key. When used, CEKpub is carried in Evidence; the response is encrypted to CEKpub.
* Credential Bundle: container that may include an X.509 chain, WIMSE WIC(s), and optionally a signing key and metadata.


# Architecture

The mechanisms described below work equally well in both Passport and Background Check RATS modes. Only the Passport mode is illustrated, but the differences between the two modes are immaterial for purposes of this specification.

## Attested Enrollment Mode (IdP/CA-mediated issuance)

* Attester generates keypair and CSR.
* Attester obtains Evidence that includes a binding to the CSR and freshness data.
* Attester, through the EST client, submits CSR + Evidence to the EST server.
* EST server verifies PoP for CSR key and obtains Attestation Results via a Verifier.
* EST server authorizes and requests credential issuance (or issues credentials directly), returning an EST enrollment response.

~~~~ ascii-art
{::include attested_enrollment.txt}
~~~~
{: #fig-enroll title="Attested Enrollment Mode (Passport)"}

1. Attester initiates Remote Attestation Challenge
2. EST client forwards Remote Attestation Challenge to EST server
3. EST server forwards Remote Attestation Challenge to Verifier, which generates the Challenge and returns it to the Attester via EST server and EST client
4. Attester sends Evidence and CSR to EST client
5. EST client forwards the Evidence and CSR to EST server
6. EST server forwards the Evidence to Verifier and obtains Attestation Results
7. EST server sends the CSR and Attestation Results to IdP/CA and obtains the newly issued Credential, which it returns to the Attester via EST server and EST client

## Attested Retrieval Mode (Key/Credential Broker)

* Attester generates an asymmetric CEK keypair and obtains Evidence that includes CEKpub and freshness data.
* Attester submits Evidence to the EST server.
* EST server forwards the Evidence to the Verifier and obtains from the Verifier Attestation Results, which include CEKpub from the Evidence
* EST server forwards the Attestation Results to the Key/Credential Store
* The Key/Credential Store identifies the Attester's identity from the Attestation Results, retrieves the Key or Credential matching that identity and encrypts that Key or Credential to CEKpub before returning the encrypted blob to the EST server
* EST server returns the results from the Key/Credential Store to the EST client
* EST client returns the encrypted credential to the Attester which decrypts it with CEKpri

~~~~ ascii-art
{::include attested_retrieval.txt}
~~~~
{: #fig-retrieve title="Attested Retrieval Mode (Passport)"}

1. Attester initiates Remote Attestation Challenge
2. EST client forwards Remote Attestation Challenge to EST server
3. EST server forwards Remote Attestation Challenge to Verifier, which generates the Challenge and returns it to the Attester via EST server and EST client
4. Attester sends Evidence (including CEKpub) to EST client
5. EST client forwards the Evidence to EST server
6. EST server forwards the Evidence to Verifier and obtains Attestation Results, which also include CEKpub
7. EST server sends the Attestation Results to the Key or Credential Store and obtains the credentials and associated secrets, encrypted to CEKpub, which it returns to the Attester via EST server and EST client


# Protocol Overview

These extensions define new resources under the existing EST “/.well-known/” prefix:

* `/.well-known/est/attest-challenge`
* `/.well-known/est/attest-enroll`
* `/.well-known/est/attest-retrieve`

These resources are used in addition to, not in place of, existing EST resources.

All exchanges in this document MUST use HTTPS as required by RFC 7030. Server authentication via TLS is REQUIRED. Client authentication is performed through the remote attestation-driven mechanisms defined herein (and optionally additional mechanisms).


# Resources and Methods

## attest-challenge (GET)

Returns Remote Attestation Verifier challenge material and server constraints for Attested Enrollment Mode.

* Method: GET
* Success: 200 OK
* Response: AttestationChallenge

## attest-enroll (POST)

Attested enrollment request (CSR + Evidence).

* Method: POST
* Success: 200 OK with an EST enrollment response body (see Section 8)
* Request: AttestedEnrollRequest

## attest-retrieve (POST)

Attested credential retrieval request (Evidence includes CEKpub).

* Method: POST
* Success: 200 OK
* Response: CredentialBundle


# Media Types and Encodings

This document defines abstract message structures. Implementations MUST support at least one interoperable encoding. Two encoding families are permitted:

* CBOR-based envelopes (recommended for compactness), using a to-be-registered media type (Section 13).
* JSON-based envelopes (for debugging and ecosystems standardized on JOSE), using a to-be-registered media type.

A server MUST indicate supported request and response media types via the HTTP Content-Type and Accept headers. Clients MUST send a supported media type.

Note: Evidence and Endorsements formats are intentionally opaque to this document; they are carried as byte strings.


# Common Structures

## Attestation Challenge

Fields:

TODO: validate everything below

* nonce (bytes): server-generated random value
* expires_in (integer): seconds until nonce expiration
* acceptable_evidence (array): identifiers for evidence formats
* required_freshness (object): policy hints (e.g., max age)
* required_bindings (array): required binding mechanisms for the mode
* acceptable_cek (array, only for credential retrieval): acceptable CEK algorithms/suites
* verifier_hint (optional): identifier or parameters to aid interoperability

Servers MUST ensure nonce uniqueness within a replay window and MUST reject reuse.

# Attested Credential Acquisition Modes

## Credential Enrollment Mode

### Request: AttestedEnrollmentRequest

Fields:

TODO: validate everything below

* challenge_nonce (bytes, REQUIRED)
* csr (bytes, REQUIRED): DER-encoded PKCS#10 CSR
* evidence (bytes, REQUIRED)
* endorsements (bytes, OPTIONAL)
* binding (object, REQUIRED): declares how the CSR key is bound to Evidence (Section 8.3)
* profile (string, OPTIONAL): requested issuance profile identifier

### Response (Success)

On success, the EST server returns credentials as in EST enrollment, consistent with RFC 7030 for simpleenroll. This document does not change the EST enrollment response formats; it only defines new request paths and authorization inputs.

### Key Binding and PoP Requirements

Attested Enrollment Mode MUST provide:

TODO: validate everything below

1. PoP for CSR Key: The server MUST verify that the requester possesses the private key corresponding to the CSR public key, using the CSR’s standard proof mechanisms or an equivalent channel-bound PoP defined by profile.
2. Evidence-to-CSR Binding: The request MUST include a binding that prevents substitution of a different CSR by an attacker reusing Evidence.

At least one of the following binding mechanisms MUST be implemented by both client and server:

* CSR Hash Claim Binding: Evidence (as appraised by the Verifier) contains a claim equal to H(csr_der) (hash of the CSR DER), and the verifier attests to its integrity.
* Public Key Thumbprint Binding: Evidence contains a claim equal to a thumbprint of the CSR SubjectPublicKeyInfo (SPKI).
* Key Certification Binding (Hardware-backed): Evidence contains a verifiable statement that the CSR key is resident in protected hardware/TEE and corresponds to the CSR public key.

The binding object MUST indicate which method is used and include any required identifiers (e.g., hash algorithm ID).

### Enrollment Server Processing

Upon receiving AttestedEnrollRequest, the EST server MUST:

1. Validate message syntax, media type, and size limits.
2. Validate challenge_nonce freshness and single-use (replay protection).
3. Verify CSR PoP.
4. Obtain Attestation Results by sending evidence (+ endorsements if provided) to a Verifier or by verifying locally.
5. Apply authorization policy mapping Attestation Results to:
    * whether issuance is permitted
    * issuance profile (subject/SAN constraints, EKU, validity, etc.)
6. Perform issuance (directly or via a CA) and return an EST enrollment response.

The server MUST NOT mint identities (e.g., DNS names) beyond what policy explicitly allows for the attested identity context.

## Credential Retrieval Mode

### Request: AttestedRetrievalRequest

Fields:

TODO: validate everything below

* challenge_nonce (bytes, REQUIRED)
* evidence (bytes, REQUIRED) -- MUST include CEKpub
* endorsements (bytes, OPTIONAL)
* credential_type (string, OPTIONAL): e.g., x509, wimse-wit, bundle
* profile (string, OPTIONAL): profile identifier -- TODO: is it needed here? Why?

### Evidence-to-CEK Binding

TODO: validate everything below

Evidence, as appraised by the Verifier, MUST integrity-protect:

* the challenge_nonce (or a server-provided freshness token that is cryptographically bound to it), and
* a claim conveying CEKpub or a stable thumbprint identifier for CEKpub.

Servers MUST reject requests where the Verifier cannot attest to the integrity of these bindings.

### Credential Group ID Determination

TODO: validate everything below

The holder of the requested Credential - the EST Server or the Credential/Key Store that stores it, MUST be able to map the Attestation Results from the Verifier to the Credential "group ID" that the EST Client is expecting. The mapping MUST be stable for “replica” workloads intended to receive identical credentials and MUST vary across security domains/tenants/profiles.

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
    * OPTIONAL: a shared signing key (high risk; see Section 11)
* metadata (optional): validity, refresh hints, rotation epoch, key identifiers
* (implicit or explicit): associated data binding at least {group_id, challenge_nonce, profile, server_id}

Mandatory-to-implement encryption mechanism: The specification MUST choose one baseline.

* CMS EnvelopedData (aligned with EST’s CMS usage), OR
* HPKE (RFC 9180) with a specific required ciphersuite, or
* COSE_Encrypt0 with a required AEAD suite.

TODO: Select exactly one as MUST in the final draft; multiple MAY be supported.

### Retrieval Server Processing

Upon receiving AttestedCredRequest, the EST server MUST:

1. Validate syntax, size limits, and challenge_nonce freshness/single-use.
2. Verify Evidence via a Verifier and obtain Attestation Results.
3. By itself or delegate to Credential/Key store depending on whether EST Server is permitted to see plaintext secrets:
    1. Compute group_id using policy.
    2. Authorize the requested credential type/profile for group_id.
    3. Fetch the credential bundle corresponding to (group_id, profile, credential_type).
    4. Encrypt the bundle to CEKpub with AAD binding including challenge_nonce and group_id.
5. Return EncryptedCredentialBundle.


# Error Handling

Servers SHOULD reuse HTTP status codes consistent with RFC 7030 and provide a machine-readable error body for these resources.

Recommended error conditions:

* 400 Bad Request: malformed envelope, missing fields
* 401 Unauthorized: missing/invalid authentication required by deployment policy
* 403 Forbidden: attestation failed or policy denies enrollment/retrieval
* 409 Conflict: nonce replay detected
* 415 Unsupported Media Type: unsupported encoding
* 429 Too Many Requests: rate limiting
* 500/503: verifier unavailable or internal error

Error bodies MUST NOT leak sensitive attestation details. Servers MAY provide a correlation identifier for debugging.


# Security Considerations {#security}

TODO: Validate everything below

## Common to Both Modes

* Freshness and Replay Protection: Servers MUST provide a nonce and MUST reject reuse within an enforcement window. Evidence MUST be bound to the nonce (directly or through verifier-issued freshness tokens).
* Verifier Trust: If the EST server delegates verification, the channel to the verifier MUST provide integrity, authenticity, and replay protection.
* Attestation Results SHOULD be audience-restricted to the EST server.
* DoS Considerations: Evidence appraisal can be expensive; servers SHOULD enforce size limits, rate limits, and caching of verifier results where safe.
* Downgrade Resistance: If both classic EST and attested resources exist, servers MUST ensure policy can require attestation and MUST NOT silently fall back to weaker modes.

## Specific to Attested Enrollment Mode

* Key Substitution: Prevented only if Evidence-to-CSR binding is mandatory and verified. Implementations MUST NOT treat “attestation success” as sufficient absent key-binding + PoP.
* Identity Over-Issuance: Policies must constrain subject/SAN issuance to attested identity context.

## Specific to Attested Retrieval Mode

* Shared Signing Key Distribution: If the credential bundle includes a private signing key shared across replicas, compromise of one replica compromises the group. This mode SHOULD be restricted to environments where unwrap and key use are strongly protected.
* Non-Exportability Requirements: Deployments that transport a signing key SHOULD require Evidence to attest that CEKpriv is non-exportable and that decryption/unwrapping occurs only within an approved protected environment (e.g., TEE/TPM-sealed key usage).
* Attribution: Shared keys eliminate per-instance attribution. If accountability is required, consider per-instance keys with identical identity claims, or a centralized signing service.

# IANA Considerations {#iana}

TODO: Treat as early draft, revisit later

This document requests registrations for:

* New EST well-known paths (if applicable under EST registries).
* Media types for:
    * AttestationChallenge
    * AttestedEnrollmentRequest
    * AttestedRetrievalRequest
    * EncryptedCredentialBundle
* Registry of acceptable_evidence identifiers and credential_type identifiers (if not reused from existing registries).

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
