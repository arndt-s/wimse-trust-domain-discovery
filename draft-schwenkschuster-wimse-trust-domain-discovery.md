---
title: "WIMSE Trust Domain Discovery"
abbrev: "WIMSE Trust Domain Discovery"
category: std

docname: draft-schwenkschuster-wimse-trust-domain-discovery-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
area: "Applications and Real-Time"
workgroup: "Workload Identity in Multi System Environments"
keyword:
 - workload
 - identity
 - trust domain
 - discovery
venue:
  group: "Workload Identity in Multi System Environments"
  type: "Working Group"
  mail: "wimse@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/wimse/"

author:
 -
    fullname: "Arndt Schwenkschuster"
    organization: Defakto Security
    email: arndts.ietf@gmail.com

normative:
  RFC7517:
  RFC7518:
  RFC8615:
  RFC9110:
  RFC9525:

informative:
  RFC8414:
  RFC9364:
  IANA.MEDIA.TYPES: IANA.media-types
  IANA.WELL.KNOWN: IANA.well-known-uris
  SPIFFE-FEDERATION:
    title: The SPIFFE Federation Standard
    target: https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Federation.md
    author:
      org: CNCF SPIFFE Project
  OIDC-DISCOVERY:
    title: OpenID Connect Discovery 1.0
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
    author:
      org: OpenID Foundation

--- abstract

The WIMSE architecture scopes workload identifiers to a trust domain. To validate a workload identity credential, a relying party must securely obtain the cryptographic trust anchors associated with the credential's trust domain. The WIMSE architecture requires that this mapping between a trust domain and its trust anchors is obtained through a secure mechanism, but leaves that mechanism undefined.

This document defines such a mechanism for open environments in which no prior trust relationship exists between the relying party and the trust domain. It specifies the WIMSE Trust Bundle, a document that conveys the trust anchors of a trust domain for both JWT-based and X.509-based workload identity credentials, and a discovery mechanism based on a well-known HTTPS endpoint that allows a relying party to resolve a trust domain name to its trust bundle on demand.

--- middle

# Introduction

Workload identity credentials, such as the Workload Identity Token (WIT) and the Workload Identity Certificate (WIC) defined in {{!WIMSE-CREDS=I-D.ietf-wimse-workload-creds}}, carry a Workload Identifier {{!WIMSE-ID=I-D.ietf-wimse-identifier}} that is scoped to a trust domain. A relying party that receives such a credential can only validate it if it possesses the trust anchors of the issuing trust domain: a set of JWT signing keys for WITs, or a set of certification authority (CA) certificates for WICs.

The WIMSE architecture {{?I-D.ietf-wimse-arch}} states that a trust domain maps to one or more trust anchors and that this mapping "MUST be obtained through a secure mechanism that ensures the authenticity and integrity of the mapping is fresh and not compromised", while declaring the mechanism itself out of scope. In closed environments this mapping is typically provisioned out of band, for example through static federation configuration or through a local workload API. In open environments, where workloads from previously unknown trust domains present credentials, no such prior provisioning exists. Today a relying party in this situation has no interoperable way to obtain the trust anchors for a trust domain such as `example.com` and therefore cannot validate the credential at all.

This document closes that gap with two components:

* The WIMSE Trust Bundle ({{trust-bundle}}), a JSON document that conveys the trust anchors of a trust domain for both WIT and WIC validation, together with freshness metadata.

* A discovery mechanism ({{discovery}}) that derives, from the trust domain name alone, the location of a Trust Domain Metadata document served from a well-known URI {{RFC8615}} under the trust domain name. The metadata document in turn references the trust bundle endpoint. This follows the well-established pattern of OpenID Connect Discovery {{OIDC-DISCOVERY}} and OAuth 2.0 Authorization Server Metadata {{RFC8414}}.

The mechanism defined in this document authenticates the mapping between a trust domain name and its trust anchors. It deliberately does not establish that the trust domain itself should be trusted; see {{discovery-not-trust}}.

## Relationship to Existing Mechanisms

The trust bundle defined in this document is conceptually similar to the SPIFFE trust bundle and the SPIFFE bundle endpoint {{SPIFFE-FEDERATION}}. SPIFFE Federation, however, requires that the bundle endpoint URL, endpoint profile, and trust domain name are configured out of band prior to bundle retrieval. This document removes that requirement by making the location of the trust domain's metadata derivable from the trust domain name itself. Deployments already operating a SPIFFE bundle endpoint can expose the same key material through the mechanism defined here.

Existing provisioning mechanisms remain valid and take precedence over discovery; see {{precedence}}.

