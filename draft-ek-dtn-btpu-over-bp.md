---
title: "Bundle Transfer Protocol - Unidirectional (BTPU) over BPv7"
abbrev: "BTPU over BPv7 (TUBE)"
docname: draft-ek-dtn-btpu-over-bp-latest
category: exp
submissiontype: IETF
ipr: trust200902
v: 3
stand_alone: true
workgroup: "Delay/Disruption Tolerant Networking"
venue:
  group: "Delay/Disruption Tolerant Networking"
  type: "Working Group"
  mail: "dtn@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dtn/"
  github: "ekline/draft-dtn-btpu-over-bp"
  latest: "https://ekline.github.io/draft-dtn-btpu-over-bp/draft-ek-dtn-btpu-over-bp.html"
keyword:
  - Bundle Protocol
  - convergence layer
  - encapsulation
  - TUBE
author:
  - fullname: Erik Kline
    initials: E.
    surname: Kline
    organization: Aalyria Technologies
    email: ek.ietf@gmail.com

--- abstract

This document specifies how to carry the Bundle Transfer Protocol -
Unidirectional (BTPU) over Bundle Protocol Version 7 (BPv7). In a Tunnel
Using BTPU Encapsulation (TUBE), the application data unit of each outer
bundle carries exactly one BTPU link-layer protocol data unit. BTPU can
pack several inner bundles together or segment one across outer bundles.
The tunnel can carry an existing BPv7 fragment without waiting for the
other fragments. TUBE specifies when a channel can be reused despite
out-of-order delivery of outer bundles.

--- middle

# Introduction

A spacecraft sends outer bundles through two ground stations to the same
tunnel egress. An outer bundle queued at one station can arrive days
after later transfers through the other station have completed.

A Tunnel Using BTPU Encapsulation (TUBE) combines BPv7 delivery
{{!RFC9171}} with BTPU framing, packing, segmentation, and reassembly
{{!I-D.ietf-dtn-btpu}}. The tunnel is one convergence-layer hop for an
inner bundle, regardless of the outer path. Each direction operates
independently.

Bundle-in-Bundle Encapsulation (BIBE) {{?I-D.ietf-dtn-bibe}} uses the
same forwarding model but a different payload format. An implementation
can reuse BTPU across TUBE and other bindings, such as BTPU over Ethernet
{{?I-D.ek-dtn-ethernet}}. A node offering both BIBE and TUBE configures
distinct receiving endpoints for the two formats.

## Conventions and terminology

{::boilerplate bcp14-tagged}

An *outer bundle* is a BPv7 bundle whose application data unit (ADU)
carries a BTPU link-layer protocol data unit (*PDU*). An *inner bundle*
is a BPv7 bundle forwarded through the tunnel. *Session*, *Transfer*,
*Transfer Window*, and the message names are defined in {{I-D.ietf-dtn-btpu}}.

# Endpoints and channel identification

The ordered pair (outer source EID, outer destination EID) identifies one
unidirectional BTPU logical channel. Compare the full endpoint identifiers
(EIDs) using their scheme's equality rules. The egress MUST select the
channel before applying a BTPU state transition. Each channel has its own
session and transfer window.

For each direction, configure a singleton destination EID at the egress
and one or more source EIDs the ingress is authorized to use as its BP
source node ID. Outer bundles MUST NOT have an anonymous source. Configure
the egress to accept those sources and dispatch the outer ADU to TUBE.
If several processes serve the destination, they MUST maintain consistent
channel state.

Under the ipn scheme, the proposed well-known TUBE destination service
number is TBD1.

An example of a configured TUBE egress EID is
`dtn://egress.example/.well-known/tube`. The demux string
".well-known/tube" is illustrative and follows the spelling used
by {{?RFC8615}}. The proposed well-known dtn demux is TBD2.

Both identifiers apply only to the outer destination EID. A deployment
can configure a different ipn service number or dtn demux for that
destination. Neither identifier constrains the choice of source EID.

The ingress MAY use an ephemeral source EID for a segmented transfer.
All outer bundles for that transfer MUST use the same source and
destination EIDs. For example, the ingress can select an ipn service
number from a pool on its own fully qualified node number (Sections 5.3
and 9.3 of {{!RFC9758}}). The egress can authorize the node and the
entire service-number range while distinguishing channels to the same
destination by their source service numbers.

# Encapsulation

## Payload mapping and forwarding

After its Bundle Protocol Agent (BPA) performs normal forwarding-time
processing under Section 5.4 of {{RFC9171}}, the ingress MUST capture one
serialization of the inner bundle and reuse those octets for every segment
and repetition of the transfer. The tunnel treats the octets as opaque,
even for a protected inner bundle. The ingress can encapsulate a BPv7
fragment without waiting for any other fragment of the same bundle.

