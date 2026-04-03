---
title: "HTTP Header Fields for Proxying DNS SVCB Information"
category: std

docname: draft-gakiwate-dnsop-proxy-dns-svcb-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Domain Name System Operations"
keyword:
venue:
  group: "Domain Name System Operations"
  type: "Working Group"
  mail: "dnsop@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dnsop/"
  github: "gakiwate/draft-gakiwate-dnsop-proxy-dns-svcb"
  latest: "https://gakiwate.github.io/draft-gakiwate-dnsop-proxy-dns-svcb/draft-gakiwate-dnsop-proxy-dns-svcb.html"

author:
 -
    fullname: Gautam Akiwate
    organization: Apple Inc
    email: "gakiwate@apple.com"
 -
    fullname: Tommy Pauly
    organization: Apple Inc
    email: "tpauly@apple.com"

normative:

informative:

...

--- abstract


This document defines a mechanism for utilizing Service Binding (SVCB and HTTPS)
DNS records in environments where clients connect to application servers via
HTTP proxies. In such deployments, the proxy, rather than the client, performs
the DNS resolution. To support this model, this document specifies a set of HTTP
header fields that enable proxies to convey SVCB / HTTPS RRs received from the
resolution.  This mechanism allows clients to benefit from SVCB-based service
discovery and configuration (e.g., alternative endpoints, ALPN, or transport
hints) while preserving existing privacy constraints.

--- middle

# Introduction

The CONNECT method {{!RFC7231}} and the CONNECT-UDP method {{!RFC9298}} are HTTP
request methods that allow clients to establish TCP or UDP flows to target
servers via an HTTP proxy. Clients identify the target using authority-form
(Section 5.3 of {{!RFC7230}}), which includes the server name or IP address and
an associated port number. When the target is specified by name, the proxy
resolves the name to an IPv4 or IPv6 address using A or AAAA queries.

However, this arrangement prevents clients from using Service Binding (SVCB or
HTTPS) RRs {{!RFC9460}}, which increasingly carry important configuration
information that influences how connections should be formed.  For example, SVCB
records may indicate that a target server supports `h3`, enabling the client to
use CONNECT-UDP immediately. Similarly, SVCB records may indicate support for
Encrypted Client Hello (`ech`), which could affect how the client initiates the
connection to the target server.

Although clients could, in principle, perform their own SVCB resolution, this is
often impractical in deployed environments due to privacy constraints or
performance considerations. To that end, this document specifies HTTP header
fields that allow proxy servers to relay information obtained from Service
Binding (SVCB and HTTPS) records to clients when handling CONNECT or CONNECT-UDP
requests.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Structured HTTP Header Fields for Proxying SVCB Information

## Client Structured HTTP Header Fields

### Proxy-DNS-Request
Clients can request SVCB parameters with the Structured Header {{!RFC8941}}
"Proxy-DNS-Request". The value MUST be an sf-list whose members are sf-integer
items that MUST NOT contain parameters. Each list member corresponds to the
numeric version of an SvcParamKey. For example, a client wanting to receive ALPN
and ECH Config parameters would send a request for 1 (alpn) and 5 (echconfig).

~~~ requestexample
proxy-dns-request = 1, 5
~~~

### Proxy-DNS-Request-Close-On

The "Proxy-DNS-Request-Close-On" header field is a Structured Header
{{!RFC8941}} that indicate conditions if met should trigger a close on the
request. 

The header value in this case is a list of tokens. This set of tokens should
ideally be pre-defined and agreed upon by the client and the proxy.  In this
document, we define two token values.

~~~ abnf
Proxy-DNS-Request-Close-On= sf-list
~~~

#### Defined Token Values

The following tokens are defined for the "Proxy-DNS-Request-Close-On" header:

h3-alpn: This token indicates that the client requests the proxy should close
the connection if the ALPN (Application-Layer Protocol Negotiation) field in DNS
SVCB/HTTPS records for the target contains the "h3" protocol identifier. A
client may choose to add this token if it prefers CONNECT-UDP over CONNECT. 