# Conventions and Definitions

All terminology in this document follows {{?I-D.ietf-wimse-arch}} and {{WIMSE-ID}}.

Relying Party:
: An entity that receives a workload identity credential and needs to validate it. This corresponds to the Consumer role in {{WIMSE-ID}}.

Discoverable Trust Domain:
: A trust domain that participates in the discovery mechanism defined in this document.

{::boilerplate bcp14-tagged}

# Requirements on Discoverable Trust Domains {#requirements}

{{WIMSE-ID}} states that a trust domain SHOULD be a fully qualified domain name (FQDN) belonging to the organization defining the trust domain. This document strengthens that requirement for participation in discovery:

* A Discoverable Trust Domain MUST be a fully qualified DNS name.

* The DNS name MUST be under the administrative control of the trust domain owner, and the owner MUST be able to serve HTTPS content for that exact name.

Trust domains that do not meet these requirements (for example, names that are not registered in the public DNS) are simply not discoverable and MUST rely on out-of-band provisioning of their trust anchors. This document does not change the requirements of {{WIMSE-ID}} for such trust domains.

Note that these requirements are largely self-enforcing: an entity that does not control the DNS name of a trust domain cannot serve the corresponding well-known resource, and discovery for that trust domain will either fail or return material published by the legitimate name owner.

Discovery operates on the exact trust domain name only. A relying party MUST NOT fall back to a parent domain if discovery for the trust domain name fails. For example, discovery for the trust domain `prod.example.com` is performed against `prod.example.com` only, never against `example.com`. This prevents a parent zone or parent host operator from implicitly answering for trust domains it does not operate.

# The WIMSE Trust Bundle {#trust-bundle}

A WIMSE Trust Bundle conveys the trust anchors of a single trust domain. It is a JSON document based on the JWK Set format {{RFC7517}} with additional top-level members for freshness. The media type of a trust bundle is `application/wimse-trust-bundle+json`.

A trust bundle contains the following top-level members:

* `keys`: REQUIRED. A JWK Set `keys` array as defined in {{Section 5.1 of RFC7517}}. The interpretation of each JWK is determined by its `use` member as defined below.

* `refresh_hint`: OPTIONAL. A JSON number indicating, in seconds, how often a consumer SHOULD re-fetch the bundle. See {{freshness}}.

* `sequence_number`: OPTIONAL. A strictly monotonically increasing JSON integer, incremented whenever the contents of the bundle change. Consumers use this value to compare the recency of two bundles.

## Bundle Entries

Each element of the `keys` array is a JWK {{RFC7517}} and MUST contain a `use` member with one of the following values:

* `wimse-jwt`: The entry carries a public key used to verify the signature of JWT-based workload identity credentials (such as the WIT defined in {{WIMSE-CREDS}}) issued by this trust domain. The key type and parameters follow {{RFC7517}} and {{RFC7518}}. Entries with this use value SHOULD contain a `kid` member to support key selection during rotation.

* `wimse-x509`: The entry carries a CA certificate used as a trust anchor for validating X.509-based workload identity credentials (such as the WIC defined in {{WIMSE-CREDS}}) issued by this trust domain. The entry MUST contain an `x5c` member whose value contains exactly one base64-encoded DER certificate. The certificate SHOULD be self-signed. The JWK key parameters MUST represent the public key of that certificate.

Consumers MUST ignore entries with an unrecognized `use` value. A trust domain that issues only one credential type publishes only the corresponding entries.

## Freshness and Rotation {#freshness}

Trust bundle contents change over time as trust domains rotate their keys. Publishers MUST publish new keys in the bundle before using them to issue credentials, and SHOULD do so sufficiently in advance that consumers refreshing at the `refresh_hint` interval obtain the new keys before they are needed.

Consumers SHOULD re-fetch the bundle at the interval indicated by `refresh_hint`, or at a locally configured default interval if the hint is absent. Consumers MUST NOT accept a newly fetched bundle whose `sequence_number` is lower than that of the currently cached bundle for the same trust domain.

Removal of a key from the bundle indicates that the key is no longer a valid trust anchor. Consumers MUST stop using removed keys after refresh. The refresh interval therefore bounds the time during which a compromised and subsequently removed key is still accepted; see {{sec-revocation}}.

# Trust Domain Discovery {#discovery}

## Trust Domain Metadata {#metadata}

A Discoverable Trust Domain publishes a Trust Domain Metadata document. The media type of this document is `application/wimse-trust-domain-metadata+json`. It is a JSON object with the following members:

* `trust_domain`: REQUIRED. The trust domain name this metadata describes.

