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
Using BTPU Encapsulation (TUBE), each outer bundle carries one BTPU
link-layer protocol data unit in its application data unit. BTPU can carry several whole bundles in one outer bundle or
divide a serialized bundle among several outer bundles. The tunnel can carry an existing BPv7 fragment without waiting for the other fragments.

Outer bundles can take different routes and arrive in any order during
their lifetimes. This document defines how tunnel endpoints identify BTPU
channels, bound the size of each protocol data unit, and reuse channels
while preserving transfers that can still complete. It also specifies when
a change in outer-bundle security policy requires a new session.

--- middle

# Introduction

Consider a spacecraft that uses a Tunnel Using BTPU Encapsulation (TUBE)
to forward bundles through either of two ground
stations. A large bundle takes three downlink transmissions. All three
pieces are addressed to the same tunnel egress at the mission operations
center. The last piece travels through a station with immediate onward
connectivity. The first piece waits at the other station for several days.
Meanwhile, later transfers reach the operations center in their entirety.
Those transfers should not prevent completion of the earlier one.

BPv7 {{!RFC9171}} provides the outer delivery service. BTPU
{{!I-D.ietf-dtn-btpu}} provides message framing, packing, segmentation,
and reassembly. Together they present one convergence-layer hop between
the tunnel ingress and egress, even when the outer bundles traverse several
BP nodes. Each direction operates independently. A deployment can use a
single direction throughout its lifetime.

Bundle-in-Bundle Encapsulation (BIBE) {{?I-D.ietf-dtn-bibe}} describes
the same forwarding model and defines its own payload format for whole
bundles and segments. This specification uses BTPU on the wire. An
operator can reuse BTPU implementations across this tunnel and other
bindings, such as BTPU over Ethernet {{?I-D.ek-dtn-ethernet}}. A node
that provides both BIBE and this service configures distinct receiving
endpoints for the two payload formats.

## Conventions and terminology

{::boilerplate bcp14-tagged}

A *carrier* is an outer BPv7 bundle. An *inner bundle* is the BPv7
bundle being forwarded through the tunnel. A *PDU* is a BTPU link-layer
protocol data unit. *Session*, *Transfer*, *Transfer Window*, and the
message names have the meanings defined by BTPU. In particular, a BTPU
Transfer carries a segmented object. A Bundle Message carries a whole
object without using a Transfer Number.

# Endpoints and channel identification

For each sending direction, configure the ingress with a singleton
destination endpoint identifier (EID) at the egress and one or more
source EIDs that the ingress is entitled to use as its BP source node
ID. Configure the egress to accept the corresponding sources, apply its
outer-bundle security policy, and dispatch the delivered application
data unit (ADU) to this service. Anonymous carriers MUST NOT be used.
The singleton destination ensures that carriers taking different paths
reach the same reassembly function. A receiver implemented by several
processes MUST provide consistent channel state across those processes.

Under the ipn scheme, the proposed well-known TUBE destination service
number is TBD1.

An example of a configured TUBE egress EID is
`dtn://egress.example/.well-known/tube`. The demux string
".well-known/tube" is illustrative and follows the spelling used
by {{?RFC8615}}. The proposed well-known dtn demux is TBD2.

Both identifiers apply only to the outer destination EID. A deployment
can configure a different ipn service number or dtn demux for that
destination. Neither identifier constrains the choice of source EID.

The ordered pair (outer source EID, outer destination EID) identifies one
unidirectional BTPU logical channel. Compare the complete EIDs according
to their scheme's equality rules. Each pair has its own BTPU instance,
current session, and transfer window. A Session ID has meaning within this
channel. The egress MUST select the channel before applying any BTPU
session or transfer state transition.

An ingress MAY use an ephemeral source EID for one segmented transfer.
All carriers of that transfer MUST use the same source and destination
EIDs. For the ipn scheme, an ingress can select a service number from
a pool assigned to this purpose on its own fully qualified node number
({{!RFC9758}}, Sections 5.3 and 9.3). The egress can authorize the peer
by node number and service-number range. It MUST retain the service number
when identifying the channel. The service number in the destination EID
can remain fixed. The availability and selection of source EIDs under
other schemes depend on the scheme and the deployment.

# Encapsulation

## Payload mapping and forwarding

The ingress takes the serialized inner bundle after its BPA has performed
normal forwarding-time processing under Section 5.4 of {{RFC9171}}. It
MUST use one serialization for all segments and repetitions of a given
transfer. The tunnel treats the serialized inner bundle as opaque, even
when it is protected. The ingress can encapsulate a BPv7 fragment without
waiting for any other fragment of the same bundle.

