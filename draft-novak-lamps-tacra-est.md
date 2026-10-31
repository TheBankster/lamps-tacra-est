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
   org:  Fraunhofer SIT
   email: Henk.Birkholz@ietf.contact

 - ins: S. Nikolaichuk
   name: Serhii Nikolaichuk
   org: The Capital Index LLC
   email: nikolaichuk.s.f@gmail.com

normative:
  RFC7030: EST
  RFC9180: HPKE
  RFC8785: JCS
  RFC5480:
  RFC9334: RATS
  RFC4648:
  RFC6838:
  RFC8259:
  RFC8610: CDDL
  RFC8951:
  RFC9110:
  RFC9741:

informative:
  INTERACTION-MODELS: I-D.ietf-rats-reference-interaction-models
  ATTESTATION-FRESHNESS: I-D.ietf-lamps-attestation-freshness
  CSR-ATTEST: I-D.ietf-lamps-csr-attestation
  HPKE-PQ: I-D.ietf-hpke-pq
  TACRA: I-D.novak-rats-tacra
  RFC6585:
  RFC9148:
  RFC9457:
  RFC8792:
  RFC5280:
  RFC7519:
  WIMSE: I-D.ietf-wimse-workload-creds
  TWISIGDef:
    -: TWISIGDef
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Definitions.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group - Definitions
    author:
      org: Confidential Computing Consortium Trustworthy Workload Identity SIG
  TWISIGReq:
    -: TWISIGReq
    target: https://github.com/confidential-computing/twi/blob/main/TWI_Requirements.md
    title: Trustworthy Workload Identity (TWI) Special Interest Group - Requirements
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
* Credential Bundle: the plaintext the Secret Vault seals ({{bundle}}): credential items with their attributes, and metadata
* Container: the sealed object that carries a Credential Bundle to the Attester, the EncryptedCredentialBundle ({{bundle}})
* Credential Hint: optional, implementation-defined information supplied by the Attester about the credential it expects. The Credential Authority or Secret Vault MAY use the hint, ignore it, or reject the request. The hint does not authorize issuance or release.
* Freshness Kind: the recency method, one of `absent-timestamp`, `absent-none`, `absent-epoch`, `present-nonce` or `present-epoch` ({{TACRA}})
* Freshness Handle: the value `attest-initiate` returns for `present-nonce` and `present-epoch` ({{TACRA}}, {{INTERACTION-MODELS}})
* Freshness element: the octets the binding digest covers ({{TACRA}}): the Freshness Handle, the Attester's timestamp, a locally held epoch marker, or nothing for `absent-none`. On `attest-enroll` and `attest-retrieve` it is the `handle` member, when present


# Architecture

The EST encoding is the same in Passport and Background Check modes.
The modes differ in 1) who originates a Freshness Handle and 2) whether the EST Server contacts the Verifier and then the RATS Relying Party (Passport) or the RATS Relying Party directly (Background Check).
Only Passport mode is illustrated.

Both modes use two EST legs: optional `attest-initiate` ({{attest-initiate}}), followed by either `attest-enroll` ({{attest-enroll}}) or `attest-retrieve` ({{attest-retrieve}}).
Note that `attest-initiate` is optional: in cases where the Attester can know with certainty which Freshness Kind to use (and that the Freshness Kind is not either `present-nonce` or `present-epoch`), whether to ask for new or existing credentials, as well as which ciphers the RRP expects, it can be omitted.

Neither EST Client nor EST Server has a RATS role.
Both are subject to the following restrictions:

* MUST NOT generate keys, Evidence, or CSRs
* MUST NOT mint `present-nonce` or `present-epoch` Freshness Handles
* MUST NOT appraise Evidence or Attestation Results
* MUST NOT authorize credential issuance or release. Correlating a Handle with the `attest-initiate` that returned it is not authorization. Deciding on the Credential Type or the Credential Hint is, and belongs to the RATS Relying Party.

## Attested Enrollment Mode

