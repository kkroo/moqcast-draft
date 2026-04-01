# Forward Error Correction for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: October 2026                                      April 2026

        Forward Error Correction for Media over QUIC (MoQ)
                      draft-ramadan-moq-fec-00

Abstract

   This document specifies a mechanism for transmitting Forward Error
   Correction (FEC) repair data alongside source media in Media over
   QUIC (MoQ) sessions.  It defines signaling for FEC configuration
   via catalog extensions, conventions for repair track naming, and
   the format of repair objects.  The mechanism supports RaptorQ
   (RFC 6330), Reed-Solomon (RFC 5510), and XOR-based FEC schemes,
   enabling receivers to recover from packet loss without
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
   3.  FEC Configuration
       3.1.  FEC Algorithms
       3.2.  RaptorQ Object Transmission Information
       3.3.  Reed-Solomon Parameters
       3.4.  Repair Object Format Selection
   4.  Catalog FEC Extension
       4.1.  Catalog Fields
   5.  Repair Track Convention
       5.1.  Track Naming
       5.2.  Subscription Model
   6.  Repair Object Format
       6.1.  Condensed Repair Format
       6.2.  Repair Symbol Generation
       6.3.  LOC Repair Format
       6.4.  Multicast Wire Format
   7.  MoQ Extension Headers for FEC
       7.1.  FEC Payload ID Extension
       7.2.  Object Length Extension
       7.3.  Auth Tag Extension
   8.  Interleaving
   9.  Priority and Congestion
   10. Multi-Path FEC Delivery
   11. Security Considerations
   12. IANA Considerations
       12.1. FEC Algorithm Registry
       12.2. MoQ Streaming Format Packaging
       12.3. MoQ Object Extension Header Types
   13. References
       13.1. Normative References
       13.2. Informative References
   Appendix A.  Examples
       A.1. Recovery Example
       A.2. Per-Object Overhead Comparison
   Appendix B.  ATSC 3.0 Compatibility
   Appendix C.  MMTP Repair Packaging (Informational)
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

FEC is particularly important for MoQ sessions using the Datagram
forwarding preference per [I-D.ietf-moq-transport], where QUIC
provides no retransmission.  In datagram mode, FEC is the sole
loss recovery mechanism, making repair track subscription
RECOMMENDED for subscribers using unreliable transport.

This document defines:

1. A catalog extension for FEC parameter signaling
2. A naming convention for repair tracks
3. The format of repair objects containing FEC symbols
4. Interleaving strategies to protect against burst loss
5. Compatibility mappings for ATSC 3.0 and ARIB STD-B60 broadcast systems
6. Multi-path delivery where FEC symbols arrive from multiple transport
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

## 3. FEC Configuration

### 3.1. FEC Algorithms

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

### 3.2. RaptorQ Object Transmission Information

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

### 3.3. Reed-Solomon Parameters

When FEC Algorithm is Reed-Solomon (0x03), the OTI field contains:

```
Reed-Solomon OTI {
  Field Size (8),          // Always 8 for GF(2^8)
  Max Source Symbols (16), // Maximum K value
  Max Repair Symbols (16), // Maximum P value
  Reserved (24),
}
```

### 3.4. Repair Object Format Selection

The format of repair object payloads is determined by the repair
track's packaging type in the catalog.  Each packaging type defines
its own repair object layout:

| Repair Track Packaging | Repair Object Format | Reference |
|------------------------|---------------------|-----------|
| condensed | Raw repair symbol, SBN/ESI from MoQ framing (Section 6.1) | This document |
| mmtp | MMTP repair packet (type 0x03) per ISO 23008-1 (Appendix C) | [I-D.ramadan-moq-mmt] |
| loc | LOC object with repair symbol payload (Section 6.3) | [I-D.ietf-moq-loc] |

The repair track's packaging MUST match the source track's
packaging when the source uses a format with native FEC support
(mmtp, loc).  When the source uses cmaf packaging, the repair
track uses condensed packaging because CMAF does not define a
native repair framing.

