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
   the format of repair objects.  FEC-enabled tracks use MMTP packaging
   where each MoQ object carries one MMTP packet; MMTP provides FEC
   metadata natively (FEC Type, Source/Repair FEC Payload ID, per-packet
   OTI).  MMTP fragmentation is the necessary packetization layer for
   datagram and multicast FEC delivery because LOC video objects and
   CMAF chunks are frame-sized and exceed the QUIC datagram MTU.  The
   same MMTP packets work on reliable QUIC streams, QUIC datagrams,
   and multicast UDP.  The mechanism supports RaptorQ (RFC 6330),
   Reed-Solomon (RFC 5510), and a "none" passthrough mode,
   enabling receivers to recover from packet loss without
   retransmission latency.

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
       6.1.  MMTP Repair Wire Format
       6.2.  Receiver Processing
       6.3.  FEC Configuration Delivery
       6.4.  Relay Behavior
       6.5.  Repair Symbol Generation
   7.  FEC Block Structure
   8.  Priority and Congestion
   9.  Security Considerations
   10. IANA Considerations
       10.1. FEC Algorithm Registry
       10.2. MoQ Streaming Format Packaging
   11. References
       11.1. Normative References
       11.2. Informative References
   Appendix A.  Examples
       A.1. Recovery Example
       A.2. Per-Object Overhead Comparison
   Appendix B.  ATSC 3.0 Compatibility
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

MMTP [I-D.bouazizi-mmtp] fragments media into MTU-sized packets,
providing the packetization layer for datagram and multicast FEC
delivery where LOC and CMAF frame-sized objects exceed the QUIC
datagram MTU.

This document defines:

1. A catalog extension for FEC parameter signaling
2. A naming convention for repair tracks
3. The format of repair objects using MMTP packaging
4. FEC block structure including per-group encoding and interleaving

FEC-enabled tracks use MMTP packaging [I-D.ramadan-moq-mmt] where
each MoQ object carries one MMTP packet.  MMTP provides FEC metadata
natively — FEC Type, Source/Repair FEC Payload ID, and per-packet OTI
— eliminating the need for MoQ extension headers.  The same MMTP
packets work identically on reliable QUIC streams, QUIC datagrams,
and multicast UDP, enabling multi-path FEC delivery where receivers
combine symbols from any transport for recovery.

Non-FEC tracks use LOC [I-D.ietf-moq-loc] or CMAF
[I-D.ietf-moq-cmsf] packaging for reliable QUIC stream delivery
without repair overhead.

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

**ssbg_mode0**: Single Symbol per Block Group mode — one MMTP packet
equals one source symbol.  The natural FEC model for MTU-sized MMTP
packets.

## 3. FEC Configuration

### 3.1. FEC Algorithms

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | This document |
| 0x01  | RaptorQ   | [RFC6330] |
| 0x02  | Reed-Solomon (GF2^8) | [RFC5510] |
| 0x03-0xFF | Reserved | IANA |

**RaptorQ (0x01)**: RaptorQ fountain code per [RFC6330].  Can recover
from loss of any symbols as long as K symbols (source or repair) are
received.  Recommended for most applications.

**Reed-Solomon (0x02)**: Reed-Solomon erasure code over GF(2^8) per
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

FEC-enabled tracks use MMTP packaging [I-D.ramadan-moq-mmt].
The format of repair object payloads is determined by the repair
track's packaging type in the catalog:

| Repair Track Packaging | Repair Object Format | Reference |
|------------------------|---------------------|-----------|
| mmtp | MMTP repair packet (FEC Type=2) per [I-D.bouazizi-mmtp] Section 3 | [I-D.ramadan-moq-mmt] |

Both source and repair tracks for FEC-enabled content MUST use
"mmtp" packaging.  Source packets carry a 4-byte Source FEC Payload
ID (SS_ID per [ISO.23008-1] Section C.5.2) appended after the
payload (16 bytes total overhead).  Repair packets carry a 13-byte
Repair FEC Payload ID ([ISO.23008-1] Section C.5.3) for 25 bytes
total overhead.  FEC configuration (OTI) is delivered via the
AL-FEC signaling message (Section 6.3), not per-packet.  This
overhead is amortized over the ~1300-byte symbol payload.

MMTP provides self-describing FEC metadata in every packet,
enabling:

