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

When HTTP clients use the CONNECT or CONNECT-UDP methods to reach target servers
through a proxy, the proxy performs DNS resolution on behalf of the client. This
prevents the client from accessing Service Binding (SVCB and HTTPS) DNS records
that carry service configuration such as supported protocols and Encrypted
Client Hello keys. This document defines HTTP header fields that enable the
proxy to relay SVCB information to the client, describes how the proxy can act
on SVCB/HTTPS data when establishing the forwarded connection, and introduces a
conditional early closure mechanism that allows the client to coordinate with
the proxy to abort a connection when specific SVCB parameters are present, so
that the client can retry with a different connection strategy.

--- middle

# Introduction

The CONNECT method {{!RFC7231}} and the CONNECT-UDP method {{!RFC9298}} allow
HTTP clients to establish TCP or UDP flows to target servers through an HTTP
proxy. The client identifies the target using authority-form, which includes a
server name or IP address and a port number. When the target is a name, the
proxy resolves it to an IP address using A or AAAA queries.

This arrangement prevents the client from accessing Service Binding (SVCB and
HTTPS) resource records {{!RFC9460}} for the target. SVCB records carry
configuration that influences how connections should be formed, including
supported application protocols (ALPN), Encrypted Client Hello keys, and
alternative endpoints.

This document defines three mechanisms that address this gap:

- A pair of HTTP header fields that allow the client to request, and the proxy
  to convey, SVCB parameters for the target ({{requesting-and-receiving-svcb-information}}).
- Guidance for how the proxy acts on SVCB data when establishing the forwarded
  connection, including alias resolution and the use of RFC 9532
  {{!RFC9532}} to signal followed aliases ({{proxy-use-of-svcb-information}}).
