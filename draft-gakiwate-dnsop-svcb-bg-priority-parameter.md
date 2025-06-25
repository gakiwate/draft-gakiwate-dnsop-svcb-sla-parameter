---
title: "Service Binding Mapping for Background Requests"
category: info

docname: draft-gakiwate-dnsop-svcb-bg-priority-parameter-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "Domain Name System Operations"
keyword:
venue:
  group: "Domain Name System Operations"
  type: ""
  mail: "dnsop@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dnsop/"
  github: "gakiwate/draft-gakiwate-dnsop-svcb-bg-priority-parameter"
  latest: "https://gakiwate.github.io/draft-gakiwate-dnsop-svcb-bg-priority-parameter/draft-gakiwate-dnsop-svcb-bg-priority-parameter.html"

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

--- abstract

This document defines a new SvcParamKey for use in Service Binding (SVCB) and
HTTPS DNS resource records which enables authoritative DNS servers to indicate
that a client can use an alternative endpoint for "background" requests. By
providing this information, clients can make informed decisions about which
service endpoints to use based on their specific applications needs at the
time of making connections.

--- middle

# Introduction

The SVCB and HTTPS resource records (RRs) provide clients with instructions to
connect to a service while avoiding transient connections to a suboptimal
default server {{!SVCB=RFC9460}}. However, server deployments may wish clients
to interact with a service differently depending on the context in which they
are operating.  For instance, consider a weather application. When running in
the background, the application may retrieve data infrequently and can thus
tolerate higher latency. In contrast, when an user is actively interacting with
the application, the same requests now require low-latency responses from nearby
servers.  Traditionally, such use cases are handled by provisioning distinct
endpoints --- one for "interactive" requests and another for "background"
requests.  This approach does not work in scenarios where a service with a
single hostname needs to be used for both "background" and "interactive"
requests. Moreover, this static partitioning imposes rigid service delineation
and prevents clients from adapting dynamically. In practice, a client may, based
on local context, dynamically determine that a request typically considered
"interactive" can be fulfilled using a latency-tolerant path, thereby improving
resource utilization and reducing operational cost.