- Multi-path delivery: same packet on QUIC streams, QUIC datagrams,
  and multicast UDP
- Mid-stream join: per-packet OTI eliminates need for prior catalog
- Broadcast interoperability: ATSC 3.0 and ARIB STD-B60 passthrough

Receivers determine the repair format by inspecting the repair
track's packaging field in the catalog.

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

Both the source track and repair track use "mmtp" packaging.
MMTP source packets carry FEC Type=1 with Source FEC Payload ID;
MMTP repair packets carry FEC Type=2 with Repair FEC Payload ID
and per-packet OTI.

For publishers also offering non-FEC LOC delivery of the same
content, both packaging variants are listed as alternative tracks:

```json
{
  "tracks": [
    { "name": "video",     "packaging": "loc",  "altGroup": 1 },
    { "name": "video-fec", "packaging": "mmtp", "altGroup": 1,
      "fec": {
        "algorithm": "raptorq",
        "symbolSize": 1312,
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "repairTrack": "video-fec/repair"
      }
    },
    { "name": "video-fec/repair", "packaging": "mmtp" }
  ]
}
```

Subscribers choose between the non-FEC LOC track (lower overhead,
reliable QUIC only) and the FEC-enabled MMTP track (works on all
transports including datagrams and multicast).

Field definitions for the `fec` object:

**algorithm** (string, REQUIRED if fec present): FEC scheme identifier.
  - "none": No FEC
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

This convention provides a predictable default. The catalog
`repairTrack` field (Section 4.1) is authoritative; publishers MAY
use different names.

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

FEC-enabled tracks use MMTP packaging.  Each MoQ object on a repair
track contains exactly one MMTP repair packet as defined in this
section.

### 6.1. MMTP Repair Wire Format

When the repair track's packaging is "mmtp", each MoQ repair object
payload contains one complete MMTP repair packet per
[I-D.bouazizi-mmtp] Section 3 and [ISO.23008-1] Annex C:

```
MoQ Object Payload (packaging: "mmtp") {
  MMTP Header (96),              // 12 bytes, FEC Type=2
  Repair FEC Payload ID {
    SS_Start (32),               // SS_ID of first source symbol in block
    RSB_length (24),             // Number of repair symbols (P)
    RS_ID (24),                  // Repair symbol index (0-based)
    SSB_length (24),             // Number of source symbols (K)
  },
  Repair Symbol Data (..),       // Symbol Size bytes
}
```

FEC configuration (OTI, K, P, symbol size) is delivered via the
AL-FEC signaling message per [ISO.23008-1] Section C.6 (Table C.3,
message_id=0x0203), NOT embedded in repair packets.  See Section
6.3.

Source packets (FEC Type=1) carry a 4-byte Source FEC Payload ID
appended after the payload per [ISO.23008-1] Section C.5.2:

```
MoQ Object Payload (packaging: "mmtp", FEC source) {
  MMTP Header (96),              // 12 bytes, FEC Type=1
  MPU Sub-Header (64),           // 8 bytes
  MFU DU Header (112),           // 14 bytes (timed, FI=0 or 1 only)
  MFU Payload Data (..),         // Variable (codec sample bytes)
  Source FEC Payload ID (32),    // 4 bytes, SS_ID per ISO C.5.2
}
```

The MMTP header fields relevant to FEC:

- **FEC Type** (2 bits): Set to 2 for repair packets
- **Packet ID** (16 bits): Repair track identifier
- **Timestamp** (32 bits): NTP short format, same epoch as source
- **Packet Sequence Number** (32 bits): Sequential within repair track

The Repair FEC Payload ID (13 bytes) per [ISO.23008-1] Section C.5.3
identifies the source block and repair symbol position:

- **SS_Start** (32 bits): The SS_ID of the first source symbol in the
  associated source symbol block per [ISO.23008-1] Section C.5.3.
  A source symbol belongs to this block if
  `SS_Start <= SS_ID < SS_Start + SSB_length`.
- **RSB_length** (24 bits): Number of repair symbols (P) in the
  associated repair symbol block
- **RS_ID** (24 bits): Repair symbol index, 0-based within the repair
  symbol block.  The Encoding Symbol ID (ESI) for RaptorQ is K + RS_ID.
- **SSB_length** (24 bits): Number of source symbols (K) in the
  associated source symbol block

### 6.2. Receiver Processing

