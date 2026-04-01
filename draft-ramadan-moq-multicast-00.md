# Multicast Delivery and Endpoint Discovery for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: October 2026                                      April 2026

     Multicast Delivery and Endpoint Discovery for Media over QUIC
                    draft-ramadan-moq-multicast-00

Abstract

   This document specifies multicast delivery mechanisms and catalog-
   based endpoint discovery for Media over QUIC (MoQ).  It defines how
   MoQ sessions integrate with IP multicast (SSM, ASM), Automatic
   Multicast Tunneling (AMT), and TreeDN for scalable live streaming.
   The specification includes a multicast catalog extension for endpoint
   discovery, a condensed multicast packet format for LOC and CMAF
   media over UDP, content authentication for multicast transport,
   and delivery path selection across TV, mobile, and browser
   platforms.  This document is container-format agnostic at the
   catalog and discovery layer, and defines the condensed wire format
   used by non-MMTP packaging types on multicast.

Status of This Memo

   This Internet-Draft is submitted in full conformance with the
   provisions of BCP 78 and BCP 79.

   Internet-Drafts are working documents of the Internet Engineering
   Task Force (IETF).  Note that other groups may also distribute
   working documents as Internet-Drafts.  The list of current
   Internet-Drafts is at https://datatracker.ietf.org/drafts/current/.

   Internet-Drafts are draft documents valid for a maximum of six
   months and may be updated, replaced, or obsoleted by other documents
   at any time.  It is inappropriate to use Internet-Drafts as
   reference material or to cite them other than as "work in progress."

Copyright Notice

   Copyright (c) 2026 IETF Trust and the persons identified as the
   document authors.  All rights reserved.

Table of Contents

   1.  Introduction
       1.1.  Multicast as Supplement to MoQ Unicast
   2.  Terminology
   3.  Delivery Paths
   4.  Multicast Catalog Extension
       4.1.  Multicast Endpoint Format
       4.2.  Network Source Types
             4.2.1.  AMT (Automatic Multicast Tunneling)
             4.2.2.  ATSC 3.0 (Broadcast Television)
             4.2.3.  Multiple Network Sources
   5.  Multicast Packet Formats
       5.1.  MMTP Multicast
       5.2.  Condensed Multicast Packet Format
       5.3.  Source Symbol Delivery
       5.4.  Block Manifest
       5.5.  Streaming Mode vs Recovery Mode
   6.  Multi-Path Delivery
   7.  Security Considerations
       7.1.  Multicast Security
       7.2.  Content Authentication
   8.  IANA Considerations
   9.  References
       9.1.  Normative References
       9.2.  Informative References
   Authors' Addresses