The ADU of each outer bundle MUST consist of exactly one nonempty BTPU
Link-layer PDU, as defined in {{I-D.ietf-dtn-btpu}}. There is no additional
envelope, length prefix, or administrative record header. The ingress MUST
clear the administrative record flag and set the "Bundle must not be
fragmented" flag on each outer bundle. Each new outer bundle has its own
BP identity, even when it repeats a BTPU message.

When sending an inner bundle without segmentation, the ingress MUST use a
BTPU Bundle Message. The ingress and egress MUST support segmented
transfers as specified in Section 4 of {{I-D.ietf-dtn-btpu}}. The egress
can receive unsegmented inner bundles on a stable EID pair without
session signaling (Section 8.1 of {{I-D.ietf-dtn-btpu}}).

Submitting all required outer bundles to the outer BPA completes the local
handoff, not delivery to the egress. If the ingress cannot construct or
submit them all, it reports a local forwarding failure to its BPA. To retry
the same transfer, the ingress preserves the assigned octets and indices.
If it cannot, it starts a new transfer under the channel reuse rules.

## PDU and reassembly limits {#pdu-and-reassembly-limits}

For each direction, provision a send maximum supported by the egress.
The ingress MUST NOT send a PDU longer than that maximum. Measure PDU
length in octets of the outer ADU before applying outer bundle security.
TUBE defines no in-band discovery or default limit.

The egress MAY accept a PDU above the provisioned send maximum if the
entire PDU fits within its local receive limit and the outer bundle meets
its admission policy. The egress MUST reject a PDU above its local receive
limit. Any rejection for size MUST precede processing of any message in
that PDU.

The path or contact can impose a separate limit on the serialized outer
bundle size. The ingress MUST choose a PDU that fits after accounting for
BP encoding, extension blocks, and security overhead. If the required
messages and data will not fit, the ingress defers transmission or selects
another forwarding opportunity.

The egress separately limits the maximum serialized inner bundle size it
accepts for reassembly. An inner bundle need not fit in one PDU.

## Expiry and packing

The ingress MUST select one expiry for a segmented transfer, no later than
the expiry of its inner bundle. It MUST set each outer bundle's lifetime
to the interval remaining until that expiry when it creates the outer
bundle. An outer bundle created later to repeat a message therefore has a
shorter lifetime. Where the ingress lacks an accurate clock, it uses the
remaining inner bundle lifetime and locally elapsed time, and supplies the
outer Bundle Age block as required by {{RFC9171}}. If it cannot establish
a remaining lifetime, it cannot use this tunnel for the inner bundle.
This follows the expiry model in {{I-D.ietf-dtn-bibe}}.

All segmented transfers represented in one outer bundle MUST have the same
chosen expiry. The outer bundle MUST also expire no later than any whole
inner bundle it contains. The ingress can select a common expiry when it
starts several transfers or use separate outer bundles for transfers with
different expiries. It MUST select outer bundle protection acceptable for
every object in that outer bundle.

# Sessions over a store-and-forward path

## Starting a session {#starting-a-session}

A delayed Session Start can cancel transfers on the same channel
(Section 5.1 of {{I-D.ietf-dtn-btpu}}). Outer bundles can arrive out of
order across a sender restart.

The ingress MUST begin any PDU containing a BTPU Transfer message with a
valid Session Start. It MUST ensure that every Transfer message in the PDU
belongs to the announced session; a Transfer message has no Session ID for
the egress to verify independently. A PDU MUST contain at most one Session
Start and MUST NOT contain a Session End. A PDU carrying only ordinary
Bundle Messages needs no Session Start (Section 8.1 of {{I-D.ietf-dtn-btpu}}).

An ingress can isolate a segmented transfer by assigning it a fresh source
EID. The following sections specify when the ingress can reuse an EID pair.

## Reusing a channel within a session

Sections 5 and 5.1 of {{I-D.ietf-dtn-btpu}} define how the egress
initializes and advances the Transfer Window. The ingress MAY start
another transfer on the channel when it has a reasonable basis to expect
that advancing the window will preserve every earlier transfer that can
still complete. It MUST account for the span of Transfer Numbers in outer
bundles that can still arrive in any admissible order. Consider both the
window the egress may already hold and the window it would initialize if
a later outer bundle arrived first. Without such a basis, the ingress
SHOULD defer the transfer or use a different source EID.

The ingress MUST prevent a delayed outer bundle from being mistaken for
part of a new transfer because of serial-number wrap or an ambiguous
serial-number distance. It can switch channels before the span becomes
ambiguous. The local queue alone does not bound that span.