The receiver extracts FEC parameters by reading at fixed byte
offsets:

```
offset 0:   MMTP Header (12 bytes) -- skip
offset 12:  SS_Start (4 bytes, big-endian) -> SS_ID of first source symbol
offset 16:  RSB_length (3 bytes) -> P (repair symbols per block)
offset 19:  RS_ID (3 bytes) -> repair index, ESI = K + RS_ID
offset 22:  SSB_length (3 bytes) -> K (source symbols per block)
offset 25:  Repair Symbol Data (T bytes) -> memcpy to decoder
```

Receiver processing follows a buffer-then-assign model per
[ISO.23008-1] Annex C:

1. **Source packets** (FEC Type=1): The receiver buffers source
   symbols by SS_ID (from the 4-byte Source FEC Payload ID).
   Source symbols are played immediately but not assigned to a
   FEC block until a repair packet defines block boundaries.

2. **Repair packets** (FEC Type=2): SS_Start and SSB_length
   define the block.  A source symbol belongs to this block if
   `SS_Start <= SS_ID < SS_Start + SSB_length`.  Source ESI =
   `SS_ID - SS_Start`.  Repair ESI = `SSB_length + RS_ID`.
   K = SSB_length (may vary per block).

3. On first repair for a block, the receiver drains buffered
   source symbols matching the SS_ID range into the block and
   attempts RaptorQ recovery.

4. **Buffer timeout**: Receivers MUST flush buffered source symbols
   that have not been assigned to any FEC block after 2x the
   expected block duration (e.g., 66ms for D=1 at 30fps), or upon
   receiving the first packet of a new MoQ group.  Flushed symbols
   are delivered to the application without FEC recovery.

OTI (symbol size, transfer length) is obtained from the AL-FEC
signaling message (Section 6.3), not from the repair packet.

### 6.3. FEC Configuration Delivery

FEC configuration (OTI, K, P, symbol size, interleave depth) is
delivered via the AL-FEC signaling message per [ISO.23008-1]
Section C.6 (Table C.3, message_id=0x0203).  The signaling message
is an MMTP packet with packet_type=0x02 carrying:

- **fec_code_id**: FEC algorithm (2=RaptorQ per [RFC6330])
- **length_of_repair_symbol**: Symbol size T in bytes
- **maximum_k_for_repair_flow**: Maximum K per block
- **maximum_p_for_repair_flow**: Maximum P per block
- **private_field**: 12-byte RaptorQ OTI when private_fec_flag=1

Publishers MUST send the AL-FEC signaling message on the signaling
track (packet_id=0) before the first repair packet.  For multicast,
the signaling message is sent periodically to support mid-stream
join.

For MoQ, the AL-FEC parameters are also available in the catalog
`fec` extension (Section 4.1).  Catalog fields are informational
for pre-subscription discovery (e.g., deciding whether to subscribe
to a repair track).  The AL-FEC signaling message is authoritative
for FEC parameters and supports mid-stream changes (e.g., dynamic K
at scene cuts) without requiring catalog updates.

### 6.4. Relay Behavior

Relays SHOULD pass MMTP repair packets through unmodified — no header
stripping, re-framing, or re-encoding.

### 6.5. Repair Symbol Generation

For RaptorQ, repair symbols are generated per [RFC6330] Section 5.3.
The Encoding Symbol ID (ESI) for repair symbols starts at K (the
number of source symbols).

For Reed-Solomon, repair symbols are generated per [RFC5510] using
the Vandermonde matrix construction.

## 7. FEC Block Structure

Each FEC source block is scoped to at most D consecutive MoQ groups,
where D is the interleave depth (default 1).  With D=1, each group
maps to exactly one source block and can be independently decoded.
FEC blocks never span more than D groups.  This bounded-scope model
follows ALC/ROUTE [ATSC-A331] and ensures that receivers can begin
FEC recovery within a bounded window.

### 7.1. Per-Group Encoding

Each MoQ group maps to one or more FEC source blocks.  With
interleave depth D = 1 (default), each group produces exactly one
source block:

```
Group N → Source Block N (K source symbols + P repair symbols)
```

With interleave depth D > 1, symbols from D consecutive groups are
combined into one source block:

```
Groups N*D .. (N+1)*D-1 → Source Block N
```

With D = 1, each MoQ group maps to one FEC source block.  The
block is identified by SS_Start — the SS_ID of the first source
symbol in the group.  Repair objects for a block are published
after the last source object of the group.

