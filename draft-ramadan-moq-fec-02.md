# Forward Error Correction for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: January 2027                                       July 2026

        Forward Error Correction for Media over QUIC (MoQ)
                      draft-ramadan-moq-fec-02

Abstract

   This document specifies a mechanism for transmitting Forward Error
   Correction (FEC) repair data alongside source media in Media over
   QUIC (MoQ) sessions.  It defines signaling for FEC configuration
   via both control messages and catalog extensions, conventions for
   repair track naming, and the format of repair objects.  The mechanism
   supports RaptorQ (RFC 6330), Reed-Solomon (RFC 5510), and XOR-based
   FEC schemes, enabling receivers to recover from packet loss without
   retransmission latency.  A repair container abstraction allows
   repair objects to carry either condensed-framed symbols or
   container-specific payloads (e.g., MMTP repair packets), enabling
   multi-path FEC delivery where receivers apply identical recovery
   logic regardless of whether data arrives via MoQ/QUIC, SSM
   multicast, or ATSC 3.0 broadcast.  This specification is designed
   for compatibility with ATSC 3.0 (A/331) and ARIB STD-B60 broadcast
   systems.

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
   2.  Terminology
   3.  Protocol Overview
   4.  FEC Configuration
       4.1.  FEC_CONFIG Message
       4.2.  FEC_CONFIG Extension Header (Alternative)
       4.3.  FEC Algorithms
       4.4.  RaptorQ Object Transmission Information
       4.5.  Reed-Solomon Parameters
       4.6.  Repair Container Types
   5.  Catalog FEC Extension
       5.1.  Catalog Fields
       5.2.  Relationship to FEC_CONFIG
   6.  Repair Track Convention
       6.1.  Track Naming
       6.2.  Subscription Model
   7.  Repair Object Format
       7.1.  Condensed Repair Format
       7.2.  Repair Symbols
   7A. MoQ Extension Headers for FEC
       7A.1. FEC Payload ID Extension
       7A.2. Object Length Extension
       7A.3. Auth Tag Extension
   8.  Block and Group Alignment
       8.1.  Single-Group Blocks
       8.2.  Multi-Group Blocks
   9.  Interleaving
   10. Priority and Congestion
   11. Hybrid Unicast and Multicast Delivery
       11.1. Multicast and AMT Delivery
       11.2. Relay-Generated Repair
       11.3. Multi-Path FEC Delivery
   12. ATSC 3.0 Compatibility
       12.1. Mapping to A/331 AL-FEC
       12.2. S-TSID to FEC_CONFIG Conversion
   13. Interaction with CMAF Packaging
   14. Security Considerations
       14.1. FEC Does Not Provide Confidentiality
       14.2. Repair Symbol Manipulation
       14.3. Amplification
       14.4. Multicast FEC Security
   15. IANA Considerations
       15.5. MoQ Object Extension Header Types
   16. References
       16.1. Normative References
       16.2. Informative References
   Appendix A.  Example Message Flows
       A.3. Per-Object Overhead Comparison
   Appendix B.  Catalog Example
   Appendix C.  MMTP Repair Container Mapping (Informational)
   Authors' Addresses
```

## 1. Introduction

Media over QUIC (MoQ) [I-D.ietf-moq-transport] provides a
publish/subscribe transport for real-time media.  While QUIC provides
reliable delivery through retransmission, the latency incurred by
retransmission can be unacceptable for ultra-low-latency (ULL)
applications such as live sports, gaming, and interactive broadcasts.

Forward Error Correction (FEC) enables receivers to recover lost
packets using redundant repair data, without waiting for
retransmissions.  This is particularly valuable in wireless
environments where burst packet loss is common.

This document defines:

1. A FEC_CONFIG message for signaling FEC parameters per subscription
2. An alternative FEC_CONFIG extension header for alignment with
   MoQ transport extension mechanisms
3. A catalog extension for out-of-band FEC signaling
4. A naming convention for repair tracks
5. The format of repair objects containing FEC symbols
6. Interleaving strategies to protect against burst loss
7. Compatibility mappings for ATSC 3.0 and ARIB STD-B60 broadcast systems
8. Multi-path delivery where FEC symbols arrive from multiple transport
   paths (MoQ/QUIC, SSM multicast, ATSC 3.0 broadcast) with identical
   recovery at the receiver

The mechanism is designed to complement existing MoQ media packaging
formats including CMAF [I-D.ietf-moq-cmsf], LOC [I-D.ietf-moq-loc], and
MMT [I-D.ramadan-moq-mmt] by operating as a separate protection layer.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

**Source Track**: A MoQ track carrying media data (e.g., video, audio).

**Repair Track**: A MoQ track carrying FEC repair symbols for a
corresponding source track.

**Source Block**: A sequence of K source symbols over which FEC
encoding is performed.

**Source Symbol**: A fixed-size unit of source data used for FEC
encoding.

**Repair Symbol**: A symbol generated by the FEC encoder that can be
used to recover lost source symbols.

**FEC Block**: The combination of K source symbols and P repair
symbols.

**Interleave Depth**: The number of media units (e.g., frames, chunks)
spanned by a single source block.

**Object Transmission Information (OTI)**: Parameters required to
configure a RaptorQ decoder, as defined in [RFC6330].

**Source Block Number (SBN)**: A zero-based index identifying a
specific source block within a session.

## 3. Protocol Overview

```
Publisher                              Subscriber
    |                                       |
    |  SUBSCRIBE (source track)             |
    |<--------------------------------------|
    |                                       |
    |  SUBSCRIBE_OK                         |
    |-------------------------------------->|
    |                                       |
    |  FEC_CONFIG (optional)                |
    |-------------------------------------->|
    |                                       |
    |  SUBSCRIBE (repair track)             |
    |<--------------------------------------|
    |                                       |
    |  SUBSCRIBE_OK                         |
    |-------------------------------------->|
    |                                       |
    |  OBJECT (source, group=N, obj=0..K-1) |
    |-------------------------------------->|
    |                                       |
    |  OBJECT (repair, group=N, obj=0..P-1) |
    |-------------------------------------->|
    |                                       |