## Starting another session on a used EID pair {#starting-another-session-on-a-used-eid-pair}

For a new session, including after restart, the ingress SHOULD use a fresh
EID pair unless it can establish both of the following:

* No earlier transfer can still need the channel.
* No delayed outer bundle from the old session can disrupt the new one.

Confirmation that the old transfers completed does not establish that all
old outer bundles have drained: a delayed Session Start can still arrive.
The ingress can wait until all old outer bundles, including repetitions
and control messages, are known to have expired, or establish that the
relevant queues and paths have drained. Expiry of an individual transfer
does not retire a shared channel.

An ingress relying on expiry for reuse MUST retain enough allocation
state across restart to enforce this rule, or select source EIDs known
to be available. If age estimates do not bound the reuse time, it SHOULD
use a fresh source EID. If none is available, it can defer the transfer
or use another route.

# Reception and delivery

Normal BP reception and expiry checks apply. If an outer bundle fails
required security or peer authorization ({{outer-bundle-admission}}),
or local receive admission (including the local receive limit in
{{pdu-and-reassembly-limits}}), the egress MUST reject it without changing
BTPU state or handing any inner bundle from it to the BPA.

After admission, any of the following violates TUBE session framing
({{starting-a-session}}):

* The PDU contains a Transfer message but does not begin with a valid
  Session Start.
* The PDU contains more than one Session Start.
* The PDU contains a Session End.

Session Start and Session End count by message type at known boundaries.
For a PDU with any such violation, the egress MUST leave the channel's
BTPU session and transfer state unchanged, even if valid control or
Transfer messages precede the violation. It MUST NOT infer a Transfer
message's session from its Transfer Number or the channel's current
session.

The egress MUST still process each valid, independently delimited Bundle
Message that needs no session context (Section 8.1 of {{I-D.ietf-dtn-btpu}}).
A trailing Session End therefore neither closes an existing session nor
prevents handoff of the inner bundle from an earlier valid Bundle Message.

For ordinary message and hint errors, apply Sections 7.2 through 7.4 of
{{I-D.ietf-dtn-btpu}}. Continue at the next known message boundary. If
that boundary cannot be determined, stop processing the remainder of the
PDU without discarding valid Bundle Messages already delimited.

For a PDU with valid session framing, Sections 4 and 5 of
{{I-D.ietf-dtn-btpu}} govern reassembly and the transfer window.
If otherwise accepted messages for an incomplete transfer
give conflicting octets for the same segment index or inconsistent final
indices, the egress MUST cancel only that transfer. Other transfers and
independent Bundle Messages remain eligible for processing. The egress
hands each whole or reconstructed inner bundle intact to ordinary BPA
reception processing.

The egress SHOULD persist session and window state, segments, and
completion or cancellation records across restart. It SHOULD retain
incomplete reassembly until completion or the transfer's expiry, subject
to cancellation and local resource limits. If outer bundles yield
different expiry estimates, the reassembly deadline is the earliest
observed effective expiry. Abandoning reassembly cancels the transfer
(Section 4.2 of {{I-D.ietf-dtn-btpu}}).

BTPU suppresses further messages for a canceled or completed transfer
(Sections 4.2 and 6 of {{I-D.ietf-dtn-btpu}}). Even after releasing the
segments, the egress retains enough state to suppress delayed copies
while they can still be admitted for that transfer. A repeated unsegmented
Bundle Message can present the same inner bundle more than once.

The receiving BPA SHOULD account for known tunnel transit time when
processing an inner bundle's Bundle Age block under {{RFC9171}},
particularly when its creation time is zero. Status reports for inner
and outer bundles refer to their respective bundles.

# Operational considerations

Routing policy MUST prevent an outer bundle from being forwarded back
into the same tunnel recursively. Each encapsulation creates a new outer
bundle with its own forwarding history, so an outer Hop Count block alone
cannot bound recursion. A route to the egress that excludes the tunnel
can enforce this rule.

Section 11 of {{I-D.ietf-dtn-btpu}} requires suitable underlying
congestion control where congestion can occur. Provision the offered
load, including repetitions and overhead, against queue storage and
forwarding capacity along the entire outer BP path. The first hop's rate
does not bound the queues beyond it.

No return channel is required, but authenticated confirmation that the
egress holds the object, such as a Completed status from the optional
Bundle Transfer Protocol - Cross-Link (BTPX) {{?I-D.taylor-dtn-btpx}},
can justify advancing the current session's window past that transfer.
It does not establish that all old outer bundles have drained before a
new session starts.

# Security considerations

## Outer bundle admission {#outer-bundle-admission}