ech-defined: This token indicates that client requests that the proxy should
close the connection if the ECH (Encrypted Client Hello) SvcParamKey is present
in DNS SVCB/HTTPS records for the target. A client may choose to add this token
if it prefers to use ECH when connecting to the target.

~~~ tokenexample
proxy-dns-request-close-on: h3-alpn
proxy-dns-request-close-on: ech-defined
proxy-dns-request-close-on: h3-alpn, ech-defined
~~~

### Example

~~~ example
HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-request = 1, 5
proxy-dns-request-close-on: h3-alpn, ech-defined
~~~

In this example, the client is requesting the proxy communicate back
the ALPN and ECH SvcParamKey values of the target's SVCB/HTTPS RRs.
The client is also requesting that if the target supports H3 or ECH
then it should close the connection attempt so that the client may
reattempt.

## Proxy Structured HTTP Header Fields

### Proxy-DNS-Response

A proxy MAY include the "Proxy-DNS-Response" Structured Header {{!RFC8941}} in a
CONNECT or CONNECT-UDP response to convey the SVCB/HTTPS RRset resolved for the
target authority. The "Proxy-DNS-Response" header field is a Structured Field
{{!RFC8941}} of type sf-list. Each member of the list is an sf-inner-list
representing a single SVCB/HTTPS RR, and the list as a whole represents the full
RRset.

Each inner list MUST carry the following parameters:

- **priority**: an sf-integer corresponding to the SvcPriority field of the RR as defined in {{!RFC9460}}.
- **target**: an sf-string corresponding to the TargetName field of the RR as defined in {{!RFC9460}}.

Additionally, every member of the inner list is an sf-integer corresponding to
the numeric identifier of a SvcParamKey as defined in {{!RFC9460}}. Each such
member MAY carry a single parameter whose name is "value" and whose value is an
sf-binary item containing the wire-format encoding of the corresponding
SvcParamValue as defined in {{!RFC9460}}. A SvcParamKey MUST appear at most once
within a given inner list.

For AliasMode records (SvcPriority 0), the inner list MUST be empty, as
AliasMode RRs do not carry SvcParams. The "target" parameter MUST convey the
alias target name.

A proxy SHOULD restrict the SvcParams conveyed in each inner list to only the
SvcParamKeys enumerated in the client's "Proxy-DNS-Request" header field. If the
proxy was unable to perform SVCB resolution, or if resolution returned no
SVCB/HTTPS RRs for the target, the proxy MUST include the header field with an
empty list.

~~~ responseexample
proxy-dns-response = (1;value=:aGkK: 5;value=:ACAgAA...:);priority=1;target="svc1.example.com",
                     (1;value=:bmkK:);priority=2;target="svc2.example.com"
~~~

~~~ responseexample
proxy-dns-response = ();priority=0;target="alias.example.com"
~~~

~~~ responseexample
proxy-dns-response =
~~~

Clients MUST NOT rely on the presence of specific SvcParamKeys in the response.
If the header field is absent, the client SHOULD treat the SVCB/HTTPS parameter
set for the target as unknown.

### Proxy-DNS-Response-Closed-On

The "Proxy-DNS-Response-Closed-On" header field is a Structured Field
{{!RFC8941}} of type sf-list. Each member of the list is an sf-token
corresponding to a token defined in Section XX (Defined Token Values) that was
present in the client's "Proxy-DNS-Request-Close-On" header field and whose
condition was evaluated as satisfied by the proxy.

A proxy MUST include this header field when it closes a request in response to a
satisfied condition from the client's "Proxy-DNS-Request-Close-On" header field.
The proxy MUST report all tokens whose conditions were satisfied. The proxy MUST
NOT include tokens that were absent from the client's
"Proxy-DNS-Request-Close-On" request, and MUST NOT include tokens whose
conditions were not satisfied.

~~~ responseexample
proxy-dns-response-closed-on: h3-alpn
proxy-dns-response-closed-on: ech-defined
proxy-dns-response-closed-on: h3-alpn, ech-defined
~~~

## Processing Considerations

### Client Processing Considerations

