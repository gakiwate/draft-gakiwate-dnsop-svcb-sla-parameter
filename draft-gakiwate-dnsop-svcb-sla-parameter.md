---
title: "Service Binding Mapping for Service Levels"
category: info

docname: draft-gakiwate-dnsop-svcb-sla-parameter-latest
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
  github: "gakiwate/draft-gakiwate-dnsop-svcb-sla-parameter"
  latest: "https://gakiwate.github.io/draft-gakiwate-dnsop-svcb-sla-parameter/draft-gakiwate-dnsop-svcb-sla-parameter.html"

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
that a service supports specific service levels or use types, such as
"interactive", "background" or "realtime", at that endpoint.  Specifically, a
service can use this new parameter to indicate which specific endpoints a client
MAY use for the specific service levels indicated. By providing this
information, clients can make informed decisions about which service endpoints
to use based on their specific applications needs at that time.

--- middle

# Introduction

The SVCB and HTTPS resource records (RRs) provide clients with instructions to
connect to a service while avoiding transient connections to a suboptimal
default server {{!SVCB=RFC9460}}. However, there are scenarios in which a client
might need to interact differently with a service based on their specific needs
at the time.  Traditionally, the solution to this problem has been to create
separate services --- one for "interactive" tasks and another for "background"
tasks.  However, there are scenarios where a client based on its local context,
knows that it can improve resource utilization by choosing a "background"
service level for tasks that are typically considered "interactive". The current
solution of splitting services does not allow clients to utilize its local
context to improve service adaptability without having to delineate tasks as
"background", or "interactive" apriori.

The document defines a new SvcParamKey "sla", short for Service Level Attribute,
to provide a standardized way for a client to dynamically discover different
service endpoints offering different service levels using SVCB / HTTPS RRs.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Motivation

In modern network environments, deployment scenarios often involve diverse
application traffic types, such as interactive video conferencing, real-time
gaming, and background file synchronization, each with distinct performance
requirements. Interactive traffic typically demands low latency and high
responsiveness, while background traffic prioritizes throughput and efficiency
over immediacy. Moreover, where in the network enviornment these demands can
be relaibly and efficiently met also differ.

The "sla" SvcParamKey enables authoritative DNS servers to signal these
distinctions, thereby allowing clients to route traffic intelligently.  This
capability is particularly useful in load-balancing scenarios, where providers
can allocate interactive traffic to low-latency, high-performance servers while
directing background tasks to cost-effective, lower-performance infrastructure.

Allowing clients explicit control over which service level to use benefits a
range of stakeholders, including service operators, by improving quality of
service (QoS), optimizing resource utilization, and enhancing end-user
experience.

# The "sla" SvcParamKey

The "sla" SvcParamKey, short for Service Level Attribute, is used to indicate that
a service described in SVCB RR supports a specific service level use type. A
endpoint can support multiple service levels. To support multiple service level
types at different endpoints, multiple SVCB / HTTPS RRs SHOULD be used.

The presentation value for "sla" SHALL be a comma-seperated list
of one or more integers (1-octet) indicating the service level use. A service
endpoint may serve more than one service level. A lower value indicates a more
cost-effective, lower-performance endpoint while a higher value indicates a
low-latency, high-performance endpoint.  A `sla=0` can be thought of as a
endpoint supporting "background" tasks such as performing backups.  A `sla=1`
can be thought of as a endpoint supporting "interactive" tasks such sending
messages, while a `sla=2` can be thought of as a endpoint supporting "real-time"
tasks such as invoking a voice assistant.  This documents defines three service
levels, and restricts the values of sla to 0, 1, or 2.  Future documents may
choose to extend this number.

The wire format for the "sla" parameter is a sequence of service levels each
denoted by a 1-octet numeric.

## Client Behavior

The "sla" SvcParamKey in the SVCB / HTTPS records indicates that the domain
owner recommends the use a specific endpoint for a specific service level use
type. On receiving RRs with the "sla" SvcParamKey, clients SHOULD use it as a
filter to decide which RRs to use when.  Since this document only defines three
service levels, a client SHOULD ignore any SVCB RRs with an "sla" value greater
than 2. It is possible that certain RRs are never used by some clients.

To use the SVCB / HTTPS RRs, the client first determines its service level. The
client then filters down to only SVCB / HTTPS RRs that match its own perceived
service level. A SVCB / HTTPS RR is considered to match if any of the "sla"
values associated match the client's service level.  A RR without a "sla" value
is considered equivalent to `sla=0,1,2` and thus matches all client service
levels.

Once the client filters SVCB / HTTPS RRs to ones that offer the service level it
desires, the client proceeds with processing the remaining SVCB / HTTPS RRs as
normal.  If no RRs match, SVCB resolution has failed, and the list of available
endpoints is empty. To prevent this behavior, service operators SHOULD NOT have
gaps in supported "sla" values.

## Example Use

A service that supports "interactive" and "background" endpoints can signal
support to the client by populating a SVCB / HTTPS RR as so:

```
        svc.example.com 7200 IN SVCB 1 background.svc.example.com ( alpn=h2, sla=0, mandatory=sla)
```

```
        svc.example.com 7200 IN SVCB 1 interactive.svc.example.com ( alpn=h2, sla=1,2 )
```

In the above example, a client that determines it wants a "realtime" (sla=2)
service level will first filter for RRs with an `sla` value of 2. In this case,
only one RR satisfies the constraint and the client can use
`interactive.svc.example.com` endpoint. The service operator can also use the
`mandatory` flag for the background service, so that clients who do not support
the "sla" SvcParamKey default to their endpoint of choice.

A service operator can also have the same service level offered at multiple
service endpoints with an additional SVCB record as so:

```
        svc.example.com 7200 IN SVCB 1 interactive.svc.example.com ( alpn=h2, sla=1 )
```

```
        svc.example.com 7200 IN SVCB 1 realtime.svc.example.com ( alpn=h2, sla=1,2 )
```

With this additional RR, a client that determines it wants a "interactive"
(sla=1) service level will match two RRs. After filtering down to these two RRs,
the client will the proceed to process the RRs as before. In this case, since
both the RRs have the same SvcPriority level, the client will apply a random
shuffle before using them {{SVCB}}.

# Security Considerations

# IANA Considerations

## SVCB Service Parameter

This document adds the following entry to the "Service Parameter Keys (SvcParamKeys)"
registry.

| Number  | Name    | Meaning                      | Change Controller | Reference       |
| ------- | ------- | ---------------------------- | ----------------- | --------------- |
| TBD     | sla     | Service Level Attribute      |                   |                 |

--- back

# Acknowledgments

{:numbered="false"}