```

The protocol operates as follows:

1. Subscriber sends SUBSCRIBE for source track
2. Publisher responds with SUBSCRIBE_OK
3. Publisher MAY send FEC_CONFIG to indicate FEC availability
4. Subscriber sends SUBSCRIBE for repair track if FEC desired
5. Publisher sends source objects followed by repair objects
6. Subscriber uses repair symbols to recover any lost source symbols

Alternatively, FEC parameters MAY be discovered via catalog (Section 5)
before subscribing.

## 4. FEC Configuration

### 4.1. FEC_CONFIG Message

The FEC_CONFIG message is sent by the publisher after SUBSCRIBE_OK to
advertise FEC parameters.  Subscribers use this information to decide
whether to subscribe to the repair track.

```
FEC_CONFIG Message {
  Message Type (i) = 0x50,
  Subscribe ID (i),
  FEC Enabled (1),
  FEC Algorithm (8),
  Repair Container (8),
  Source Symbols Per Block (i),
  Repair Symbols Per Block (i),
  Symbol Size (i),
  Interleave Depth (i),
  OTI Length (i),
  Object Transmission Information (..),
}
```

**Subscribe ID**: The subscription this FEC configuration applies to.

**FEC Enabled**: 1 if FEC is available for this track, 0 otherwise.

**FEC Algorithm**: The FEC algorithm identifier (see Section 4.3).

**Repair Container**: Identifies the format of repair object payloads
(see Section 4.6).  Default value 0x00 indicates the condensed repair
format defined in Section 7 (raw repair symbol, SBN/ESI derived from
MoQ framing).
Value 0x01 indicates MMTP repair packet
passthrough per Appendix C.  Receivers branch once on this value at
subscription time to select the appropriate byte-offset extraction
recipe.

**Source Symbols Per Block (K)**: Number of source symbols per FEC
block.  MUST be >= 1.

**Repair Symbols Per Block (P)**: Number of repair symbols generated
per block.  MUST be >= 1.

**Symbol Size**: Size of each symbol in bytes.  All symbols in a
block MUST have the same size.  For RaptorQ, MUST be a multiple of
the Symbol Alignment parameter (typically 8 bytes per RFC 6330).

**Interleave Depth**: Number of media units (frames/chunks) spanned by
one source block.  Higher values protect against longer burst losses
but increase latency.

**OTI Length**: Length of the OTI field in bytes.  0 if not applicable.

**Object Transmission Information**: Algorithm-specific parameters.
For RaptorQ, this is the concatenation of the 8-byte Common FEC OTI
and the 4-byte Scheme-Specific FEC OTI (12 bytes total) as defined
in [RFC6330] Sections 3.3.2 and 3.3.3.

### 4.2. FEC_CONFIG Extension Header (Provisional)

NOTE: This section is provisional and depends on a future extension
to [I-D.ietf-moq-transport] that adds extension header support on
control messages (e.g., SUBSCRIBE_OK).  As of moq-transport-15,
extension headers are defined only for Object payloads, not control
messages.  Implementations MUST use the FEC_CONFIG message
(Section 4.1) as the normative signaling mechanism until such an
extension is adopted.

For implementations where MoQ transport supports control message
extensions, FEC parameters MAY alternatively be conveyed as an
extension header in SUBSCRIBE_OK:

```
FEC_CONFIG Extension Header {
  Extension Type (i) = 0x50,
  Length (i),
  FEC Enabled (1),
  FEC Algorithm (8),
  Source Symbols Per Block (i),
  Repair Symbols Per Block (i),
  Symbol Size (i),
  Interleave Depth (i),
  OTI Length (i),
  Object Transmission Information (..),
}
```

Relays MUST cache and forward this extension header without
modification.

The FEC_CONFIG message (Section 4.1) is the normative mechanism.
The extension header is informational and provided for convenience.
Publishers MUST NOT send both simultaneously for the same
subscription.

### 4.3. FEC Algorithms

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | N/A       |
| 0x01  | XOR       | This document |
| 0x02  | RaptorQ   | [RFC6330] |
| 0x03  | Reed-Solomon (GF2^8) | [RFC5510] |
| 0x04-0xFF | Reserved | IANA |

**XOR (0x01)**: Simple XOR-based FEC.  Each repair symbol is the XOR
of all source symbols in the block.  Can recover exactly one lost
symbol per block.

**RaptorQ (0x02)**: RaptorQ fountain code per [RFC6330].  Can recover
from loss of any symbols as long as K symbols (source or repair) are
received.  Recommended for most applications.

**Reed-Solomon (0x03)**: Reed-Solomon erasure code over GF(2^8) per
[RFC5510].  Can recover up to P lost symbols.  Suitable for
applications requiring exact recovery guarantees or interoperability
with ATSC 3.0 systems using RS FEC.

### 4.4. RaptorQ Object Transmission Information

When FEC Algorithm is RaptorQ (0x02), the OTI field contains the
concatenation of the 8-byte Common FEC OTI (Section 3.3.2 of
[RFC6330]) and the 4-byte Scheme-Specific FEC OTI (Section 3.3.3
of [RFC6330]), totaling 12 bytes:

```
RaptorQ OTI {
  Transfer Length (40),
  Reserved (8),
  Symbol Size (16),
  Number of Source Blocks (8),
  Number of Sub-Blocks (16),
  Symbol Alignment (8),
}
```

Note: For MoQ usage, Transfer Length is typically computed per source
block from the cumulative size of source objects in the block.

### 4.5. Reed-Solomon Parameters

When FEC Algorithm is Reed-Solomon (0x03), the OTI field contains:

```
Reed-Solomon OTI {
  Field Size (8),          // Always 8 for GF(2^8)
  Max Source Symbols (16), // Maximum K value
  Max Repair Symbols (16), // Maximum P value
  Reserved (24),
}
```

### 4.6. Repair Object Format Selection

The format of repair object payloads is determined by the repair
track's packaging type in the catalog.  Each packaging type defines
its own repair object layout:

| Repair Track Packaging | Repair Object Format | Reference |
|------------------------|---------------------|-----------|
| condensed | Raw repair symbol, SBN/ESI from MoQ framing (Section 7.1) | This document |
| mmtp | MMTP repair packet (type 0x03) per ISO 23008-1 (Appendix C) | [I-D.ramadan-moq-mmt] |
| loc | LOC object with repair symbol payload (Section 7.3) | [I-D.ietf-moq-loc] |

The repair track's packaging MUST match the source track's
packaging when the source uses a format with native FEC support
(mmtp, loc).  When the source uses cmaf packaging, the repair
track uses condensed packaging because CMAF does not define a
native repair framing.

Receivers determine the repair format by inspecting the repair
track's packaging field in the catalog.  No separate "repair
container" signaling is needed — the packaging field IS the signal.

## 5. Catalog FEC Extension

### 5.1. Catalog Fields

MoQ catalogs [I-D.ietf-moq-catalogformat] MAY include FEC
configuration at the track level:

```json
{
  "tracks": [
    {
      "name": "video",
      "packaging": "mmtp",
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "symbolSize": 1312,
        "interleaveDepth": 4,
        "repairTrack": "video/repair"
      }
    },
    {
      "name": "video/repair",
      "packaging": "mmtp"
    }
  ]
}
```

The repair track is a regular catalog track whose packaging field
indicates the repair object format (Section 4.6).  In this example,
both source and repair use mmtp packaging — the MMTP type 0x03
repair packet format carries FEC Payload ID and OTI natively.

Field definitions for the `fec` object:

**algorithm** (string, REQUIRED if fec present): FEC scheme identifier.
  - "none": No FEC
  - "xor": Simple XOR
  - "raptorq": RaptorQ per RFC 6330
  - "reed-solomon": Reed-Solomon per RFC 5510

**sourceSymbols** (integer, REQUIRED): K value - source symbols per
FEC block.

**repairSymbols** (integer, REQUIRED): P value - repair symbols per
FEC block.

**symbolSize** (integer, REQUIRED): T value - bytes per symbol.

**interleaveDepth** (integer, OPTIONAL): Number of media units per
FEC block.  Default is 1.

**repairTrack** (string, REQUIRED): Track name for repair symbols,
following the convention in Section 6.1.

### 5.2. Relationship to FEC_CONFIG

The FEC_CONFIG message (Section 4.1) is the normative signaling
mechanism.  The catalog FEC extension (Section 5.1) provides
out-of-band discovery before subscription.

Precedence rules when multiple signaling sources are present:

1. **FEC_CONFIG message** is authoritative.  If present, it defines
   the FEC parameters for the subscription.
2. **Catalog FEC fields** are informational for discovery purposes.
   Subscribers use catalog data to decide whether to subscribe to
   repair tracks before receiving FEC_CONFIG.
3. If FEC_CONFIG parameters differ from catalog values, subscribers
   MUST use FEC_CONFIG parameters for decoding.
4. Subscribers SHOULD log a warning if FEC_CONFIG differs from
   catalog, as this may indicate a configuration error.
5. The provisional extension header (Section 4.2) is informational
   only and MUST NOT be used as the sole FEC signaling mechanism.

For multicast delivery where FEC_CONFIG cannot be sent (no back
channel), catalog-based signaling is REQUIRED.

Relay behavior: Relays MUST forward FEC_CONFIG messages unmodified
to downstream subscribers.  Relays MUST NOT strip FEC_CONFIG in
favor of catalog or extension header signaling.

## 6. Repair Track Convention

### 6.1. Track Naming

For a source track with name components [namespace, track_name], the
corresponding repair track MUST be named:

```
[namespace, track_name, "repair"]
```

Examples:

| Source Track | Repair Track |
|--------------|--------------|
| ["live", "video"] | ["live", "video", "repair"] |
| ["broadcast", "video", "1080p"] | ["broadcast", "video", "1080p", "repair"] |
| ["stream", "audio", "en"] | ["stream", "audio", "en", "repair"] |

This convention allows subscribers to discover repair tracks without
additional signaling.

### 6.2. Subscription Model

Subscribers SHOULD subscribe to the source track first, then subscribe
to the repair track if:

1. The FEC_CONFIG message indicates FEC is available, AND
2. The subscriber desires FEC protection

Publishers MUST NOT require subscription to repair tracks.  Repair
tracks are optional and subscribers MAY choose to rely solely on
QUIC retransmission.

Subscribers MAY subscribe to repair tracks without first receiving
FEC_CONFIG if they have out-of-band knowledge of FEC availability
(e.g., from catalog).

## 7. Repair Object Format

### 7.1. Condensed Repair Format

This section defines the condensed repair format, used when the
repair track's packaging is "condensed" in the catalog.  This is
the default repair format for CMAF source tracks, which do not
have a native FEC framing.

When the repair track uses a different packaging (e.g., "mmtp" per
Appendix C, or "loc" per Section 7.3), repair object payloads
follow the packaging-specific format and this section does not apply.

Each MoQ object on a repair track contains exactly one repair symbol.
The object payload is the raw repair symbol data (Symbol Size bytes),
with no additional header.

```
Repair Object Payload {
  Repair Symbol (Symbol Size bytes),
}
```

SBN and ESI are derived from MoQ transport framing:

```
SBN = floor(group_id / interleave_depth)
ESI = K + object_id
```

where K (source symbols per block) and interleave_depth are known
from the catalog or FEC_CONFIG.  Repair symbols have ESI >= K by
convention: object_id 0 on the repair track corresponds to ESI K,
object_id 1 to ESI K+1, and so on up to object_id P-1 (ESI K+P-1).

### 7.1.1. Repair Track Group Structure

Publishers MUST structure repair tracks so that group_id equals
the Source Block Number (SBN).  Object IDs within each repair group
MUST be sequential starting from 0, where object_id R corresponds
to repair symbol ESI K+R.  This convention enables receivers to
derive SBN and ESI from MoQ framing without per-object metadata,
regardless of the source track's container format (LOC, CMAF, etc.).

This convention holds for all container modes.  While CMAF source
objects require explicit FEC Payload ID extensions (Section 7A.1)
because CMAF chunks do not map 1:1 to source symbols, repair
objects ARE 1:1 with symbols (each object = one T-byte repair
symbol), so the derivation is valid.

Publishers MUST publish exactly one repair symbol per MoQ object.
Bundling multiple repair symbols in a single object would create a
single point of failure — losing that object loses all repair for
the block, defeating the purpose of FEC.  Separate objects allow
independent delivery, priority, and congestion handling per symbol.

The number of repair symbols per block (P) is known from the catalog
or FEC_CONFIG and does not need to be signaled per object.

### 7.2. Repair Symbol Generation

For RaptorQ, repair symbols are generated per [RFC6330] Section 5.3.
The Encoding Symbol ID (ESI) for repair symbols starts at K (the
number of source symbols).

For Reed-Solomon, repair symbols are generated per [RFC5510] using
the Vandermonde matrix construction.

For XOR, a single repair symbol is generated as:

```
repair_symbol = source_symbol[0] XOR source_symbol[1] XOR ... XOR source_symbol[K-1]
```

### 7.3. LOC Repair Format

When the repair track's packaging is "loc", each MoQ object on
the repair track is a standard LOC object containing one repair
symbol as its payload.  SBN and ESI are derived from MoQ transport
framing identically to the condensed format (Section 7.1):

```
SBN = floor(group_id / interleave_depth)
ESI = K + object_id
```

LOC repair objects MAY include an Auth Tag extension header
[I-D.ietf-moq-loc] for per-symbol authentication.  No FEC-specific
extension headers are needed — the LOC framing plus MoQ object
coordinates provide all metadata required for FEC recovery.

LOC repair is the natural pairing for LOC source tracks.  Both
source and repair use the same LOC packaging, enabling receivers
to apply a single LOC parser to both tracks.

### 7.4. Condensed Multicast Wire Format

When condensed-packaged content is delivered over IP multicast (SSM/ASM),
each UDP datagram carries one condensed multicast packet.  This format
provides audio/video demuxing (Track ID), FEC metadata (SBN/ESI), and
optional authentication — all catalog-driven with no per-packet flags.

```
Condensed Multicast Packet {
  Track ID (16),              // 2 bytes — audio/video routing
  Repair (8),                 // 1 byte — 0=source, 1=repair
  Source Block Number (24),   // 3 bytes — SBN (wraps at ~16M blocks)
  Encoding Symbol ID (8),    // 1 byte — ESI (max 255 symbols/block)
  Auth Length (8),            // 1 byte — 0=no auth, else tag size
  Payload (..),               // Source chunk or repair symbol
  [Auth Tag (Auth Length)],   // Variable-length (HMAC, ALTA, etc.)
}
```

**Track ID** (16 bits): Identifies the media track within the SSM
  group.  Assigned per-track by the publisher and signaled in the
  catalog.  Receivers use Track ID to route packets to the correct
  decoder (e.g., video vs audio).  Analogous to MMTP packet_id.

**Repair** (8 bits): 0 for source packets, 1 for repair symbols.
  Receivers branch on this field to route source data to the media
  decoder and repair data to the FEC recovery engine.  The explicit
  flag is necessary because K may vary per block (e.g., blocks
  spanning keyframes), making ESI >= K unreliable for source/repair
  discrimination.

**SBN** (24 bits): Source Block Number.  Identifies the FEC block.
  Wraps at 16,777,216 — at 30fps with interleave depth 4 (~7.5
  blocks/sec), this provides ~25 days before wraparound.

**ESI** (8 bits): Encoding Symbol ID.  For source packets, ESI < K.
  For repair packets, ESI >= K.  Maximum 255 symbols per block
  (e.g., K=247 source + P=8 repair).

**Auth Length** (8 bits): Length of the authentication tag in bytes.
  0 if no authentication for this packet.  Maximum 255 bytes,
  sufficient for HMAC-SHA256 (32), Ed25519 (64), and ALTA
  [I-D.krose-mboned-alta] variable-length tags (chained MACs +
  optional signature).  The per-packet length field is necessary
  because schemes like ALTA produce variable-size tags depending
  on MAC chain depth and signature presence.

**Payload**: Source chunk data (CMAF moof+mdat fragment or portion
  thereof) or repair symbol (Symbol Size bytes from FEC encoding).

**Auth Tag** (Auth Length bytes): Authentication tag.  Present when
  Auth Length > 0.  Format is scheme-specific (HMAC, ALTA, etc.)
  as declared in the catalog.

Fixed overhead: 8 bytes.  Compare to MMTP repair (33 bytes) and
LOC with extensions (~14+ bytes).  The 4x reduction in per-packet
overhead is significant for high-rate multicast streams.

On the MoQ unicast path, condensed packaging uses the headerless
format (Section 7.1) with SBN/ESI derived from MoQ transport framing.
The multicast wire format in this section applies only to UDP delivery.

## 7A. MoQ Extension Headers for FEC

This section defines MoQ transport extension headers used by FEC-enabled
streams.  These extensions use the MoQ transport extension header
mechanism defined in [I-D.ietf-moq-transport].  Each extension is
self-describing (type + length).  Presence is determined by the
catalog — no per-object flags byte is needed.

### 7A.1. FEC Payload ID Extension

```
FEC Payload ID Extension {
  Extension Type (i) = 0xNN,
  Length (i) = 8,
  Source Block Number (32),
  Encoding Symbol ID (32),
}
```

The FEC Payload ID extension carries the combined Source Block Number
(SBN, 32 bits) and Encoding Symbol ID (ESI, 32 bits) for a total of
8 bytes.

This extension is REQUIRED on CMAF source track objects when FEC is
enabled.  CMAF chunks are variable-size due to VBR encoding, and the
object-to-source-block mapping is not 1:1 — a single CMAF chunk may
span multiple source blocks, or multiple chunks may fall within one
block.  The explicit SBN+ESI enables receivers to correctly associate
each object with its FEC source block.

This extension is NOT needed on LOC source objects.  For LOC, SBN and
ESI are derivable from MoQ object coordinates:

```
SBN = floor(group_id / interleave_depth)
ESI = object_id
```

This derivation works for all cases including late join and relay
forwarding because the catalog (which contains interleave_depth) is
always received before the first media object, and group_id/object_id
are present in every MoQ object header.

This extension is NOT present on repair track objects.  Repair objects
derive SBN and ESI from MoQ framing (Section 7.1): SBN =
floor(group_id / interleave_depth), ESI = K + object_id.

### 7A.2. Object Length Extension

```
Object Length Extension {
  Extension Type (i) = 0xNN,
  Length (i),
  Original Object Length (i),
}
```

The Object Length extension carries the original object size as a
QUIC variable-length integer (1-8 bytes, per RFC 9000 Section 16).
The encoding is self-describing: the two most significant bits of
the first byte indicate the field width.

This extension is REQUIRED on CMAF source track objects when FEC is
enabled.  The receiver needs the original object size to strip
zero-padding from the last FEC source symbol.  CMAF chunks are
variable-size (VBR encoding), and a single object may span multiple
source blocks, making the padding length non-trivial to derive
without this field.

This extension is NOT needed on LOC source objects.  MoQ framing
combined with fixed symbol size is sufficient to determine original
object boundaries and strip padding.

### 7A.3. Auth Tag Extension

```
Auth Tag Extension {
  Extension Type (i) = 0xNN,
  Length (i) = N,
  Auth Tag (N),
}
```

The Auth Tag extension carries an authentication tag whose length
is given by the MoQ extension Length field.  Tag size may be fixed
(HMAC-SHA256: always 32 bytes) or variable (ALTA
[I-D.krose-mboned-alta]: varies per packet depending on MAC chain
depth and signature presence).  The catalog declares the auth
scheme and key distribution mechanism; the per-object Length field
carries the actual tag size.

This extension is OPTIONAL on LOC source and repair track objects.
MMTP tracks use MMTP-native authentication.  Condensed multicast
packets carry auth via the Auth Length field (Section 7.4).

The Auth Tag provides end-to-end content authentication across
untrusted relays (e.g., ALTA).  MoQ transport security (TLS/QUIC)
protects individual hops but does not protect content integrity
across the full delivery chain — a compromised or malicious relay
could modify object payloads before re-encrypting for the next hop.
The Auth Tag extension enables receivers to verify that object
content originated from the publisher, regardless of how many
relay hops it traversed.

Auth on repair objects is particularly important: a compromised
relay could inject malicious repair symbols that cause receivers
to reconstruct corrupted source data after FEC recovery.  When
auth is enabled, receivers SHOULD verify repair object auth tags
before feeding symbols to the FEC decoder.

## 8. Block and Group Alignment

### 8.1. Single-Group Blocks (Interleave Depth = 1)

When Interleave Depth is 1, each FEC block corresponds to exactly
one MoQ Group:

```
Group 0:  [Obj 0] [Obj 1] ... [Obj K-1]  →  FEC Block 0
Group 1:  [Obj 0] [Obj 1] ... [Obj K-1]  →  FEC Block 1
...
```

The Source Block Number (SBN) in repair objects equals the Group ID.

### 8.2. Multi-Group Blocks (Interleave Depth > 1)

When Interleave Depth is D > 1, each FEC block spans D consecutive
Groups:

```
Groups 0-3 (D=4):  →  FEC Block 0, SBN = 0
Groups 4-7 (D=4):  →  FEC Block 1, SBN = 1
...
```

The SBN is computed as: `SBN = floor(Group_ID / Interleave_Depth)`

The first Group ID covered by a block is: `First_Group = SBN * Interleave_Depth`

Repair objects for Block N are published after the last source object
of Group `(N+1) * D - 1`.

## 9. Interleaving

Interleaving spreads source symbols across time to protect against
burst loss.  With interleave depth D, symbols from D consecutive
media units are grouped into one source block:

```
Media Units:    M0   M1   M2   M3   M4   M5   M6   M7
                 |    |    |    |    |    |    |    |