* `trust_bundle_endpoint`: REQUIRED. An HTTPS URL from which the current WIMSE Trust Bundle ({{trust-bundle}}) of this trust domain can be retrieved.

Additional members MAY be present and MUST be ignored if not understood. Future specifications may register additional metadata members, for example to describe supported credential types or federation policy.

The `trust_bundle_endpoint` URL is not required to share an origin with the trust domain name. Its authority derives from the authenticated metadata document that references it, analogous to the `jwks_uri` member of {{RFC8414}}. This permits serving the bundle from shared infrastructure such as a content delivery network.

## Well-Known URI {#well-known}

The Trust Domain Metadata document is retrieved from the following URL, constructed from the trust domain name:

~~~
https://<trust-domain>/.well-known/wimse-trust-domain
~~~

For example, for the trust domain `prod.example.com` the metadata URL is:

~~~
https://prod.example.com/.well-known/wimse-trust-domain
~~~

The path suffix `wimse-trust-domain` is registered in the "Well-Known URIs" registry; see {{iana-well-known}}.

## Discovery Procedure {#procedure}

To discover the trust anchors of a trust domain, a relying party performs the following steps:

1. Verify that the trust domain name is a syntactically valid fully qualified DNS name. If it is not, discovery fails.

2. Construct the metadata URL as defined in {{well-known}} and retrieve it with an HTTP GET request {{RFC9110}} over TLS. The TLS server certificate MUST be validated according to {{RFC9525}} against the trust domain name, using the relying party's Web PKI trust store. The request MUST NOT require client authentication.

3. Parse the response as a Trust Domain Metadata document ({{metadata}}). The relying party MUST verify that the `trust_domain` member is byte-wise identical to the trust domain name for which discovery was initiated. If it is not, discovery fails. This check prevents mix-up attacks in which a document intended for one trust domain is served for another.

4. Retrieve the trust bundle from the `trust_bundle_endpoint` URL with an HTTP GET request over TLS. The TLS server certificate MUST be validated according to {{RFC9525}} against the host component of the `trust_bundle_endpoint` URL, using the relying party's Web PKI trust store.

5. Parse the response as a WIMSE Trust Bundle ({{trust-bundle}}) and associate its contents with the trust domain name from step 1 as that trust domain's trust anchors, subject to the precedence rules of {{precedence}} and local policy ({{discovery-not-trust}}).

Servers MAY respond with HTTP redirects at either step; clients following redirects MUST apply the respective TLS validation rules of the step to every URL in the redirect chain, and all URLs in the chain MUST use the `https` scheme.

If any step fails, discovery for the trust domain fails and the relying party MUST NOT use partially retrieved material. A discovery failure is indistinguishable from the trust domain not participating in discovery; in either case, credentials from that trust domain cannot be validated through this mechanism and are handled according to local policy (for example, validated against out-of-band configured anchors, or rejected).

## Precedence of Locally Provisioned Trust {#precedence}

Deployments frequently provision trust anchors through existing mechanisms, such as a local workload API, static federation configuration, or the mechanisms described in {{WIMSE-CREDS}}. These mechanisms take precedence over discovery:

* If a relying party has trust anchors for a trust domain provisioned through a local mechanism, it MUST use those anchors and MUST NOT perform discovery for that trust domain.

* Discovered trust anchors MUST NOT override, replace, or be merged with locally provisioned trust anchors for the same trust domain.

* If an implementation nevertheless finds itself holding both locally provisioned and discovered material for the same trust domain, this condition SHOULD be surfaced as an operational error.

This rule ensures that a workload operating inside a trust domain, whose trust anchors are delivered through a local channel, never resolves its own or a locally federated trust domain through the public mechanism defined here, even if discovery is enabled by default. Implementations MAY enable discovery by default; the precedence rule bounds the effect of misconfiguration to trust domains for which no local trust exists.

# Security Considerations

## Discovery Is Not Trust {#discovery-not-trust}

Successful discovery proves that the retrieved trust anchors were published by the entity controlling the trust domain's DNS name and its Web PKI identity. It does not establish that credentials from that trust domain should be accepted for any purpose. The decision whether to trust a given trust domain, and for what, is an authorization decision of the relying party and is out of scope for this document. Deployments may implement this decision through allowlists, trust-on-first-use with key continuity, gradual trust establishment, or any other local policy.

Relying parties MUST NOT treat the mere existence of a discoverable trust bundle as authorization to accept credentials from that trust domain.

