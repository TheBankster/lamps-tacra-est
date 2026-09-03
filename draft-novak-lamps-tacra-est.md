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

This document specifies extensions to Enrollment over Secure Transport (EST, {{!RFC7030}}) to support Remote Attestation Procedures (RATS, {{!RFC9334}}) as an authorization input for workload credential provisioning. Two modes are defined:

1. Attested Credential Enrollment Mode: a workload submits a PKCS#10 Certificate Signing Request (CSR) and Remote Attestation Evidence. The EST server (acting as a RATS Relying Party) authorizes issuance based on attestation and cryptographically binds the issued credential to the CSR key through proof-of-possession and explicit key-binding.
2. Attested Credential Retrieval Mode: a workload submits Remote Attestation Evidence that includes an asymmetric Credential Encryption Key (CEK). Upon successful verification and authorization, the EST server returns an existing shared public credential bundle (e.g., an X.509 certificate, a WIMSE WIC), and a secret, such as an associated signing key, a bearer token, or a preshared key, encrypted to the CEK.

These extensions add new EST resources, request/response envelopes, processing rules, and security requirements for replay protection, freshness, key-binding, confidentiality, auditability, and key distribution risk management.

--- middle

# Introduction {#intro}

EST ({{!RFC7030}}) defines an HTTPS-based protocol for certificate enrollment and management, typically between an EST client and an EST server acting as an interface to a Certification Authority (CA). In modern environments (e.g., cloud, containers, confidential computing), workloads often lack pre-provisioned credentials and require issuance based on runtime properties. Additionally, zero trust environments place additional restrictions limiting which parties have access to secrets and credentials. This means that EST clients and servers may not be trusted to handle such restricted information in plaintext.

RATS ({{!RFC9334}}) defines an architecture and roles for remote attestation. This document integrates RATS with EST by making an EST server a conduit for credential issuance or credential release decisions based on verified attestation.

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

{::include attested_enrollment.txt}

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

{::include attested_retrieval.txt}

1. Attester initiates Remote Attestation Challenge
2. EST client forwards Remote Attestation Challenge to EST server
3. EST server forwards Remote Attestation Challenge to Verifier, which generates the Challenge and returns it to the Attester via EST server and EST client
4. Attester sends Evidence (including CEKpub) to EST client
5. EST client forwards the Evidence to EST server
6. EST server forwards the Evidence to Verifier and obtains Attestation Results, which also include CEKpub
7. EST server sends the Attestation Results to the Key or Credential Store and obtains the credentials and associated secrets, encrypted to CEKpub, which it returns to the Attester via EST server and EST client

# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
