---
title: "MOQT URI and Discovery"
abbrev: "moq-uri"
docname: draft-jennings-moq-uri-latest
date: {DATE}
category: std

ipr: trust200902
area: "Web and Internet Transport"
submissionType: IETF
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - moqt uri
 - uri resolution
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
  github: "suhasHere/draft-jennings-moq-uri"

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
  RFC1035:
  RFC2119:
  RFC3596:
  RFC3986:
  RFC5280:
  RFC5890:
  RFC7301:
  RFC8174:
  RFC8615:
  RFC9460:
  RFC2782:
  RFC4985:
  RFC6762:
  RFC6763:
  RFC9525:
  moq-transport: I-D.ietf-moq-transport

informative:
  RFC6943:
  RFC7838:
  RFC8499:
  RFC9110:
  RFC9580:
  WebTransport: I-D.ietf-webtrans-http3

--- abstract

This document defines the `moqt` URI scheme, URI resolution mechanisms,
and discovery methods for the Media over QUIC Transport (MOQT) protocol.
It specifies the URI syntax, fragment identifiers, dereferencing
procedures, normalization rules, and X.509 certificate matching for
`moqt` URIs.  It also defines DNS-based resolution using SVCB and SRV
records, as well as local network discovery via mDNS and DNS-SD.

--- middle

# Introduction

The Media over QUIC Transport (MOQT) protocol {{moq-transport}} identifies
servers using the `moqt` URI scheme.  Clients establish MOQT sessions over
native QUIC or over WebTransport ({{moq-transport, Section 3}}).

This document consolidates the URI definition, resolution, and discovery
aspects of MOQT into a single specification.  It is organized as follows:

MOQT URI Scheme ({{moqt-uri-scheme}}):
: Defines the `moqt` URI syntax, fragment identifiers, and dereferencing
  procedures.

URI Normalization ({{uri-normalization}}):
: Specifies canonical forms for comparison and certificate matching.

X.509 Certificate Matching ({{cert-matching}}):
: Rules for validating server certificates against `moqt` URIs.

SVCB Records ({{svcb}}):
: Unicast DNS records that carry connection parameters — including supported
  ALPNs — alongside address records for native QUIC endpoints.  Section 2.4
  of {{RFC9460}} requires a mapping document for each URI scheme using SVCB;
  this document fulfills that requirement for `moqt`.  For WebTransport,
  standard HTTPS resource record processing applies to the derived `https`
  URI.

SRV Records ({{srv}}):
: Unicast DNS records that provide port and target information for load
  balancing and failover.

mDNS and DNS-SD ({{mdns}}):
: Multicast DNS {{RFC6762}} and DNS Service Discovery {{RFC6763}} for local
  network discovery without a central DNS server.

# Conventions and Definitions

This specification uses the terminology from {{RFC3986}}, {{RFC5280}},
and {{RFC9525}}.

{::boilerplate bcp14-tagged}

# MOQT URI Scheme {#moqt-uri-scheme}

An MOQT server is identified using a URI with the "moqt" scheme.  The "moqt"
URI scheme is defined as follows, using definitions from {{RFC3986}}:

~~~~~~~~~~~~~~~
moqt-URI = "moqt" "://" authority path-abempty [ "?" query ]
~~~~~~~~~~~~~~~

The `authority` portion MUST NOT contain an empty `host` portion.
The `moqt` URI scheme supports the `/.well-known/` path prefix defined in
{{RFC8615}}.

The `moqt` URI scheme follows the generic URI syntax of {{RFC3986}} for
the `authority`, `path-abempty`, and `query` components, including the
use of reserved characters and percent-encoding defined therein.  A `moqt`
URI can be converted to an `https` URI by replacing the scheme (see
{{moq-transport, Section 6.2.1}}), so the `path-abempty` and `query`
components use the same syntax as `https` URIs.

## Fragment Identifiers {#moqt-fragment}

The media type for resources identified by `moqt` URIs is
`application/moqt`.

Fragment identifiers MAY be used with `moqt` URIs.  The fragment is not
transmitted to the server; it is processed locally by the client after
establishing the MOQT session.

A `moqt` URI fragment MUST begin with a registered fragment type
identifier, followed by a colon (`:`), followed by a type-specific value:

~~~
moqt://example.com/app#<type>:<value>
~~~

Fragment type identifiers MUST consist of ASCII lowercase letters,
digits, and hyphens (`a-z`, `0-9`, `-`).  The semantics of the value
after the colon are defined by the specification that registers the
fragment type.

## Dereferencing a MOQT URI {#dereferencing}

The default operation for dereferencing a `moqt` URI is to establish a
MOQT session to the identified server.  See
{{security-considerations}} for the security implications of
dereferencing a `moqt` URI.

A `moqt` URI can be dereferenced over two distinct transports
({{moq-transport, Section 3}}):

* Native QUIC, where the client opens a QUIC connection directly to the
  authority of the `moqt` URI and negotiates a `moqt`/`moqt-N` ALPN.
* WebTransport, where the `moqt` URI is first mapped to an `https` URI
  ({{moq-transport, Section 6.2.1}}) and the client establishes a
  WebTransport session over HTTP/3 (ALPN `h3`) or HTTP/2 (ALPN `h2`)
  to that `https` origin.

The two transports use different DNS resolution and certificate
validation procedures, summarized below.

For the native QUIC transport, when `moqt`-specific SVCB records are
published for the `authority` ({{svcb}}), the client MAY use them to
learn the server's endpoints and `moqt`/`moqt-N` ALPNs before
connecting.  When those records are not available, the client falls
back to SRV records ({{srv}}) or, on a local link, uses mDNS and DNS-SD
({{mdns}}).  Otherwise the client resolves the `host` subcomponent of
the `authority` to one or more network addresses using DNS A
{{RFC1035}} and AAAA {{RFC3596}} records.  If the port is omitted in
the URI, a default port of 443 is used.  The server's X.509
certificate MUST be validated against the original `moqt` URI
authority as described in {{cert-matching}}, and the URI MUST be
normalized as specified in {{uri-normalization}} before certificate
comparison.