Interleave=4:   [--- Block 0 ---]  [--- Block 1 ---]
                 S0   S1   S2   S3   S4   S5   S6   S7
                                |                    |
                        Block 0 Repair:      Block 1 Repair:
                           R0, R1               R2, R3
```

A burst loss affecting M1-M2 loses symbols S1,S2 from Block 0.  With
P >= 2 repair symbols, Block 0 can still be recovered.

Publishers SHOULD choose interleave depth based on:

- Expected burst loss duration
- Acceptable recovery latency
- Available bandwidth for repair overhead

Typical values:

| Application | Interleave Depth | Latency Impact |
|-------------|------------------|----------------|
| Interactive (gaming, WebRTC) | 1-4 | 33-133ms at 30fps |
| Low-latency live | 4-8 | 133-267ms at 30fps |
| Broadcast | 30 | ~1 second at 30fps |
| ATSC 3.0 default | 60 | ~2 seconds at 30fps |

## 10. Priority and Congestion

Repair tracks SHOULD use lower priority than source tracks so that
repair data is dropped first under congestion.

Recommended priority assignment:

| Track Type | Priority | Notes |
|------------|----------|-------|
| Control/Signaling | 0-1 | Highest priority |
| Source Media | 2-4 | Protected |
| Repair Symbols | 6-7 | Dropped first |

Publishers set priority via the QUIC stream priority mechanism or
MoQ-specific priority signaling.

Under congestion, this priority separation ensures:

1. Source media is preserved as long as possible
2. Repair overhead is gracefully shed
3. Receivers degrade to QUIC retransmission when FEC is unavailable

## 11. Hybrid Unicast and Multicast Delivery

This FEC mechanism is designed to support hybrid delivery architectures
where the same media stream is delivered via multiple transport paths
simultaneously.  Multicast delivery paths (Section 3), TreeDN
integration (Section 5), and transport hierarchy (Section 6) are
defined in [I-D.ramadan-moq-multicast]; this section covers
FEC-specific considerations for hybrid delivery.

### 11.1. Multicast and AMT Delivery

IP multicast (including Source-Specific Multicast per [RFC4607]) and
Automatic Multicast Tunneling (AMT per [RFC7450]) provide efficient
one-to-many delivery but are inherently unreliable.  FEC enables receivers
to recover from packet loss without retransmission:

- **Native SSM**: Receivers join multicast group directly
- **AMT Tunneling**: Receivers without native multicast connect via AMT
  relays discovered through DRIAD ([RFC8777])
- **TreeDN Distribution**: Hierarchical multicast trees per [RFC9706]
  with FEC protection at each hop.  ISP edge routers with inline AMT
  relay capability [JUNIPER-TREEDN] can serve as TreeDN nodes,
  replacing x86 CDN servers with in-router replication while
  preserving FEC repair across the delivery path

The same MoQ objects (source and repair) can be transmitted over both
unicast (QUIC/WebTransport) and multicast (UDP/SSM/AMT) paths.  Receivers
on reliable unicast paths MAY skip FEC decoding, while receivers on lossy
multicast paths use FEC for recovery.

```
Publisher
    |
    v