The ADU of each carrier MUST consist of exactly one nonempty BTPU
Link-layer PDU, as defined in {{I-D.ietf-dtn-btpu}}. The outer ADU boundary
delimits the PDU. There is no additional envelope, length prefix, or
administrative record header. The ingress MUST clear the administrative
record flag and set the "Bundle must not be fragmented" flag on each
carrier. Each carrier, including a newly created carrier that repeats a
BTPU message, receives its own BP identity. Normal BP forwarding of a
carrier preserves that carrier's identity.

A sender using this binding MUST encode a whole inner bundle in a BTPU
Bundle Message. When the whole bundle exceeds the available space or the
message length limit, use Transfer Segment and Transfer End messages as
specified by BTPU. Receivers MUST support both forms. A PDU may pack
messages for several objects when the channel, expiry, and security
rules in this document permit it. A Bundle Message consumes no window
slot and can be processed without a current session. Whole bundles alone
can therefore use a stable EID pair without session signaling in the
base binding.

Submitting all necessary carriers to the outer BPA completes the local
handoff. It gives the ingress no assurance that the egress reconstructed
the inner bundle. If the ingress cannot construct or submit all required
carriers, it reports a local forwarding failure to its BPA. A retry that
continues an existing transfer preserves the bytes and indices already
assigned. If that is impossible, the ingress uses a new transfer and
applies the channel reuse rules below.

## PDU and reassembly limits

Each direction MUST have a provisioned maximum PDU length, expressed in
octets of the outer ADU before outer-bundle security is applied. It
includes all BTPU headers, hints, content, and padding. The ingress MUST
use a maximum supported by the egress and MAY emit smaller PDUs. The
egress MUST discard an oversized PDU before processing any of its
messages. This binding defines no in-band discovery procedure or universal
default. The maximum serialized inner bundle size that the egress will
accept for reassembly is a separate deployment parameter.

Where a route or contact limits the serialized carrier size, the ingress
MUST account for the outer primary block, canonical blocks and their
encoding, CRCs where present, and the overhead of the selected security
operations. It MUST use a PDU small enough for the resulting carrier to
fit the applicable limit. A change in outer-bundle overhead can reduce
the usable PDU size. If the available space cannot accommodate the
required messages and data, the ingress defers transmission or selects
another forwarding opportunity.

A BTPU message's 20-bit Length field covers the hints and content after
its four-octet header. Thus a Bundle Message with no hints can carry up
to 1,048,575 inner-bundle octets, even if the configured PDU limit is
larger. Several messages can occupy one larger PDU. As a sizing example,
a 65,536-octet PDU containing a 14-octet Session Start and one Transfer
Segment can carry 65,510 segment-data octets. This assumes no hints or
padding and allows four octets for the second message header and eight
for its Transfer Number and Segment Index. The outer BP overhead is
additional.

## Expiry and packing

A sender MUST select one expiry for a segmented transfer, no later than
the expiry of its inner bundle. It MUST set the lifetime of each carrier
from the interval remaining until that expiry when the carrier is
created. A repetition sent later has correspondingly less lifetime.
Where the ingress lacks an accurate clock, it uses the remaining inner
bundle lifetime and locally elapsed time, and supplies the outer Bundle
Age block as required by BPv7. If it cannot establish a remaining
lifetime, it cannot use this tunnel for the bundle. This follows the
expiry model in {{I-D.ietf-dtn-bibe}}.

All segmented transfers represented in one carrier MUST have the same
chosen expiry. The carrier MUST also expire no later than any whole
inner bundle it contains. The sender can select a common expiry when it
starts several transfers or pack separate carriers for transfers that
already have different expiries. It MUST also select outer protection
acceptable for every object represented in a carrier.

# Sessions over a store-and-forward path

## Starting a session

BTPU maintains one current session per logical channel. A Session Start
for a new session cancels transfers in the preceding session. Reordering
can cause a delayed Session Start to restore an earlier session and
cancel work in the later one. A Transfer message itself carries no
Session ID. It would be interpreted under whichever session is then
current (Section 5.1 of {{I-D.ietf-dtn-btpu}}).

For this binding, every PDU containing a Transfer message MUST begin with
a valid Session Start for the session to which all Transfer messages in
that PDU belong. The sender sets Next Transfer Number as specified by
BTPU at the time of emission. A repeated Session Start for the current
session does not reset the receiver's window. A PDU MUST carry at most
one Session Start and MUST NOT carry a Session End. The binding retires
channel state by expiry. These rules let the carrier with the final
segment initialize the session when that carrier arrives first.

A simple sending policy assigns a fresh source EID, one session, and one
segmented transfer to a channel. Other channels can then carry any number
of later transfers without advancing this channel's window. The sender
still announces a valid BTPU window size.

