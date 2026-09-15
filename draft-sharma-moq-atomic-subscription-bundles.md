---
title: "Atomic Subscription Bundles for Media over QUIC Transport"
abbrev: "moq-atomic-subscription-bundles"
category: std

docname: draft-sharma-moq-atomic-subscription-bundles-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - subscription
 - atomic switch

venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "sharmafb/draft-sharma-moq-atomic-subscription-bundles"
  latest: "https://sharmafb.github.io/draft-sharma-moq-atomic-subscription-bundles/draft-sharma-moq-atomic-subscription-bundles.html"

smart_quotes: no

author:
 -
    ins: A. Sharma
    fullname: Aman Sharma
    organization: Meta
    email: amsharma@meta.com

normative:
  MOQT: I-D.ietf-moq-transport

--- abstract

This document defines a Media over QUIC Transport (MOQT) extension for
atomically changing the Forward State of a set of established subscriptions.
It allows a subscriber to replace one set of Tracks with another without an
intermediate partially switched state at its peer.


--- middle

# Introduction {#introduction}

Applications often consume Tracks as a unit. A sports presentation can contain
video, commentary, and captions, while a monitoring view can contain several
camera Tracks. Switching each subscription independently can briefly activate
only part of the destination set or forward both the old and new sets.

This document defines BUNDLE_SWITCH, a one-shot operation over established
subscriptions on one MOQT Session. The subscriber normally prepares the
destination subscriptions with Forward State 0, then atomically deactivates the
old set and activates the new set. The subscriptions remain independent after
the switch; this extension creates no persistent bundle object.

Atomicity applies to Forward State at the immediate peer. It does not guarantee
simultaneous Object arrival or presentation, and data already in flight can
still arrive after a switch.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms Object, Track, Subscription, Forward State,
Publisher, Subscriber, Relay, and Request ID as defined in {{MOQT}}.

Activation Set:
: Established subscriptions whose Forward State changes from 0 to 1.

Deactivation Set:
: Established subscriptions whose Forward State changes from 1 to 0.

Commit:
: The single logical transition at which every requested Forward State change
  takes effect.


# Negotiation {#negotiation}

Support is negotiated with the ATOMIC_SUBSCRIPTION_BUNDLES Setup Option. Its
even-numbered Option Type is TBD1 and its value is a variable-length integer
capability bit mask. Bit 0x01 enables BUNDLE_SWITCH; unknown bits are ignored.

The capability is negotiated when both endpoints advertise bit 0x01. An
endpoint MUST NOT send BUNDLE_SWITCH otherwise. The 0-RTT requirements of
{{MOQT}} apply; a client MUST NOT use the extension in 0-RTT unless it has
remembered peer support.


# Preparing a Bundle {#preparing}

Every bundle member is an Established subscription on the same Session. The
subscriber SHOULD establish each destination subscription with Forward State
0 and wait for SUBSCRIBE_OK before sending BUNDLE_SWITCH. Filters, priorities,
and other subscription parameters are configured on the subscription's own
request stream and are not changed by BUNDLE_SWITCH.

Preparing subscriptions in Forward State 0 allows failures to be detected
while the current bundle continues forwarding. It can consume state at the
Publisher or its upstream peers, but does not send Object payloads to the
subscriber.

After sending BUNDLE_SWITCH and before receiving its response, the subscriber
MUST NOT update or cancel a referenced subscription.


# BUNDLE_SWITCH {#bundle-switch}

BUNDLE_SWITCH is a request sent as the first message on a new bidirectional
stream. It consumes a Request ID under the rules of {{MOQT}}.

~~~
BUNDLE_SWITCH Message {
  Type (vi64) = TBD2,
  Length (16),
  Request ID (vi64),
  Number of Deactivating Subscriptions (vi64),
  Deactivating Subscription Request ID (vi64) ...,
  Number of Activating Subscriptions (vi64),
  Activating Subscription Request ID (vi64) ...,
}
~~~

