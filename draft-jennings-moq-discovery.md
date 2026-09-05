---
title: "DNS and mDNS Discovery for MOQT"
abbrev: "moq-discovery"
docname: draft-jennings-moq-discovery-latest
date: {DATE}
category: std

ipr: trust200902
area: "Web and Internet Transport"
submissionType: IETF
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - svcb
 - https records
 - service discovery
 - srv
 - mdns
 - dns-sd
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "suhasHere/draft-jennings-moq-discovery"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: C. Jennings
    name: Cullen Jennings
    organization: Cisco
    email: fluffy@iii.ca

  -
    ins: S. Nandakumar
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

normative:
  RFC2119:
  RFC8174:
  RFC9460:
  RFC7301:
  RFC3986:
  RFC2782:
  RFC6762:
  RFC6763:
  RFC5890:
  RFC5280:
  RFC9525:
  RFC4985:
  moq-transport: I-D.ietf-moq-transport

informative:
  RFC7838:
  RFC8499:
  RFC6943:
  RFC9110:

--- abstract

This document defines how MOQT clients discover server endpoints using
DNS and Multicast DNS (mDNS).  It specifies SVCB and HTTPS DNS record
mappings for the `moqt` URI scheme, SRV records as a fallback
mechanism, and DNS-SD over mDNS for local network discovery.

--- middle

# Introduction

The Media over QUIC Transport (MOQT) protocol {{moq-transport}} identifies
servers using the `moqt` URI scheme.  Clients establish MOQT sessions over
native QUIC or over WebTransport ({{moq-transport, Section 3}}).

This document defines three complementary discovery mechanisms:

SVCB and HTTPS records ({{svcb}}):
: Unicast DNS records that carry connection parameters — including supported
  ALPNs — alongside address records.  Section 2.4 of {{RFC9460}} requires a
  mapping document for each URI scheme using SVCB; this document fulfills
  that requirement for `moqt`.

SRV records ({{srv}}):
: Unicast DNS records that provide port and target information for load
  balancing and failover.  They carry no ALPN or connection parameters.

mDNS and DNS-SD ({{mdns}}):
: Multicast DNS {{RFC6762}} and DNS Service Discovery {{RFC6763}} for local
  network discovery without a central DNS server.

# Conventions and Definitions

This specification uses the terminology from {{RFC3986}}, {{RFC5280}},
and {{RFC9525}}.

{::boilerplate bcp14-tagged}

# SVCB and HTTPS Records for MOQT {#svcb}

## Record Types

MOQT uses two DNS record types:

SVCB RRs (RR type 64):
: For native QUIC MOQT endpoints.  Published at a `moqt`-specific owner
  name ({{svcb-name}}); carries `moqt`/`moqt-N` ALPNs only.

HTTPS RRs (RR type 65):
: For WebTransport MOQT endpoints.  MAY carry both `moqt` and HTTP ALPN
  (`h3`).  `moqt` URIs using WebTransport map to `https`
  URIs ({{moq-transport, Section 3}}); both schemes default to port 443.

## Record Names {#svcb-name}

### SVCB Record Name

For a `moqt` URI with host `H` and port `P`, the SVCB (type 64) owner
name is:

* `_moqt.H` when `P` is 443 or omitted (defaulting to 443).
* `_P._moqt.H` when `P` is any other value.

### HTTPS Record Name

For a `moqt` URI with host `H` and port `P`, the HTTPS (type 65) owner
name is:

* `H` when `P` is 443 or omitted (defaulting to 443).
* `_P._https.H` when `P` is any other value, as specified in
  {{RFC9460, Section 9.1}}.

## Default ALPN Identifiers {#default-alpn}

Unlike `https` (which implies `h2`/`h3`), the `moqt` URI scheme has no
implicit default ALPN.  Every ServiceMode record MUST include an explicit
`alpn` SvcParamKey listing all supported protocols.

The following ALPN identifiers are defined for MOQT:

`moqt`:
: Native QUIC, published MOQT specification ({{!RFC7301}}).  MUST appear
  in SVCB records; MAY appear in HTTPS records when the server supports
  both native QUIC and WebTransport.

`moqt-N` (where N is a non-negative integer):
: Native QUIC, draft version N (e.g., `moqt-15`).  Ephemeral; MUST NOT
  be used in production.  Same record-placement rules as `moqt`.

`h3`:
: WebTransport over HTTP/3 ({{moq-transport}}).  HTTPS records only.

A client MUST treat a record whose `alpn` contains none of its supported
identifiers as unusable and proceed as if no record were present.

TODO: Define ALPN identifier and record-placement rules for qmux once
the qmux specification matures.

## Automatically Mandatory SvcParamKeys {#mandatory-keys}

The `moqt` scheme introduces no additional automatically mandatory
SvcParamKeys; standard {{RFC9460}} processing rules apply.  Omitting `alpn`
is equivalent to publishing a record with no supported protocols
({{default-alpn}}), which clients will treat as unusable.

## Relevant SvcParamKeys {#svcparams}

### alpn and no-default-alpn

The `alpn` SvcParamKey MUST be present in every ServiceMode record
(see {{default-alpn}}).  The `no-default-alpn` SvcParamKey MUST NOT
appear; with no default ALPNs defined, it has no effect and would be
misleading.

### port

When present, `port` specifies the UDP port (native QUIC or WebTransport
over HTTP/3).  When absent, the port from the `moqt` URI is used,
defaulting to 443.

### ech

ECH {{?RFC9580}} MAY appear in SVCB or HTTPS records.  Clients that
support ECH SHOULD use it; without it, the `moqt` URI authority is
exposed in the TLS SNI extension ({{moq-transport}}).

