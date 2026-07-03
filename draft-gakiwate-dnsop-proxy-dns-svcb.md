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
 -
    fullname: Erik Nygren
    organization: Akamai Technologies
    email: "erik+ietf@nygren.org"

normative:

informative:

...

--- abstract

When HTTP clients use the CONNECT or CONNECT-UDP methods to reach target servers
through a proxy, the proxy performs DNS resolution on behalf of the client. This
prevents the client from accessing Service Binding (SVCB and HTTPS) DNS records
that carry service configuration such as supported protocols and Encrypted
Client Hello keys. This document defines HTTP header fields that enable the
proxy to relay SVCB information to the client, and introduces a conditional
early closure mechanism that allows the client to coordinate with the proxy to
abort a connection when specific SVCB parameters are present, so that the
client can retry with a different connection strategy.

--- middle

# Introduction

The CONNECT method {{!RFC9110}} and the CONNECT-UDP method {{!RFC9298}} allow
HTTP clients to establish TCP or UDP flows to target servers through an HTTP
proxy. The client identifies the Proxy Target using authority-form (in {{!RFC9110}}),
or a URI Template {{!RFC6570}} for CONNECT-UDP.  The Proxy Target includes a
server name or IP address and a port number. When the Proxy Target is a name, the
proxy resolves it to an IP address using A or AAAA queries.

This arrangement prevents the client from accessing Service Binding (SVCB and
HTTPS) resource records {{!RFC9460}} for the target. SVCB records carry
configuration that influences how connections should be formed, including
supported application protocols (ALPN), Encrypted Client Hello keys, and
alternative endpoints.

This document defines two mechanisms that address this gap:

- A pair of HTTP header fields that allow the client to request, and the proxy
  to convey, SVCB parameters for the target ({{requesting-and-receiving-svcb-information}}).
- A conditional closure mechanism that allows the client to request that the
  proxy abort the connection if certain conditions are met, returning the
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
- **Selected Endpoint**: The SVCB or HTTPS resource record selected
  by the client to use, as defined by the algorithm in {{!RFC9460}} Section 3.
- **Selected TargetName**: The TargetName of the Selected Endpoint.
  When the TargetName of the resource record is ".",
  the Selected TargetName is the owner name of the resource record.
- **Proxy Target**: The target server name (or IP address), target port, and protocol
  that the client has requested the proxy connect to,
  as specified in the authority-form (for CONNECT)
  or in a URI Template (for CONNECT-UDP).

# Requesting and Receiving SVCB Information

This section defines two HTTP header fields that allow a client to request SVCB
parameters from the proxy and receive the resolved data in the response.


## The Proxy-DNS-SVCB-Request Header Field

The "Proxy-DNS-SVCB-Request" header field is a Structured Field {{!RFC8941}}
whose value MUST be an sf-token. A client includes this header field to
signal that it wants the proxy to resolve and return DNS resource records
of the indicated type for the target. The token value MUST be `SVCB`
or another SVCB-compatible record type (such as `HTTPS`):

- `SVCB`: the proxy is requested to resolve SVCB resource records.
- `HTTPS`: the proxy is requested to resolve HTTPS resource records.

For SVCB records that are on a different name from the Proxy Target name,
the field MAY include a `name` parameter, whose value is an sf-string
specifying the DNS name to resolve. If the `name` parameter is omitted,
the proxy resolves the record(s) for the target's own origin.

If the header field is absent, the client is not requesting DNS
resolution from the proxy.

A proxy that receives a "Proxy-DNS-SVCB-Request" header field whose value is not
`SVCB`, `HTTPS`, or another SVCB-compatible RR type that it recognizes
MUST ignore the header field.

~~~ example
proxy-dns-svcb-request: HTTPS
~~~

~~~ example
proxy-dns-svcb-request: SVCB;name="_dns.example.com"
~~~

A proxy MAY choose to ignore the "Proxy-DNS-SVCB-Request" header field
if it is unrelated to the Proxy Target name.


## The Proxy-DNS-SVCB-Response Header Field