+----------------+
|   MoQ Relay    |
| (Origin/Root)  |
+-------+--------+
        |
   +----+----+--------------------+
   |         |                    |
   v         v                    v
Unicast   SSM Multicast      AMT Tunnel
(QUIC)    (Native)           (Encapsulated)
   |         |                    |
   v         v                    v
[Web]    [Native SSM         [Mobile/
Client    Clients]            Remote]

FEC:      FEC:                FEC:
Optional  Required            Required
```

### 11.2. Relay-Generated Repair

Relays MAY generate additional repair symbols for local network conditions.
This is enabled by the fountain code property of RaptorQ, which allows
generation of arbitrary repair symbols from decoded source data.

Use cases for relay-generated repair include:

- Edge relays serving lossy last-mile networks (WiFi, cellular) can increase
  repair overhead beyond what the publisher provides
- Regional relays can adapt FEC parameters to local loss patterns
- Relays can regenerate repair if upstream repair symbols were lost
- IWA home gateways receiving multicast over wired Ethernet can apply
  local FEC repair before redistributing via WebTransport to wireless
  clients, reducing retransmission round trips to the origin

When a relay generates its own repair symbols:

1. The relay MUST successfully decode the source block first
2. Generated repair symbols SHOULD use ESIs that do not conflict with
   publisher-generated repair (e.g., ESI >= K + P_publisher)
3. The relay MAY advertise relay-specific repair via a separate track
   namespace or track path suffix

Security considerations for relay-generated repair:

A compromised relay could inject malicious repair symbols that cause
receivers to reconstruct corrupted source data.  This risk is
elevated on multicast paths where QUIC's integrity guarantees are
absent.  To mitigate:

1. **Block-level integrity**: Publishers SHOULD use the Auth Tag
   extension (Section 7A.3) on source objects so receivers can
   verify that decoded source data matches the publisher's
   original after FEC recovery.
2. **Trust model**: Receivers MUST distinguish between
   publisher-generated repair (trusted, from the origin) and
   relay-generated repair (semi-trusted).  If decoded data fails
   integrity verification using relay-generated repair, receivers
   SHOULD discard it and request retransmission via unicast.
3. **Separate tracks**: Relay-generated repair SHOULD be published
   on a distinct track (e.g., "video/repair/relay") so receivers
   can apply different trust policies.

```
Publisher (K=32, P=8, 25% overhead)
    |
    | Source ESI: 0-31, Repair ESI: 32-39
    v