This document defines a new SvcParamKey, `bg-priority` (short for "background
priority"), to enable clients to dynamically discover service endpoints suited
for latency-tolerant requests. The presence of `bg-priority=1` on a service
endpoint signals that it is optimized for background requests and can be selected
as needed by clients. This key allows clients to filter down to endpoints
that apply to them at the time of using DNS answers to establish connections.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Motivation

In modern network environments, deployment scenarios often involve diverse
types of application traffic, such as interactive video conferencing, real-time
gaming, and background file synchronization, each with distinct performance
requirements. Interactive traffic typically demands low latency and high
responsiveness, whereas background traffic prioritizes throughput and efficiency
over immediacy. Moreover, where in the network environment these demands can
be relaibly and efficiently met also differ.

The `bg-priority` SvcParamKey enables authoritative DNS servers to signal
these distinctions, thereby allowing clients to route traffic intelligently.
This capability is particularly valuable in load-balancing resource-constrained
scenarios, where providers can allocate interactive traffic to low-latency,
high-performance servers while directing background requests to higher-latency but
more cost-effective underutilized servers.

Allowing clients explicit control over which service level to use benefits a
range of stakeholders, including service operators, by improving quality of
service (QoS), optimizing resource utilization, and enhancing end-user
experience.

# The "bg-priority" SvcParamKey

The `bg-priority` SvcParamKey, short for "background priority," is used to
indicate that a service endpoint described in an SVCB or HTTPS record is
optimized for background, latency-tolerant traffic.  The presentation value
for bg-priority SHALL be either 0 or 1.

A value of 1 indicates that the endpoint is designated for background requests,
while a value of 0 indicates that the endpoint may be optimized for interactive
or real-time requests. If the `bg-priority` paramter is absent, then the client
must assumes that the endpoint is designated for both background and interactive
requests. The wire format for the `bg-priority` parameter is a single 1-octet
value, with only the values `0x00` and `0x01` currently defined.

## Client Behavior

The `bg-priority` SvcParamKey in SVCB and HTTPS records indicates that the
service operator recommends the use of a particular endpoint for background,
latency-tolerant requests. Upon receiving RRs containing the `bg-priority` key,
clients should treat it as a signal to guide endpoint selection based on their
current operational context.

To use SVCB or HTTPS RRs, a client first determines whether the current network
request is suitable for background handling. The mechanism for making this
determination is implementation specific and left to the discretion of
the client. For example, a client may provide an explicit signalling
interface that allows an application or user to indicate a given request is
background in nature. The client may also rely on the local operational
context to infer whether background handling is appropriate.

If the client determines background handling to be appropriate, the client
filters the RRSet to include only those records with `bg-priority=1`. If not,
the client filters the RRSet to include only those records with
`bg-priority=0`. A record that does not include the `bg-priority` key is
considered to be suitable for all traffic types and should always be included.

Once the client filters the SVCB / HTTPS RRs based on this designation, it
proceeds with the remaining records using standard selection logic {{SVCB}}.  If
no suitable records remain after filtering, SVCB resolution is considered to
have failed, and the client should fallback appropriately. To avoid unintended
resolution failures, service operators should ensure that if any service in a set contains the `bg-priority` parameters,
then at least one service exists for each `bg-priority=0` and `bg-priority=1`, or else at least one service exists that does not contain
the `bg-priority` parameter to catch all traffic.

## Example Use

A service that supports endpoints optimized for background i.e.,
latency-tolerant requests can advertise support to clients using the
`bg-priority` SvcParamKey as illustrated below:

~~~
svc.example.com 7200 IN HTTPS 1 . (
        alpn=h2, bg-priority=0 )
svc.example.com 7200 IN HTTPS 2 background.svc.example.com (
        alpn=h2, bg-priority=1, mandatory=bg-priority )
~~~

In this example, a client that determines a request is suitable for background
handling filters the SVCB / HTTPS RRs to select only those records with `bg-priority=1`.
In this case, only one RR satisfies the constraint, and the client uses the
`background.svc.example.com` endpoint despite its higher `SvcPriority` (i.e.,
lower preference) compared to the other advertised RR. If a client does not
support the `bg-priority` key, the client will ignore it and continue
using the default endpoint (`SvcPriority=1`) as before. The service operator can
use `mandatory=bg-priority` for the background designated RR to ensure that
a client lacking support will skip this RR entirely.

## Operational Considerations

If a service operator does not explicitly include `bg-priority=0` in the default
HTTPS RR (as shown in the example above), a client that filters for
background-eligible endpoints will treat both RRs as valid matches. This is
because a record without the `bg-priority` key is considered suitable for all
traffic types. After filtering, the client will proceed with standard SVCB
selection logic and prefer the record with the lowest SvcPriority. In such
cases, the client will select the default endpoint, even if an endpoint
optimized for background requests is available.

To avoid unintended selection behavior, service operators should be cautious
when mixing RRs that include `bg-priority` key with those that omit it.
Explicitly setting `bg-priority` can help ensure more predictable client
behavior.

Additionally, partitioning traffic across different service tiers may interfere
with connection pooling in some client implementations. This could lead to
increased connection overhead or degraded performance in scenarios that rely
heavily on connection reuse. Service operators should evaluate the impact on
connection management when deploying `bg-priority` based routing.

# Security Considerations

TODO: Security considerations

# IANA Considerations

## SVCB Service Parameter

Once standardized, this document would ask to add the following entry to the
"Service Parameter Keys (SvcParamKeys)" registry.

| Number  |      Name         |                   Meaning                      | Change Controller | Reference       |
| ------- | ----------------- | ---------------------------------------------- | ----------------- | --------------- |
| TBD     | bg-priority       |   Endpoint is suitable for background traffic  |                   | (this document) |

--- back

# Acknowledgments
{:numbered="false"}

Thank you to Ben Schwartz, Ryan Watson, and many others for their feedback and
suggestions on this document.
