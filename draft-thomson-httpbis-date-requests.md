---
title: "Using The Date Header Field In HTTP Requests"
abbrev: "Date Requests"
category: std

docname: draft-thomson-httpbis-date-requests-latest
ipr: trust200902
area: "Applications and Real-Time"
workgroup: "HTTP"
keyword: Internet-Draft
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "martinthomson/http-request-date"
  latest: "https://martinthomson.github.io/http-request-date/draft-thomson-httpbis-date-requests.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    name: Martin Thomson
    organization: Mozilla
    email: mt@lowentropy.net

normative:

informative:


--- abstract

HTTP clients rarely make use of the Date field when making requests.  This
document describes considerations for using Date fields in requests.  A method
of correcting erroneous in Date fields that might arise from a bad clock is
defined. The risks of applying that correction technique are discussed.


--- middle

# Introduction

Many HTTP requests are timeless.  That is, the contents of the request are not
bound to a specific point in time.  Thus, the use of the HTTP `Date` header field
in requests is rare; see {{Section 6.6.1 of ?HTTP=I-D.ietf-httpbis-semantics}}.

However, in some contexts, it is important that a request only be valid over a
small period of time.  One such context is when requests are signed
{{?SIGN=I-D.ietf-httpbis-message-signatures}}, where including a time in a
request might prevent a signed request from being reused at another time.
Similarly, some uses of OHTTP {{?OHTTP=I-D.ietf-ohai-ohttp}} might depend on the
same sort of replay protection.  It is possible to make anti-replay protections
at servers more efficient if requests from either far in the past or into the
future can be rejected.

This document describes some considerations for using the `Date` request header
field.  The 4xx (Date Not Acceptable) header field is defined in {{status-code}}
for use in rejecting requests with a missing or incorrect `Date` header field.

{{skew}} explores the consequences of using `Date` in requests when client and
server clocks do not agree.  A method for recovering from differences in clocks
is described in {{correction}}.  {{scope}} describes the privacy considerations
that apply to this technique.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Date in HTTP Requests

Most HTTP clients have no need to use the `Date` header field in requests.  This
only changes if it is important that the request not be considered valid at
another time.  As requests are - by default - trivially copied, stored, and
modified by any entity that can read them, the addition of a `Date` header field
is unlikely to be useful in many cases.

Signed HTTP requests are one example of where requests might be available to
entities that are not permitted to alter their contents.  Adding a `Date`
request header field - and signing it - ensures that the request cannot be used
at a very different time to what was intended.

OHTTP {{?OHTTP}} is another example of where capture and replay of a request
might be undesirable.  Here, a partially trusted intermediary, an oblivious
proxy resource, receives encapsulated HTTP requests.  Though this entity cannot
read or modify these messages, it is able to delay or replay them.  The
inclusion of a `Date` header field in these requests might be used to limit the
time over which delay or replay is possible.

In both cases, the inclusion of a `Date` header field might be part of an
anti-replay strategy at a server.  A simple anti-replay scheme starts by
choosing a window of time anchored at the current time.  Requests with
timestamps that fall within this period are remembered and rejected if they
appear again; requests with timestamps outside of this window are rejected.
This scheme works for any monotonic value (see for example {{Section 3.4.3 of
?RFC4303}}) and allows for efficient rejection of duplicate requests with
minimal state.


# 4xx (Date Not Acceptable) Status Code {#status-code}

A server sends a 4xx (Date Not Acceptable) status code in response to a request
where the `Date` request header field is either absent or indicates a time that
is not acceptable to the server.  A server MUST include a `Date` response header
field in any responses with a 4xx (Date Not Acceptable) status code.

In processing a `Date` header field in a request, a server MUST allow for delays
in transmitting the request, retransmissions performed by transport protocols,
plus any processing that might occur in the client and any intermediaries, and
those parts of the server prior to processing the field.  Additionally, the
`Date` header field is only capable of expressing time with a resolution of one
second.  These factors could mean that the value a server receives could be some
time in the past.

Differences between client and server clocks are likely to be a source of most
disagreements between the server time and the time expressed in `Date` request
header field.  {{skew}} will explore this problem in more detail and offer some
means of handling these disagreements.


# Clock Skew {#skew}