+------------------+
| Backbone Relay   |  (passes through, low loss)
+--------+---------+
         |
    +----+----+
    |         |
    v         v
+--------+ +--------+
| Edge A | | Edge B |
| P=16   | | P=4    |
| (50%)  | | (12%)  |
+--------+ +--------+
    |         |
    v         v
[Lossy    [Clean
 WiFi]     Fiber]
```

### 11.3. Multi-Path FEC Delivery

When source and repair data are delivered via multiple transport
paths simultaneously (e.g., MoQ/QUIC unicast and SSM multicast,
or MoQ/QUIC and ATSC 3.0 RF broadcast), receivers combine FEC
symbols from all available paths:

```
          +------ MoQ/QUIC (reliable) ------+
          |                                  |
Publisher --> MoQ Relay                  Receiver
          |                                  |
          +------ SSM Multicast (lossy) ----+
                  or ATSC 3.0 RF broadcast
```

Key multi-path considerations:

1. **Format consistency**: All paths delivering repair symbols
   for the same source track SHOULD use the same repair container
   format.  This ensures the receiver applies one byte-offset
   extraction recipe per track, branching once on container type
   at subscription time.

2. **Deduplication**: Receivers receiving the same source symbol
   via multiple paths MUST deduplicate before FEC decoding.  Source
   Block Number (SBN) and Encoding Symbol ID (ESI) uniquely
   identify each symbol across all paths.

3. **Per-packet OTI**: When using container formats that carry OTI
   per-packet (e.g., MMTP, Appendix C), receivers that join
   mid-stream obtain OTI from the first repair packet received on
   any path, without waiting for FEC_CONFIG delivery.

4. **Reliable + unreliable combination**: Source symbols on the
   reliable MoQ/QUIC path and repair symbols on the lossy multicast
   path is a valid configuration.  Receivers on multi-path MAY skip
   FEC decoding when the reliable path delivers all source symbols,
   using repair only when the reliable path experiences elevated
   loss or congestion.

## 12. ATSC 3.0 Compatibility

This specification is designed for interoperability with ATSC A/331
[ATSC-A331] Application-Layer FEC.

### 12.1. Mapping to A/331 AL-FEC

| ATSC A/331 Concept | MoQ FEC Equivalent |
|--------------------|--------------------|
| `<FECParameters>` | FEC_CONFIG message or catalog `fec` field |
| `<RepairFlow>` | Repair track subscription |
| `fecOTI` (in S-TSID) | OTI field in FEC_CONFIG |
| Source TOI range | Group ID range per Section 8 |
| `maximumDelay` | Derived from Interleave Depth × frame duration |
| `overhead` | Computed as (P / K) × 100 |

Publishers ingesting ATSC 3.0 broadcasts SHOULD preserve the original
FEC parameters and pass them through in FEC_CONFIG.

For ATSC 3.0 MMT mode ingest, the repair track uses "mmtp"
packaging — repair packets are passed through as-is.  See
Appendix C for the MMTP repair format and multi-path deployment
scenarios.

### 12.2. S-TSID to FEC_CONFIG Conversion

When converting ATSC S-TSID FEC parameters to MoQ FEC_CONFIG:

```
S-TSID (per ATSC A/331):
<FECParameters maximumDelay="1000" overhead="25"
               fecOTI="F=32;T=1312;Z=4;N=1;Al=8">
  <ProtectedObject tsi="1">
    <SourceTOI x="0" y="255"/>
  </ProtectedObject>