Conversely, an attacker that mints credentials carrying a trust domain name it does not control gains nothing from this mechanism: discovery for that name either fails or returns the legitimate owner's trust anchors, under which the forged credentials do not verify.

## Reliance on DNS and the Web PKI {#sec-webpki}

This mechanism anchors the trust domain to trust anchor mapping in the DNS and the Web PKI. An attacker that can obtain a fraudulently issued Web PKI certificate for the trust domain name, or that gains control of the DNS name (for example through registrar compromise or domain expiry and re-registration), can publish attacker-controlled trust anchors and thereby impersonate every workload in the trust domain towards relying parties that bootstrap trust through discovery.

This trade-off is inherent to the design and identical to that accepted by OpenID Connect Discovery. Publishers SHOULD employ standard mitigations, including CAA records, Certificate Transparency monitoring, DNSSEC {{RFC9364}} for their zones, and registrar security controls. Deployments with stronger requirements can restrict discovery to specific trust domains, pin discovered material, or continue to use out-of-band provisioning, which always takes precedence ({{precedence}}).

After first discovery, consumers retain the cached bundle and its `sequence_number`. Implementations MAY apply key-continuity policies across refreshes, for example flagging bundle replacements that discard all previously seen keys, to reduce the impact of a temporary compromise of the discovery channel.

## Revocation and Compromise Recovery {#sec-revocation}

This mechanism has no online revocation protocol. The removal of a compromised key from the published bundle propagates to relying parties only upon refresh. The effective revocation latency is therefore the consumers' refresh interval. Publishers SHOULD choose a `refresh_hint` that reflects their tolerance for this latency, and consumers SHOULD honor it. Trust domains SHOULD additionally keep the lifetime of issued credentials short, as recommended in {{WIMSE-CREDS}}, so that both the credential lifetime and the bundle refresh interval bound the exposure after a compromise.

## Trust Domain Name Handling

The trust domain name used for discovery is taken from an unauthenticated position, namely the authority component of the workload identifier in a not-yet-validated credential. Implementations MUST treat it as untrusted input: it MUST be parsed with a standards-compliant URI parser as required by {{WIMSE-ID}}, and MUST be validated as a DNS name before any network request is made. Implementations MUST NOT perform discovery for trust domains represented by IP addresses.

## Denial of Service

Because discovery is triggered by inbound credentials, an attacker can present credentials with arbitrary trust domain names to induce outbound requests. Implementations SHOULD rate-limit discovery attempts, cache negative results, and bound concurrent discovery operations.

# Privacy Considerations

Performing discovery reveals to the network, to DNS resolvers, and to the operators of the contacted endpoints that the relying party is processing credentials of the queried trust domain. Deployments for which this metadata is sensitive can pre-fetch and cache bundles for expected peer trust domains, use encrypted DNS transports, or restrict discovery to an allowlist.

Publishing a trust bundle reveals the existence of the trust domain and its current trust anchor material. Publishers should not include information in bundle or metadata documents beyond what is required for validation.

# IANA Considerations

## Well-Known URI Registration {#iana-well-known}

IANA is requested to register the following entry in the "Well-Known URIs" registry {{IANA.WELL.KNOWN}}:

* URI Suffix: wimse-trust-domain
* Change Controller: IETF
* Specification Document: RFC XXX, {{well-known}}
* Status: permanent

## Media Type Registration

IANA is requested to register the following entries in the "Media Types" registry {{IANA.MEDIA.TYPES}}:

* application/wimse-trust-domain-metadata+json, per {{metadata}}.
* application/wimse-trust-bundle+json, per {{trust-bundle}}.

Registration templates to be completed in a future revision of this document.

## JSON Web Key Use Values

IANA is requested to register the values `wimse-jwt` and `wimse-x509` in the "JSON Web Key Use" registry established by {{RFC7517}}, per {{trust-bundle}}.

--- back

# Document History
<cref>RFC Editor: please remove before publication.</cref>

## draft-schwenkschuster-wimse-trust-domain-discovery-00

* Initial version.
* Define the WIMSE Trust Bundle format (JWK Set with `wimse-jwt` and `wimse-x509` use values, refresh hint, sequence number).
* Define discovery through a Trust Domain Metadata document at the `/.well-known/wimse-trust-domain` URI, referencing the trust bundle endpoint.
* Require Discoverable Trust Domains to be DNS names under the owner's control; exact-match resolution only.
* Normative precedence of locally provisioned trust anchors over discovered ones.
* Security considerations: discovery is not trust, Web PKI/DNS reliance, revocation latency, untrusted input handling, denial of service.

# Acknowledgments
{:numbered="false"}

TBD.