The "Proxy-DNS-SVCB-Response" header field is a Structured Field {{!RFC8941}}
that conveys the HTTPS resource records resolved by the proxy for the target
authority. The field value is an sf-list where each member is an sf-binary item
containing a single HTTPS resource record in DNS wire format as defined in
Section 3.2.1 of {{!RFC1035}}. Each encoded record includes the owner name,
TYPE, CLASS, TTL, and RDATA. Owner names in the encoded records MUST NOT use
DNS name compression.

The proxy SHOULD include every SVCB or HTTPS record encountered during
resolution, including any AliasMode records that were followed. This
gives the client visibility into the full resolution path, not just
the terminal records.

~~~ example
proxy-dns-svcb-response: :AAEGYW...:, :AAIHc3...:
~~~

Each sf-binary item encodes one complete DNS resource record. The owner name
identifies the name at which the record was resolved; the RDATA contains the
SvcPriority, TargetName, and SvcParams as defined in Section 2.2 of
{{!RFC9460}}.

### Empty and Absent Responses

If the proxy was unable to resolve HTTPS records, or if resolution returned no
HTTPS records for the target, the proxy MUST include the header field with
an empty list. If the header field is absent from the response, the client
SHOULD treat the HTTPS record information as unknown.

~~~ example
proxy-dns-svcb-response:
~~~

### Signaling DNS Name Resolution

When returning a `Proxy-DNS-SVCB-Response` header field the proxy SHOULD also
return a `Proxy-Staus` header field ({{!RFC9209}}) to return a `next-hop` parameter
and also the `next-hop-aliases` parameter ({{!RFC9532}}).
These will allow the client to determine how the proxy's connection was made
and thus if is compatible with a given Selected Endpoint.


## Processing Rules

### Client Behavior

A client that wishes to receive SVCB/HTTPS information SHOULD include the
"Proxy-DNS-SVCB-Request" header field in its CONNECT or CONNECT-UDP request.

Upon receiving a "Proxy-DNS-SVCB-Response" header field, the client parses each
sf-binary member as a complete DNS resource record and uses the resulting
records to inform connection establishment, following the algorithm defined
in {{!RFC9460}} Section 3 to pick a Selected Endpoint.
If the header field contains an empty list or is
absent, the client treats the SVCB/HTTPS record information as unknown
meaning that SVCB resolution has failed, and the list of available endpoints is empty.

Prior to using a connection established by the proxy as a Selected Endpoint,
the client MUST ensure that it is compatible.

An established connection is compatible if the protocol (TCP or UDP) of
the Selected Endpoint matches what was used
to establish the connection (ie, via CONNECT or CONNECT-TCP),
if the port of the Selected Endpoint matches that used for the Proxy Target,
*and* if at least one of the following applies:

* The Selected TargetName matches the Proxy Target for the connection.
* An `ipv4hint` or `ipv6hint` SvcParam from the Selected Endpoint matches the `next-hop`
  parameter returned from the `Proxy-Status` header field.
* The final name in the `next-hop-aliases` of the `Proxy-Status` header field
  matches the Selected TargetName.  (ie, that the Selected TargetName shares
  an owner name with the A or AAAA DNS records used to resolve the `next-hop`.)

If the proxy connection is compatible and connection establishment succeeded,
the client SHOULD continue to use as the Selected Endpoint.

If the Selected Endpoint is not compatible, the client MUST abandon this
CONNECT request and establish a new CONNECT/CONNECT-UDP where the client
provides the final Selected TargetName and port to the proxy as a new Proxy Target,
as described in {{!RFC9460}} Section 3.2.

Clients SHOULD obey the TTLs in the DNS RRs and request new ones
on subsequent requests if previously cached ones have expired in cache.


### Proxy Behavior

Support for the "Proxy-DNS-SVCB-Request" header field is OPTIONAL. A proxy that
does not support it MUST ignore the header and continue processing the request
normally. In that case, the response will not include the
"Proxy-DNS-SVCB-Response" header field.

A proxy that supports the header field:

- SHOULD perform a SVCB or HTTPS RR resolution for either the Proxy Target
  name or the value of the `name` attribute if specified.
- SHOULD include the `Proxy-DNS-SVCB-Response` header field in its response.
- SHOULD include the `Proxy-Staus` header field with both the
  `next-hop` parameter containing the IP address and `next-hop-aliases`
  parameter containing any CNAMEs.