```

## 1. Introduction

Live streaming audiences routinely reach tens of millions of
concurrent viewers.  Traditional HTTP-based CDN architectures
replicate streams at x86 server farms close to viewers, but this
approach does not scale cost-effectively for 4K/8K/360-degree
bitrates.

IP multicast provides an alternative where a single stream is
replicated at the network layer, but lacks the economic incentives
for widespread ISP deployment.  Media over QUIC (MoQ)
[I-D.ietf-moq-transport] provides a modern transport for real-time
media, but its unicast model faces the same scaling limitations as
HTTP CDNs for mass-audience events.

This document specifies how MoQ integrates with IP multicast to
combine the scalability of multicast with the reliability and
signaling of QUIC:

1. **Delivery paths**: Platform-specific multicast reception for
   TV, mobile, and browser clients
2. **Catalog extension**: A container-agnostic multicast endpoint
   discovery mechanism for MoQ catalogs [I-D.ietf-moq-catalogformat]
3. **Condensed wire format**: A compact multicast packet format for
   LOC and CMAF media over UDP, with FEC metadata, object boundary
   signaling, and optional authentication
4. **Multi-path delivery**: Combining MoQ unicast and multicast for
   seamless failover and FEC symbol deduplication

MoQ relays MAY operate as TreeDN [RFC9706] nodes for hierarchical
distribution.

This specification works with any MoQ packaging format.  MMTP
[I-D.ramadan-moq-mmt] uses its native self-describing framing on
multicast.  LOC [I-D.ietf-moq-loc] and CMAF [I-D.ietf-moq-cmsf]
use the condensed multicast packet format defined in Section 5.2.

### 1.1. Multicast as Supplement to MoQ Unicast

Multicast delivery as defined in this document is a supplement to
MoQ/QUIC unicast.  For LOC and CMAF packaging, subscribers MUST
establish a MoQ session first to receive the catalog, negotiate
parameters, and receive initial media.  Multicast MAY then be used
as an optimized delivery path for ongoing media data.  This ensures
that codec configuration (LOC Video Config property, CMAF init
segment) is available before multicast reception begins.

Exception: MMTP-packaged streams [I-D.ramadan-moq-mmt] delivered
via ATSC 3.0 broadcast or native SSM are self-describing and MAY
operate as unidirectional data streams without a MoQ session.  MMTP
carries per-packet routing (packet_id), timing (timestamp),
sequencing (Packet Sequence Number), FEC metadata (FEC Type, OTI),
and signaling (PA, MPI messages) natively.  ATSC 3.0 and ARIB
STD-B60 receivers consume MMTP over SSM as their native delivery
path.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY",
and "OPTIONAL" in this document are to be interpreted as described
in BCP 14 [RFC2119] [RFC8174].

**SSM**: Source-Specific Multicast [RFC4607]

**AMT**: Automatic Multicast Tunneling [RFC7450]

**DRIAD**: DNS Reverse IP AMT Discovery [RFC8777]

**TreeDN**: Tree-based Content Delivery Network [RFC9706]

**IWA**: Isolated Web App -- a Chrome packaging format that grants
DirectSocket API access

**Block Manifest**: A header at the start of each source block's
FEC-protected data containing object count and sizes, enabling
receivers to split recovered byte streams into individual media
objects after FEC recovery.

## 3. Delivery Paths

MoQ content can reach receivers via multiple delivery paths depending
on platform capabilities:

| Client Type | Multicast Path |
|-------------|----------------|
| Native (TV, mobile) | SSM direct via OS multicast API |
| Native (no multicast) | AMT tunneling over UDP [RFC7450] |
| Browser (IWA) | SSM/AMT via DirectSocket [WICG-DirectSockets] |
| Browser (standard) | MoQ/WebTransport (unicast) |

Both native SSM and AMT tunneling require UDP socket access.
MoQ/WebTransport is the universal fallback and requires no
special platform capabilities.

Broadcast receivers (ATSC 3.0, ARIB STD-B60) consume MMTP streams
natively over RF tuner hardware.  MoQ integration occurs at the
gateway/head-end level via TreeDN [RFC9706] and AMT [RFC7450] /
DRIAD [RFC8777].

Multiple transports MAY be available for a given stream.  Applications
select transports based on availability and local policy.

## 4. Multicast Catalog Extension

For tracks available via multicast, the MoQ catalog includes a
top-level `multicast` field containing an `endpoints` array for
endpoint discovery.

### 4.1. Multicast Endpoint Format

The `multicast` field is a catalog extension per
[I-D.ietf-moq-catalogformat] Section 3.1.  Parsers that do not
support multicast MUST ignore it.

The `multicast` field contains an `endpoints` array listing one or
more multicast groups.  A single-endpoint deployment uses a
one-element array:

```json
{
  "multicast": {
    "endpoints": [{
      "sourceAddress": "69.25.95.10",
      "groupAddress": "232.0.10.1",
      "port": 8000,
      "tracks": [
        { "name": "video",        "id": 0 },
        { "name": "video/repair", "id": 1 },
        { "name": "audio",        "id": 2 },
        { "name": "audio/repair", "id": 3 }
      ],
      "bandwidth": 6500000
    }]
  }
}
```

Multi-group deployments (e.g., per-quality ABR tiers or separate
audio/video groups) use multiple elements:

```json
{
  "multicast": {
    "endpoints": [
      {
        "sourceAddress": "10.0.0.1",
        "groupAddress": "232.1.1.1",
        "port": 5004,
        "tracks": [
          { "name": "video", "id": 0 },
          { "name": "video/repair", "id": 1 }
        ]
      },
      {
        "sourceAddress": "10.0.0.1",
        "groupAddress": "232.1.1.2",
        "port": 5004,
        "tracks": [
          { "name": "audio", "id": 2 }
        ]
      }
    ],
    "networkSource": {
      "type": "amt",
      "discovery": "driad",
      "relay": "amt.example.com"
    }
  }
}
```

Endpoint field definitions:

**protocol** (string, OPTIONAL): Transport protocol.
  - "ssm": Source-Specific Multicast (RFC 4607).  This is the default
    when `sourceAddress` is present.
  - "asm": Any-Source Multicast.

**sourceAddress** (string, OPTIONAL): SSM source IP address per
  [RFC4607].  Required for Source-Specific Multicast.  If omitted,
  implies ASM.

**groupAddress** (string, REQUIRED): Multicast group address.
  - SSM range: 232.0.0.0/8 (IPv4), ff3x::/32 (IPv6)
  - ASM range: 224.0.0.0/4 (IPv4), ff0x::/16 (IPv6)

**port** (integer, REQUIRED): UDP port number.

**tracks** (array, REQUIRED): Tracks available on this endpoint.
  Each element is an object with the following fields:

  - **name** (string, REQUIRED): Track name corresponding to the
    track identifier used in the MoQ catalog.
  - **id** (integer, REQUIRED): Numeric Track ID used in the
    condensed multicast packet header (Section 5.2) for packet-level
    track routing.  Values MUST be unique within an (sourceAddress,
    groupAddress, port) tuple.  For MMTP-packaged tracks, this field
    is informational -- MMTP uses its native packet_id for routing.

**bandwidth** (integer, OPTIONAL): Aggregate bandwidth of this
  endpoint in bits per second.  Subscribers SHOULD check available
  network capacity before joining high-bandwidth multicast groups.

**networkSource** (object or array, OPTIONAL): Network delivery
  configuration describing how subscribers can reach the multicast
  stream when native IP multicast routing is not available.  May
  appear on individual endpoints or at the top-level `multicast`
  object to apply to all endpoints.  See Section 4.2.

Each (sourceAddress, groupAddress, port) tuple MUST be associated
with at most one MoQ namespace.  Publishers requiring multiple
independent streams MUST use distinct multicast groups or ports.
This constraint ensures that Track ID values are unambiguous
within a multicast group.

Subscribers receiving a catalog with multicast endpoints SHOULD
auto-connect to the multicast group when multicast APIs are
available (IWA DirectSocket, native UDP sockets, AMT tunneling).
This enables a seamless upgrade path: subscribers first connect via
MoQ/QUIC to receive the catalog and media, then optionally switch
to multicast for lower-latency, FEC-protected delivery.

During multicast join (which may take 1-3 seconds for IGMP/MLD),
subscribers SHOULD continue receiving via MoQ/QUIC.  Once multicast
data arrives, subscribers switch to the multicast path.  This
dual-path startup avoids join latency gaps.

If multicast reception fails or degrades, subscribers fall back
to MoQ/QUIC unicast by subscribing with filter LatestGroup or
NextGroup.  No multicast-to-MoQ coordinate mapping is needed --
the relay provides the current group position.

### 4.2. Network Source Types

The `networkSource` field describes how subscribers can reach the
multicast stream when native IP multicast routing is not available.
It appears at the `multicast` level (applying to all endpoints) or
on individual endpoints.

**type** (string, REQUIRED): Delivery technology identifier.

Defined types:

#### 4.2.1. AMT (Automatic Multicast Tunneling)

```json
{
  "networkSource": {
    "type": "amt",
    "relay": "69.25.95.1",
    "discovery": "driad"
  }
}
```

**relay** (string, OPTIONAL): AMT relay address (IP or hostname).
  When present, subscribers SHOULD connect directly to this relay.

**discovery** (string, OPTIONAL): AMT relay discovery method.
  - "driad": DNS Reverse IP AMT Discovery per [RFC8777].
    Subscribers query the source IP's reverse DNS for AMT relay
    records.  This is the RECOMMENDED discovery method.
  - "manual": Relay address is provided in the `relay` field.

Subscribers SHOULD attempt relay discovery in this order:
1. Use `relay` directly if provided
2. Use DRIAD discovery on the SSM source IP if `discovery` is
   "driad" or omitted
3. Fall back to MoQ/QUIC unicast

Note: DRIAD requires TYPE260 reverse DNS records on the source IP.
Publishers without reverse DNS control SHOULD provide the `relay`
field directly.

#### 4.2.2. ATSC 3.0 (Broadcast Television)

```json
{
  "networkSource": {
    "type": "atsc3",
    "frequency": 533000,
    "plpId": 0,
    "serviceId": 1
  }
}
```

**frequency** (integer, REQUIRED): RF center frequency in kHz.

**plpId** (integer, OPTIONAL): Physical Layer Pipe ID.
  Defaults to 0 (base PLP).

**serviceId** (integer, OPTIONAL): Service ID within the
  broadcast multiplex.

Receivers with ATSC 3.0 tuner hardware can receive the stream
directly over RF.  MoQ subscribers without tuner hardware ignore
this networkSource type.

#### 4.2.3. Multiple Network Sources

When a stream is available via multiple delivery technologies,
`networkSource` MAY be an array:

```json
{
  "networkSource": [
    { "type": "amt", "relay": "69.25.95.1", "discovery": "driad" },
    { "type": "atsc3", "frequency": 533000, "plpId": 0 }
  ]
}
```

Subscribers select the highest-priority available source per the
transport hierarchy defined in Section 3.

## 5. Multicast Packet Formats

Multicast UDP wire formats are determined by the source track's
packaging type in the catalog.

### 5.1. MMTP Multicast

For MMTP-packaged tracks, MMTP packets are transmitted as UDP
datagrams with the standard MMTP header intact per
[I-D.ramadan-moq-mmt].  MMTP is self-describing -- each packet
carries track routing (packet_id), timestamps, sequence numbers,
and FEC metadata natively.  MMTP multicast operates as a
unidirectional data stream (Section 1.1) and does not use the
condensed format.

### 5.2. Condensed Multicast Packet Format

For LOC and CMAF packaged tracks, source and repair data are
delivered over IP multicast using the condensed multicast packet
format defined in this section.  Each UDP datagram carries one
condensed multicast packet containing either one source symbol
(Flags bit 0 = 0) or one repair symbol (Flags bit 0 = 1).

The condensed format provides track demuxing (Track ID), FEC
metadata (SBN/ESI), random access signaling (RAP flag), object
boundary signaling (Close flag), optional timestamps, optional
codec configuration, and optional authentication -- all
catalog-driven.

```
Condensed Multicast Packet {
  Track ID (16),              // 2 bytes -- catalog-assigned per-track ID
  Flags (8),                  // 1 byte -- see below
  Source Block Number (16),   // 2 bytes -- SBN (wraps, modular arithmetic)
  Encoding Symbol ID (16),    // 2 bytes -- ESI (up to 65535 symbols/block)
  Auth Length (8),            // 1 byte -- 0=no auth, else tag size
  [Timestamp (32)],           // 4 bytes -- NTP-short, if Flags bit 2 = 1
  [Config Data (..)],         // Variable -- codec config, if Flags bit 3 = 1
  Payload (..),               // Source symbol or repair symbol
  [Auth Tag (Auth Length)],   // Variable-length (HMAC, ALTA, etc.)
}
```

**Track ID** (16 bits): Identifies the media track for packet
  routing (analogous to MMTP packet_id).  The value corresponds
  to the `id` field assigned to each track in the multicast
  catalog endpoint (Section 4.1).  Publishers MUST use consistent
  Track ID values across all packets for a given track.

**Flags** (8 bits): Bit field:
  - Bit 0: Repair (0=source, 1=repair)
  - Bit 1: RAP (0=delta frame, 1=keyframe/random access point)
  - Bit 2: Timestamp (0=absent, 1=32-bit NTP-short follows header)
  - Bit 3: Config (0=absent, 1=codec configuration data follows
    Timestamp if present, else follows Auth Length).  SHOULD only
    be set on RAP=1 source packets.  Config data is prefixed with
    a 16-bit big-endian length.
  - Bit 4: Close (0=not last, 1=last source symbol of current
    object).  Enables streaming-mode receivers to process objects
    in real-time without waiting for FEC block completion, similar
    to the LCT B flag in ROUTE/ALC [ATSC-A331].  This flag is in
    the condensed header (not FEC-protected); receivers requiring
    reliable object boundaries after FEC recovery MUST use the
    Block Manifest (Section 5.4) instead.
  - Bits 5-7: Reserved (MUST be 0, receivers MUST ignore)

**Timestamp** (32 bits, OPTIONAL): Present when Flags bit 2 is set.
  NTP short format (16-bit seconds + 16-bit fractional seconds).
  RECOMMENDED for LOC source packets where in-band timestamps are
  not available.  Not needed for CMAF (timestamps in moof box) or
  repair symbols.

**Config Data** (variable, OPTIONAL): Present when Flags bit 3 is
  set.  Prefixed by a 16-bit big-endian length in bytes, followed
  by codec configuration data (e.g., SPS/PPS for H.264, VPS/SPS/PPS
  for HEVC, or other codec-specific extradata as declared in the
  catalog's `selectionParams.codec`).  Publishers SHOULD include
  Config on every RAP=1 source packet to enable mid-stream join on
  multicast without requiring MoQ unicast bootstrap for codec
  configuration.

**SBN** (16 bits): Source Block Number.  When FEC is in use (per
  [I-D.ramadan-moq-fec]), identifies the FEC source block.  Wraps
  using modular arithmetic.  Receivers distinguish wrap-around from
  late/duplicate packets using the half-space rule: if
  `new_SBN - current_SBN > 32768` (unsigned), treat as wrap.
  When FEC is not in use, SBN serves as a sequence counter for
  packet ordering and loss detection.  Publishers SHOULD reset SBN
  on stream restart.

**ESI** (16 bits): Encoding Symbol ID.  When FEC is in use, source
  symbols have ESI < K and repair symbols have ESI >= K.  Supports
  up to 65535 symbols per block.  When FEC is not in use, ESI
  serves as a packet sequence number within the current SBN.

**Auth Length** (8 bits): Auth tag length in bytes.  0 = no auth.
  Max 255, sufficient for HMAC-SHA256 (32), Ed25519 (64), and
  ALTA variable-length tags.  Auth scheme is declared in the
  catalog (Section 7.2).

**Payload**: Source symbol data or repair symbol (Symbol Size bytes
  when FEC is in use, or variable when FEC is not in use).  For
  source symbols, the first source symbol of each FEC block starts
  with a Block Manifest (Section 5.4) when FEC is in use.

**Auth Tag** (Auth Length bytes): Scheme-specific tag (HMAC, ALTA,
  etc.) as declared in catalog.

Fixed overhead: 8 bytes without optional fields (vs MMTP 33 bytes).

Each (sourceAddress, groupAddress, port) multicast tuple MUST
be associated with at most one MoQ namespace.  Publishers
requiring multiple streams MUST use distinct multicast groups
or ports.

### 5.3. Source Symbol Delivery

On multicast, media frames or chunks that exceed T bytes (the FEC
symbol size from the catalog) are fragmented into T-byte source
symbols by the FEC encoder before transmission.  Each source symbol
is carried in one UDP datagram with a condensed header (Flags
bit 0 = 0).

Source payloads MAY be smaller than T bytes (e.g., the last
symbol of a frame, or small audio frames).  Receivers determine
the actual payload size from the UDP datagram length:

    payload_size = datagram_length - header_size - Auth_Length

where header_size depends on the Flags bits present.  When FEC is
in use, receivers pad source payloads to T bytes with zeros for
FEC decoding.

Publishers MUST set the Close flag (Flags bit 4) on the last
source symbol of each media object (frame or chunk).  This enables
streaming-mode receivers (Section 5.5) to begin processing objects
before the full FEC block is available.

When FEC is not in use, each condensed packet carries one complete
media frame or chunk (or a fragment thereof with Close signaling
the final fragment).

### 5.4. Block Manifest

When FEC is in use and multiple media objects (frames, chunks) are
packed into a single FEC source block, the publisher MUST prepend a
Block Manifest to the source block's byte stream before symbol
fragmentation.  The Block Manifest is part of the FEC-protected
source data, ensuring reliable object boundary recovery even when
condensed headers (including the Close flag) are lost for
FEC-recovered symbols.

```
Block Manifest {
  Object Count (QUIC varint),                   // 1-8 bytes
  Object Lengths [Object Count] (QUIC varint),  // 1-8 bytes each
}
```

**Object Count**: Number of media objects in this source block,
  encoded as a QUIC variable-length integer (RFC 9000 Section 16).

**Object Lengths**: Array of `Object Count` entries, each a QUIC
  variable-length integer giving the byte size of one media object.
  Objects are listed in transmission order.

After FEC recovery, the receiver reads the Block Manifest from the
first bytes of the recovered source block, then uses the Object
Lengths to split the remaining bytes into individual media objects.

Overhead examples:

| Block content | Manifest size |
|--------------|--------------|
| 60 video frames, avg 5KB | 1 + 60x2 = 121 bytes |
| 4 CMAF chunks, avg 100KB | 1 + 4x4 = 17 bytes |
| 30 audio frames, avg 200B | 1 + 30x1 = 31 bytes |

When a source block contains exactly one object (e.g., interleave
depth 1 with one frame per group), the Block Manifest is still
present (Object Count = 1, one Object Length entry) for format
consistency.

### 5.5. Streaming Mode vs Recovery Mode

The condensed multicast format supports two receiver modes:

**Streaming mode**: Under low loss, receivers process source symbols
  as they arrive, using the Close flag (Flags bit 4) to detect
  object boundaries in real-time.  Each completed object is passed
  to the decoder immediately.  FEC recovery is not invoked.  This
  is analogous to ROUTE/ALC [ATSC-A331] streaming delivery.

**Recovery mode**: Under loss, receivers collect source and repair
  symbols for a full FEC block, perform FEC recovery (per
  [I-D.ramadan-moq-fec] when available), then use the Block Manifest
  (Section 5.4) to split the recovered byte stream into individual
  objects.  The Block Manifest is FEC-protected and always available
  after successful recovery.

Receivers MAY operate in streaming mode by default and fall back to
recovery mode when loss is detected within a source block.

## 6. Multi-Path Delivery

The same media content (source and repair) can be transmitted over
both MoQ unicast (QUIC/WebTransport) and multicast (UDP/SSM/AMT)
paths simultaneously.  Receivers on reliable unicast paths MAY
skip FEC decoding; receivers on lossy multicast paths use FEC for
recovery when available per [I-D.ramadan-moq-fec].

When symbols arrive from multiple paths simultaneously, receivers:

1. MUST deduplicate symbols using SBN+ESI as the unique key
2. MAY combine source symbols from reliable MoQ/QUIC with repair
   symbols from lossy multicast -- skipping FEC when all source
   symbols arrive reliably

Multicast-to-unicast failover: if multicast reception fails,
subscribers fall back to MoQ/QUIC unicast by subscribing with
filter LatestGroup or NextGroup per [I-D.ietf-moq-transport].
No coordinate mapping between multicast SBN and MoQ group_id is
needed -- the relay provides the current position.

## 7. Security Considerations

### 7.1. Multicast Security

SSM inherently limits traffic to authorized sources via (S,G)
filtering.  Sequence numbers enable replay detection.  For AMT,
trust is delegated to the relay per [RFC7450].

### 7.2. Content Authentication

Multicast UDP lacks QUIC's integrity guarantees.  Publishers MAY
include per-packet authentication tags in the condensed multicast
format (Auth Length + Auth Tag fields, Section 5.2) for end-to-end
content authentication across untrusted relay chains.

The authentication scheme is declared in the catalog as part of the
multicast configuration:

```json
{
  "multicast": {
    "auth": {
      "scheme": "hmac-sha256",
      "keyId": "stream-2026-04"
    },
    "endpoints": [...]
  }
}
```

**scheme** (string, REQUIRED if auth present): Authentication
  algorithm identifier.  Defined values:
  - "hmac-sha256": HMAC-SHA-256, 32-byte tag
  - "ed25519": Ed25519 signature, 64-byte tag
  - "alta": ALTA per [I-D.krose-mboned-alta], variable-length tag

**keyId** (string, OPTIONAL): Key identifier for key distribution.
  Key distribution mechanism is out of scope of this document.

For MMTP-packaged tracks, MMTP-native authentication mechanisms
apply per [ISO.23008-1].

On MoQ unicast, end-to-end content authentication MAY be provided
by Secure Objects which provides object-level encryption and
authentication independent of multicast.

## 8. IANA Considerations

This document has no IANA actions.  The multicast catalog extension
(Section 4) and condensed multicast packet format (Section 5.2) are
defined by this document and do not require IANA registration.

## 9. References

### 9.1. Normative References

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
           Requirement Levels", BCP 14, RFC 2119,
           DOI 10.17487/RFC2119, March 1997.

[RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
           2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174,
           May 2017.

[RFC4607]  Holbrook, H. and B. Cain, "Source-Specific Multicast for IP",
           RFC 4607, DOI 10.17487/RFC4607, August 2006.

[RFC7450]  Bumgardner, G., "Automatic Multicast Tunneling", RFC 7450,
           DOI 10.17487/RFC7450, February 2015.

[RFC8777]  Holland, J., "DNS Reverse IP Automatic Multicast Tunneling
           (AMT) Discovery", RFC 8777, DOI 10.17487/RFC8777, April 2020.

[RFC9000]  Iyengar, J., Ed. and M. Thomson, Ed., "QUIC: A UDP-Based
           Multiplexed and Secure Transport", RFC 9000,
           DOI 10.17487/RFC9000, May 2021.

[RFC9706]  Holland, J., et al., "TreeDN: Tree-Based Content Delivery
           Network (CDN) for Live Streaming to Mass Audiences",
           RFC 9706, December 2024.

[I-D.ietf-moq-transport]
           Curley, L., Pugin, K., Nandakumar, S., Vasiliev, V., and
           I. Swett, "Media over QUIC Transport",
           draft-ietf-moq-transport (work in progress).

### 9.2. Informative References

[I-D.ramadan-moq-mmt]
           Ramadan, O., "MPEG Media Transport (MMT) Packaging for
           Media over QUIC", draft-ramadan-moq-mmt (work in progress).

[I-D.ramadan-moq-fec]
           Ramadan, O., "Forward Error Correction for Media over QUIC",
           draft-ramadan-moq-fec (work in progress).

[I-D.ietf-moq-cmsf]
           Law, W., "CMSF: A CMAF Compliant Implementation of
           MOQT Streaming Format", draft-ietf-moq-cmsf
           (work in progress).

[I-D.ietf-moq-loc]
           Zanaty, M., et al., "Low Overhead Media Container",
           draft-ietf-moq-loc (work in progress).

[I-D.ietf-moq-catalogformat]
           Nandakumar, S., et al., "Common Catalog Format for
           MoQ", draft-ietf-moq-catalogformat (work in progress).

[I-D.krose-mboned-alta]
           Krose, B., "Asymmetric Loss-Tolerant Authentication",
           draft-krose-mboned-alta (work in progress).

[ISO.23008-1]
           ISO, "Information technology - High efficiency coding and
           media delivery in heterogeneous environments - Part 1:
           MPEG media transport (MMT)", ISO/IEC 23008-1:2023.

[ATSC-A331]
           ATSC, "Signaling, Delivery, Synchronization, and Error
           Protection", A/331:2025, February 2025.

[WICG-DirectSockets]
           Rayskiy, A., "Direct Sockets API", W3C Community Group
           Draft Report, https://wicg.github.io/direct-sockets/

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