- A conditional closure mechanism that allows the client to request that the
  proxy abort the connection if certain SVCB conditions are met, returning the
  data so the client can retry with a different strategy ({{conditional-connection-closure}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

- **ServiceMode RR**: An SVCB or HTTPS resource record with a non-zero
  SvcPriority value, as defined in {{!RFC9460}}.
- **AliasMode RR**: An SVCB or HTTPS resource record with SvcPriority 0,
  as defined in {{!RFC9460}}.
- **Highest-priority RR**: The ServiceMode RR in a resolved RRset with the
  lowest non-zero SvcPriority value.

# Requesting and Receiving SVCB Information

This section defines two HTTP header fields that allow a client to request SVCB
parameters from the proxy and receive the resolved data in the response.

## The Proxy-DNS-SVCB-Request Header Field

The "Proxy-DNS-SVCB-Request" header field is a Structured Field {{!RFC8941}} of
type sf-boolean. A client includes this header field with a value of `?1` to
signal that it wants the proxy to return SVCB/HTTPS information for the target.

~~~ example
proxy-dns-svcb-request: ?1
~~~

## The Proxy-DNS-SVCB-Response Header Field

The "Proxy-DNS-SVCB-Response" header field is a Structured Field {{!RFC8941}}
that conveys the SVCB/HTTPS records resolved by the proxy for the target
authority. The field value is an sf-list where each member is an sf-binary item
containing the RDATA of a single SVCB/HTTPS resource record in wire format as
defined in Section 2.2 of {{!RFC9460}}.

The proxy SHOULD include every SVCB/HTTPS record encountered during resolution,
including any AliasMode records that were followed. This gives the client
visibility into the full resolution path, not just the terminal records.

~~~ example
proxy-dns-svcb-response: :AAEGYW...:, :AAIHc3...:
~~~

Each sf-binary item encodes one complete SVCB/HTTPS RDATA (SvcPriority,
TargetName, and SvcParams). The client parses each item using the standard SVCB
wire format.

### Empty and Absent Responses

If the proxy was unable to resolve SVCB records, or if resolution returned no
SVCB/HTTPS records for the target, the proxy MUST include the header field with
an empty list. If the header field is absent from the response, the client
SHOULD treat the SVCB/HTTPS information as unknown.

~~~ example
proxy-dns-svcb-response:
~~~

## Processing Rules

### Client Behavior

A client that wishes to receive SVCB/HTTPS information SHOULD include the
"Proxy-DNS-SVCB-Request" header field in its CONNECT or CONNECT-UDP request.

Upon receiving a "Proxy-DNS-SVCB-Response" header field, the client parses each
sf-binary member as SVCB/HTTPS RDATA and uses the resulting records to inform
connection establishment, respecting SvcPriority ordering as defined in
{{!RFC9460}}. If the header field contains an empty list or is absent, the
client treats the SVCB/HTTPS information as unknown.

### Proxy Behavior

Support for the "Proxy-DNS-SVCB-Request" header field is OPTIONAL. A proxy that
does not support it MUST ignore the header and continue processing the request
normally. In that case, the response will not include the
"Proxy-DNS-SVCB-Response" header field.

A proxy that supports the header field:

- SHOULD perform SVCB/HTTPS resolution for the target authority.
- SHOULD include the "Proxy-DNS-SVCB-Response" header field in its response.
- MUST include the header field with an empty list if resolution fails or
  returns no SVCB/HTTPS records.

## Example

The client sends a CONNECT request requesting SVCB information:

~~~ example
HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-svcb-request: ?1
~~~

The proxy resolves SVCB records for `svc.example.com` and finds a ServiceMode
RR with `h3` ALPN and an ECH configuration. The proxy returns the SVCB RDATA:

~~~ example
HEADERS
:status = 200
proxy-dns-svcb-response: :AAEGYW...:
~~~

The client may use the conveyed parameters to use CONNECT-UDP for any future
connections to `svc.example.com`.

# Proxy Use of SVCB Information

When a proxy resolves SVCB records for the target, it may act on the data when
establishing the forwarded connection. This section describes how the proxy
signals its actions back to the client.

## Signaling DNS Name Resolution

When the proxy connects to a name different from the client's original target
authority, whether due to AliasMode following or a ServiceMode TargetName, the
proxy SHOULD signal the resolution chain to the client using the
`next-hop-aliases` parameter of the Proxy-Status header field defined in
{{!RFC9532}}.

## Example

The client sends a CONNECT request for a target whose SVCB records include an
alias chain and ECH configuration:

~~~ example
HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-svcb-request: ?1
~~~

The proxy resolves `svc.example.com` and finds a ServiceMode RR at
`cdn.example.net` with `h2` ALPN and an ECH configuration. The proxy connects
to `cdn.example.net` and returns:

~~~ example
HEADERS
:status = 200
proxy-status: proxy.example; next-hop-aliases="cdn.example.net"
proxy-dns-svcb-response: :AAEGYW...:
~~~

# Conditional Connection Closure {#conditional-closure}

## Motivation

A client may prefer a different connection strategy based on the SVCB
information for the target. For example, the client might prefer CONNECT-UDP
when the target supports `h3`, or use Encrypted Client Hello when the target
publishes ECH keys.

Without coordination, the client learns this only after the proxy has
established the forwarded connection. Tearing that connection down and
retrying wastes a round trip. The mechanism in this section lets the client
signal conditions under which the proxy should abort before connecting and
return the SVCB data immediately.

## The Proxy-DNS-SVCB-Request-Close-On Header Field

The "Proxy-DNS-SVCB-Request-Close-On" header field is a Structured Field
{{!RFC8941}} whose value is an sf-list of sf-token items. Each token names a
condition that, if satisfied by the resolved SVCB data, should cause the proxy
to close the request without establishing the forwarded connection.

The set of tokens is open-ended, and any token that the client and proxy
mutually understand can be used; this document defines an initial set of tokens
that clients and proxies must support, but deployments are free to use
additional tokens registered through the process described in
{{iana-considerations}} or agreed upon out-of-band.

~~~ example
proxy-dns-svcb-request-close-on: h3-alpn, ech-defined
~~~

### Initial Tokens

This document defines the following tokens. These are starting points covering
common cases; they do not constrain what other tokens may be defined or used.

h3-alpn:
: The condition is satisfied if the ALPN SvcParamKey of the highest-priority RR
  contains the "h3" protocol identifier. A client includes this token when it
  prefers CONNECT-UDP for targets that support HTTP/3.

ech-defined:
: The condition is satisfied if the ECH SvcParamKey is present in the
  highest-priority RR. A client includes this token when it prefers to use
  Encrypted Client Hello when connecting to the target.

### Evaluation Rules

The proxy evaluates each recognized token against the resolved SVCB/HTTPS RRset
as follows:

- Evaluation is performed against only the highest-priority RR (the RR with the
  lowest non-zero SvcPriority value).
- AliasMode records MUST be followed before evaluation.
- All recognized tokens are evaluated independently. The order in which tokens
  appear does not indicate precedence.
- The "h3-alpn" token MUST NOT be evaluated when the client's request uses the
  CONNECT-UDP method, since the client is already using a UDP-based tunnel.
- Unknown tokens MUST be silently ignored.

### Extensibility

New tokens can carry new evaluation semantics. For example, a token could
trigger closure if any RR in the RRset satisfies a condition, rather than only
the highest-priority RR. The evaluation rules above apply to the tokens defined
in this document; new tokens define their own.

## The Proxy-DNS-SVCB-Response-Closed-On Header Field

The "Proxy-DNS-SVCB-Response-Closed-On" header field is a Structured Field
{{!RFC8941}} whose value is an sf-list of sf-token items. The proxy includes
this header field when it closes a request due to a satisfied condition.

- The proxy MUST report all tokens whose conditions were satisfied.
- The proxy MUST NOT include tokens that were absent from the client's
  "Proxy-DNS-SVCB-Request-Close-On" header field.
- The proxy MUST NOT include tokens whose conditions were not satisfied.

~~~ example
proxy-dns-svcb-response-closed-on: h3-alpn
~~~

## Processing Rules

### Proxy Behavior

A proxy that supports the "Proxy-DNS-SVCB-Request-Close-On" header field MUST
evaluate each recognized token against the resolved SVCB/HTTPS RRset prior to
establishing the forwarded connection.

If one or more conditions are satisfied:

- The proxy SHOULD NOT establish the forwarded connection.
- The proxy SHOULD close the request with HTTP status code 307 and include the
  "Proxy-DNS-SVCB-Response-Closed-On" header field indicating the satisfied
  tokens.
- The proxy MUST include the "Proxy-DNS-SVCB-Response" header field with the
  resolved records, so the client can retry using the conveyed information
  without an additional DNS resolution.

\[TODO: 307 (Temporary Redirect) is used as a placeholder to signal that the
request was not fulfilled and the client should retry with different
parameters. Future revisions of this document may select a different status
code or define a new one if 307 proves unsuitable.\]

A proxy that does not support the "Proxy-DNS-SVCB-Request-Close-On" header field
MUST ignore it and proceed normally.

### Client Behavior

Upon receiving a "Proxy-DNS-SVCB-Response-Closed-On" header field, the client
parses the accompanying "Proxy-DNS-SVCB-Response" and inspects the satisfied
tokens to determine its retry strategy:

- If "h3-alpn" is present, the client SHOULD retry using CONNECT-UDP with ALPN
  values from the highest-priority RR in the conveyed records.
- If "ech-defined" is present, the client SHOULD retry using Encrypted Client
  Hello with the ECH keys from the highest-priority RR in the conveyed records.

## Example

The client sends a CONNECT request and indicates that it strongly prefers
CONNECT-UDP if the target supports it by including the h3-alpn token:

~~~ example
HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-svcb-request: ?1
proxy-dns-svcb-request-close-on: h3-alpn
~~~

The proxy resolves a single ServiceMode RR:

~~~ dns
svc.example.com. 300 IN HTTPS 1 . alpn=h2,h3
~~~

The highest-priority RR contains `h3` in ALPN, so the condition is satisfied.
The proxy closes the request:

~~~ example
HEADERS
:status = 307
proxy-dns-svcb-response-closed-on: h3-alpn
proxy-dns-svcb-response: :AAEGYW...:
~~~

The client observes the satisfied condition and retries with CONNECT-UDP:

~~~ example
HEADERS
:method = CONNECT-UDP
:authority = svc.example.com:443
proxy-dns-svcb-request: ?0
~~~

The client need not request the SVCB/HTTPS data again since it already has the
information it needs. The client may instead omit the "Proxy-DNS-SVCB-Request"
header field entirely.

# Combined Examples

## Successful CONNECT with SVCB Parameters

The client requests SVCB information via CONNECT:

~~~ example
HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-svcb-request: ?1
~~~

The proxy resolves a single ServiceMode RR:

~~~ dns
svc.example.com. 300 IN HTTPS 1 . alpn=h2 ech=...
~~~

The TargetName "." indicates the target is the same as the owner name
(`svc.example.com`). The proxy establishes the connection and returns the
SVCB RDATA:

~~~ example
HEADERS
:status = 200
proxy-dns-svcb-response: :AAEGYW...:
~~~

## Multiple RRs with Different Priorities

The client sends CONNECT requesting SVCB information with `h3-alpn` close-on:

~~~ example
HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-svcb-request: ?1
proxy-dns-svcb-request-close-on: h3-alpn
~~~

The proxy resolves two RRs:

~~~ dns
svc.example.com. 300 IN HTTPS 1 svc1.example.com. alpn=h2,h3 ech=...
svc.example.com. 300 IN HTTPS 2 svc2.example.com. alpn=h2
~~~

The highest-priority RR (priority 1) contains `h3` in ALPN. The proxy closes:

~~~ example
HEADERS
:status = 307
proxy-dns-svcb-response-closed-on: h3-alpn
proxy-dns-svcb-response: :AAEGYW...:, :AAIHc3...:
~~~

The client retries with CONNECT-UDP using the original authority. The proxy
re-resolves and connects to the highest-priority RR's target:

~~~ example
HEADERS
:method = CONNECT-UDP
:authority = svc.example.com:443
~~~

The proxy establishes the connection and responds, signaling the resolved
target via Proxy-Status:

~~~ example
HEADERS
:status = 200
proxy-status: proxy.example; next-hop-aliases="svc1.example.com"
~~~

## AliasMode with next-hop-aliases and Close-On

The client sends CONNECT requesting SVCB information with both close-on conditions:

~~~ example
HEADERS
:method = CONNECT
:authority = svc.example.com:443
proxy-dns-svcb-request: ?1
proxy-dns-svcb-request-close-on: h3-alpn, ech-defined
~~~

The proxy resolves `svc.example.com` and encounters an AliasMode RR:

~~~ dns
svc.example.com. 300 IN HTTPS 0 cdn.example.net.
~~~

The proxy follows the alias and resolves `cdn.example.net`:

~~~ dns
cdn.example.net. 300 IN HTTPS 1 cdn.example.net. alpn=h2,h3 ech=...
~~~

Both `h3-alpn` and `ech-defined` conditions are satisfied. The proxy closes
and returns both the AliasMode RR for `svc.example.com` and the terminal
ServiceMode RR for `cdn.example.net` in "Proxy-DNS-SVCB-Response", and signals
the resolved alias via Proxy-Status:

~~~ example
HEADERS
:status = 307
proxy-status: proxy.example; next-hop-aliases="cdn.example.net"
proxy-dns-svcb-response-closed-on: h3-alpn, ech-defined
proxy-dns-svcb-response: :AAAGYW...:, :AAEGYW...:
~~~

The client retries with CONNECT-UDP, using ECH with the keys from the response:

~~~ example
HEADERS
:method = CONNECT-UDP
:authority = cdn.example.net:443
proxy-dns-svcb-request: ?1
~~~

The proxy establishes the connection and responds:

~~~ example
HEADERS
:status = 200
proxy-dns-svcb-response: :AAEGYW...:
~~~

# Security Considerations

## Privacy of Client Preferences

The "Proxy-DNS-SVCB-Request" header field reveals that the client is interested
in SVCB information for the target. The "Proxy-DNS-SVCB-Request-Close-On" header
field further reveals the client's connection strategy preferences.

## Alias Chain Verification

When the proxy follows SVCB aliases, the "Proxy-Status" header field with
`next-hop-aliases` allows the client to observe the alias chain. Clients MAY use
this information to audit the proxy's resolution path and verify that the
terminal target is consistent with the SVCB response.

# IANA Considerations

## HTTP Header Field Registrations

This document registers the following HTTP header fields in the "Hypertext
Transfer Protocol (HTTP) Field Name Registry" defined in Section 16.3.1 of
{{!RFC9110}}:

| Field Name | Status |
|---|---|
| Proxy-DNS-SVCB-Request | permanent |
| Proxy-DNS-SVCB-Response | permanent |
| Proxy-DNS-SVCB-Request-Close-On | permanent |
| Proxy-DNS-SVCB-Response-Closed-On | permanent |

## Proxy-DNS-SVCB-Request-Close-On Token Registry

This document establishes a new "Proxy-DNS-SVCB-Request-Close-On Tokens"
registry. The registration policy is Specification Required ({{!RFC8126}}).

Initial entries:

| Token | Description | Reference |
|---|---|---|
| h3-alpn | ALPN contains "h3" in highest-priority RR | This document |
| ech-defined | ECH SvcParamKey present in highest-priority RR | This document |

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.