- MUST include the header field with an empty list if resolution fails or
  returns no SVCB/HTTPS records.


## Example

The client sends a CONNECT request requesting SVCB information:

~~~ example
HEADERS
:method = CONNECT
:authority = www.example.com:443
proxy-dns-svcb-request: HTTPS
~~~

The proxy resolves HTTPS records for `www.example.com` and finds a ServiceMode
RR with `h3` ALPN and an ECH configuration. The proxy returns the HTTPS
resource record:

~~~ example
HEADERS
:status = 200
proxy-dns-svcb-response: :AAEGYW...:
proxy-status: proxy.example; next-hop="2001:db8:a::b"
~~~

where the proxy-dns-svcb-response contains the DNS RR:

~~~ example
www.example.com. 300 IN HTTPS 1 . alpn=h2,h3 ipv6hint="2001:db8:a::b" ech=...
~~~

The client may use the conveyed parameters to use CONNECT-UDP for any future
connections to `www.example.com`.

It may also reuse the established connection with the HTTPS record containing
the ECH configuration since the Selected TargetName from the HTTPS record was
`www.example.com` and because its `ipv6hint` SvcParam includes `2001:db8:a::b`.


# Conditional Connection Closure {#conditional-connection-closure}

## Motivation

A client may prefer a different connection strategy based on the SVCB
information for the target. For example, the client might prefer CONNECT-UDP
when the target supports `h3`, or use Encrypted Client Hello when the target
publishes ECH keys.

Without coordination, the client learns this only after the proxy has
established the forwarded connection. Tearing that connection down and
retrying wastes potentially multiple round trips in the case of HTTP/1.1.
The mechanism in this section lets the client
signal conditions under which the proxy should abort before connecting and
return the SVCB data immediately.

The mechanism in this section also enables a client using HTTP/2 or HTTP/3
to pipeline DATA frames with a TLS ClientHello with a reduced chance that
those will leak TLS SNI information to services supporting ECH.

## The Proxy-DNS-SVCB-Request-Cancel-On Header Field

The "Proxy-DNS-SVCB-Request-Cancel-On" header field is a Structured Field
{{!RFC8941}} whose value is an sf-list of sf-token items. Each token names a
condition that, if satisfied by the resolved SVCB data, should cause the proxy
to skip the request without establishing the forwarded connection.

The set of tokens is open-ended, and any token that the client and proxy
mutually understand can be used; this document defines an initial set of tokens
that clients and proxies must support, but deployments are free to use
additional tokens registered through the process described in
{{iana-considerations}} or agreed upon out-of-band.

~~~ example
proxy-dns-svcb-request-cancel-on: h3-alpn, ech-defined
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

https:
: The condition is satisfied if any HTTPS RR is found.
  This indicates that the client must not establish an insecure HTTP connection.



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

## The Proxy-DNS-SVCB-Response-Cancelled-On Header Field

The "Proxy-DNS-SVCB-Response-Cancelled-On" header field is a Structured Field
{{!RFC8941}} whose value is an sf-list of sf-token items. The proxy includes
this header field when it cancels making the connect
request due to a satisfied condition.

- The proxy MUST report all tokens whose conditions were satisfied.
- The proxy MUST NOT include tokens that were absent from the client's
  "Proxy-DNS-SVCB-Request-Cancel-On" header field.
- The proxy MUST NOT include tokens whose conditions were not satisfied.

~~~ example
proxy-dns-svcb-response-cancelled-on: h3-alpn
~~~

## Processing Rules

### Proxy Behavior

A proxy that supports the "Proxy-DNS-SVCB-Request-Cancel-On" header field MUST
evaluate each recognized token against the resolved SVCB/HTTPS RRset prior to
establishing the forwarded connection.

If one or more conditions are satisfied:

- The proxy SHOULD NOT establish the forwarded connection.
- The proxy SHOULD cancel the request with HTTP status code 307 and include the
  "Proxy-DNS-SVCB-Response-Cancelled-On" header field indicating the satisfied
  tokens.
- The proxy MUST include the "Proxy-DNS-SVCB-Response" header field with the
  resolved HTTPS records, so the client can retry using the conveyed information
  without an additional DNS resolution.