A client that wishes to receive SVCB/HTTPS parameters from the proxy SHOULD
include the "Proxy-DNS-Request" header field in its CONNECT or CONNECT-UDP
request, enumerating the SvcParamKeys it is interested in. A client MAY
additionally include the "Proxy-DNS-Request-Close-On" header field to request
that the proxy close the connection if specific conditions are satisfied.

Upon receipt of a "Proxy-DNS-Response" header field, the client SHOULD use the
conveyed RRset to inform how it establishes the connection to the target,
respecting the SvcPriority ordering of the records as defined in {{!RFC9460}}.
If the header field is present but contains an empty list, the client SHOULD
treat the SVCB/HTTPS RRset for the target as unknown. If the header field is
absent, the client SHOULD treat the SVCB/HTTPS RRset for the target as unknown.

Upon receipt of a "Proxy-DNS-Response-Closed-On" header field, the client SHOULD
inspect the tokens present and use them, in conjunction with the parameters
conveyed in the "Proxy-DNS-Response" header field, to determine an appropriate
retry strategy. For example:

- If the token "h3-alpn" is present, the client SHOULD reattempt the connection
using CONNECT-UDP rather than CONNECT, using the ALPN and other SvcParam values
from the highest priority RR conveyed in the "Proxy-DNS-Response" header field.
- If the token "ech-defined" is present, the client SHOULD reattempt the
connection using Encrypted Client Hello, using the ECH keys from the highest
priority RR conveyed in the "Proxy-DNS-Response" header field.

If the "Proxy-DNS-Response" header field is absent in a response containing
"Proxy-DNS-Response-Closed-On", the client SHOULD perform its own SVCB
resolution before reattempting the connection.

### Proxy Processing Considerations

Support for the "Proxy-DNS-Request" and "Proxy-DNS-Request-Close-On" header
fields is OPTIONAL. A proxy that does not support these header fields MUST
ignore them and MUST continue processing the CONNECT or CONNECT-UDP request as
if they were absent.

A proxy that supports the "Proxy-DNS-Request" header field SHOULD perform
SVCB/HTTPS resolution for the target authority and SHOULD include the
"Proxy-DNS-Response" header field in its response, restricted to the
SvcParamKeys enumerated in the client's request. If the proxy was unable to
perform SVCB resolution, or if resolution returned no SVCB/HTTPS RRs for the
target, the proxy MUST include the "Proxy-DNS-Response" header field with an
empty list.

A proxy that supports the "Proxy-DNS-Request-Close-On" header field MUST
evaluate each recognized token against the resolved SVCB/HTTPS RRset prior to
establishing the forwarded connection to the target. Token evaluation MUST be
performed as follows:

- **h3-alpn**: The proxy MUST evaluate this token against only the highest
priority RR in the resolved RRset, i.e., the RR with the lowest non-zero
SvcPriority value. The condition is satisfied if the ALPN SvcParamKey of that RR
contains the "h3" protocol identifier. AliasMode records (SvcPriority 0) MUST be
followed before evaluation.  
- **ech-defined**: The proxy MUST evaluate this token against only the highest
priority RR in the resolved RRset. The condition is satisfied if the ECH
SvcParamKey is present in that RR.

New tokens MAY be defined in future documents to address evaluation against
other RRs in the RRset, or to define alternative evaluation semantics. For
example, a future token could be defined to trigger closure if any RR in the
RRset satisfies a condition, rather than only the highest priority RR. Unknown
tokens MUST be silently ignored.

If one or more recognized conditions are satisfied, the proxy SHOULD NOT
establish the forwarded connection and SHOULD instead close the request,
including the "Proxy-DNS-Response-Closed-On" header field indicating the tokens
that triggered the closure. The proxy SHOULD also include the
"Proxy-DNS-Response" header field populated with the resolved RRset, allowing
the client to reattempt the connection without requiring an additional
resolution step.

A proxy MUST silently ignore any tokens in "Proxy-DNS-Request-Close-On" that it
does not recognize. The order in which tokens appear in the list does not
indicate precedence; all recognized tokens are evaluated independently, and the
presence of any single satisfied condition is sufficient to trigger closure.

# Examples

## Scenario 1: Successful CONNECT with SVCB Parameters Returned