## Reusing a channel within a session

A sender MAY start another transfer on the channel when it has a
reasonable basis to expect that advancing the window will preserve every
earlier transfer on the channel that can still complete. In making this
determination it MUST account for any admissible arrival order, including
a receiver that learns the session from a later carrier. A sender lacking
such a basis SHOULD defer the new transfer or use a different source EID.

For a window of size W, if the greatest Transfer Number the receiver
might observe is G, the window extends back through G - W + 1 inclusive,
using BTPU serial number arithmetic. With W = 4, transfers 100 through
103 can coexist even when 103 arrives first. Starting 104 can evict 100.
Finishing 101, 102, and 103 does not change that fact. Evidence that
100 has expired or that the egress already holds its complete object
can justify advancing the window. Elapsed time since submission or
transmission over one outer hop is insufficient by itself.

The sender MUST also prevent delayed carriers from being mistaken for
new transfers because of serial number wrap or an ambiguous serial
number distance. One simple policy is to stop assigning numbers on the
channel before the range of numbers in still-deliverable carriers can
exceed BTPU's unambiguous comparison range, and to allocate another
channel. A count of locally queued transfers alone cannot bound that range.

## Starting another session on a used EID pair

A sender starting a new session, including after sender restart, SHOULD
use a fresh EID pair unless it can establish both that outstanding work
from the old session no longer needs the channel and that a carrier from
that session cannot disrupt the new one. Waiting for all carriers of the
old session, including repetitions and control messages, to expire
satisfies the latter condition when their latest possible delivery time
can be bounded. An operator can also establish that all relevant queues
and paths have drained. Completion of the old transfers alone leaves open
the possibility of a delayed duplicate Session Start.

An ingress relying on expiry for reuse MUST retain enough allocation
state across restart to enforce this rule, or select source EIDs it knows
are available. Where age estimates do not provide a safe reuse time, it
SHOULD use a fresh source EID. Pool exhaustion is a reason to defer a
transfer or select another route.

# Reception and delivery

The egress applies normal outer BP reception, expiry, and configured
security checks. Before allowing a PDU to alter channel state, it MUST
verify that the carrier is admissible for the complete EID pair and
within the receive PDU limit. It MUST discard a PDU that violates the
Session Start placement or Session End rules above. For admitted PDUs,
BTPU governs message parsing, unknown types and hints, invalid messages,
and window processing.

The egress accepts segments in any order within an admitted transfer.
A Transfer End with final index N supplies the last segment and tells
the receiver which indices to await. It hands the reconstructed octets
to ordinary BPA reception processing only after it has every segment
from zero through N. Identical repetitions contribute no new octets.
If two admissible messages give conflicting data for an index or
inconsistent final indices, the egress MUST cancel that transfer.
A valid Transfer Cancel has its usual BTPU effect.

For an admitted transfer the egress SHOULD persist the session, window,
segments, and completion or cancellation state across restart. It SHOULD
retain incomplete state until completion or the transfer's expiry,
subject to BTPU's resource limits and explicit cancellation. Where
carriers indicate different local estimates of the same expiry, use the
earliest observed effective expiry as the incomplete reassembly deadline.
Silence and progress on another channel are insufficient grounds to
declare a transfer obsolete.

The egress MUST suppress repeated delivery of a completed segmented
transfer while additional copies can still be admitted for it. An
implementation can commit the completed-transfer record together with
the handoff to a persistent BPA receive queue, and retain the record
until the last admissible copy expires. A receiver that has discarded
partial state cannot reconstruct the missing octets without another
copy. A whole Bundle Message has no Transfer Number. Duplicate carriers
or repeated Bundle Messages can therefore result in repeated presentation
of the inner bundle to normal BPA reception processing.

The tunnel hands the inner serialization to the BPA intact. Inner
extension blocks are opaque during transit. The receiving BPA SHOULD
account for known tunnel transit time when processing a Bundle Age
block under {{RFC9171}}, particularly for an inner bundle whose creation
time is zero. Transit through outer forwarding nodes counts as one
inner convergence-layer hop. Inner status reports and outer status
reports refer to their respective bundles.

# Operational considerations

Routing policy MUST prevent a carrier from being forwarded into the
same tunnel recursively. Each new carrier starts its own BP forwarding
history. An outer Hop Count block alone cannot bound repeated
re-encapsulation. A separate outer forwarding table or a route to the
egress service that excludes this tunnel can provide the restriction.

The congestion-control applicability limits of {{I-D.ietf-dtn-btpu}}
apply. Provision the offered load, including repetitions and overhead,
against available queue storage and forwarding capacity across the
outer path. An uncontested, capacity-managed path is one deployment
model. Where congestion can occur, suitable congestion control is
required. A known downlink rate alone says little about the queues
beyond the first receiving station.