~~~~ ascii-art
{::include attested_enrollment.txt}
~~~~
{: #fig-enroll title="Attested Enrollment Mode (Passport)"}

1. Attester initiates Remote Attestation
2. EST Client forwards to EST Server (`attest-initiate`)
3. If Freshness is required: EST Server obtains Freshness from the Verifier or RATS Relying Party and returns it via the EST Client
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
3. If Freshness is required: EST Server obtains Freshness from the Verifier or RATS Relying Party and returns it via the EST Client
4. Attester generates CEK and Evidence including CEKpub, bound to Freshness
5. EST Client POSTs Evidence (`attest-retrieve`)
6. EST Server forwards Evidence to the Verifier
7. EST Server forwards Attestation Results to the Secret Vault, which seals the matching Credential Bundle to CEKpub in a container. The EST Server, through the EST Client, returns that container to the Attester, which checks it and decrypts it with CEKpri


# Protocol Overview

Three new EST operations are added under the EST well-known prefix (Section 3.2.2 of {{RFC7030}}):

1. `/.well-known/est/attest-initiate`
2. `/.well-known/est/attest-enroll`
3. `/.well-known/est/attest-retrieve`

They sit beside the operations of {{RFC7030}}.
An EST Server that uses the optional path segment of Section 3.2.2 of {{RFC7030}}, `/.well-known/est/arbitraryLabel1/simpleenroll` for instance, MUST offer the `attest-*` operations under that segment as well, `/.well-known/est/arbitraryLabel1/attest-initiate` and likewise for the other two.
As that section requires, it serves them without the segment too.
The label MUST NOT be `attest-initiate`, `attest-enroll` or `attest-retrieve`.
The segment selects the Credential Authority or Secret Vault the EST Server uses, as it selects a CA in {{RFC7030}}.
It is not the Target, which stays the `target` query parameter or member ({{TACRA}}).
For `attest-*` the client uses the same segment it uses for `simpleenroll` at that issuer.
Example exchanges are in {{examples}}.
They support TACRA Initiate-Credential-Acquisition, Enroll-Credential, and Retrieve-Credential {{TACRA}} commands, encoded with EST semantics.

All exchanges MUST use HTTPS as required by {{RFC7030}}.
EST Server authentication via TLS is REQUIRED.


# Resources and Methods

## attest-initiate {#attest-initiate}

The first leg of both modes is where the Attester initiates Remote Attestation by invoking the EST Client.

* Method: GET
* Request: no body; query parameters `target` (REQUIRED) and `credential_type` (OPTIONAL)
* Success: 200 OK
* Response: AttestationInitiationResponse ({{initiation-response}}): Freshness kind, Credential Acquisition Mode (`enroll` or `retrieve`), acceptable ciphers
* Error: 400 when `target` is absent, 404 when the EST Server does not serve that Target, 406 when the RATS Relying Party does not serve that `credential_type` for it ({{errors}})

Query parameters:

* `target`: the Target of Initiate-Credential-Acquisition (Section 5.1 of {{TACRA}}). The client URI-encodes it; the EST Server compares the decoded UTF-8 string.
* `credential_type`: the Credential Type the Attester plans to use with this Target ({{credential-type}}). If omitted, the EST Server returns the mode and algorithms its policy sets for the Target.

* If the EST Client already knows all the information the Attester needs to proceed, i.e., it is already configured for an absent Freshness kind (`absent-timestamp`, `absent-none`, or `absent-epoch`), and it knows what Credential Acquisition Mode is expected, and which ciphers to use, it MAY complete `attest-initiate` locally and, in that case, MUST NOT contact the EST Server.
* The EST Server, if contacted, obtains the Freshness Kind and Handle, if any, from the configured Verifier or RATS Relying Party and returns that result.
* For `present-nonce`, that GET mints a single-use Handle and records it, as the nonce resource of {{ATTESTATION-FRESHNESS}} does ({{relationship}}).
* `present-nonce` and `present-epoch` MUST include a Freshness Handle from the party chosen by the EST Server. `absent-*` Freshness Kinds carry no Freshness Handle.
* The EST Client forwards the response to the Attester.
* The Attester produces Evidence as the Freshness Kind requires.
* The EST Client then POSTs `attest-enroll` or `attest-retrieve` as the Attester indicates.

A Credential Authority or Secret Vault issues or releases credentials for a Target per its policy for that Target, since the Target only recognizes credentials from an issuer it already trusts.
The Target itself takes no part in this profile: it is the RATS-unaware Relying Party of {{TACRA}}, named in the request and in the binding.
* If the second leg fails because a `present-epoch` moved or a `present-nonce` is no longer valid, the Attester retries `attest-initiate`.

## attest-enroll {#attest-enroll}

* Method: POST
* Request: AttestedEnrollmentRequest ({{enrollment-request}}): CSR and Evidence
* Success: 200 OK
* Response: enrollment response as for simpleenroll (Section 4.2.3 of {{RFC7030}})

## attest-retrieve {#attest-retrieve}

* Method: POST
* Request: AttestedRetrievalRequest ({{retrieval-request}}): CEKpub and Evidence
* Success: 200 OK
* Response: EncryptedCredentialBundle ({{bundle}})


# Relationship to Other LAMPS Work {#relationship}

Two LAMPS documents cover parts of the exchange defined here.

{{ATTESTATION-FRESHNESS}} defines a nonce request and response for certificate management protocols, including an EST resource at `/.well-known/est/nonce` (GET, or POST with `application/est-attestation-freshness+json`) whose response carries a `nonce` of 8 to 64 bytes and an OPTIONAL `expiry` in seconds (Section 5.1 of {{ATTESTATION-FRESHNESS}}).
`attest-initiate` ({{attest-initiate}}) is the first leg that returns the Freshness Handle: such a nonce for `present-nonce`, the current epoch marker for `present-epoch`, and none for the `absent-*` kinds.
It returns in addition the Freshness Kind itself, the Credential Acquisition Mode and the acceptable CSK or CEK algorithms ({{initiation-response}}), none of which the nonce resource carries.
A `present-nonce` Handle of this profile is a nonce as {{TACRA}} and Section 5.1 of {{ATTESTATION-FRESHNESS}} specify: 8 to 64 octets, at least 64 bits of entropy, issued once and not reused.
`expires_in` is that document's `expiry`.
That section requires the server to associate the nonce with the subsequent request.
This profile does so by correlating the Handle with the `attest-initiate` that returned it, in the server processing of both modes, and by the RATS Relying Party's recomputation of the digest that carries the Handle ({{est-binding}}).
It does not use a TLS session or a cookie for that: either would bind the EST Client, the conduit, and not the Attester.
A request of this profile takes no nonce length: the Handle reaches the Evidence through the digest, so no platform needs a Handle of a particular size.
An EST Server MAY offer both resources.
They are not interchangeable: a nonce the nonce resource returned is not a Freshness Handle of this profile, and `attest-enroll` and `attest-retrieve` take a Handle from `attest-initiate`, or none for the `absent-*` kinds.

{{CSR-ATTEST}} conveys attestations inside the CSR, in its `id-aa-attestation` attribute.
It makes the Certification Authority or Registration Authority that processes the CSR responsible for ensuring that the binding between those attestations and the CSR's public key has been validated (Section 6.1 of {{CSR-ATTEST}}).
This profile conveys the Evidence that carries the binding of {{TACRA}} beside the CSR, in the AttestedEnrollmentRequest ({{enrollment-request}}).
That Evidence is produced over the digest of {{est-binding}}, which includes the finished CSR, so it cannot travel inside it, and {{CSR-ATTEST}} does not replace it.
Evidence beside the CSR is what this profile requires.
Attestations in the CSR as well, per {{CSR-ATTEST}}, are a second attestation, optional, about properties of the CSR key that this digest does not prove, under the rules of {{TACRA}} and of Section 6.1 of {{CSR-ATTEST}}.
Whether a Credential Authority requires them is agreed out of band.
This choice is specific to Enrollment; Retrieval has no CSR and binds Evidence to CEKpub ({{evidence-to-cek}}).


# Media Types and Encodings {#media-types}

The messages of the `attest-*` resources are JSON ({{RFC8259}}), as defined in CDDL ({{RFC8610}}) in {{cddl}}; implementations MUST support this encoding.
A CBOR encoding, for EST over secure CoAP ({{RFC9148}}), is left for future work.
Evidence is an opaque byte string.

As each EST message type has its own media type (Section 3.2.4 of {{RFC7030}}), so does each JSON message of the `attest-*` resources ({{iana}}):

| Resource | Request | Response |
|---|---|---|
| `attest-initiate` | none: a GET, query string, no body | `application/est-attest-initiate+json` |
| `attest-enroll` | `application/est-attest-enroll+json` | `application/pkcs7-mime; smime-type=certs-only`, as simpleenroll |
| `attest-retrieve` | `application/est-attest-retrieve+json` | `application/est-attest-bundle+json` |
{: title="Media types of the attest-* messages"}

A client MUST send a POST of the resource's media type.
An `attest-initiate` GET has no body; the client SHOULD set Accept to `application/est-attest-initiate+json`.
An EST Server responds to a request of another media type with 415 (Unsupported Media Type) and SHOULD name the resource's media type in an Accept header field (Section 15.5.16 of {{RFC9110}}).

## Message Definitions {#cddl}

In the definitions below, a byte string is carried as a text string holding its unpadded base64url encoding (Section 5 of {{RFC4648}}), as the control operator `.b64u` defines it (Section 2.1 of {{RFC9741}}): only characters of the base64url alphabet, no padding, and the bits that encode no octet set to zero.
The group choices tie each member to the Freshness Kinds and modes its description below names: in a response, `handle` and `expires_in` to `present-nonce` and `present-epoch`, `max_age` to `absent-timestamp`, `acceptable_csk` to `enroll` and `acceptable_cek` to `retrieve`; in a request, `handle` to every Freshness Kind but `absent-none`.
The EncryptedCredentialBundle and its plaintext are the last three rules ({{bundle}}).

~~~ cddl
{::include est-attest.cddl}
~~~


# Common Structures

## AttestationInitiationResponse {#initiation-response}

Fields:

* `freshness_kind` (string, REQUIRED): one of the Freshness Kinds of {{TACRA}}
    * `absent-timestamp`: stamp Evidence from a trusted clock
    * `absent-none`: no freshness claim
    * `absent-epoch`: embed an epoch marker already held locally
    * `present-nonce`: single-use Freshness Handle; embed it in Evidence
    * `present-epoch`: current epoch marker as Freshness Handle; embed it in Evidence; retry `attest-initiate` if the epoch moved
* `handle` (bytes, 8 to 64 octets): Freshness Handle, REQUIRED for `present-nonce` and `present-epoch`, and MUST be absent otherwise ({{attest-initiate}}). A `present-nonce` Handle is a nonce as {{TACRA}} and Section 5.1 of {{ATTESTATION-FRESHNESS}} specify: at least 64 bits of entropy, issued once and not reused
* `expires_in` (integer, OPTIONAL): seconds until a `present-nonce` or `present-epoch` Handle is no longer valid, the `expiry` of {{ATTESTATION-FRESHNESS}}
* `max_age` (integer, OPTIONAL): for `absent-timestamp`, the maximum Evidence age in seconds acceptable to the Verifier or RATS Relying Party
* `mode` (string, REQUIRED): `enroll` or `retrieve`
* `acceptable_cek` (array, only when `mode` is `retrieve`): the CEK suites the Secret Vault accepts, tokens of {{alg-tokens}}
* `acceptable_csk` (array, only when `mode` is `enroll`): the CSK algorithms the Credential Authority accepts, tokens of {{alg-tokens}}
* `acceptable_evidence` (array, OPTIONAL): the Evidence formats the EST Server forwards, each named by its media type ({{alg-tokens}}). A request of the second leg MAY name the format of its Evidence in `profile`, one of them. When the list is absent, or `attest-initiate` was completed locally, the EST Client sends the format its configuration names, or none

The Freshness Handle originator MUST ensure `present-nonce` uniqueness and MUST correlate the second-leg request with the Handle from this `attest-initiate`.
The Verifier appraises whether Evidence is bound to a still-valid Freshness Handle.


## The Binding in This Profile {#est-binding}

This profile uses the digest of Section "Binding Credential Keys to Evidence" of {{TACRA}}.
The Attester writes that digest into the freshness input of its Evidence, the Direct form of that section.
The Verifier reports the value it read.
The Credential Authority or the Secret Vault recomputes the digest from the request and refuses the request with 400 Bad Request unless the two are equal.

The three octet strings of that construction are, in this profile:

* `freshness`: the freshness element of the request, the octets of `handle`. For `present-nonce` and `present-epoch` that is the Freshness Handle `attest-initiate` returned. For `absent-epoch` it is the epoch marker the Attester holds. For `absent-timestamp` it is the Attester's timestamp, 8 octets, as {{TACRA}} encodes it. For `absent-none` it is empty, and `handle` is absent.
* `target`: the UTF-8 encoding of the `target` member, used as an exact octet string.
* `subject`: for Enrollment, the DER encoding of the PKCS#10 CSR, the octets of `csr`. For Retrieval, CEKpub as the request carries it, the octets of `cek_pub`, in the encoding of its KEM ({{bundle}}).

The digest is SHA-512, as {{TACRA}} sets for a freshness input of 64 octets, which REPORT_DATA of AMD SEV-SNP and REPORTDATA of Intel TDX are.
A platform whose freshness input is not 64 octets is outside this document's mandatory method.

The `binding` object ({{cddl}}) carries no octets of its own.
It names the method and the hash:

* `method` (string, REQUIRED): `binding-input`, the construction above and the only method this document defines.
* `hash` (string, REQUIRED): `sha512`.

A Credential Authority or Secret Vault that does not recognize `method` refuses the request with 400 Bad Request.

## Credential Type Tokens {#credential-type}

`credential_type`, on `attest-initiate` and in both requests, takes one of these tokens:

* `x509`: an X.509 certificate ({{RFC5280}}). Enrollment mints one. Retrieval MAY return one that was issued before.
* `wic`: a WIMSE Workload Identity Certificate (Section 6.1 of {{WIMSE}}), an X.509 certificate whose Workload Identifier is its one URI SubjectAltName. Enrollment mints one, and the Credential Authority sets the Workload Identifier under its policy, as it sets the subject names of any certificate it issues. Retrieval MAY return one that was issued before.
* `wit`: a WIMSE Workload Identity Token ({{WIMSE}}). Retrieval only.
* `jwt`: a JSON Web Token used as a bearer token ({{RFC7519}}). Retrieval only.
* `api-key`: an API key. Retrieval only.
* `psk`: a pre-shared symmetric key. Retrieval only.
* `private-key`: a private key shared by the instances of a replicated service. Retrieval only, under the conditions of {{security}}.

`attest-enroll` mints `x509` or `wic`.
A `credential_type` other than these on `attest-enroll` is a Credential Type the RATS Relying Party does not serve, and the EST Server refuses the request with 406 Not Acceptable ({{errors}}).
When `credential_type` is absent, on `attest-enroll` and on `attest-retrieve` alike, the client takes what the RATS Relying Party issues or releases under its policy.
A client that needs a Credential Type names it.
`attest-retrieve` MAY carry any token above.
The Secret Vault selects what it releases from the Attestation Results, its policy and the Credential Hint.
Where the request names a `credential_type`, it releases that Credential Type or refuses the request.
A Credential Type the RATS Relying Party does not serve is refused with 406 Not Acceptable, and the response names the Credential Types it serves for that `target` ({{errors}}).
The `type` of a credential item in the released bundle ({{bundle}}) uses the same tokens.


## Protocol Tokens {#alg-tokens}

`acceptable_csk`, `acceptable_cek` and `acceptable_evidence` are arrays of ASCII tokens, and `profile` is one such token.
This document defines them.

CSK algorithms, for Enrollment:

* `ecdsa-p256-sha256`: ECDSA on P-256 with SHA-256, the signature of the CSR. Implementations MUST support it.

CEK suites, for Retrieval: the HPKE suites of {{bundle}}, each named by its KEM, its KDF and its AEAD as the HPKE registries name them ({{RFC9180}}), joined with `+`:

* `DHKEM(P-256, HKDF-SHA256)+HKDF-SHA256+AES-256-GCM`: the mandatory suite. Implementations MUST support it.
* `MLKEM768-P256+HKDF-SHA256+AES-256-GCM`: the recommended suite.

A Credential Authority or Secret Vault names in `acceptable_csk` or `acceptable_cek` the tokens it accepts, and the Attester generates its key for one of them.

Evidence formats: the media type of the Evidence, in the format the Verifier recognizes, for example `application/vnd.veraison.tsm-report+cbor`.

A client MUST NOT use a token the `attest-initiate` response did not list.
A client that completed `attest-initiate` locally uses the tokens its configuration names.

# Attested Credential Acquisition Modes

## Credential Enrollment Mode

### Request: AttestedEnrollmentRequest {#enrollment-request}

Fields:

* target (string, REQUIRED): the Target of Enroll-Credential (Section 5.2 of {{TACRA}})
* freshness_kind (string, REQUIRED): MUST match the preceding `attest-initiate` response
* handle (bytes): the freshness element ({{est-binding}}) - REQUIRED when `freshness_kind` is not `absent-none` and MUST be absent otherwise. For `present-nonce` and `present-epoch` it MUST equal the Freshness Handle returned by `attest-initiate`. For `absent-timestamp` it is the 8-octet timestamp and for `absent-epoch` the locally held epoch marker.
* csr (bytes, REQUIRED): DER-encoded PKCS#10 CSR
* evidence (bytes, REQUIRED): MUST be bound to the Freshness returned by `attest-initiate`, if any
* profile (string, OPTIONAL): the media type of `evidence`, one of the `acceptable_evidence` of the `attest-initiate` ({{initiation-response}}). If absent, the EST Server forwards the Evidence as it is, and the Verifier recognizes its format. A format the EST Server does not accept is refused with 415 Unsupported Media Type
* binding (object, REQUIRED): declares how the CSR key is bound to Evidence
* credential_type (string, OPTIONAL): `x509` or `wic` ({{credential-type}}). If absent, the client takes what the Credential Authority mints
* credential_hint (string, OPTIONAL): Credential Hint supplied by the Attester; the Credential Authority MAY use it, ignore it, or reject the request

### Response (Success)

On success, the EST Server returns the enrollment response produced by the Credential Authority, as for simpleenroll in {{RFC7030}}.

### Key Binding and PoP Requirements {#key-binding}

1. PoP: The Credential Authority MUST verify possession of the CSR private key.
2. Evidence-to-CSR, Evidence-to-Target and Evidence-to-Freshness: the Evidence MUST carry the digest of {{est-binding}}, the binding of Section "Binding Credential Keys to Evidence" of {{TACRA}}, with the DER-encoded CSR as its subject. As that section requires, the Verifier reports the bound value in the Attestation Results, and the Credential Authority recomputes it from the request and refuses the request with 400 Bad Request on a mismatch. A substituted CSR, Target or freshness element fails that check.

The `binding` object ({{cddl}}) is as {{est-binding}} defines it.

### Enrollment Server Processing {#enrollment-processing}

Upon receiving AttestedEnrollmentRequest, the EST Server MUST:

1. Validate syntax, media type, and size limits.
2. For `present-nonce` and `present-epoch`, correlate `handle` with the `attest-initiate` that returned it. Refuse the request with 400 Bad Request when no `attest-initiate` returned this Handle, when it has expired or was used before, or when its `target`, `freshness_kind`, or `credential_type` when it is carried, differ from that `attest-initiate`'s. A `present-nonce` Handle is single-use. For the `absent-*` Freshness Kinds no Handle was returned, and the RATS Relying Party checks the freshness element under the Freshness Kind its policy sets for the Target ({{TACRA}}). This correlation keeps the two legs of one exchange together at the EST Server. It is not authorization: the Credential Authority takes the Freshness Kind from its policy for the Target, recomputes the digest and decides ({{TACRA}}).
3. Passport: forward Evidence to the Verifier and obtain Attestation Results. Background Check: skip this step; the Credential Authority obtains Attestation Results itself.
4. Forward the CSR and Attestation Results (Passport) or the CSR and Evidence (Background Check) to the Credential Authority, which verifies PoP and authorizes issuance.
5. Return the Credential Authority's enrollment response.

The Credential Authority MUST NOT mint identities (e.g., DNS names) beyond policy for the attested identity context.

## Credential Retrieval Mode

### Request: AttestedRetrievalRequest {#retrieval-request}

Fields:

* target (string, REQUIRED): the Target of Retrieve-Credential (Section 5.3 of {{TACRA}})
* freshness_kind (string, REQUIRED): MUST match the preceding `attest-initiate` response
* handle (bytes): the freshness element ({{est-binding}}) - REQUIRED when `freshness_kind` is not `absent-none` and MUST be absent otherwise. For `present-nonce` and `present-epoch` it MUST equal the Freshness Handle returned by `attest-initiate`. For `absent-timestamp` it is the 8-octet timestamp and for `absent-epoch` the locally held epoch marker.
* evidence (bytes, REQUIRED) -- MUST include CEKpub; MUST be bound to the Freshness returned by `attest-initiate`, if any
* cek_pub (bytes, REQUIRED): CEKpub in the encoding of its KEM ({{bundle}}), to which the Secret Vault seals the bundle
* profile (string, OPTIONAL): the media type of `evidence`, one of the `acceptable_evidence` of the `attest-initiate` ({{initiation-response}}). If absent, the EST Server forwards the Evidence as it is, and the Verifier recognizes its format. A format the EST Server does not accept is refused with 415 Unsupported Media Type
* binding (object, REQUIRED): declares how CEKpub is bound to Evidence
* credential_type (string, OPTIONAL): one of the tokens of {{credential-type}}. If absent, the client takes what the Secret Vault releases
* credential_hint (string, OPTIONAL): Credential Hint supplied by the Attester; the RATS Relying Party (Secret Vault or Credential Authority) MAY use it, ignore it, or reject the request

### Evidence-to-CEK Binding {#evidence-to-cek}

The Evidence MUST carry the digest of {{est-binding}} with the octets of `cek_pub` as its subject.
As Section "Binding Credential Keys to Evidence" of {{TACRA}} requires, the Verifier reports the bound value in the Attestation Results, and the Secret Vault recomputes it from the request and denies release with 400 Bad Request on a mismatch.
The `binding` object is as {{est-binding}} defines it.

### Credential Group ID Determination {#group-id}

The Secret Vault maps Attestation Results to a credential group, as {{TACRA}} requires: stable for replica workloads, and distinct across security domains, tenants and the credentials it releases.
The mapping is the Secret Vault's policy.
This profile does not specify the construction of `group_id`, and different Secret Vaults need not compute the same value.
`credential_hint` MAY select among the credentials the policy allows for the attested context and MUST NOT enlarge that set.
A Credential Hint the policy does not know is ignored, or the request is refused with 400 Bad Request.
The `group_id` the container carries ({{bundle}}) is the one the Secret Vault chose, an opaque string to everyone else.

An example of a mapping, not a requirement: SHA-256 over the length-prefixed concatenation, as {{TACRA}} prefixes, of an octet string the Secret Vault derives from the Attestation Results, the Target and an octet string that names the release policy.
It is derived from the Attestation Results and not from the Evidence, so that freshness does not change the group id.

### Response: EncryptedCredentialBundle {#bundle}

The EST Server returns the container the Secret Vault produced, as Section "Sealing the Released Credential" of {{TACRA}} constructs it, as JSON of media type `application/est-attest-bundle+json`: the `encrypted-credential-bundle` of {{cddl}}.
The EST Client and the EST Server MUST treat it as opaque.

This profile sets the parameters of that construction as follows.

* The mandatory HPKE suite is KEM DHKEM(P-256, HKDF-SHA256) (identifier 0x0010), KDF HKDF-SHA256 (0x0001) and AEAD AES-256-GCM (0x0002), all of {{RFC9180}}. Its name in `acceptable_cek` is `DHKEM(P-256, HKDF-SHA256)+HKDF-SHA256+AES-256-GCM`. Implementations MUST support it.
* The suite with KEM MLKEM768-P256 (Section 4 of {{HPKE-PQ}}, identifier 0x0050) and the same KDF and AEAD, named `MLKEM768-P256+HKDF-SHA256+AES-256-GCM`, is RECOMMENDED where a released secret must stay confidential against a later quantum computer. A Secret Vault names in `acceptable_cek` the suites it accepts, and the Attester generates its CEK for one of them.
* CEKpub is carried in `cek_pub` and is the subject of the binding ({{est-binding}}). For DHKEM(P-256, HKDF-SHA256) it is the DER encoding of a SubjectPublicKeyInfo for id-ecPublicKey on prime256v1 ({{RFC5480}}). For MLKEM768-P256 it is the encapsulation key, 1249 octets.
* The label of `info` is the ASCII string `tacra-est bundle v1`, without a terminating NUL.
* The associated data is the JSON Canonicalization Scheme serialization ({{RFC8785}}) of the `aad` object below. The plaintext is the JCS serialization of a `credential-bundle` object ({{cddl}}).

Members of the container:

* `container` (string, REQUIRED): `hpke-base`.
* `suite` (object, REQUIRED): `kem`, `kdf` and `aead`, the names above.
* `enc` (bytes, REQUIRED): the HPKE encapsulated key.
* `ciphertext` (bytes, REQUIRED): the HPKE ciphertext.
* `aad` (object, REQUIRED): `group_id` (string), `target` (string), `handle` (bytes, when the request carried one) and `credential_hint` (string, when the request carried one). The Attester rebuilds the associated data from these members and checks them against what its Evidence bound before it decrypts.

The plaintext, a `credential-bundle`, has `credential_items` (array, at least one item) and `metadata` (object, OPTIONAL, string keys).
Each item has `type` (string, a token of {{credential-type}}) and `value` (string, the credential), and MAY have `format`, `kid`, `purpose` (array of strings), `algorithm`, `exportable` (boolean) and `shared` (boolean): the key-block attributes the Secret Vault's policy sets.
`metadata` MAY carry `validity_seconds` (integer) and `secret_version` (string).

The Attester proceeds as {{TACRA}} requires: it checks that `aad` names its Target, its freshness element and its Credential Hint, checks the suite, and only then opens the ciphertext.
It MUST refuse the container otherwise.
Who sealed the container, the Attester cannot tell: anyone who saw CEKpub can seal to it.
This profile accepts that, and {{security}} states the consequence.

### Retrieval Server Processing {#retrieval-processing}

Upon receiving AttestedRetrievalRequest, the EST Server MUST:

1. Validate syntax, media type, and size limits.
2. For `present-nonce` and `present-epoch`, correlate `handle` with the `attest-initiate` that returned it, as in Enrollment. Refuse the request with 400 Bad Request when no `attest-initiate` returned this Handle, when it has expired or was used before, or when its `target`, `freshness_kind`, or `credential_type` when it is carried, differ from that `attest-initiate`'s. A `present-nonce` Handle is single-use. For the `absent-*` Freshness Kinds no Handle was returned, and the RATS Relying Party checks the freshness element under the Freshness Kind its policy sets for the Target ({{TACRA}}). This correlation keeps the two legs of one exchange together at the EST Server. It is not authorization: the Secret Vault takes the Freshness Kind from its policy for the Target, recomputes the digest and decides ({{TACRA}}).
3. Passport: forward Evidence to the Verifier and obtain Attestation Results. Background Check: skip this step; the Secret Vault obtains Attestation Results itself.
4. Forward Attestation Results (Passport) or Evidence (Background Check) to the Secret Vault, which computes group_id, authorizes, fetches the bundle and seals it to CEKpub ({{bundle}}).
5. Return the EncryptedCredentialBundle produced by the Secret Vault.

A variant in which the EST Server receives a plaintext secret from the Secret Vault and seals it to CEKpub itself is not permitted: it would expose the secret to the EST Server ({{TACRA}}).


# Error Handling {#errors}

An error response carries one of the status codes below, and a human-readable body as a simpleenroll error does (Section 4.2.3 of {{RFC7030}}, updated by Section 5.1 of {{RFC8951}}):

* 400 Bad Request: a request that does not match its definition ({{cddl}}), or an `attest-initiate` with no `target`. A binding digest that does not recompute ({{est-binding}}). A Handle that no `attest-initiate` returned, that has expired or was used before, or a `present-epoch` marker that has moved. A Credential Hint the policy does not know and does not ignore. A request that differs from the `attest-initiate` that returned its Handle ({{enrollment-processing}}, {{retrieval-processing}})
* 401 Unauthorized: missing/invalid authentication required by deployment policy
* 403 Forbidden, forwarded from the RATS Relying Party: attestation failed, or its policy denies enrollment or retrieval, the Credential Hint included
* 404 Not Found: a `target` the EST Server does not serve (Section 15.5.5 of {{RFC9110}}: the query is part of the target URI, and a request of the second leg names the same `target`)
* 406 Not Acceptable: a `credential_type` the RATS Relying Party does not serve for the `target` (Section 15.5.7 of {{RFC9110}}). The body names the Credential Types it serves for that `target`
* 413 Content Too Large: a request larger than the server takes (Section 15.5.14 of {{RFC9110}})
* 415 Unsupported Media Type: a request not of the resource's media type ({{media-types}}), or Evidence of a format that is not accepted
* 429 Too Many Requests: rate limiting (Section 4 of {{RFC6585}})
* 500 Internal Server Error: internal error
* 502 Bad Gateway: the RATS Relying Party did not answer, or answered with a message the EST Server cannot pass on
* 503 Service Unavailable: the Verifier is unavailable

A client acts on the status code.
A machine-readable body ({{RFC9457}}) is left for a revision in which a client needs the reason.

Error bodies MUST NOT leak sensitive attestation details.
Servers MAY provide a correlation identifier for debugging.

# Security Considerations {#security}

## Specific to Attested Enrollment Mode

* Key Substitution: attestation success is not sufficient without Evidence-to-CSR binding and PoP
* Identity Over-Issuance: Credential Authority policy must constrain subject/SAN to the attested identity context

## Specific to Attested Retrieval Mode

* Shared Signing Key distribution: If the credential bundle includes a private signing key shared across replicas, compromise of one replica compromises the group. This mode SHOULD be restricted to environments where unwrap and key use are strongly protected.
* Non-Exportability Requirements: Deployments that transport a signing key SHOULD require Evidence to attest that CEKpri is non-exportable and that decryption/unwrapping occurs only within an approved protected environment (e.g., TEE/TPM-sealed key usage).
* Origin of the container: the EST Client and the EST Server saw CEKpub and can return a container of their own, sealed to it. The Secret Vault's secret stays confidential: a container the Secret Vault sealed can only be unsealed with CEKpri, and one created by the EST Client or the EST Server will not authenticate the Attester to the Target. This profile accepts that risk, and the Trust Model of {{TACRA}} states what follows by Credential Type.
* Attribution: Shared keys eliminate per-instance attribution. If accountability is required, consider per-instance keys with identical identity claims, or a centralized signing service.

## Common to Both Modes

* Freshness: the Freshness Kinds and the freshness element are those of {{TACRA}}, built on the recency methods of Section 10 of {{RFC9334}}. A `present-*` Handle comes from the Verifier or the RATS Relying Party, a timestamp from the Attester's clock, and an epoch marker from a local hold or a returned Handle. Evidence MUST carry the freshness element in the binding ({{est-binding}}). The originator of a Handle MUST reject `present-nonce` reuse and a stale `present-epoch`.
* Evidence and Attestation Results: the EST Client and the EST Server see Evidence and, in the Passport model, Attestation Results in plaintext (Section 12.2 of {{RFC9334}}, the Trust Model of {{TACRA}}). A retrieved secret stays inside the EncryptedCredentialBundle ({{bundle}}), an enrolled certificate is public, and CSKpri and CEKpri never leave the Attester.
* Channel binding: the digest of {{est-binding}} binds no TLS session and no Credential Authority or Secret Vault. It binds the Target name, so an EST Server can relay the exchange to another RATS Relying Party that serves the same Target, and that party recomputes the same digest. What it obtains, and that Evidence can reach any RATS Relying Party, is in the Trust Model of {{TACRA}}. A TLS channel binding would have to reach the Attester from the EST Client, which this architecture does not assume (Design Goal 8 of {{TACRA}}), so this profile puts none in the Evidence.
* Client authentication: {{TACRA}} leaves the authentication of the EST Client to the EST Server to profiles. This profile requires none for the `attest-*` resources: Remote Attestation authenticates the Attester, and the EST Client holds no credential of its own. A deployment MAY require TLS client authentication on them, as on simpleenroll (Section 3.3 of {{RFC7030}}). An EST Server that mints `present-nonce` Handles on an unauthenticated `attest-initiate` SHOULD rate-limit them and cap the Handles outstanding per client.
* The channels from the EST Server to the Verifier and to the Credential Authority or Secret Vault MUST provide integrity, authenticity, and replay protection.
* The EST Server SHOULD enforce size and rate limits on Evidence.
* The Credential Type and the Credential Hint are not in the binding. The RATS Relying Party decides by its policy what it issues or releases for them. The EST Server's correlation of a Handle with its `attest-initiate` only checks that the second leg repeats the same `credential_type`, and authorizes nothing ({{TACRA}}).
* If classic EST and attested resources both exist, the RATS Relying Party MUST be able to require attestation. The EST Server MUST NOT substitute a classic EST operation for an attested request.

# Privacy Considerations {#privacy}

Evidence can identify the device, its firmware level and the launched software (Section 11 of {{RFC9334}}).
The EST Client, the EST Server, the Verifier and the RATS Relying Party see it.
The Trust Model of {{TACRA}} states what the EST Client and the EST Server read, and a deployment that operates them accepts that exposure.
Error bodies MUST NOT echo Evidence or Attestation Results.

# IANA Considerations {#iana}

The `attest-*` resources are additional services of an EST server (Section 3.2.2 of {{RFC7030}}), under the well-known URI suffix "est" that {{RFC7030}} registered.
No registry of EST operation paths exists, so none is requested.

IANA is requested to register the media types of {{media-types}} in the "Media Types" registry ({{RFC6838}}).

The operation names this document adds are `attest-initiate`, `attest-enroll` and `attest-retrieve`.

## application/est-attest-initiate+json

Type name:
: application

Subtype name:
: est-attest-initiate+json

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary

Security considerations:
: See {{security}} of this document.

Interoperability considerations:
: The response of `attest-initiate`.

Published specification:
: This document.

Applications that use this media type:
: EST implementations of the `attest-*` resources of this document.

Fragment identifier considerations:
: The syntax and semantics of fragment identifiers are as specified for "application/json".
At publication of this specification, no fragment identification syntax is defined for "application/json".

Additional information:
: Deprecated alias names for this type: N/A; Magic number(s): N/A; File extension(s): N/A; Macintosh file type code(s): N/A

Person & email address to contact for further information:
: Mark F. Novak <mark.f.novak@jpmchase.com>

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: See the Authors' Addresses section of this document.

Change controller:
: IESG

## application/est-attest-enroll+json

Type name:
: application

Subtype name:
: est-attest-enroll+json

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary

Security considerations:
: See {{security}} of this document.

Interoperability considerations:
: The request of `attest-enroll`; its response is that of simpleenroll.

Published specification:
: This document.

Applications that use this media type:
: EST implementations of the `attest-*` resources of this document.

Fragment identifier considerations:
: The syntax and semantics of fragment identifiers are as specified for "application/json".
At publication of this specification, no fragment identification syntax is defined for "application/json".

Additional information:
: Deprecated alias names for this type: N/A; Magic number(s): N/A; File extension(s): N/A; Macintosh file type code(s): N/A

Person & email address to contact for further information:
: Mark F. Novak <mark.f.novak@jpmchase.com>

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: See the Authors' Addresses section of this document.

Change controller:
: IESG

## application/est-attest-retrieve+json

Type name:
: application

Subtype name:
: est-attest-retrieve+json

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary

Security considerations:
: See {{security}} of this document.

Interoperability considerations:
: The request of `attest-retrieve`; its response is `application/est-attest-bundle+json`.

Published specification:
: This document.

Applications that use this media type:
: EST implementations of the `attest-*` resources of this document.

Fragment identifier considerations:
: The syntax and semantics of fragment identifiers are as specified for "application/json".
At publication of this specification, no fragment identification syntax is defined for "application/json".

Additional information:
: Deprecated alias names for this type: N/A; Magic number(s): N/A; File extension(s): N/A; Macintosh file type code(s): N/A

Person & email address to contact for further information:
: Mark F. Novak <mark.f.novak@jpmchase.com>

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: See the Authors' Addresses section of this document.

Change controller:
: IESG

## application/est-attest-bundle+json

Type name:
: application

Subtype name:
: est-attest-bundle+json

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary

Security considerations:
: See {{security}} of this document.

Interoperability considerations:
: The response of `attest-retrieve`, the EncryptedCredentialBundle ({{bundle}}).

Published specification:
: This document.

Applications that use this media type:
: EST implementations of the `attest-*` resources of this document.

Fragment identifier considerations:
: The syntax and semantics of fragment identifiers are as specified for "application/json".
At publication of this specification, no fragment identification syntax is defined for "application/json".

Additional information:
: Deprecated alias names for this type: N/A; Magic number(s): N/A; File extension(s): N/A; Macintosh file type code(s): N/A

Person & email address to contact for further information:
: Mark F. Novak <mark.f.novak@jpmchase.com>

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: See the Authors' Addresses section of this document.

Change controller:
: IESG

--- back

# Example Messages {#examples}

These examples are informative.
They are exchanges recorded on 29 October 2026 between an EST Client and an EST Server that implement this profile, on an AMD SEV-SNP guest, with the mandatory CEK suite.
The Targets, the Handles and the structures are as they crossed the wire.
The base64url values are cut to their first 24 characters and marked `...`, and the Host header names the EST Server as this document's examples do.
Lines longer than 69 characters are folded as {{RFC8792}} describes.

## attest-initiate for Enrollment {#ex-initiate-enroll}

~~~
NOTE: '\' line wrapping per RFC 8792

GET /.well-known/est/attest-initiate?credential_type=x509&target=htt\
    ps%3A%2F%2Fdb-nonce.tacra.example HTTP/1.1
Host: est.tacra.example
Accept: application/est-attest-initiate+json

HTTP/1.1 200 OK
Content-Type: application/est-attest-initiate+json

{
  "freshness_kind": "present-nonce",
  "handle": "JguWe4G_4VRGo0tZY4URL8xT...",
  "expires_in": 300,
  "acceptable_evidence": [
    "application/vnd.veraison.tsm-report+cbor"
  ],
  "acceptable_csk": [
    "ecdsa-p256-sha256"
  ],
  "mode": "enroll"
}
~~~

## attest-enroll {#ex-enroll}

~~~
POST /.well-known/est/attest-enroll HTTP/1.1
Host: est.tacra.example
Content-Type: application/est-attest-enroll+json

{
  "target": "https://db-nonce.tacra.example",
  "credential_type": "x509",
  "freshness_kind": "present-nonce",
  "handle": "JguWe4G_4VRGo0tZY4URL8xT...",
  "csr": "MIHkMIGMAgEAMCoxKDAmBgNV...",
  "evidence": "o2dhdXhibG9iWRKbY9p1jeZk...",
  "binding": {
    "method": "binding-input",
    "hash": "sha512"
  },
  "profile": "application/vnd.veraison.tsm-report+cbor",
  "credential_hint": "payments.workload.tacra.example"
}

HTTP/1.1 200 OK
Content-Type: application/pkcs7-mime; smime-type=certs-only
Content-Transfer-Encoding: base64

MIICswYJKoZIhvcNAQcCoIICpDCCAqACAQExADAL...
~~~

## attest-initiate for Retrieval {#ex-initiate-retrieve}

~~~
NOTE: '\' line wrapping per RFC 8792

GET /.well-known/est/attest-initiate?credential_type=api-key&target=\
    https%3A%2F%2Fledger-nonce.tacra.example HTTP/1.1
Host: est.tacra.example
Accept: application/est-attest-initiate+json

HTTP/1.1 200 OK
Content-Type: application/est-attest-initiate+json

{
  "freshness_kind": "present-nonce",
  "handle": "JRAAJxYwTJ-tsqU1Kqk_PQpA...",
  "expires_in": 300,
  "acceptable_evidence": [
    "application/vnd.veraison.tsm-report+cbor"
  ],
  "acceptable_cek": [
    "DHKEM(P-256, HKDF-SHA256)+HKDF-SHA256+AES-256-GCM"
  ],
  "mode": "retrieve"
}
~~~

## attest-retrieve {#ex-retrieve}

~~~
NOTE: '\' line wrapping per RFC 8792

POST /.well-known/est/attest-retrieve HTTP/1.1
Host: est.tacra.example
Content-Type: application/est-attest-retrieve+json

{
  "target": "https://ledger-nonce.tacra.example",
  "credential_type": "api-key",
  "freshness_kind": "present-nonce",
  "handle": "JRAAJxYwTJ-tsqU1Kqk_PQpA...",
  "cek_pub": "MFkwEwYHKoZIzj0CAQYIKoZI...",
  "evidence": "o2dhdXhibG9iWRKbY9p1jeZk...",
  "binding": {
    "method": "binding-input",
    "hash": "sha512"
  },
  "profile": "application/vnd.veraison.tsm-report+cbor",
  "credential_hint": "payments.workload.tacra.example"
}

HTTP/1.1 200 OK
Content-Type: application/est-attest-bundle+json

{
  "container": "hpke-base",
  "suite": {
    "kem": "DHKEM(P-256, HKDF-SHA256)",
    "kdf": "HKDF-SHA256",
    "aead": "AES-256-GCM"
  },
  "enc": "BFR-7sH5nfSJ7HuGs5K0dydE...",
  "ciphertext": "FrhDDRwvpBSZgk-1tjX-IBHR...",
  "aad": {
    "group_id": "0156b1ec5fc72a6ccc6c04b33e23cb114c9e19979cd3fe68bdd\
    7e9495544a46c",
    "target": "https://ledger-nonce.tacra.example",
    "handle": "JRAAJxYwTJ-tsqU1Kqk_PQpA...",
    "credential_hint": "payments.workload.tacra.example"
  }
}
~~~


# Acknowledgments
{:numbered="false"}

The authors thank the Confidential Computing Consortium's Trustworthy Workload Identity (TWI) Special Interest Group ({{TWISIGDef}}, {{TWISIGReq}}) for creating the TACRA architecture.