With D > 1, symbols from D consecutive groups are combined into
one source block.  Repair objects are published after the last
source object of the final group in the block.

Each block may contain a variable number of source symbols (e.g.,
fewer packets after a scene cut).  SSB_length in the Repair FEC
Payload ID (Section 6.1) carries the actual K for each block.
Receivers MUST NOT assume K is constant across blocks.  Block
membership is determined by SS_Start range matching (Section 6.2),
not by arithmetic on SS_ID.

### 7.2. Sub-Blocks

When a single group produces a large number of source symbols, the
FEC encoder MAY divide the source block into Z sub-blocks per the
RFC 6330 Z parameter.  Sub-block boundaries are signaled in the
AL-FEC signaling message OTI (Section 6.3).

### 7.3. ssbg_mode0: One Packet = One Symbol

In ssbg_mode0 (Single Symbol per Block Group mode 0), each MMTP
packet carries exactly one source symbol.  This is the natural model
for MMTP because:

- MMTP packets are sized to fit in UDP datagrams (~T bytes)
- One MoQ object = one MMTP packet = one source symbol
- No fragmentation or reassembly needed at the FEC layer
- The same packet works on QUIC streams, QUIC datagrams, and
  multicast UDP

Publishers using MMTP packaging SHOULD use ssbg_mode0.  The symbol
size T is chosen to fit within the network MTU minus protocol
overhead (typically T = 1312 bytes for standard 1500-byte MTU).

### 7.4. Interleaving

Interleaving spreads source symbols across time to protect against
burst loss.  With interleave depth D, symbols from D consecutive
media units are grouped into one source block.

| Application | Interleave Depth | Latency Impact |
|-------------|------------------|----------------|
| Interactive (gaming, WebRTC) | 1-4 | 33-133ms at 30fps |
| Low-latency live | 4-8 | 133-267ms at 30fps |
| Broadcast / ATSC 3.0 | 30-60 | ~1-2 seconds at 30fps |

Note: For broadcast interleave depths D=30-60, the publisher
buffers D groups (~1-2 seconds at 30fps) before emitting repair
symbols.  Relay forwarding is stateless — relays forward source
and repair objects transparently with no per-stream FEC state.
The FEC encoding buffer exists only at the publisher.

## 8. Priority and Congestion

Repair tracks SHOULD use lower priority than their corresponding
source tracks so that repair data is dropped first under congestion.
The RECOMMENDED priority offset is +2: if a source track has
subscriber priority S, the corresponding repair track SHOULD use
priority S + 2.

| Track Type | Priority | Derivation |
|------------|----------|------------|
| Control/Signaling | 0-1 | Fixed |
| Source Media | S (2-4) | Set by application |
| Repair Symbols | S + 2 (4-6) | Source priority + 2 |

Under congestion, this separation ensures source media is preserved
while repair overhead is gracefully shed.

When using the Datagram forwarding preference per
[I-D.ietf-moq-transport], QUIC datagrams lack stream-level priority.
MoQ implementations SHOULD implement application-level scheduling
that respects the priority model above when multiplexing source and
repair datagrams.

## 9. Security Considerations

FEC does not provide confidentiality — it operates on whatever data
it receives (plaintext or ciphertext).  Integrity protection of
repair tracks is RECOMMENDED to prevent attackers from causing
incorrect source reconstruction.  The repair overhead (P/K) SHOULD
be limited to reasonable values (e.g., <= 50%) to prevent bandwidth
amplification.

End-to-end content authentication on MoQ unicast MAY be provided by
Secure Objects [SecureObjects] which provides object-level encryption
and authentication.  For multicast delivery, MMTP-native
authentication mechanisms per [I-D.bouazizi-mmtp] apply.

## 10. IANA Considerations

### 10.1. FEC Algorithm Registry

This document requests creation of a "MoQ FEC Algorithms" registry
with the following initial values:

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | This document |
| 0x01  | RaptorQ   | [RFC6330] |
| 0x02  | Reed-Solomon | [RFC5510] |
| 0x03-0xFF | Unassigned | |

New registrations require Specification Required policy.

### 10.2. MoQ Streaming Format Packaging

The "mmtp" packaging value used by FEC repair tracks is registered by
[I-D.ramadan-moq-mmt].  No additional packaging registrations are
needed by this document.

