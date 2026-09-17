# Changelog

All notable changes to packet-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `pktspan` — the load-bearing decision. A header is a SPAN over the
  caller's bytes: an offset and a length into a `Bytes` the caller
  owns, not a struct of copied fields. A parse hands back offsets and a
  field read is a read at a known offset, so walking a capture of ten
  million packets allocates nothing per packet where a copying parser
  would allocate three. `narrowed` is the function that keeps Ethernet
  padding out of a payload, and `bytes_of` is the only call in the
  package that copies, named plainly so a reader can find the
  allocation. The cost is stated rather than hidden: a span holds no
  reference to its buffer, so reading one against a different buffer of
  the same size is wrong and undetectable. `PktSpan` is a boxed struct
  and not a `@value` one, because a `@value` struct may not be a
  `Result` payload and every parse here answers one.
- `pkterror` — ten reasons in three groups. `is_truncation` is true of
  `PktTruncated` alone, because a capture taken with a snapshot length
  truncates every full-sized frame and a tool counting that as
  corruption reports a working link as broken. `is_malformed` is the
  group that describes the wire and `is_caller_fault` the group no
  packet can produce. Three groups, three responses: raise the snapshot
  length, investigate the sender, fix the program.
- `pktsum` — `PktChecksum` has FOUR cases, because two of them cannot
  be said with a boolean. A zero UDP checksum over IPv4 means the
  sender computed none (RFC 768) and is `PktChecksumAbsent`, which is
  neither valid nor wrong; a packet whose covered bytes the capture cut
  off, or one taken from a sending host before its network card filled
  the field in, is `PktChecksumUnverifiable`. `is_valid` is documented
  as false for both, so `not is_valid` does not mean corrupt.
  `PktPseudo` holds the pseudo-header's BYTES rather than two addresses,
  which is what lets one type serve the twelve-byte IPv4 form and the
  forty-byte IPv6 one. `udp_transmit_value` carries the rule that a
  computed zero is written as 0xFFFF, and `incremental_update` is RFC
  1624's equation rather than RFC 1141's, which could produce the zero
  that means "no checksum".
- `pktether` — the field after the addresses is an ethertype at 1536
  and above and a LENGTH below 1500 (IEEE 802.3 clause 3.2.6), and
  `length_or_type` answers it raw rather than deciding for the caller.
  VLAN tags stack, so the ethertype is not at a fixed offset: `parse`
  walks the chain and `tag_count` says how many it found, with a bound
  of two. The parse side answers `PktVlan`, a view; the build side
  takes `PktVlanTag`, four fields, and refuses identifier 4095 rather
  than masking it into something else.
- `pktarp` — the one header here with no fixed layout. Four address
  fields sit at offsets computed from two length bytes, which is why
  every accessor for them takes the buffer. `sender_ipv4_span` refuses
  a packet whose protocol type is not IPv4 rather than reading four
  bytes of a sixteen-byte address. `is_probe` and `is_gratuitous` are
  separate, because RFC 5227's probe carries a zero sender address on
  purpose and treating it as an announcement records a claim before it
  is made.
- `pktipv4` — three lengths disagree in ordinary traffic and each
  disagreement means something different: shorter than the span is
  Ethernet padding, longer is truncation, and a header longer than the
  total length is malformed. The payload is narrowed to the total
  length. A total length of ZERO is segmentation offload seen from the
  sending host, which is most of a busy server's own outgoing traffic:
  `parse` refuses it and `parse_offloaded` takes it, two entry points
  rather than a flag. The option list is WALKED — the type byte split
  into its copied flag, class and number, and a length byte of 0 or 1
  refused, because accepting one is an infinite loop from a packet
  anybody can send.
- `pktipv6` — the extension-header chain, which is the whole difficulty
  of IPv6 parsing. The fixed header's next-header byte names the first
  extension header and not the transport, so `next_header` and
  `upper_layer` are two different answers; 59 ends the chain WITHOUT
  being a header. Three length rules, not one: eight-octet units for
  most, a fixed eight for the fragment header whose length byte is
  reserved, and four-octet units minus two for the authentication
  header. The chain is bounded by `extension_limit`, because RFC 8200
  bounds it nowhere and a chain of empty destination-options headers is
  a denial of service. `PktIpv6Ext` carries its `kind` because an
  extension header's own bytes never say what it is. The TLV options
  inside hop-by-hop and destination-options headers are walked, with
  `ipv6_option_action` answering the two bits that say what to do with
  an option a reader does not know.
- `pkttcp` — `segment_length` is the number most callers actually want:
  the payload length plus one for SYN and plus one for FIN, because a
  tool using the payload length alone is one byte out from the
  handshake onwards and reports the first data segment as a
  retransmission. The window is answered UNSCALED and `scaled_window`
  is a pure function, because applying the shift needs the peer's
  announcement and the knowledge that the handshake is over, and this
  package holds no connection state. The options are walked and the six
  a modern connection negotiates are decoded; an unknown option keeps
  its bytes, so traffic using an option newer than this package is
  still readable.
- `pktudp` — eight bytes and two rules. The length field counts the
  header, so a length below eight is malformed and a payload is the
  length less eight. `build_ipv4_without_checksum` is named for what it
  is and takes no pseudo-header at all, because RFC 8200 section 8.1
  forbids the zero checksum over IPv6.
- `pkticmp` — two types and no function taking both, because type 1 is
  an echo reply in one numbering and destination unreachable in the
  other. `icmpv6_is_error` is one bit (RFC 4443 section 2.1) and
  `icmp_is_error` is a list, and treating either as the other gets half
  the type space wrong. The embedded datagram is what makes an error
  attributable to a flow, and its addresses are the original sender's
  and therefore reversed.
- `pktproto` — one enum for the IPv4 protocol field and the IPv6 next
  header, which use the same numbering, with `is_extension_header` as
  the IPv6 walk's stop condition.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the eleven suites reaches
  `not implemented: packet-nv.<module>.<fn>`.
- **Six address accessors are total and cannot report a reason.**
  ipaddr-nv's `Ipv4` and `Ipv6` are `@value` structs, and a `@value`
  struct may be neither a `Result` payload nor an option (`E2015`,
  SPEC section 14.5). So `pktipv4.source`, `pktipv4.destination`,
  `pktipv6.source`, `pktipv6.destination`, `pktarp.sender_ipv4` and
  `pktarp.target_ipv4` answer the zero address where they would
  otherwise answer a `PktError`. The bounds proof lives in the `parse`
  that produced the header, and a `_span` form beside each of them
  reports the reason. The README says so in its own section.
- **`PktPseudo` holds bytes partly for the same rule.** A struct with
  two `Ipv4` fields is `@value` itself and could not come back from
  `pktipv4.pseudo`, which answers a `Result` because computing the
  upper-layer length can fail.
- The IPv6 chain walk stops at an encapsulating security payload
  (protocol 50) and at no-next-header (59), so `upper_layer_span`
  answers `PktNoSuchLayer` for both. Reading inside an ESP needs a key
  and is not this package's work.
- No fragment or stream reassembly, and none planned here: both need
  state with a lifetime the caller owns. The fields a reassembler keys
  on are published.