\[TODO: 307 (Temporary Redirect) is used as a placeholder to signal that the
request was not fulfilled. This may change based on discussions.
We may wish to specify that Location not be sent.  We may
also wish to consider moving Proxy-DNS-SVCB-Response-Cancelled-On into
Proxy-Status.
\]

A proxy that does not support the "Proxy-DNS-SVCB-Request-Cancel-On" header field
MUST ignore it and proceed normally.

### Client Behavior

Clients establishing an insecure HTTP connection to port 80 SHOULD
send `https` as a "Proxy-DNS-SVCB-Request-Cancel-On" value.

Upon receiving a "Proxy-DNS-SVCB-Response-Cancelled-On" header field, the client
parses the accompanying "Proxy-DNS-SVCB-Response" and inspects the satisfied
tokens to determine its retry strategy:

- If `h3-alpn` is present, the client SHOULD retry using CONNECT-UDP with ALPN
  values from the highest-priority RR in the conveyed records.
- If `ech-defined` is present, the client SHOULD retry using Encrypted Client
  Hello with the ECH keys from the highest-priority RR in the conveyed records.
- If `https` is present, the client MUST upgrade to an HTTPS request
  as described in {{!RFC9460}}.


## Example

The client sends a CONNECT request and indicates that it strongly prefers
CONNECT-UDP if the target supports it by including the h3-alpn token:

~~~ example
HEADERS
:method = CONNECT
:authority = www.example.com
proxy-dns-svcb-request: HTTPS
proxy-dns-svcb-request-cancel-on: h3-alpn
~~~

The proxy resolves a single ServiceMode RR:

~~~ dns
www.example.com. 300 IN HTTPS 1 www.example.com alpn=h2,h3
~~~

The highest-priority RR contains `h3` in ALPN, so the condition is satisfied.
The proxy cancels the request and does not make the connection:

~~~ example
HEADERS
:status = 307
proxy-dns-svcb-response-cancelled-on: h3-alpn
proxy-dns-svcb-response: :AAEGYW...:
~~~

The client observes the satisfied condition and retries with CONNECT-UDP:

~~~ example
HEADERS
:method = CONNECT
:protocol = connect-udp
:scheme = https
:path = /.well-known/masque/udp/www.example.com/443/
:authority = proxy.example
capsule-protocol = ?1
~~~

The client need not request the HTTPS RR data again since it already has the
information it needs. The client signals this by omiting the
"Proxy-DNS-SVCB-Request" header field entirely.

# Combined Examples

## Successful CONNECT with SVCB Parameters

The client requests SVCB information via CONNECT:

~~~ example
HEADERS
:method = CONNECT
:authority = svc.example.com:8000
proxy-dns-svcb-request: SVCB,name=_8000._foo.svc.example.com
~~~

The proxy resolves the SVCB RR for svc.example.com and returns
it in a header:

~~~ example
HEADERS
:status = 200
proxy-dns-svcb-response: :AAEGYW...:
proxy-status: proxy.example; next-hop="2001:db8:a::b"
~~~

Within `proxy-dns-svcb-response` is the single DNS RR:

~~~ dns
_8000._foo.svc.example.com. 300 IN SVCB 1 svc.example.com port=8000 ech=...
~~~

The client selecting to use this ServiceMode RR
confirms that the Selected TargetName matches the Proxy Target used
(also `svc.example.com`) and that the ports match (8000)
so it is able to use the proxy connection
and send a TLS ClientHello using ECH and the SVCB record.


## Multiple RRs with Different Priorities

The client sends CONNECT requesting SVCB information with `h3-alpn` cancel-on:

~~~ example
HEADERS
:method = CONNECT
:authority = www.example.com:443
proxy-dns-svcb-request: HTTPS
proxy-dns-svcb-request-cancel-on: h3-alpn
~~~

The proxy resolves two RRs:

~~~ dns
www.example.com. 300 IN HTTPS 1 svc1.example.com. alpn=h2,h3 ech=...
www.example.com. 300 IN HTTPS 2 svc2.example.com. alpn=h2
~~~

The highest-priority RR (priority 1) contains `h3` in ALPN. The proxy cancels
making the connect request:

~~~ example
HEADERS
:status = 307
proxy-dns-svcb-response-cancelled-on: h3-alpn
proxy-dns-svcb-response: :AAEGYW...:, :AAIHc3...:
~~~

The client retries with CONNECT-UDP using the Selected TargetName
of the Selected Endpoint. The proxy re-resolves and connects
this new Proxy Target requested by the client:

~~~ example
HEADERS
:method = CONNECT
:protocol = connect-udp
:scheme = https
:path = /.well-known/masque/udp/svc1.example.com/443/
:authority = proxy.example
capsule-protocol = ?1
~~~

The proxy establishes the connection and responds:

~~~ example
HEADERS
:status = 200
proxy-status: proxy.example; next-hop="2001:db8:a::c"
~~~

Note that even if the proxy had been instructed to cancel making
the CONNECT request, it still would not have been usable by
the client as neither TargetName (`svc1.example.com` nor `svc2.example.com`)
matches the Proxy Target (`www.example.com`).


## AliasMode and Cancel-On

The client sends CONNECT requesting SVCB information with both cancel-on conditions:

~~~ example
HEADERS
:method = CONNECT
:authority = www.example.com:443
proxy-dns-svcb-request: HTTPS
proxy-dns-svcb-request-cancel-on: h3-alpn, ech-defined
~~~

The proxy resolves `www.example.com` and encounters an AliasMode RR:

~~~ dns
www.example.com. 300 IN HTTPS 0 cdn.example.
~~~

The proxy follows the alias and resolves `cdn.example`
which is an alias to `cdn2.example.net`:

~~~ dns
cdn.example.      900 IN CNAME cdn2.example.net.
cdn2.example.net. 300 IN HTTPS 1 . alpn=h2,h3 ech=...
~~~

Both `h3-alpn` and `ech-defined` conditions are satisfied. The proxy cancels
and returns both the AliasMode RR for `svc.example.com`,
the CNAME for `cdn.example`, and the terminal
ServiceMode RR for `cdn2.example.net` in "Proxy-DNS-SVCB-Response":

~~~ example
HEADERS
:status = 307
proxy-dns-svcb-response-cancelled-on: h3-alpn, ech-defined
proxy-dns-svcb-response: :AAAGYW...:, :AAEGYW..., :AAEGYW...:
~~~

Where the "proxy-dns-svcb-response" contains all three RRs:

~~~ dns
www.example.com. 300 IN HTTPS 0 cdn.example.
cdn.example.      900 IN CNAME cdn2.example.net.
cdn2.example.net. 300 IN HTTPS 1 . alpn=h2,h3 ech=...
~~~

The client retries with CONNECT-UDP, using ECH with the keys from the response:

~~~ example
HEADERS
:method = CONNECT
:protocol = connect-udp
:scheme = https
:path = /.well-known/masque/udp/cdn2.example.net/443/
:authority = proxy.example
capsule-protocol = ?1
~~~

The proxy establishes the connection and responds:

~~~ example
HEADERS
:status = 200
proxy-status: proxy.example; next-hop="2001:db8:a::c"
~~~

Over this connection the client establishes its HTTP/3 HTTPS connection
to `www.example.com`.


## Cancel-On and HTTP connections to insecure port 80

The client sends CONNECT requesting HTTPS RRs
information with only the cancel-on for `https`:

~~~ example
HEADERS
:method = CONNECT
:authority = www.example.com:80
proxy-dns-svcb-request: HTTPS
proxy-dns-svcb-request-cancel-on: https
~~~

The proxy resolves `www.example.com` and encounters
that it has a CNAME to a CDN which has an HTTPS RR:

~~~ dns
www.example.com. 3600 IN CNAME cdn.example.
cdn.example.      600 IN HTTPS 1 . alpn=h2 ech=...
~~~

Because the `https` condition is satified, the proxy cancels
and returns both DNS RRs in "Proxy-DNS-SVCB-Response":

~~~ example
HEADERS
:status = 307
proxy-dns-svcb-response-cancelled-on: https
proxy-dns-svcb-response: :AAAGYW...:, :AAEGYW...
~~~

The client retries the CONNECT but this time to port 443 for HTTPS,
using ECH with the keys from the response, and makes the connection
to the TargetName from the HTTPS record:

~~~ example
HEADERS
:method = CONNECT
:authority = cdn.example:443
~~~

The proxy establishes the connection and responds:

~~~ example
HEADERS
:status = 200
proxy-status: proxy.example; next-hop="2001:db8:a::c"
~~~

Over this connection the client establishes its HTTP/2 HTTPS connection
to `www.example.com` with ECH configuration from the HTTPS RR.



# Security Considerations

## Client Trust of Proxy

As the client is relying on the proxy to resolve the SVCB and HTTPS
DNS records for it, the client is trusting the proxy as if it was a recursive
DNS resolver for the client. By omitting or modifying
SVCB/HTTPS RRs, the proxy could prevent the client from using ECH
thus exposing the TLS SNI to the proxy. The proxy could also omit
HTTPS RRs.

## Proxies Not Supporting Proxy-DNS-SVCB-Request-Cancel-On

If a client assumes that a proxy supports Proxy-DNS-SVCB-Request-Cancel-On
(or assumes it supports a given attribute), the client might end up having
the proxy establish a connection where it would have preferred one
not be created. This would be of particular concern if the client
pipelined DATA frames with payload (eg, the ClientHello with
a cleartext TLS SNI) if it was assuming the proxy would cancel
in the case of ECH being present.

Clients MUST NOT rely on "Proxy-DNS-SVCB-Request-Cancel-On" with `ech-defined`
for sending H/2 or H/3 DATA frames along with the CONNECT request unless
they have an out-of-band way of ensuring this is supposted by the Proxy.

## Reuse of HTTP connections to the proxy

While "Proxy-DNS-SVCB-Request-Cancel-On" allows the connection to an
HTTP/1.1 proxy to be reused for issuing a subsequent CONNECT or CONNECT-UDP
request, this does have risks for HTTP Request Smuggling and request
framing confusion. Clients SHOULD NOT reuse a connection to an HTTP/1.1
proxy after making the initial cancelled CONNECT request and instead
use a fresh connection to the proxy.

## Lack of DNSSEC signal

The DNS RRs returned in "Proxy-DNS-SVCB-Response" do not contain
information about whether the proxy performed DNSSEC validation
of the DNS responses, nor does it contain enough information
for the client to perform DNSSEC validation itself.

Clients with security or privacy needs relying on DNSSEC SHOULD NOT use
this mechanism and instead should use a separate secure connection
to a DNS resolver for obtaining HTTPS/SVCB RRs along with DNSSEC proofs.

\[TODO: Explore ways for the Proxy to signal DNSSEC information.
Options include:
1) Proxy returns the DNS "AD" flag as part of "Proxy-DNS-SVCB-Response
2) Client adds a "dnssec" attribute to "Proxy-DNS-SVCB-Request"
   which prompts the proxy to include ALL records needed by the client
   to validate DNSSEC proofs as part of the "Proxy-DNS-SVCB-Response" header
   field (which could be quite large).
\]


## Privacy of Client Preferences

The "Proxy-DNS-SVCB-Request" header field reveals that the client is interested
in SVCB information for the target. The "Proxy-DNS-SVCB-Request-Cancel-On" header
field further reveals the client's connection strategy preferences.

# IANA Considerations

## HTTP Header Field Registrations

This document registers the following HTTP header fields in the "Hypertext
Transfer Protocol (HTTP) Field Name Registry" defined in Section 16.3.1 of
{{!RFC9110}}:

| Field Name | Status |
|---|---|
| Proxy-DNS-SVCB-Request | permanent |
| Proxy-DNS-SVCB-Response | permanent |
| Proxy-DNS-SVCB-Request-Cancel-On | permanent |
| Proxy-DNS-SVCB-Response-Cancelled-On | permanent |

## Proxy-DNS-SVCB-Request-Cancel-On Token Registry

This document establishes a new "Proxy-DNS-SVCB-Request-Cancel-On Tokens"
registry. The registration policy is Specification Required ({{!RFC8126}}).

Initial entries:

| Token | Description | Reference |
|---|---|---|
| h3-alpn | ALPN contains "h3" in highest-priority RR | This document |
| ech-defined | ECH SvcParamKey present in highest-priority RR | This document |
| https | HTTPS DNS RR is present indicating HSTS-like behavior | This document |

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