</FECParameters>

Note: The fecOTI attribute uses ATSC A/331 field names, which
differ from RFC 6330 terminology.  F denotes the number of source
symbols per block (equivalent to K in this document), T is symbol
size, Z is the number of source blocks, N is sub-blocks, and Al
is symbol alignment.  RFC 6330 uses F for Transfer Length (40 bits),
which is a different concept; implementors should use the A/331
schema as the reference for S-TSID parsing.

Converts to FEC_CONFIG:
  FEC Algorithm: 0x02 (RaptorQ)
  Source Symbols Per Block: 32  (from F parameter)
  Repair Symbols Per Block: 8  (25% of 32, from overhead)
  Symbol Size: 1312  (from T parameter)
  Interleave Depth: 4  (from Z parameter or derived from maximumDelay)
  OTI: [12-byte concatenation of Common FEC OTI + Scheme-Specific
        FEC OTI per RFC 6330]
```

## 13. Interaction with CMAF Packaging

This specification is designed to work alongside CMAF packaging for
MoQ [I-D.ietf-moq-cmsf].  The relationship is:

```
┌─────────────────────────────────────────────────┐
│              MoQ Track Namespace                │
├─────────────────────────────────────────────────┤
│                                                 │
│  Source Track (CMAF packaged)                   │
│  ┌─────────────────────────────────────────┐   │
│  │ Object 0: CMAF Header (ftyp, moov)      │   │
│  │ Object 1: CMAF Chunk (moof + mdat)      │   │
│  │ Object 2: CMAF Chunk (moof + mdat)      │   │
│  │ ...                                      │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  Repair Track (this specification)              │
│  ┌─────────────────────────────────────────┐   │
│  │ Object 0: Repair for Block 0            │   │
│  │ Object 1: Repair for Block 1            │   │
│  │ ...                                      │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
└─────────────────────────────────────────────────┘
```

FEC encoding is applied to CMAF chunk payloads, treating each chunk
(or portion thereof) as source symbols.  The FEC layer is agnostic to
the media container format.

### 13.1. CARP Integration

The CMAF-Aware Real-time Protocol (CARP) [I-D.law-moq-carp] uses
the same FEC catalog fields (Section 5.1) and multicast endpoint
format ([I-D.ramadan-moq-multicast] Section 7) as any other MoQ
packaging format.  No CARP-specific FEC signaling is needed — CARP
catalogs include the `fec` and `multicast` objects defined in this
document and [I-D.ramadan-moq-multicast] respectively.

CMAF source objects with FEC enabled MUST carry the FEC Payload ID
and Object Length extension headers (Section 7A.1, 7A.2).  CARP
receivers use these extensions identically to non-CARP CMAF
receivers.

## 14. Security Considerations

### 14.1. FEC Does Not Provide Confidentiality

FEC repair symbols are derived from source data.  If source data is
encrypted, repair symbols will also be encrypted by virtue of
operating on ciphertext.  However, FEC itself provides no
confidentiality guarantees.

### 14.2. Repair Symbol Manipulation

An attacker who can modify repair symbols could cause receivers to
reconstruct incorrect source data.  Integrity protection of repair
tracks is RECOMMENDED, using the same mechanisms as source tracks
(e.g., QUIC packet protection).

### 14.3. Amplification

FEC repair symbols represent additional bandwidth.  Publishers MUST
NOT generate excessive repair symbols that could be used for bandwidth
amplification attacks.  The repair overhead (P/K) SHOULD be limited
to reasonable values (e.g., <= 50%).

### 14.4. Multicast FEC Security

When FEC is used over multicast (SSM/AMT), additional considerations
apply:

1. **No QUIC Protection**: Multicast UDP lacks QUIC's integrity
   guarantees.  Publishers SHOULD use DTLS-SRTP or equivalent for
   payload encryption when confidentiality is required.

2. **Repair Injection**: Attackers could inject malicious repair
   symbols on multicast networks.  Receivers SHOULD verify FEC block
   consistency by checking that decoded data matches expected patterns
   (e.g., valid media container structure).

3. **Source Authentication**: For SSM, receivers verify source address
   matches (S,G) subscription.  For AMT, trust is delegated to the
   AMT relay's authentication mechanisms.

4. **Block Integrity**: Receivers MAY implement block-level checksums
   to detect corrupted recovery.  If decoded source data fails
   integrity checks, receivers SHOULD request retransmission via
   unicast fallback.

## 15. IANA Considerations

### 15.1. MoQ Message Type

This document requests registration of a new MoQ message type in the
"MoQ Message Types" registry:

| Type | Name | Reference |
|------|------|-----------|
| 0x50 | FEC_CONFIG | This document |

Note: The value 0x50 is tentative and subject to coordination with
the MOQ WG chairs to avoid collision with other allocations.
Implementations SHOULD use a value in the experimental range
(0xF000-0xFFFF) until IANA allocation is confirmed.  This
registration is shared with [I-D.ramadan-moq-mmt], which references
this document for the normative FEC_CONFIG definition.

### 15.2. MoQ Extension Header Type

This document requests registration of a new MoQ extension header
type in a separate "MoQ Extension Header Types" registry (note: this
is a different IANA registry than the message type registry in 15.1,
so reuse of the value 0x50 does not create a collision):

| Type | Name | Reference |
|------|------|-----------|
| 0x50 | FEC_CONFIG | This document |

Note: This registration is provisional pending adoption of control
message extension headers in [I-D.ietf-moq-transport] (see
Section 4.2).

### 15.3. FEC Algorithm Registry

This document requests creation of a "MoQ FEC Algorithms" registry
with the following initial values:

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | This document |
| 0x01  | XOR       | This document |
| 0x02  | RaptorQ   | [RFC6330] |
| 0x03  | Reed-Solomon | [RFC5510] |
| 0x04-0xFF | Unassigned | |

New registrations require Specification Required policy.

### 15.4. MoQ Streaming Format Packaging

This document requests registration of a MoQ Streaming Format
packaging value in the registry established by
[I-D.ietf-moq-catalogformat]:

| Packaging | Description | Reference |
|-----------|-------------|-----------|
| "condensed" | Condensed FEC repair format (Section 7.1) | This document |

The "condensed" packaging is used for CMAF repair tracks where the
source packaging (cmaf) does not have native FEC framing.  Each MoQ
object contains one raw repair symbol with SBN/ESI derived from MoQ
transport framing (Section 7.1).

Note: The "mmtp" packaging value is registered by
[I-D.ramadan-moq-mmt].  The "loc" packaging value is registered by
[I-D.ietf-moq-loc].  Repair tracks using those packagings follow
the respective format's native FEC framing.

### 15.5. MoQ Object Extension Header Types

This document requests registration of the following MoQ object
extension header types in the "MoQ Extension Header Types" registry
(note: these are object-level extensions per [I-D.ietf-moq-transport],
distinct from the control message extension in Section 15.2):

| Type | Name | Reference |
|------|------|-----------|
| 0xNN | FEC Payload ID | This document, Section 7A.1 |
| 0xNN | Object Length | This document, Section 7A.2 |
| 0xNN | Auth Tag | This document, Section 7A.3 |

Note: The values 0xNN are placeholders.  Actual values are to be
assigned in coordination with the MOQ WG.  Implementations SHOULD
use values in the experimental range until IANA allocation is
confirmed.

## 16. References

### 16.1. Normative References

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
           Requirement Levels", BCP 14, RFC 2119,
           DOI 10.17487/RFC2119, March 1997.

[RFC5510]  Lacan, J., Roca, V., Peltotalo, J., and S. Peltotalo,
           "Reed-Solomon Forward Error Correction (FEC) Schemes",
           RFC 5510, DOI 10.17487/RFC5510, April 2009.

[RFC6330]  Luby, M., Shokrollahi, A., Watson, M., Stockhammer, T.,
           and L. Minder, "RaptorQ Forward Error Correction Scheme
           for Object Delivery", RFC 6330, DOI 10.17487/RFC6330,
           August 2011.

[RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
           2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174,
           May 2017.

[RFC9000]  Iyengar, J., Ed. and M. Thomson, Ed., "QUIC: A UDP-Based
           Multiplexed and Secure Transport", RFC 9000,
           DOI 10.17487/RFC9000, May 2021.

[I-D.ietf-moq-transport]
           Curley, L., Pugin, K., Nandakumar, S., Vasiliev, V., and
           I. Swett, "Media over QUIC Transport",
           draft-ietf-moq-transport (work in progress).

### 16.2. Informative References

[I-D.ietf-moq-loc]
           Zanaty, M., et al., "Low Overhead Media Container",
           draft-ietf-moq-loc (work in progress).

[I-D.ietf-moq-catalogformat]
           Nandakumar, S., et al., "Common Catalog Format for
           MoQ", draft-ietf-moq-catalogformat (work in progress).

[I-D.ietf-moq-cmsf]
           Law, W., "CMSF: A CMAF Compliant Implementation of
           MOQT Streaming Format", draft-ietf-moq-cmsf
           (work in progress).

[I-D.ramadan-moq-mmt]
           Ramadan, O., "MPEG Media Transport (MMT) Packaging for
           Media over QUIC", draft-ramadan-moq-mmt (work in progress).

[I-D.ramadan-moq-multicast]
           Ramadan, O., "Multicast Delivery and Endpoint Discovery
           for Media over QUIC", draft-ramadan-moq-multicast
           (work in progress).

[RFC4607]  Holbrook, H. and B. Cain, "Source-Specific Multicast for IP",
           RFC 4607, DOI 10.17487/RFC4607, August 2006.

[RFC7450]  Bumgardner, G., "Automatic Multicast Tunneling", RFC 7450,
           DOI 10.17487/RFC7450, February 2015.

[RFC8777]  Holland, J., "DNS Reverse IP Automatic Multicast Tunneling
           (AMT) Discovery", RFC 8777, DOI 10.17487/RFC8777, April 2020.

[RFC9706]  Holland, J., et al., "TreeDN: Tree-Based Content Delivery
           Network (CDN) for Live Streaming to Mass Audiences",
           RFC 9706, December 2024.

[ATSC-A331]
           ATSC, "Signaling, Delivery, Synchronization, and Error
           Protection", A/331:2025, February 2025.

[ARIB-B60]
           ARIB, "MMT-Based Media Transport Scheme in Digital
           Broadcasting Systems", ARIB STD-B60, Version 1.14.

[ISO.23008-1]
           ISO, "Information technology - High efficiency coding and
           media delivery in heterogeneous environments - Part 1:
           MPEG media transport (MMT)", ISO/IEC 23008-1:2023.

[JUNIPER-TREEDN]
           Giuliano, L., "TreeDN - The Fix for Catastrophically
           Successful Live Streaming Events", Juniper TechPost,
           August 2025, https://community.juniper.net/blogs/lenny/
           2025/08/21/introduction-to-TreeDN

## Appendix A. Example Message Flows

### A.1. Publisher-Initiated FEC

```
Publisher                              Subscriber
    |                                       |
    |  SUBSCRIBE(sub_id=1, track=video)     |
    |<--------------------------------------|
    |                                       |
    |  SUBSCRIBE_OK(sub_id=1)               |
    |-------------------------------------->|
    |                                       |
    |  FEC_CONFIG(sub_id=1,                 |
    |    algorithm=RaptorQ,                 |
    |    K=32, P=8, symbol_size=1312,       |
    |    interleave=4, oti=...)             |
    |-------------------------------------->|
    |                                       |
    |  SUBSCRIBE(sub_id=2, track=video/repair)
    |<--------------------------------------|
    |                                       |
    |  SUBSCRIBE_OK(sub_id=2)               |
    |-------------------------------------->|
    |                                       |
    |  [Source objects, group=0, obj=0..31] |
    |-------------------------------------->|
    |                                       |
    |  [Repair objects, group=0, obj=0..7]  |
    |-------------------------------------->|
    |                                       |
