---
title: "Using Dates and Times In HTTP Requests"
abbrev: "Date Requests"
category: std

docname: draft-thomson-httpapi-date-requests-latest
ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Building Blocks for HTTP APIs"
keyword: Internet-Draft
venue:
  group: "Building Blocks for HTTP APIs"
  type: "Working Group"
  mail: "httpapi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/httpapi/"
  github: "martinthomson/http-request-date"
  latest: "https://martinthomson.github.io/http-request-date/draft-thomson-httpapi-date-requests.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    fullname: Martin Thomson
    organization: Mozilla
    email: mt@lowentropy.net

normative:

informative:

  UWT:
    title: "Unsanctioned Web Tracking"
    author:
      fullname: Mark Nottingham
    date: 2015-07-17
    target: https://www.w3.org/2001/tag/doc/unsanctioned-tracking/


--- abstract

HTTP clients rarely make use of the `Date` header field when making requests.
This document describes considerations for using the `Date` header field - or
other timestamp information - in requests.  A method is described for correcting
erroneous timestamps that might arise from differences in client and server
clocks. The risks of applying that correction technique are discussed.


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
field in {{date-req}}.  These apply to any use of timestamps in requests.  A new
type of problem report {{!PROBLEM=I-D.ietf-httpapi-rfc7807bis}} is defined in
{{problem-type}} for use in rejecting requests with a missing or incorrect
timestamps in requests.