## 11. References

### 11.1. Normative References

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

[I-D.ietf-moq-transport]
           Curley, L., Pugin, K., Nandakumar, S., Vasiliev, V., and
           I. Swett, "Media over QUIC Transport",
           draft-ietf-moq-transport (work in progress).

[I-D.bouazizi-mmtp]
           Bouazizi, I., "MMT Protocol (MMTP)",
           draft-bouazizi-mmtp-01 (work in progress).

### 11.2. Informative References

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
           (Informative.  A freely available description of the MMTP
           wire format is provided by [I-D.bouazizi-mmtp].)

## Appendix A. Examples

### A.1. Recovery Example

Subscriber receives source objects 0,1,3,4,5,6,7,8,9 (object 2 lost)
and repair objects 0,1,2.

With K=10, P=3, subscriber has:
- 9 source symbols (ESI 0,1,3,4,5,6,7,8,9)
- 3 repair symbols (ESI 10,11,12)
- Total: 12 symbols >= K=10

RaptorQ decoder can recover the missing source symbol (ESI 2).

### A.2. Per-Object Overhead

MMTP per-object FEC signaling overhead:

| Packet Type | Per-Object Overhead | Components |
|-------------|---------------------|------------|
| MMTP source | 16 bytes | MMTP Header (12B) + Source FEC PID (4B, SS_ID) |
| MMTP repair | 25 bytes | MMTP Header (12B) + Repair FEC PID (13B) |

Source FEC Payload ID: SS_ID (4B) per [ISO.23008-1] Section C.5.2.
Repair FEC Payload ID: SS_Start (4B) + RSB_length (3B) + RS_ID (3B)
+ SSB_length (3B) = 13B per [ISO.23008-1] Section C.5.3.
OTI is delivered once via AL-FEC signaling (Section 6.3), not
per-packet.

For a typical symbol size of T=1312 bytes, the source overhead is
1.2% (16/1312) and the repair overhead is 1.9% (25/1312).  The
wire format is fully compatible with [ISO.23008-1] Annex C, ATSC
3.0 [ATSC-A331], and ARIB STD-B60 [ARIB-B60].

## Appendix B. ATSC 3.0 Compatibility

This specification is designed for interoperability with ATSC A/331
[ATSC-A331] Application-Layer FEC.

| ATSC A/331 Concept | MoQ FEC Equivalent |
|--------------------|--------------------|
| `<FECParameters>` | Catalog `fec` field (Section 4.1) |
| `<RepairFlow>` | Repair track subscription |
| `fecOTI` (in S-TSID) | OTI derived from catalog fec fields |
| Source TOI range | Group ID range per Section 7 |
| `maximumDelay` | Derived from Interleave Depth x frame duration |
| `overhead` | Computed as (P / K) x 100 |

Publishers ingesting ATSC 3.0 broadcasts SHOULD preserve the original
FEC parameters and pass them through in the catalog fec fields.

For ATSC 3.0 MMT mode ingest, the repair track uses "mmtp"
packaging — repair packets are passed through as-is (Section 6).

### B.1. ATSC 3.0 Gateway FEC Parameter Extraction

An ATSC 3.0 gateway extracts FEC parameters from the broadcast
signaling chain and populates the MoQ catalog `fec` extension:

1. **SLT → S-TSID**: Parse the SLT (Service List Table) to discover
   services, then acquire the S-TSID for each service.

2. **S-TSID → FEC OTI**: Each S-TSID `<RS>` (Repair Session) element
   contains `<FECParameters>` with `fecSchemeID`, `symbolSize`,
   `maxSourceBlockLength` (K), and `maxNumberEncSymbols` (K + P).

3. **Catalog mapping**:

| S-TSID Field | Catalog `fec` Field |
|--------------|---------------------|
| fecSchemeID | algorithm (e.g., scheme 6 → "raptorq") |
| symbolSize | symbolSize |
| maxSourceBlockLength | sourceSymbols |
| maxNumberEncSymbols − maxSourceBlockLength | repairSymbols |
| Repair flow TSI | repairTrack (gateway assigns track name) |

For ATSC 3.0 MMT-mode services, the gateway extracts FEC parameters
from the AL-FEC signaling message (message_id=0x0203) and passes
MMTP packets through to MoQ objects without re-encoding.

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