The subscription Request IDs identify existing SUBSCRIBE requests, not Track
Aliases. Each ID MUST occur exactly once, the two sets MUST be disjoint, and
the combined number of IDs MUST be non-zero. All referenced subscriptions MUST
have been initiated by the sender on this Session. Each deactivating
subscription MUST be Established with Forward State 1, and each activating
subscription MUST be Established with Forward State 0.

The Publisher validates every member before changing any Forward State. If a
member is invalid, unauthorized, terminated, or cannot be activated, the
Publisher MUST send REQUEST_ERROR and leave all referenced subscriptions
unchanged. INVALID_BUNDLE (TBD3) is used for an invalid member list or state;
other errors such as UNAUTHORIZED, TIMEOUT, and EXCESSIVE_LOAD retain their
MOQT meanings.

Once all members are ready, the Publisher commits the switch. No Object from
an activating subscription can be scheduled as a result of the switch until
the Publisher has stopped scheduling Objects from every deactivating
subscription. The normal MOQT rules for Forward State changes and outstanding
Subgroup streams apply. At Commit, the Publisher MUST save the current Largest
Location of each activating subscription as its Joining Location when Objects
have been published on that Track.

The Publisher then sends REQUEST_OK on the BUNDLE_SWITCH stream; this response
is called BUNDLE_SWITCH_OK. Its Parameters and Track Properties MUST be empty.
The Publisher MUST serialize BUNDLE_SWITCH with any other operation that
changes a referenced Forward State, so validation and Commit use one consistent
state.

The deactivated subscriptions remain Established with Forward State 0. They
can be reactivated by a later BUNDLE_SWITCH or terminated using normal MOQT
procedures. A failure after Commit affects subscriptions independently and
does not roll the switch back.


# Atomicity and Media Alignment {#atomicity}

Commit is an atomic control-plane transition at one Publisher or Relay. QUIC
streams are independent, so Objects sent before Commit can arrive after Objects
from the Activation Set. The extension does not align Groups, timestamps, or
media presentation points across Tracks. Applications that require a seamless
presentation switch need compatible media boundaries or application metadata.

The operation is scoped to one MOQT Session. It cannot atomically switch
subscriptions held on different peers or Sessions.


# Relay Processing {#relay}

Request IDs are Session-local, so a Relay MUST NOT forward BUNDLE_SWITCH
unchanged. It performs the downstream Commit locally and MAY prepare or update
upstream subscriptions as needed. If it cannot make every activating
subscription ready, it fails the downstream operation without changing the
referenced downstream Forward States.

Other downstream subscribers and upstream Forward States are outside the
atomicity guarantee.


# Security Considerations {#security}

The security considerations of {{MOQT}} apply. BUNDLE_SWITCH does not bypass
authorization on any member subscription. A Publisher MUST reject a Request ID
that does not identify an eligible subscription initiated by the sender.

Large bundles and prepared subscriptions can consume state. Implementations
SHOULD limit bundle size and pending duration and MAY reject an operation with
EXCESSIVE_LOAD. A malicious peer cannot use BUNDLE_SWITCH to alter another
Session's subscriptions because Request IDs are Session-local.


# IANA Considerations {#iana}

This document requests the following provisional registrations:

| Registry | Value | Name | Specification |
|:---------|------:|:-----|:--------------|
| MOQT Setup Options | TBD1 (even) | ATOMIC_SUBSCRIPTION_BUNDLES | {{negotiation}} |
| MOQT Message Types | TBD2 | BUNDLE_SWITCH | {{bundle-switch}} |
| MOQT REQUEST_ERROR Codes | TBD3 | INVALID_BUNDLE | {{bundle-switch}} |


--- back

# Acknowledgments
{:numbered="false"}

This work was motivated by discussion of atomic multi-Track switching in the
MOQT working group.

# Change Log
{:numbered="false"}

## draft-sharma-moq-atomic-subscription-bundles-00
{:numbered="false"}

* Initial version.

# Use of Generative AI
{:numbered="false"}

OpenAI Codex was used to assist with drafting and editing this document. All
generated text was reviewed and approved by the author.