{{skew}} explores the consequences of using timestamps in requests when client
and server clocks do not agree.  A method for recovering from differences in
clocks is described in {{correction}}.  {{scope}} describes the privacy
considerations that apply to this technique.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Dates or Timestamps in HTTP Requests {#date-req}

Most HTTP clients have no need to use the `Date` header field in requests.  This
only changes if it is important that the request not be considered valid at
another time.  As requests are - by default - trivially copied, stored, and
modified by any entity that can read them, the addition of a `Date` header field
is unlikely to be useful in many cases.

OHTTP {{?OHTTP}} is an example of where capture and replay of a request
might be undesirable.  Here, a partially trusted intermediary, an oblivious
proxy resource, receives encapsulated HTTP requests.  Though this entity cannot
read or modify these messages, it is able to delay or replay them.  The
inclusion of a `Date` header field in these requests might be used to limit the
time over which delay or replay is possible.

Signed HTTP requests {{SIGN}} and JSON Web Tokens {{?JWT=RFC7519}} are other
examples of where requests - or parts of requests - might be protected to
prevent certain attacks by intermediaries.  Signatures are often used to enhance
protections for requests for privileged action on resources.  Adding a `Date`
header field or some other field (`created` for {{?SIGN}} or `iat` for {{?JWT}})
and signing that information can ensure that the request cannot be captured and
used at a very different time to what was intended.

The inclusion of a timestamp might only be used to limit the period over which
information might be used.  It might be acceptable for the same request to be
made multiple times over a fixed period.

In other cases, the inclusion of a date or timestamp might be part of an
anti-replay strategy at a server.  A simple anti-replay scheme starts by
choosing a window of time anchored at the current time.  Requests with
timestamps that fall within this period are remembered and rejected if they
appear again; requests with timestamps outside of this window are rejected.
This scheme works for any monotonic value (see for example {{Section 3.4.3 of
?RFC4303}}) and allows for efficient rejection of duplicate requests with
minimal state.


# Date Not Acceptable Problem Type {#problem-type}

A server can send a 400-series status code in response to a request where the
`Date` request header field or other timestamp is either absent or not
acceptable to the server.  Including content of type "application/problem+json"
(or "application/problem+xml"), as defined in {{!PROBLEM}}, in that response
allows the server to provide more information about the error.

This document defines a problem type of
"https://iana.org/assignments/http-problem-types#date" for indicating that a
timestamp is missing or incorrect.  {{ex1}} shows an example response in
HTTP/1.1 format.

~~~ http-message
HTTP/1.1 400 Bad Request
Date: Mon, 07 Feb 2022 00:28:05 GMT
Content-Type: application/problem+json
Content-Length: 128

{"type":"https://iana.org/assignments/http-problem-types#date",
"title": "date field in request outside of acceptable range"}
~~~
{: #ex1 title="Example Response"}

A server MUST include a `Date` response header field in any responses that use
this problem detail type.  This allows the client to learn what the server
believes to be a correct time, which might allow for correction; see
{{correction}}.

In processing a `Date` header field or timestamp in a request, a server MUST
allow for delays in transmitting the request, retransmissions performed by
transport protocols, plus any processing that might occur in the client and any
intermediaries and any server processing prior to generating the response.
Additionally, the `Date` header field is only capable of expressing time with a
resolution of one second.  These factors could mean that the value a server
receives could be some time in the past.

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

For a server that uses timestamps in requests to limit the state kept for
anti-replay purposes, the amount of state might be all that determines the range
of values it accepts.


## Date Correction {#correction}

Even when a server is tolerant of small clock errors, a valid request from a
client can be rejected if the client clock is outside of the range of times that
a server will accept.  A server might also reject a request when the client
makes a request without a timestamp.

A client can recover from a failure that caused by a bad clock by adjusting the
time and re-attempting the request.

For a fresh response (see {{Section 4.2 of !CACHING=I-D.ietf-httpbis-cache}}),
the client can re-attempt the request immediately, copying the `Date` header
field from the response into its new request.  If the response is stale, the
client can add the age of the response to determine the time to use in a
re-attempt; see {{intermediaries}} for more.

An immediate retry of a request reveals the round trip time between the server
and the client.  This might reveal some information about the network location
of the client, even if the client uses intermediaries to obscure this
information.  A client might add a delay before retrying a request to obscure
this information.

In addition to adjusting for response age, the client could adjust the time it
uses based on the elapsed time since it estimates when the response was
generated.  If the client retries a request immediately, any additional
increment is likely to be less than the one second resolution of the `Date`
header field under most network conditions.  Such an adjustment might be used by
the server to learn what the client believes the round-trip time to the server
is.  This depends on the resolution of the timestamp field that is used, but
this could also be used to gain some information about the network location of
the client.


## Limitations of Date Correction {#scope}

Clients MUST NOT accept the time provided by an arbitrary HTTP server as the
basis for system-wide time.  Even if the client code in question were able to
set the time, altering the system clock in this way exposes clients to attack.
The source of system time information needs to be trustworthy as the current
time is a critical input to security-relevant decisions, such as whether to
accept a server certificate {{?RFC6125}}.

Use of date correction allows requests that use the correction to be correlated.
Limitations on use of date corrections is necessary to ensure privacy.  An
immediate retry of an identical request with an update `Date` header field is
safe in that it only provides the server with the ability to match the retry to
the original request.

Anything other than a single retry requires careful consideration of the privacy
implications.  Use of the same date correction for other requests can be used to
link those requests to the same client.  Using the same date correction is
equivalent to connection reuse, cookies, TLS session tickets, or other state a
client might carry between requests.  Linking requests might be acceptable, but
in general only where other forms of linkage already exist.

Clients MUST NOT use the time correction from one server when making requests of
another server.  Using the same date correction across different servers might
be used by servers to link client identities and to exchange information via a
channel provided by the client.

For clients that maintain per-server state, the specific date correction that is
used for each server MUST be cleared when removing other state for that server
to prevent re-identification.  For instance, a web browser that remembers a date
correction would forget that correction when removing cookies and other state.


## Intermediaries and Date Corrections {#intermediaries}

Some intermediaries, in particular those acting as reverse proxies or gateways,
will rewrite the `Date` header field in responses. This applies especially to
responses served from cache, but this might also apply to those that are
forwarded directly from an origin server.

For responses that are forwarded by an intermediary, changes to the `Date`
response header field will not change how the client corrects its clock. Errors
only occur if the clock at the intermediary differs significantly from the clock
at the origin server or if the intermediary updates the `Date` response header
field without also adjusting or removing the `Age` header field on a stale
response.

Servers that condition their responses on the `Date` header field SHOULD either
ensure that intermediaries do not cache responses (by including a
`Cache-Control` directive of `no-store`) or designate the response as
conditional on the value of the `Date` request header field (by including the
token "date" in a `Vary` header field).


# Security Considerations

Including a timestamp in requests reveals information about the client clock.
This might be used to fingerprint clients {{UWT}} or to identify clients with
vulnerability to attacks that depend on incorrect clocks.

{{scope}} contains a discussion of the security and privacy concerns associated
with date correction.


# IANA Considerations

IANA are requested to create a new entry in the "HTTP Problem Type" registry
established by {{!PROBLEM}}.

Type URI:
: https://iana.org/assignments/http-problem-types#date

Title:
: Date Not Acceptable

Recommended HTTP Status Code:
: 400

Reference:
: {{problem-type}} of this document

--- back

# Acknowledgments
{:numbered="false"}

This document is a result of discussions about how to provide anti-replay
protection for OHTTP in which Mark Nottingham, Eric Rescorla, and Chris Wood
were instrumental.