Receivers determine the repair format by inspecting the repair
track's packaging field in the catalog.  No separate "repair
container" signaling is needed — the packaging field IS the signal.

## 4. Catalog FEC Extension

### 4.1. Catalog Fields

MoQ catalogs [I-D.ietf-moq-catalogformat] MAY include FEC
configuration as an extension field at the track level.  The `fec`
object is a custom extension per [I-D.ietf-moq-catalogformat]
Section 3.1; parsers that do not support FEC MUST ignore it.

```json
{
  "version": 1,
  "streamingFormat": 1,
  "streamingFormatVersion": "0.2",
  "tracks": [
    {
      "name": "video",
      "packaging": "mmtp",
      "selectionParams": {
        "codec": "avc1.64001f",
        "width": 1920,
        "height": 1080,
        "framerate": 30,
        "bitrate": 5000000
      },
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
indicates the repair object format (Section 3.4).  In this example,
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
following the convention in Section 5.1.

Media properties (codec, width, height, framerate, bitrate, samplerate,
channelConfig) are defined in the `selectionParams` object per
[I-D.ietf-moq-catalogformat] Section 3.2.17 and are not repeated here.

## 5. Repair Track Convention

### 5.1. Track Naming

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

This convention provides a predictable default. The catalog `repairTrack` field (Section 4.1) is authoritative; publishers MAY use different names.

### 5.2. Subscription Model

Subscribers SHOULD subscribe to the source track first, then subscribe
to the repair track if:

1. The catalog indicates FEC is available for the source track, AND
2. The subscriber desires FEC protection

Publishers MUST NOT require subscription to repair tracks.  Repair
tracks are optional and subscribers MAY choose to rely solely on
QUIC retransmission.

Subscribers discover FEC availability and repair track names from the
catalog (Section 4.1).

## 6. Repair Object Format

### 6.1. Condensed Repair Format

This section defines the condensed repair format, used when the
repair track's packaging is "condensed" in the catalog.  This is
the default repair format for CMAF source tracks, which do not
have a native FEC framing.

When the repair track uses a different packaging (e.g., "mmtp" per
Appendix C, or "loc" per Section 6.3), repair object payloads
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
from the catalog.  Repair symbols have ESI >= K by
convention: object_id 0 on the repair track corresponds to ESI K,
object_id 1 to ESI K+1, and so on up to object_id P-1 (ESI K+P-1).

### 6.1.1. Repair Track Group Structure

Publishers MUST structure repair tracks so that group_id equals
the Source Block Number (SBN).  Object IDs within each repair group
MUST be sequential starting from 0, where object_id R corresponds
to repair symbol ESI K+R.  This convention enables receivers to
derive SBN and ESI from MoQ framing without per-object metadata,
regardless of the source track's container format (LOC, CMAF, etc.).

This convention holds for all container modes.  While CMAF source
objects require explicit FEC Payload ID extensions (Section 7.1)
because CMAF chunks do not map 1:1 to source symbols, repair
objects ARE 1:1 with symbols (each object = one T-byte repair
symbol), so the derivation is valid.

Publishers MUST publish exactly one repair symbol per MoQ object.
The number of repair symbols per block (P) is known from the catalog.

### 6.2. Repair Symbol Generation

For RaptorQ, repair symbols are generated per [RFC6330] Section 5.3.
The Encoding Symbol ID (ESI) for repair symbols starts at K (the
number of source symbols).

For Reed-Solomon, repair symbols are generated per [RFC5510] using
the Vandermonde matrix construction.

For XOR, a single repair symbol is generated as:

```
repair_symbol = source_symbol[0] XOR source_symbol[1] XOR ... XOR source_symbol[K-1]
```

### 6.3. LOC Repair Format

When the repair track's packaging is "loc", each MoQ object on
the repair track is a standard LOC object containing one repair
symbol as its payload.  SBN and ESI are derived from MoQ transport
framing identically to the condensed format (Section 6.1):

```
SBN = floor(group_id / interleave_depth)
ESI = K + object_id
```

LOC repair objects MAY include an Auth Tag extension header
[I-D.ietf-moq-loc] for per-symbol authentication.  No FEC-specific
extension headers are needed.

### 6.4. Multicast Wire Format

For packaging types without a native multicast framing (LOC, CMAF),
source and repair symbols are delivered over IP multicast (SSM/ASM)
using the condensed multicast packet format defined in this section.
MMTP packets are self-describing and use native MMTP framing on
multicast per [I-D.ramadan-moq-mmt] Section 7.

Each UDP datagram carries one condensed multicast packet containing
either one source symbol (Flags bit 0 = 0) or one repair symbol
(Flags bit 0 = 1).  The format provides track demuxing (Track ID),
FEC metadata (SBN/ESI), random access signaling (RAP flag), optional
timestamps, and optional authentication — all catalog-driven.

```
Condensed Multicast Packet {
  Track ID (16),              // 2 bytes — catalog-assigned per-track ID
  Flags (8),                  // 1 byte — see below
  Source Block Number (24),   // 3 bytes — SBN (wraps at ~16M blocks)
  Encoding Symbol ID (8),    // 1 byte — ESI (max 255 symbols/block)
  Auth Length (8),            // 1 byte — 0=no auth, else tag size
  [Timestamp (32)],           // 4 bytes — NTP-short, if Flags bit 2 = 1
  Payload (..),               // Source symbol or repair symbol
  [Auth Tag (Auth Length)],   // Variable-length (HMAC, ALTA, etc.)
}
```

**Track ID** (16 bits): Identifies the media track for packet
  routing (analogous to MMTP packet_id).  The value corresponds
  to the `id` field assigned to each track in the multicast
  catalog endpoint ([I-D.ramadan-moq-multicast] Section 4.1).
  Publishers MUST use consistent Track ID values across all
  packets for a given track.

**Flags** (8 bits): Bit field:
  - Bit 0: Repair (0=source, 1=repair)
  - Bit 1: RAP (0=delta frame, 1=keyframe/random access point)
  - Bit 2: Timestamp (0=absent, 1=32-bit NTP-short follows header)
  - Bits 3-7: Reserved (MUST be 0, receivers MUST ignore)

**Timestamp** (32 bits, OPTIONAL): Present when Flags bit 2 is set.
  NTP short format (16-bit seconds + 16-bit fractional seconds).
  RECOMMENDED for LOC source packets where in-band timestamps are
  not available.  Not needed for CMAF (timestamps in moof box) or
  repair symbols.  When present, the fixed header is 12 bytes
  instead of 8.

**SBN** (24 bits): Source Block Number.  Wraps at ~16M blocks (~25 days at 30fps depth=4).

**ESI** (8 bits): Encoding Symbol ID.  Source: ESI < K.  Repair: ESI >= K.  Max 255 symbols/block.

**Auth Length** (8 bits): Auth tag length in bytes.  0 = no auth.  Max 255, sufficient for HMAC-SHA256 (32), Ed25519 (64), and ALTA variable-length tags.

**Payload**: Source chunk data or repair symbol (Symbol Size bytes).

**Auth Tag** (Auth Length bytes): Scheme-specific tag (HMAC, ALTA, etc.) as declared in catalog.

Fixed overhead: 8 bytes without timestamp, 12 bytes with timestamp
(vs MMTP 33 bytes, LOC ~14+ bytes).

Limitation: The 8-bit ESI field limits blocks to 255 symbols (e.g., K=247 source + P=8 repair). For broadcast deployments requiring larger blocks (K>247), the MMTP repair format (Appendix C) or LOC repair format (Section 6.3) SHOULD be used instead.

The 24-bit SBN wraps at 16,777,216 blocks. At 60fps with interleave depth 1, this wraps in ~3.2 days. At 30fps with depth 4, ~25 days. Publishers SHOULD reset SBN on stream restart.

Each (sourceAddress, groupAddress, port) multicast tuple MUST
be associated with at most one MoQ namespace.  Publishers
requiring multiple streams MUST use distinct multicast groups
or ports.  See [I-D.ramadan-moq-multicast] Section 4.1.

### 6.4.1. Source Symbol Delivery on Multicast

On multicast, media frames or chunks that exceed T bytes are
fragmented into T-byte source symbols by the FEC encoder before
transmission.  Each source symbol is carried in one UDP datagram
with a condensed header (Flags bit 0 = 0).

Source payloads MAY be smaller than T bytes (e.g., the last
symbol of a frame, or small audio frames).  Receivers determine
the actual payload size from the UDP datagram length:

    payload_size = datagram_length - header_size - Auth_Length

where header_size is 8 bytes (Flags bit 2 = 0) or 12 bytes
(Flags bit 2 = 1).  Receivers pad source payloads to T bytes
with zeros for FEC decoding.

For MMTP-packaged tracks, multicast delivery uses native MMTP
packets (not condensed framing) per [I-D.ramadan-moq-mmt].

On the MoQ unicast path, condensed packaging uses the headerless
format (Section 6.1) with SBN/ESI derived from MoQ transport framing.
The multicast wire format in this section applies only to UDP delivery.

## 7. MoQ Extension Headers for FEC

This section defines MoQ transport extension headers used by FEC-enabled
streams.  These extensions use the MoQ transport extension header
mechanism defined in [I-D.ietf-moq-transport].  Each extension is
self-describing (type + length).  Presence is determined by the
catalog — no per-object flags byte is needed.

These extensions are NOT needed when each MoQ object contains
exactly one source symbol (object payload <= T bytes).  This
applies to:

- MMTP source objects (MMTP-level fragmentation produces
  symbol-sized packets)
- LOC audio source objects (audio frames are typically <= T)
- All repair objects (each object = one T-byte symbol, SBN/ESI
  from framing per Section 6.1)

These extensions ARE REQUIRED when a single MoQ object spans
multiple source symbols (object payload > T bytes).  This
applies to:

- LOC video source objects (video frames are typically >> T)
- CMAF source objects (fMP4 chunks are typically >> T)

The receiver determines which model applies from the catalog:
if the track's media type produces objects larger than symbolSize,
the FEC Payload ID extension (Section 7.1) and Object Length
extension (Section 7.2) MUST be present on source objects.

For CMAF packaging [I-D.ietf-moq-cmsf], the concatenated byte
stream of chunk payloads (moof+mdat) within a source block is
divided into T-byte source symbols.  The FEC Payload ID on each
CMAF source object identifies the first ESI of that chunk, and
Object Length carries the original chunk size for zero-pad
stripping.  The source track uses CMAF packaging; the repair
track uses condensed packaging.

### 7.1. FEC Payload ID Extension

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

### 7.2. Object Length Extension

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
the first byte indicate the field width.  The receiver needs the
original object size to strip zero-padding from the last FEC source
symbol.

### 7.3. Auth Tag Extension

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
packets carry auth via the Auth Length field (Section 6.4).

The Auth Tag provides end-to-end content authentication across
untrusted relays — receivers verify that content originated from
the publisher regardless of relay hops (see Section 11).

## 8. Interleaving

Interleaving spreads source symbols across time to protect against
burst loss.  With interleave depth D, symbols from D consecutive
media units are grouped into one source block.  The SBN is computed
as `floor(Group_ID / D)`.  Repair objects for block N are published
after the last source object of Group `(N+1) * D - 1`.

| Application | Interleave Depth | Latency Impact |
|-------------|------------------|----------------|
| Interactive (gaming, WebRTC) | 1-4 | 33-133ms at 30fps |
| Low-latency live | 4-8 | 133-267ms at 30fps |
| Broadcast / ATSC 3.0 | 30-60 | ~1-2 seconds at 30fps |

## 9. Priority and Congestion

Repair tracks SHOULD use lower priority than source tracks so that
repair data is dropped first under congestion.

Recommended priority assignment:

| Track Type | Priority | Notes |
|------------|----------|-------|
| Control/Signaling | 0-1 | Highest priority |
| Source Media | 2-4 | Protected |
| Repair Symbols | 6-7 | Dropped first |

Under congestion, this separation ensures source media is preserved
while repair overhead is gracefully shed.

## 10. Multi-Path FEC Delivery

The same MoQ objects (source and repair) can be transmitted over
both unicast (QUIC/WebTransport) and multicast (UDP/SSM/AMT) paths
per [I-D.ramadan-moq-multicast].  Receivers on reliable unicast
paths MAY skip FEC decoding; receivers on lossy multicast paths
use FEC for recovery.

When symbols arrive from multiple paths simultaneously, receivers:

1. MUST deduplicate symbols using SBN+ESI as the unique key
2. SHOULD use the same repair container format across all paths
3. MAY combine source from reliable MoQ/QUIC with repair from
   lossy multicast — skipping FEC when all source symbols arrive

## 11. Security Considerations

FEC does not provide confidentiality — it operates on whatever data
it receives (plaintext or ciphertext).  Integrity protection of
repair tracks is RECOMMENDED to prevent attackers from causing
incorrect source reconstruction.  The repair overhead (P/K) SHOULD
be limited to reasonable values (e.g., <= 50%) to prevent bandwidth
amplification.

When FEC is used over multicast (SSM/AMT), multicast UDP lacks
QUIC's integrity guarantees.  Receivers SHOULD verify decoded media
structure to detect repair injection attacks.  Additional multicast
security considerations (source authentication, replay protection,
AMT relay trust) are defined in [I-D.ramadan-moq-multicast]
Section 6.

## 12. IANA Considerations

### 12.1. FEC Algorithm Registry

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

### 12.2. MoQ Streaming Format Packaging

This document requests registration of a MoQ Streaming Format
packaging value in the registry established by
[I-D.ietf-moq-catalogformat]:

| Packaging | Description | Reference |
|-----------|-------------|-----------|
| "condensed" | Condensed FEC repair format (Section 6.1) | This document |

The "condensed" packaging is used for:
- Repair tracks paired with CMAF or LOC source tracks, where the
  source packaging does not have native FEC repair framing.  Each
  MoQ object contains one raw repair symbol with SBN/ESI derived
  from MoQ transport framing (Section 6.1).
- Multicast delivery of LOC and CMAF source and repair symbols,
  where each UDP datagram carries one condensed multicast packet
  (Section 6.4).

Note: The "mmtp" packaging value is registered by
[I-D.ramadan-moq-mmt].  The "loc" packaging value is registered by
[I-D.ietf-moq-loc].  Repair tracks using those packagings follow
the respective format's native FEC framing.

### 12.3. MoQ Object Extension Header Types

This document requests registration of the following MoQ object
extension header types in the "MoQ Extension Header Types" registry
(note: these are object-level extensions per [I-D.ietf-moq-transport]):

| Type | Name | Reference |
|------|------|-----------|
| 0xNN | FEC Payload ID | This document, Section 7.1 |
| 0xNN | Object Length | This document, Section 7.2 |
| 0xNN | Auth Tag | This document, Section 7.3 |

Note: The values 0xNN are placeholders.  Actual values are to be
assigned in coordination with the MOQ WG.  Implementations SHOULD
use values in the experimental range until IANA allocation is
confirmed.

## 13. References

### 13.1. Normative References

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

### 13.2. Informative References

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

## Appendix A. Examples

### A.1. Recovery Example

Subscriber receives source objects 0,1,3,4,5,6,7,8,9 (object 2 lost)
and repair objects 0,1,2.

With K=10, P=3, subscriber has:
- 9 source symbols (ESI 0,1,3,4,5,6,7,8,9)
- 3 repair symbols (ESI 10,11,12)
- Total: 12 symbols >= K=10

RaptorQ decoder can recover the missing source symbol (ESI 2).

### A.2. Per-Object Overhead Comparison

The following table compares per-object FEC signaling overhead across
container formats.  "MoQ Extensions (CMAF+FEC)" uses the extension
headers defined in Section 7.

| Format | Per-Object Overhead | Components |
|--------|---------------------|------------|
| LOC audio source | 0 bytes | Audio frames <= T; SBN/ESI derived from MoQ coordinates |
| LOC video source | 9-12 bytes | Video frames >> T; FEC Payload ID (8) + Object Length (1-4) |
| LOC source + auth | +N bytes | Add Auth Tag extension (N from catalog) |
| Condensed (0x00), repair | 0 bytes (no auth) | Raw symbol payload, SBN/ESI derived (Section 6.1.1) |
| Condensed (0x00), repair + auth | N bytes | Raw symbol payload + Auth Tag extension |
| MoQ Extensions (CMAF+FEC) | 9-12 bytes | FEC Payload ID (8) + Object Length (1-4 varint) |
| MoQ Extensions (CMAF+FEC+auth) | 9+N - 12+N bytes | FEC PID + Object Length + Auth Tag |
| MMTP (0x01) | 33 bytes | MMTP Header (12) + FEC Payload ID (9) + OTI (12) |

LOC audio and condensed repair formats derive SBN/ESI from MoQ
coordinates (zero FEC overhead).  LOC video and CMAF source objects
require explicit FEC Payload ID and Object Length extensions (9-12
bytes) because media objects exceed the symbol size T.  MMTP carries
full per-packet OTI (33 bytes) for self-describing operation and
ATSC 3.0 compatibility.

## Appendix B. ATSC 3.0 Compatibility

This specification is designed for interoperability with ATSC A/331
[ATSC-A331] Application-Layer FEC.

| ATSC A/331 Concept | MoQ FEC Equivalent |
|--------------------|--------------------|
| `<FECParameters>` | Catalog `fec` field (Section 4.1) |
| `<RepairFlow>` | Repair track subscription |
| `fecOTI` (in S-TSID) | OTI derived from catalog fec fields |
| Source TOI range | Group ID range per Section 8 |
| `maximumDelay` | Derived from Interleave Depth x frame duration |
| `overhead` | Computed as (P / K) x 100 |

Publishers ingesting ATSC 3.0 broadcasts SHOULD preserve the original
FEC parameters and pass them through in the catalog fec fields.

For ATSC 3.0 MMT mode ingest, the repair track uses "mmtp"
packaging — repair packets are passed through as-is (Appendix C).

## Appendix C. MMTP Repair Packaging (Informational)

NOTE: This appendix is informational.  It creates no normative
dependency on [ISO.23008-1].  Implementations that do not interwork
with MMTP sources need not implement this packaging type.

### C.1. Motivation

MMTP repair packaging enables FEC interoperability between MoQ CDN
delivery and MMTP-based broadcast systems (SSM/ASM multicast, ATSC 3.0
MMT mode).  Repair payloads are unmodified MMTP packets per
[ISO.23008-1], so receivers apply identical recovery regardless of path.

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
(determined at subscription time from the catalog), then fixed-offset
integer reads and a memcpy of symbol data:

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

OTI is carried in every MMTP repair packet at offset 21 (12 bytes),
enabling mid-stream join without catalog and supporting mixed-K
streams on a shared repair track.

### C.5. Relay Behavior

Relays SHOULD pass MMTP repair packets through unmodified — no header
stripping, re-framing, or re-encoding.

### C.6. Relationship to Catalog

Catalog fec fields are OPTIONAL when packaging is "mmtp" (OTI is
per-packet).  If present, they SHOULD match per-packet OTI values
and are useful for repair track discovery before the first packet.

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