Operation requires no return channel. A deployment that provides one
can use authenticated feedback to inform the window decision. For
example, the Bundle Transfer Protocol - Cross-Link (BTPX) proposal {{?I-D.taylor-dtn-btpx}} defines a
Completed status indicating that the reporting peer holds the object.
Such a report can justify advancing the window past that transfer. It
does not establish that old carriers have drained. A deployment using
BTPX also observes its requirement for distinct Session IDs across a
sender's concurrent sessions on associated channels.

# Security considerations

## Carrier admission

The egress MUST apply its configured outer-bundle security and peer
authorization policy before processing any BTPU message, including
Session Start and Transfer Cancel. An unauthorized control message
could otherwise cancel a transfer before the carrier is rejected.
An EID is a routing and dispatch identifier. Its presence alone does
not authenticate the peer authorized to use it.

Where authenticated carriage is required, the policy MUST bind the PDU
and the outer source and destination EIDs to an authorized sender. The
security source may differ from the bundle source. The egress verifies
that the security source is authorized to act for that carrier source.
BPSec {{!RFC9172}} can supply carrier protection. When using the BPSec
COSE context {{?I-D.ietf-dtn-bpsec-cose}}, the egress checks the actual
protection and scope, including coverage of the channel-identifying
fields. Inner-bundle protection remains subject to independent BP policy.

## Changes in security policy

When the effective policy for protecting or accepting carriers changes,
the sender MUST reassess channel reuse. It MUST start a new session
for subsequent segmented transfers when the receiver cannot authorize
joint reassembly of carriers protected under the old and new policies.
The normal cross-session EID reuse rules apply. The sender MUST NOT
continue an inner transfer across such a boundary. If the inner bundle
still requires forwarding, it may start a new transfer under the new
policy. Previously authorized transfers can finish on their old
channels for as long as receiver policy permits.

For example, routine key rotation can preserve a session when overlapping
credentials authorize the same peer, recipient set, and protection.
A change of confidentiality recipients can require a fresh channel and
a new transfer. On revocation, the egress MUST apply the revocation
policy to both arriving carriers and retained segments of incomplete
transfers. If policy no longer permits using those segments, it MUST
discard the affected reassembly. An authenticated change of Session ID
does not establish authorization for data already received.

## Resource use and replay

Bound the PDU length, accepted inner size, number of channels and
transfers, segment storage, and retention time. Apply aggregate limits
to the authorized peer across its ephemeral EIDs. Check lengths and
indices before allocating storage. A declared final index or Bundle
Length hint alone gives no reason to reserve the advertised capacity.
A replayed valid carrier can consume resources until expiry. Completion
and cancellation records, bounded lifetimes, and channel reuse rules
limit the effects of such replays.

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

# Example of reordered carriers

The ingress has a pool of authorized source EIDs. The egress has one
configured destination EID. Channel A uses one source EID and a window
of four. Channel B uses a second source EID and the same destination.
Both channels carry Session Start at the beginning of each PDU
containing Transfer messages.

1. On A, the ingress sends transfer 100 in three carriers, with segment
   indices 0, 1, and 2. Only the carrier with Transfer End and index 2
   reaches the egress promptly. The egress stores it.
2. Transfers 101 through 103 complete on A. Transfer 100 is still within
   the window. The ingress has no confirmation of its completion.
3. The ingress sends the next object on B. Assigning transfer number
   104 on A could displace 100. The Transfer Number on B is chosen
   independently. Receipt on B leaves A's window untouched.
4. The egress restarts and recovers its receive state. Before transfer
   100 expires, segments 1 and 0 arrive on A. The egress concatenates
   segments 0, 1, and 2 and hands the inner bundle to its BPA. A later
   duplicate of segment 2 produces no second handoff.

If transfer 100 carries a BPv7 fragment, the egress reconstructs that
fragment from BTPU segments 0 through 2. It passes the fragment to its
BPA with the fragment offset unchanged. Reassembly of the original ADU
is a separate BPv7 operation.

# Experiment

This experiment tests whether a BPv7 network can serve as a BTPU link
across contacts with substantially different delays. Two independent
implementations can use the preceding example to check the reconstructed
octets, number of BPA handoffs, and retained state after restart.
Additional cases include two transfers sharing a carrier with a common
expiry, an oversized PDU containing Session Start, a routine key rollover
during reassembly, and rejection of a carrier after key revocation.
Useful observations include the peak number of source EIDs in use,
the duration of reassembly state, and the fraction of late carriers
discarded by reason.