```

### A.2. Recovery Example

Subscriber receives source objects 0,1,3,4,5,6,7,8,9 (object 2 lost)
and repair objects 0,1,2.

With K=10, P=3, subscriber has:
- 9 source symbols (ESI 0,1,3,4,5,6,7,8,9)
- 3 repair symbols (ESI 10,11,12)
- Total: 12 symbols >= K=10

RaptorQ decoder can recover the missing source symbol (ESI 2).

### A.3. Per-Object Overhead Comparison

The following table compares per-object FEC signaling overhead across
container formats.  "MoQ Extensions (CMAF+FEC)" uses the extension
headers defined in Section 7A.

| Format | Per-Object Overhead | Components |
|--------|---------------------|------------|
| Condensed (0x00), LOC source | 0 bytes (FEC only) | SBN/ESI derived from MoQ coordinates |
| Condensed (0x00), LOC source + auth | N bytes | Auth Tag extension only (N from catalog) |
| Condensed (0x00), repair | 0 bytes (no auth) | Raw symbol payload, SBN/ESI derived (§7.1.1) |
| Condensed (0x00), repair + auth | N bytes | Raw symbol payload + Auth Tag extension |
| MoQ Extensions (CMAF+FEC) | 9-12 bytes | FEC Payload ID (8) + Object Length (1-4 varint) |
| MoQ Extensions (CMAF+FEC+auth) | 9+N - 12+N bytes | FEC PID + Object Length + Auth Tag |
| MMTP (0x01) | 33 bytes | MMTP Header (12) + FEC Payload ID (9) + OTI (12) |

For LOC source objects, FEC-specific overhead is zero — SBN and ESI
are derived from group_id and object_id (Section 8).  However, if
content authentication is enabled, the Auth Tag extension
(Section 7A.3) adds N bytes per object as a MoQ extension header.

The MoQ Extensions format adds 9-12 bytes per CMAF source object:
8 bytes for the FEC Payload ID extension (SBN+ESI) and 1-4 bytes
for the Object Length extension (QUIC varint encoding — 1 byte for
objects up to 63 bytes, 2 bytes up to 16383, 4 bytes up to ~1GB).
At typical CMAF video rates (30fps, interleave depth 4, ~7.5
objects/second), this is approximately 68-90 bytes/second.

The MMTP format carries full per-packet OTI for self-describing
operation and ATSC 3.0 compatibility.

## Appendix B. Catalog Example

Complete catalog with FEC and multicast configuration, showing all
three packaging types with their corresponding repair tracks:

```json
{
  "version": 1,
  "namespace": "live/broadcast",
  "tracks": [
    {
      "name": "video",
      "packaging": "mmtp",
      "codec": "avc1.64001f",
      "width": 1920,
      "height": 1080,
      "framerate": 30,
      "bitrate": 5000000,
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "symbolSize": 1312,
        "interleaveDepth": 4,
        "repairTrack": "video/repair"
      }
    },
    {
      "name": "video/repair",
      "packaging": "mmtp",
      "priority": 7
    },
    {
      "name": "video.loc",
      "packaging": "loc",
      "codec": "avc1.64001f",
      "width": 1920,
      "height": 1080,
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "symbolSize": 1312,
        "interleaveDepth": 4,
        "repairTrack": "video.loc/repair"
      }
    },
    {
      "name": "video.loc/repair",
      "packaging": "loc",
      "priority": 7
    },
    {
      "name": "video.cmaf",
      "packaging": "cmaf",
      "codec": "avc1.64001f",
      "width": 1920,
      "height": 1080,
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "symbolSize": 1312,
        "interleaveDepth": 4,
        "repairTrack": "video.cmaf/repair"
      }
    },
    {
      "name": "video.cmaf/repair",
      "packaging": "condensed",
      "priority": 7
    },
    {
      "name": "audio",
      "packaging": "mmtp",
      "codec": "mp4a.40.2",
      "sampleRate": 48000,
      "channelCount": 2,
      "bitrate": 128000,
      "fec": {
        "algorithm": "xor",
        "sourceSymbols": 10,
        "repairSymbols": 1,
        "symbolSize": 512,
        "repairTrack": "audio/repair"
      }
    },
    {
      "name": "audio/repair",
      "packaging": "mmtp",
      "priority": 7
    }
  ],
  "multicast": {
    "groupAddress": "232.1.1.50",
    "port": 5000,
    "sourceAddress": "192.168.1.100"
  }
}
```

The `multicast` field uses the simple format defined in
[I-D.ramadan-moq-multicast] Section 7.1.  For multi-endpoint
deployments, the extended format (Section 7.2) with `endpoints`
array is also available.

## Appendix C. MMTP Repair Packaging (Informational)

NOTE: This appendix is informational.  It creates no normative
dependency on [ISO.23008-1].  Implementations that do not interwork
with MMTP sources need not implement this packaging type.

### C.1. Motivation

MMTP repair packaging enables FEC interoperability between MoQ CDN
delivery and MMTP-based broadcast systems including SSM/ASM multicast
[RFC7450] and ATSC 3.0 MMT mode [ATSC-A331].  The repair object
payload is an unmodified MMTP repair packet per [ISO.23008-1],
enabling receivers to apply identical FEC recovery regardless of
delivery path.

In hybrid architectures, the same content may arrive via multiple
paths simultaneously.  The receiver does not need to distinguish
which path delivered a given repair packet — all paths produce
byte-identical MMTP payloads:

| Broadcast Path | MoQ CDN Path | Repair Track Packaging |
|---|---|---|
| ATSC 3.0 MMT (RF) | MMTP passthrough | mmtp |
| SSM multicast + ATSC 3.0 | MMTP passthrough | mmtp |
| MoQ only (no broadcast) | condensed | condensed |
| ATSC 3.0 ROUTE (RF) | CMAF over MoQ | condensed |
| LOC unicast | LOC over MoQ | loc |

The MMTP rows share one repair format across broadcast and CDN paths.
The condensed rows use the lightweight format (Section 7.1) for
pure-MoQ deployments.  LOC repair uses LOC packaging (Section 7.3).

### C.2. Wire Format

When the repair track's packaging is "mmtp", each MoQ repair object
payload contains one complete MMTP repair packet:

```
MoQ Object Payload (packaging: "mmtp") {
  MMTP Header (96),              // 12 bytes, FEC Type=2
  FEC Payload ID {
    Subsequence Start (24),      // SS_Start
    Repair Symbol Block Len (24),// RSB_length
    Repair Symbol ID (24),       // RS_ID
  },
  Object Transmission Info (96), // 12 bytes, RFC 6330 OTI
  Repair Symbol Data (..),       // Symbol Size bytes
}
```

### C.3. Receiver Processing

The receiver extracts FEC parameters by reading at fixed byte
offsets.  This requires one branch on the repair container type
(determined at subscription time from FEC_CONFIG or catalog),
then fixed-offset integer reads and a memcpy of symbol data:

```
offset 0:   MMTP Header (12 bytes) — skip
offset 12:  SS_Start (3 bytes, big-endian) → Source Block Number
offset 15:  RSB_length (3 bytes) — skip
offset 18:  RS_ID (3 bytes) → Encoding Symbol ID derivation
offset 21:  OTI (12 bytes) → RFC 6330 Common + Scheme-Specific OTI
  offset 21: Transfer Length F (5 bytes, 40-bit big-endian)
  offset 26: Reserved (1 byte)
  offset 27: Symbol Size T (2 bytes, big-endian)
  offset 29: Num Source Blocks Z (1 byte)
  offset 30: Num Sub-Blocks N (2 bytes)
  offset 32: Alignment Al (1 byte)