Perfect synchronization of client and server clocks is an ideal state that
generally only exists in tightly controlled settings.  In practice, despite good
availability of time services like NTP {{?NTP=RFC5905}} Internet-connected
endpoints often disagree about the time (see for example Section 7.1 of
{{?CLOCKSKEW=DOI.10.1145/3133956.3134007}}).

The prevalence of clock skew could justify servers being more tolerant of a
larger range of values for the `Date` request header field.  This includes
accepting times that are a short duration into the future in addition to times
in the past.

For a server that uses the `Date` request header field to limit the state kept
for anti-replay purposes, the amount of state might be all that determines the
range of values it accepts.


## Date Correction {#correction}

Even when a server is tolerant of small clock errors, a valid request from a
client can be rejected if the client clock is outside of the range of times that
a server will accept.  A server might also reject a request when the client
makes a request without a `Date` field.

A client can recover from a failure that caused by a bad clock by adjusting the
time and re-attempting the request.

For a fresh 4xx (Date Not Acceptable) response (see {{Section 4.2 of
!CACHING=I-D.ietf-httpbis-cache}}), the client can re-attempt the request,
copying the `Date` header field from the response into its new request.  If the
response is stale, the client can add the age of the response to determine the
time to use in a re-attempt; see {{intermediaries}} for more.

In addition to adjusting for response age, the client can adjust the time it
uses based on the elapsed time since it estimates when the response was
generated.  Note however that if the client retries a request immediately, any
additional increment is likely to be less than the one second resolution of the
`Date` header field under most network conditions.


## Limitations of Date Correction {#scope}

Clients MUST NOT accept the time provided by an arbitrary HTTP server as the
basis for system-wide time.  Even if the client code in question were able to
set the time, this exposes clients to attack.  The source of system time
information needs to be trustworthy as the current time is a critical input to
security-relevant decisions, such as whether to accept a server certificate
{{?RFC6125}}.

Use of date correction allows requests that use the correction to be correlated.
An immediate retry of an identical request with an update `Date` header field
only provides the server with the ability to identify where the correction
originated, but making multiple requests with the same correction links all of
those requests.

Anything other than an immediate retry requires careful consideration of the
privacy implications.  Use of the same date correction for other requests can be
used to link those requests to the same client.  Using the same date correction
is equivalent to connection reuse, cookies, TLS session tickets, or other state
a client might carry between requests.  Linking requests might be acceptable
only where other forms of linkage already exist.

Limitations on use of date corrections is necessary to ensure privacy.  At a
minimum, clients MUST NOT use the time correction from one server when making
requests of another server.  Using the same date correction across different
servers might be used by servers to create a communication channel.  For clients
that maintain per-server state, the specific date correction that is used for
each server MUST be cleared when removing other state for that server.  For
instance, a web browser that remembers a date correction would forget that
correction when removing cookies and other state.


## Intermediaries and Date Corrections {#intermediaries}

Some intermediaries, in particular those acting as reverse proxies or gateways,
will rewrite the Date header field in responses. This applies especially to
responses served from cache, but this might also apply to those that are
forwarded directly from an origin server.

Servers that condition their responses on the Date header field SHOULD either
ensure that intermediaries do not cache responses (by including a
`Cache-Control` directive of `no-store`) or designate the response as
conditional on the value of the `Date` request header field (by including the
token "date" in a `Vary` header field).

For responses that are forwarded by an intermediary, changes to the `Date`
response header field will not change how the client corrects its clock. Errors
only occur if the clock at the intermediary differs significantly from the clock
at the origin server or if the intermediary updates the `Date` response header
field without also adjusting or removing the `Age` header field on a stale
response.


# Security Considerations

Including a `Date` header field in requests reveals information about the client
clock.  This might be used to identify clients with vulnerability to attacks
that depend on incorrect clocks.

{{scope}} contains a discussion of the security and privacy concerns associated
with date correction.


# IANA Considerations

IANA are requested to add a value to the "Hypertext Transfer Protocol (HTTP)
Status Code Registry" at [](https://www.iana.org/assignments/http-status-codes)
as follows:

Value:
: 4xx (to be assigned)

Description:
: Date Not Acceptable

Reference:
: {{status-code}} of this document

--- back

# Acknowledgments
{:numbered="false"}

This document is a result of discussions about how to provide anti-replay
protection for OHTTP in which Chris Wood and Eric Rescorla were instrumental.