For the WebTransport transport, the client applies standard `https`
processing to the derived `https` URI: HTTPS resource records
({{RFC9460}}) are resolved as for any `https` origin, and the server's
X.509 certificate matching following the same procedures as defined 
for native QUIC.  No `moqt`-specific DNS records are
consulted on this path.

# URI Normalization {#uri-normalization}

For comparison purposes, the URI, or part of the URI, is put in a
canonical form and then bitwise compared to the data from the X.509
certificate after putting it into canonical form.  This section defines
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


# X.509 Certificate Matching {#cert-matching}

The fields referenced in this section are defined in {{RFC5280}} and
{{RFC4985}}.

The CN-ID field as defined in {{RFC9525}} MUST NOT be used.

Clients MUST implement dNSName, uniformResourceIdentifier and 
iPAddress matching.

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

## WebTransport Certificate Validation

When connecting via WebTransport, the TLS handshake terminates at an
`https` origin derived from the `moqt` URI ({{moq-transport,
Section 6.2.1}}).  In this case, the client validates the server
certificate against that `https` URI using standard Web PKI procedures;
the `moqt`-specific matching rules above do not apply.  The TLS SNI
extension MUST contain the host from the derived `https` URI.

The `moqt`-specific certificate matching defined in this section applies
only to native QUIC connections where the client connects directly to
the authority of the `moqt` URI.  On such connections the TLS SNI
extension MUST contain the host from the original `moqt` URI.

## SVCB and SRV Indirection

When a client is redirected to a different target host via SVCB or SRV,
certificate matching MUST be performed against the authority from the
original `moqt` URI, not the resolved target name.  The TLS SNI
extension likewise MUST contain the original authority's host.  This is
consistent with the requirements in {{RFC9460, Section 2.3}} and with
the SRV rules in {{srv}} of this document.

SRVName matching ({{RFC4985}}) applies regardless of whether the client
discovered the endpoint via SRV records or SVCB records, because the
`_moqt` service type is the same in both cases.

# SVCB Records for MOQT {#svcb}

MOQT defines SVCB records {{RFC9460}} for native QUIC endpoints.
For WebTransport, a `moqt` URI maps to an `https` URI
({{moq-transport, Section 6.2.1}}); the client resolves that `https`
URI using HTTPS resource records following standard `https` processing
rules ({{RFC9460, Section 9.1}}).

## SVCB Record Name {#svcb-name}

For a `moqt` URI with host `H` and port `P`, the SVCB owner name is:

* `_moqt.H` when `P` is 443 or omitted (defaulting to 443).
* `_P._moqt.H` when `P` is any other value.

## Default ALPN Identifiers {#default-alpn}

Unlike `https` (which implies `h2`/`h3`), the `moqt` URI scheme has no
implicit default ALPN.  Every ServiceMode record MUST include an explicit
`alpn` SvcParamKey listing all supported protocols.

The following ALPN identifiers are defined for MOQT:

`moqt`:
: Connected with raw QUIC, published MOQT specification. 

`moqt-N` (where N is a non-negative integer):
: Connect with raw QUIC using draft version N (e.g., `moqt-15`) of
  the MOQT Transport draft.

Note to RFC Editor: Remove this bullet point in RFC

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

When present, `port` specifies the UDP port for the native QUIC
endpoint.  When absent, the port from the `moqt` URI is used,
defaulting to 443.

### ech

ECH {{?RFC9580}} MAY appear in SVCB records.  Clients that
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
failover but carry no ALPN or connection parameters.  SRV support is
retained because it is required by DNS-SD ({{mdns}}) and provides a
transitional fallback for the native QUIC path in deployments where
SVCB infrastructure is not yet available.

## SRV Record Names

For a `moqt` URI with host `H`, SRV queries are performed at:

* `_moqt._udp.H`

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

When SVCB records are available and usable for a `moqt` authority,
clients SHOULD prefer them over SRV records and MAY skip the SRV query
entirely.

Operators SHOULD publish SVCB records rather than (or in addition to)
SRV records for new deployments.

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

## Interaction with SVCB

This document does not define SVCB-based service parameter delivery
for MOQT DNS-SD instances; the TXT record ({{mdns-txt}}) is the sole
mechanism for conveying ALPN information in DNS-SD.

# Security Considerations {#security-considerations}

Dereferencing a `moqt` URI exposes the following information to on-path
observers and intermediaries:

- The `authority` component is sent in the TLS SNI extension during
  connection establishment, exposing the target server identity to
  on-path observers.  Encrypted Client Hello (ECH) {{?RFC9580}} can
  mitigate this exposure.

- The `path-abempty` and `query` components are visible to the relay
  that terminates the client's connection.

TODO: Expand this section with additional considerations.

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

## URI Scheme Registrations

This document requests the registration of the following URI schemes in the
"Uniform Resource Identifier (URI) Schemes" registry, per {{!RFC7595}}:

### "moqt" URI Scheme Registration

Scheme name: moqt

Status: Permanent

Applications/protocols that use this scheme name: Media over QUIC Transport
(MOQT) over native QUIC or WebTransport, as defined in this document.

Contact: IETF MoQ Working Group (moq@ietf.org)

Change controller: IETF

References: This document

## ALPN Registration

TODO 

--- back

# Acknowledgments
{:numbered="false"}

TODO: Acknowledgments to be added.