offset 33:  Repair Symbol Data (T bytes) → memcpy to decoder
```

K (source symbols per block) is derived: K = ceil(F / T).

### C.4. Per-Packet OTI

Unlike the condensed format (Section 7) where repair objects
carry only the raw symbol data and OTI is signaled once via
FEC_CONFIG, the MMTP container carries OTI in every
repair packet (12 bytes at offset 21).  This provides:

- Immediate OTI availability when joining mid-stream via any path
- Robustness to FEC_CONFIG loss or late delivery
- Support for mixed-K streams (e.g., video K=32, audio K=4)
  on a shared repair track, where per-packet OTI disambiguates

The overhead is 12 bytes per repair symbol.  At typical rates
(8 repair symbols/block x ~7.5 blocks/second for 30fps video
with depth=4), this is ~720 bytes/second — negligible relative
to video bitrate.

### C.5. Relay Behavior

MoQ relays ingesting MMTP sources (multicast or direct) SHOULD
pass repair packets through without modification:

1. Receive MMTP repair packet (FEC Type=2)
2. Publish as MoQ object on the repair track
3. Set Repair Container to 0x01 in FEC_CONFIG

No header stripping, no re-framing, no re-encoding.

### C.6. Relationship to FEC_CONFIG

When Repair Container is 0x01:

- FEC_CONFIG is OPTIONAL (OTI is per-packet)
- If sent, FEC_CONFIG fields (K, P, T, depth) are informational
  and SHOULD match the per-packet OTI values
- FEC_CONFIG is still useful for repair track discovery and to
  signal FEC availability before the first repair packet arrives

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