### ipv4hint and ipv6hint

MAY be used to provide address hints that reduce DNS round trips.
Hints are advisory and do not replace A/AAAA resolution.

## ALPN Selection {#alpn-selection}

A client uses the `alpn` SvcParamKey to determine which connection modes
the server supports and connects using any ALPN identifier from the record
that it supports.  Selection among multiple supported ALPNs is a matter of
local client policy and is out of scope for this document.

# SRV Records for MOQT {#srv}

SRV records {{RFC2782}} provide port and target for load balancing and
failover but carry no ALPN or connection parameters.

## SRV Record Names

For a `moqt` URI with host `H`, SRV queries are performed at:

* `_moqt._udp.H` (both native QUIC and WebTransport over HTTP/3 use UDP).

## Using SRV Target and Port

When SRV records are found:

* The target hostname and port from the SRV record MUST be used as the
  connection endpoint.
* The original `moqt` URI's `host` component MUST be used as the TLS
  SNI value and for certificate validation; it MUST NOT be replaced by
  the SRV target hostname.
* SRV targets with a port of 0 and a dot (`.`) target indicate that the
  service is not available at this name and MUST be treated as indicating
  no service.

## Interaction with SVCB

When SVCB (type 64) or HTTPS (type 65) records are available and usable
for a `moqt` authority, clients SHOULD prefer them over SRV records and
MAY skip the SRV query entirely.

Operators SHOULD publish SVCB and/or HTTPS records rather than (or in
addition to) SRV records for new deployments.

# mDNS and DNS-SD Discovery for MOQT {#mdns}

mDNS {{RFC6762}} and DNS-SD {{RFC6763}} enable MOQT discovery on local
links without a central DNS server.

## DNS-SD Service Names {#mdns-names}

MOQT uses the following DNS-SD service types, which are also registered
for SRV use ({{iana-service}}):

`_moqt._udp`:
: Advertises MOQT endpoints (native QUIC and WebTransport over HTTP/3).

A MOQT relay on the local network announces itself by publishing PTR,
SRV, and TXT records under the appropriate service type in the `.local.`
domain, as specified in {{RFC6763}}.

## TXT Record Parameters {#mdns-txt}

The TXT record for a MOQT DNS-SD instance MUST contain the following
key-value pair:

`alpn`:
: The ALPN identifier for the transport mode advertised by this instance
  (e.g., `moqt`, `moqt-15`, `h3`).

A client MUST treat an instance whose TXT record does not contain a
recognized `alpn` value as unusable.

# URI Normalization

For comparison purposes, the URI, or part of the URI, is put in a
canonical form and then bitwise compared to the data from the X.509
certificate after putting it into canonical form. This section defines
how to create the canonical form.

The URI MUST be normalized using Case Normalization and Percent-Encoding
Normalization as specified in Sections 6.2.2.1 and 6.2.2.2 of
{{RFC3986}}.

The "." and ".." sequences have no special meaning in MOQT URIs and are
not changed during normalization.

If the port is missing, it MUST be replaced with the default port (443).

If the URI reg-name ends in a ".", that MUST be removed.

Internationalized Domain Names in the reg-name MUST be converted to IDNA
format as defined in {{RFC5890}}.

When comparing hostnames, wildcard matching MUST NOT be supported and
the "*" character has no special meaning.


# X.509 Certificate Matching

The fields referenced in this section are defined in {{RFC5280}} and
{{RFC4985}}.

The CN-ID field as defined in {{RFC9525}} MUST NOT be used.

If any one of the following identifiers in the subjectAltName of the
certificate matches the URI, then the certificate is valid for that URI.

* dNSName: The canonicalized dNSName from the certificate matches the
  host part of the authority from the canonicalized URI.

* iPAddress: The host part of the authority from the canonicalized URI,
  parsed as an IP address, matches the iPAddress value from the
  certificate.

* uniformResourceIdentifier: The canonicalized uniformResourceIdentifier
  MUST match the canonicalized URI with the query and fragment removed.

* SRVName: Matched as described in Section 4 of {{RFC4985}} using the
  host part of the authority from the canonicalized URI as the name
  restriction and a SRVName restrictions of "_moqt".

## SVCB and SRV Indirection

When a client is redirected to a different target host, certificate
matching MUST be performed against the authority from the original
`moqt` URI, not the resolved target name.  This is consistent with the requirements in
{{RFC9460, Section 2.3}} and with the SRV rules in {{srv}} of this
document.

SRVName matching ({{RFC4985}}) applies regardless of whether the client
discovered the endpoint via SRV records or SVCB records, because the
`_moqt` service type is the same in both cases.

# Security Considerations {#security-considerations}

TODO

# IANA Considerations

## Service Name Registration {#iana-service}

IANA is requested to register the following entry in the "Service Name
and Transport Protocol Port Number Registry":

* Service Name: moqt
* Transport Protocol(s): udp
* Assignee: IETF
* Contact: moq@ietf.org
* Description: Media over QUIC Transport
* Reference: This document and {{moq-transport}}
* Port Number: 443

This registration covers use of the `_moqt._udp` service label in SRV
records ({{RFC2782}}) and DNS-SD ({{RFC6763}}).

## ALPN Registration

The ALPN identifier `moqt` is already registered in the "TLS Application-
Layer Protocol Negotiation (ALPN) Protocol IDs" registry by {{moq-transport}}.
No additional registrations are requested by this document.

--- back

# Acknowledgments
{:numbered="false"}

TODO: Acknowledgments to be added.