The egress MUST apply its configured outer bundle security and peer
authorization policy before processing any BTPU message. An EID alone
does not authenticate a sender. Where authentication is required, the
policy MUST bind the PDU and the outer source and destination EIDs to an
authorized sender. If the security source differs from the outer bundle
source, the egress verifies that it is authorized to act for that source.

BPSec {{!RFC9172}} can protect outer bundles. When using the BPSec COSE
context {{?I-D.ietf-dtn-bpsec-cose}}, check that the selected protection
covers the PDU and the channel-identifying fields. Protection of the
inner bundle remains subject to independent BP policy.

## Changes in security policy

When the policy for protecting or accepting outer bundles changes, the
ingress MUST reassess channel reuse. If the egress cannot authorize joint
reassembly of segments protected under the old and new policies, the
ingress MUST NOT continue a transfer across that change and MUST start a
new session before its next segmented transfer
({{starting-another-session-on-a-used-eid-pair}}). Old transfers can
finish on their old channels only while egress policy permits.

Routine key rollover can preserve a session when overlapping credentials
authorize the same peer, recipient set, and protection. A change of
confidentiality recipients can require a new transfer. On revocation, the
egress MUST apply its revocation policy to both arriving outer bundles and
retained segments of incomplete transfers. It MUST discard the affected
reassembly if the policy no longer permits using those segments.

## Resource use and replay

In addition to BTPU's reassembly limits (Sections 4 and 10 of
{{I-D.ietf-dtn-btpu}}), bound the number of channels and transfers,
segment storage, and retention time for each authorized peer across all
its ephemeral EIDs. A replayed outer bundle can consume resources until
expiry even when duplicate delivery is suppressed.

# IANA considerations

IANA is requested to assign TBD1 from the Specification Required range
(0x8000 through 0xFFFF) of the "'ipn' Scheme URI Well-Known Service
Numbers for BPv7" registry (Section 9.3 of {{RFC9758}}):

* Value: TBD1
* Description: Tunnel Using BTPU Encapsulation (TUBE) destination service
* Reference: This document

Editor's note: TBD2 denotes the proposed TUBE demux for the outer dtn
destination EID. The registration mechanism for well-known dtn demux
strings remains open. No registration under {{RFC8615}} is requested.

This binding requires no new BP block type, administrative record type,
or BTPU message type.

--- back

# Example of reordered outer bundles {#example-of-reordered-outer-bundles}

The ingress has two authorized source EIDs and one egress destination
EID. Channel A uses the first source EID and a window of four. The
ingress reserves the second source EID for a fresh channel B.

1. On A, the ingress sends transfer 100 in three outer bundles carrying
   segments 0, 1, and 2. Only the outer bundle with segment 2 (Transfer
   End) arrives promptly. Its PDU begins with a Session Start, so the
   egress can accept segment 2 first.
2. Transfers 101 through 103 complete on A. The window now spans 100
   through 103; the ingress has no confirmation that 100 is complete.
3. The ingress sends the next inner bundle on B. Assigning transfer number
   104 on A could displace 100. The transfer number on B is independent
   and does not advance A's window.
4. After a restart, the egress recovers its session, window, and stored
   segment 2. Segments 1 and 0 arrive on A before transfer 100 expires.
   The egress hands the reconstructed inner bundle to its BPA. A later
   copy of segment 2 does not cause a second handoff.

If the inner bundle was a BPv7 fragment, the BPA receives that fragment
with its original offset. Reassembly of the original ADU is separate.

# Experiment

Two independent implementations can use the scenario in
{{example-of-reordered-outer-bundles}} to compare reconstructed octets,
BPA handoff counts, and state recovered after restart across contacts
with different delays. Record handoffs and retained state for transfers
packed with a common expiry, compatible key rollover, and revocation
during reassembly. Add the following receiver cases:

* With a transfer incomplete on channel A, deliver an authorized outer
  bundle containing Session Start(A), a complete Bundle Message carrying
  inner bundle X, and an unexpected Session End(A), in that order. Both
  control messages name A's current session. Verify that X reaches the
  BPA and that the PDU changes neither A's session nor its transfer state.
  Deliver the remaining segments in correctly framed PDUs and verify that
  the original transfer completes.
* Set the local receive limit above the provisioned send maximum. Use
  otherwise admissible outer bundles whose PDUs contain complete Bundle
  Messages. For a PDU above the send maximum but within the receive limit,
  verify inner bundle handoff when the egress is configured to accept
  that size. For a PDU above the local receive limit, verify rejection
  before any message processing, with no BTPU state change or inner
  bundle handoff.

Measure peak source EID use, reassembly state retention time, and the
fraction of late outer bundles discarded for each reason.