The client sends a CONNECT request requesting ALPN and ECH parameters:

~~~ example
REQUEST HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-request = 1, 5
~~~

The proxy performs SVCB/HTTPS resolution for `svc.example.com` and finds a
single RR indicating support for `h2` in the ALPN SvcParamKey and an ECH
configuration. The proxy establishes the forwarded connection and responds
successfully, conveying the resolved SVCB parameters:

~~~ example
RESPONSE HEADERS
:status = 200
proxy-dns-response = (1;value=:aGkK: 5;value=:ACAgAA...:);priority=1;target="svc.example.com"
~~~

The client uses the conveyed ALPN and ECH values to inform how it establishes the connection to the target.

### Scenario 2: Close-On Triggered, Client Retries with CONNECT-UDP

The client sends a CONNECT request, requesting ALPN parameters and indicating that the connection should be closed if the target supports `h3`:

~~~ example
REQUEST HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-request = 1
proxy-dns-request-close-on: h3-alpn
~~~

The proxy performs SVCB/HTTPS resolution for `svc.example.com` and finds that
the highest priority RR contains `h3` in the ALPN SvcParamKey. The condition
specified in the "Proxy-DNS-Request-Close-On" header field is satisfied. The
proxy does not establish a forwarded connection and instead responds with the
satisfied condition and the resolved SVCB parameters:

~~~ example
RESPONSE HEADERS
:status = 307
proxy-dns-response-closed-on: h3-alpn
proxy-dns-response = (1;value=:aGkK:);priority=1;target="svc.example.com"
~~~

Upon receipt of the "Proxy-DNS-Response-Closed-On" header field, the client
observes that the `h3-alpn` condition was satisfied. Using the ALPN values
conveyed in the "Proxy-DNS-Response" header field, the client reattempts the
connection using CONNECT-UDP:

~~~ example
REQUEST HEADERS
:method = CONNECT-UDP
:authority = svc.example.com:443
proxy-dns-request = 1
~~~

The proxy establishes the forwarded connection and responds successfully:

~~~ example
RESPONSE HEADERS
:status = 200
proxy-dns-response = (1;value=:aGkK:);priority=1;target="svc.example.com"
~~~

## Scenario 3: Multiple RRs with Different Priorities

The client sends a CONNECT request requesting ALPN and ECH parameters:

~~~ example
REQUEST HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-request = 1, 5
proxy-dns-request-close-on: h3-alpn
~~~

The proxy performs SVCB/HTTPS resolution for `svc.example.com` and finds two RRs with different priorities:

~~~ dns
svc.example.com. 300 IN HTTPS 1 svc1.example.com. alpn=h2,h3 ech=...
svc.example.com. 300 IN HTTPS 2 svc2.example.com. alpn=h2
~~~

The highest priority RR (priority 1) indicates support for `h3` in the ALPN
SvcParamKey. The `h3-alpn` condition is therefore satisfied. The proxy does not
establish a forwarded connection and responds with the full RRset and the
satisfied condition:

~~~ example
RESPONSE HEADERS
:status = 307
proxy-dns-response-closed-on: h3-alpn
proxy-dns-response = (1;value=:aGkK: 5;value=:ACAgAA...:);priority=1;target="svc1.example.com",
                     (1;value=:bmkK:);priority=2;target="svc2.example.com"
~~~

Upon receipt of the "Proxy-DNS-Response-Closed-On" header field, the client
observes that the `h3-alpn` condition was satisfied. The client selects the
highest priority RR and reattempts the connection using CONNECT-UDP to
`svc1.example.com`:

~~~ example
REQUEST HEADERS
:method = CONNECT-UDP
:authority = svc1.example.com:443
proxy-dns-request = 1, 5
~~~

The proxy establishes the forwarded connection and responds successfully:

~~~ example
RESPONSE HEADERS
:status = 200
proxy-dns-response = (1;value=:aGkK: 5;value=:ACAgAA...:);priority=1;target="svc1.example.com"
~~~

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.

# TODO List

Interaction with rfc9532
h3-alpn cannot be used with CONNECT-UDP

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
