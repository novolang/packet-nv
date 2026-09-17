# packet-nv

This package reads and writes the protocol headers a network packet is
made of: Ethernet with its VLAN tags, ARP
([RFC 826](https://www.rfc-editor.org/rfc/rfc826)), IPv4
([RFC 791](https://www.rfc-editor.org/rfc/rfc791)), IPv6
([RFC 8200](https://www.rfc-editor.org/rfc/rfc8200)), ICMP
([RFC 792](https://www.rfc-editor.org/rfc/rfc792)), ICMPv6
([RFC 4443](https://www.rfc-editor.org/rfc/rfc4443)), TCP
([RFC 9293](https://www.rfc-editor.org/rfc/rfc9293)) and UDP
([RFC 768](https://www.rfc-editor.org/rfc/rfc768)). It is the analysis
half of a protocol stack: it decodes packets and builds them, and it
sends nothing. The Rust crate `pnet`'s packet module and Python's
`scapy` layers are the references.

A header is read as a **span** — an offset and a length into a buffer
the caller already holds — so walking a capture file allocates nothing
per packet.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a packet is

A packet on an Ethernet network is a series of headers, each naming
what comes after it. A **header** is a fixed number of bytes with named
fields at known offsets, and most of the fields are numbers written
most significant byte first, which RFC 791 appendix B calls **network
byte order**.

An **Ethernet frame** begins with a six-byte destination address, a
six-byte source address, and two bytes that are either an **ethertype**
naming what the frame carries, or a length. Between the source address
and that field a sender may insert one or two four-byte **VLAN tags**,
each holding a twelve-bit VLAN identifier.

An **IPv4 datagram** has a twenty-byte header that may be followed by
**options**, a list of variable-length items in the space between the
fixed header and the header length the header itself gives. An **IPv6
datagram** has a forty-byte header with no options, and may be followed
by a chain of **extension headers**, each naming the next; the chain
ends at the upper-layer protocol.

**TCP** and **UDP** are the two transports. A TCP header has options of
its own, negotiated in the first segment of each direction.

A **checksum** is a sixteen-bit number computed over some of the bytes,
defined once in [RFC 1071](https://www.rfc-editor.org/rfc/rfc1071) and
used by all of these protocols. The TCP, UDP and ICMPv6 checksums are
computed over a **pseudo-header** as well — the source and destination
addresses, the protocol and the length, which are never transmitted and
exist so that a datagram delivered to the wrong host fails its check.

A **capture** is a recording of packets, usually taken with a **snapshot
length** that stores only the first so many bytes of each. Packets
longer than it arrive with their tails missing.

## Install

```
novo pkg add packet-nv
```

## Example

```novo
use std.bytes
use pktspan
use pktether
use pktipv4
use pkttcp

fn main() [io]
    // The bytes of one captured frame. A real program gets these from a
    // capture file or from a socket.
    let frame = bytes.zeros(60)

    // Read the Ethernet header. Nothing is copied: `e` holds offsets.
    match pktether.parse(frame, pktspan.whole(frame))
        Err(x) => println("not a frame: ${x.message()}")
        Ok(e)  =>
            // Read the IPv4 header out of the frame's payload.
            match pktipv4.parse(frame, pktether.payload(e))
                Err(x) => println("not IPv4: ${x.message()}")
                Ok(ip) =>
                    // The payload span stops at the IP total length, so
                    // the frame's padding is not part of it.
                    match pkttcp.parse(frame, pktipv4.payload(ip))
                        Err(x) => println("not TCP: ${x.message()}")
                        Ok(t)  =>
                            // Read one field out of the buffer.
                            match pkttcp.destination_port(frame, t)
                                Err(x) => println(x.message())
                                Ok(p)  => println("to port ${p}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: packet-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `pktspan` | The span: an offset and a length into a buffer, the readers over one, and the one call that copies. |
| `pkterror` | Every refusal, and whether each is the recording's fault, the packet's, or the caller's. |
| `pktproto` | The protocol numbers IPv4 and IPv6 share, and which of them name extension headers. |
| `pktsum` | The checksum arithmetic, the two pseudo-headers, and a verdict with four cases. |
| `pktether` | Ethernet frames, the VLAN tag chain, and the ethertype at the end of it. |
| `pktarp` | ARP packets, whose four address fields are at offsets computed from two length bytes. |
| `pktipv4` | The IPv4 header and its option list. |
| `pktipv6` | The IPv6 header, the extension-header chain, and the options inside two of those headers. |
| `pkticmp` | ICMP for IPv4 and ICMPv6, which share a header shape and no numbering. |
| `pkttcp` | The TCP header, its options, and how far a segment advances the sequence number. |
| `pktudp` | The UDP header. |

## How to choose an entry point

**Each layer has a `parse` that takes a buffer and a span** and answers
a header made of spans. A walk starts with `pktspan.whole(buffer)` and
passes each layer's payload span to the next layer's `parse`.

**`pktipv4.parse_offloaded` and `pktipv6.parse_offloaded` accept a zero
length field.** Use them for a capture taken on the host that sent the
packets, where the network card has not yet written the length. Use
`parse` for a capture taken from a tap or a mirror port, where a zero
length is a malformed packet.

**`pktipv4.parse_verified` refuses a header whose checksum is wrong.**
Use it when a header or nothing is wanted. Use `pktipv4.parse` with
`pktipv4.verify_checksum` when the four checksum outcomes should be
counted separately.

**`pktipv6.upper_layer` answers what an IPv6 datagram carries.**
`pktipv6.next_header` answers what the fixed header names, which is the
first extension header when there is one.

**Each layer has a `build` that takes values and answers `Bytes`.** The
parse side is a view over bytes that exist; the build side has no bytes
yet, so the two do not share a type.

**`pktspan.bytes_of` copies a span's bytes into a buffer of its own.**
Every other function in this package reads in place.

## The rules a user needs

1. **A span is only meaningful with the buffer it was cut from.** It
   holds no reference to that buffer, does not keep it alive, and does
   not notice if it is modified. Reading a span against a different
   buffer of the same size gives a wrong answer and no error.
   `pktspan.fits` catches only a span that does not fit.
2. **The IP total length, and not the buffer, says where a datagram
   ends.** An Ethernet frame is padded to sixty bytes (IEEE 802.3
   section 4.2.3.3), so a short datagram arrives with padding behind it.
   `pktipv4.parse` narrows the payload span to the total length.
3. **A total length or payload length of zero means the packet was
   captured on the host that sent it**, before the network card wrote
   the field, or that it is an IPv6 jumbogram
   ([RFC 2675](https://www.rfc-editor.org/rfc/rfc2675)). It does not
   mean an empty datagram. `parse` refuses it; `parse_offloaded` accepts
   it.
4. **The IPv4 header length field counts four-byte words** (RFC 791
   section 3.1) and the TCP data offset field does too (RFC 9293 section
   3.1). `pktipv4.header_length` and `pkttcp.data_offset` answer bytes.
5. **The IPv4 fragment offset field counts eight-byte units** (RFC 791
   section 3.1), as the IPv6 one does (RFC 8200 section 4.5).
   `pktipv4.fragment_offset` and `pktipv6.fragment_offset` answer bytes.
6. **A datagram is a fragment when the more-fragments flag is set OR the
   fragment offset is non-zero.** The last fragment has the flag clear.
   `pktipv4.is_fragment` tests both.
7. **Only the fragment at offset zero carries the transport header.**
   `pktipv4.has_transport_header` answers which. Parsing a TCP header
   out of any other fragment reads payload bytes as ports.
8. **The IPv4 option type byte is three fields** (RFC 791 section 3.1):
   a copied flag in bit 7, a class in bits 6 and 5, and a number in bits
   4 to 0. Record route is number 7 and arrives as 0x87 when the copied
   flag is set. `pktipv4.option_number` is the field to compare.
9. **An option length byte of 0 or 1 is refused**, in IPv4 and in TCP. A
   walk that accepted one would not advance. `PktBadOption` is the
   reason.
10. **The IPv6 next-header field of the fixed header names the first
    extension header**, not the transport protocol.
    `pktipv6.upper_layer` walks the chain; `pktipv6.next_header` does
    not.
11. **Next header 59 means there is nothing after this header** (RFC
    8200 section 4.7). The following bytes are not a header of any kind.
12. **Three extension headers measure their length differently.** Most
    count eight-octet units not including the first eight; the fragment
    header is fixed at eight bytes and its length byte is reserved (RFC
    8200 section 4.5); the authentication header counts four-octet units
    minus two (RFC 4302 section 2.2). `pktipv6.ext_length` answers bytes
    under whichever rule applies.
13. **The extension chain has a limit here.** RFC 8200 sets none, and a
    chain of empty destination-options headers can be made as long as a
    datagram allows. `pktipv6.extension_limit` is the bound and
    `PktExtChainTooLong` is what a longer chain gets.
14. **The top two bits of an IPv6 option type say what to do with an
    option a reader does not recognise** (RFC 8200 section 4.2): skip,
    discard, discard and report, or discard and report unless the
    destination was multicast. `pktipv6.ipv6_option_action` answers
    which.
15. **A TCP segment advances the sequence number by its payload length
    plus one for SYN and plus one for FIN** (RFC 9293 section 3.4).
    `pkttcp.segment_length` answers that number.
16. **The TCP window field is not scaled in the header.** The shift is
    announced in each direction's SYN, applies to the other direction's
    windows, and applies to no segment of the handshake (RFC 7323
    section 2.2). `pkttcp.scaled_window` applies a shift the caller
    supplies.
17. **A SACK block is a half-open range** (RFC 2018 section 3): `left`
    is the first sequence number and `right` is one past the last.
18. **The UDP length field counts the eight-byte header too** (RFC 768).
    A length below eight is refused.
19. **A zero UDP checksum over IPv4 means the sender computed none**
    (RFC 768) and is `PktChecksumAbsent`, which is neither valid nor
    wrong. Over IPv6 the checksum is required (RFC 8200 section 8.1), so
    a zero there is wrong.
20. **`pktsum.is_valid` is false for `PktChecksumAbsent` and
    `PktChecksumUnverifiable`.** Counting `not is_valid` as corruption
    counts every zero-checksum UDP datagram and every truncated capture
    as corrupt.
21. **A UDP checksum that computes to zero is transmitted as 0xFFFF**
    (RFC 768), because zero in the field means "no checksum".
    `pktsum.udp_transmit_value` applies the rule.
22. **The pseudo-header's length is the upper-layer packet length**, not
    the IP length. For IPv4 it is the total length less the header
    length; for IPv6 it is the payload length less every extension
    header. `pktipv4.pseudo` and `pktipv6.pseudo` compute it.
23. **The ICMPv6 checksum is mandatory and covers the pseudo-header**
    (RFC 4443 section 2.3). The ICMP for IPv4 checksum covers the
    message alone (RFC 792). `pkticmp.build_v6` therefore takes a
    pseudo-header and `pkticmp.build` does not.
24. **An ICMPv6 type below 128 is an error message and one of 128 and
    above is informational** (RFC 4443 section 2.1). ICMP for IPv4 has
    no such rule; its error types are a list. `pkticmp.icmpv6_is_error`
    and `pkticmp.icmp_is_error` are two different tests.
25. **The addresses inside an ICMP error are the original sender's**, so
    they are reversed with respect to the ICMP message's own.
    `pkticmp.icmp_embedded_span` is the datagram that caused the error.
26. **An ARP packet's address fields are at offsets computed from its
    two length bytes** (RFC 826). `pktarp.sender_ipv4_span` refuses a
    packet whose protocol type is not IPv4.
27. **`PktTruncated` means the capture did not keep the bytes**, not
    that the packet is malformed. `pkterror.is_truncation` and
    `pkterror.is_malformed` separate them.
28. **The field after an Ethernet frame's addresses is an ethertype at
    1536 and above and a length at 1500 and below** (IEEE 802.3 clause
    3.2.6). `pktether.is_ethertype` is the test.
29. **VLAN tags stack**, so the ethertype is not at a fixed offset.
    `pktether.tag_count` says how many there are and
    `pktether.ethertype` answers the value at the end of the chain. Tag
    0 is the outermost, which under IEEE 802.1ad is the service
    provider's.
30. **VLAN identifier 4095 is reserved** and is refused by
    `pktether.vlan_tag`.

## Address accessors are total

`pktipv4.source`, `pktipv4.destination`, `pktipv6.source`,
`pktipv6.destination`, `pktarp.sender_ipv4` and `pktarp.target_ipv4`
answer an address and never a `Result`. ipaddr-nv's `Ipv4` and `Ipv6`
are `@value` structs, which may be neither a `Result` payload nor an
option (SPEC section 14.5), so no reason can come back from them.

Each of them has already been made safe by the `parse` that produced
the header, which proved the header fits the buffer. Called with a
different buffer, or on an ARP packet that is not Ethernet over IPv4,
they answer the zero address. `pktipv4.source_span`,
`pktipv4.destination_span`, `pktipv6.source_span`,
`pktipv6.destination_span`, `pktarp.sender_ipv4_span` and
`pktarp.target_ipv4_span` are the forms that report a reason.

## What is not included

- **Sockets.** Every function here takes bytes and returns bytes or
  values. Nothing opens a socket, reads a clock or touches a file.
- **IP fragment reassembly.** Joining fragments needs a table keyed on
  the source, the destination, the protocol and the identification, a
  buffer per datagram, a timeout, and a policy for overlapping
  fragments. All of it is state with a lifetime, and the program that
  owns the packets owns that state. The fields to key on are here:
  `pktipv4.identification`, `pktipv4.fragment_offset`,
  `pktipv4.more_fragments`, and their counterparts on the IPv6 fragment
  extension header.
- **TCP stream reassembly.** The same reason, with more of it: a table
  of connections, a reorder buffer per direction, a policy for
  overlapping segments, and a timeout. `pkttcp.segment_length` and
  `pkttcp.sequence` are what a reassembler is built on.
- **A routing table, an ARP cache or a neighbour cache.** Each is state
  with an eviction rule and a conflict policy that belongs to the
  program holding the interface.
- **Reading capture files.** `pcapfile-nv` reads the pcap and pcapng
  formats and yields the packets this package parses.
- **Address parsing and formatting.** `ipaddr-nv` does both, and this
  package answers its types.
- **Protocols above the transport.** DNS, HTTP and TLS each have their
  own package.
- **Encrypted payloads.** The IPv6 extension walk stops at an
  encapsulating security payload, because what follows it cannot be read
  without the key.

## Related packages

- [ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv) holds IP
  addresses and CIDR networks as value types. This package depends on it
  and answers its `Ipv4` and `Ipv6`, so an address read out of a header
  can be handed to `Net4.contains` without going through bytes.
- [pcapfile-nv](https://novo-lang.org/packages/pcapfile-nv) reads and
  writes capture files. Take it to get the bytes; take this package to
  read them.
- [cidr-nv](https://novo-lang.org/packages/cidr-nv) matches an address
  against a set of prefixes. Take it for a filter over many networks.
- [dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv) decodes
  what a UDP datagram to port 53 carries.
- [bitstream-nv](https://novo-lang.org/packages/bitstream-nv) reads
  fields that are not whole bytes. Take it for a format whose fields
  cross byte boundaries arbitrarily; the bit fields here are few and are
  read by named accessors.

## Tests

```bash
novo test tests/pktspan_tests.nv    # the span rules every module rests on
novo test tests/pkterror_tests.nv   # truncated, malformed, and the caller's own fault
novo test tests/pktether_tests.nv   # the ethertype boundary and the tag chain
novo test tests/pktarp_tests.nv     # computed offsets, probes and announcements
novo test tests/pktipv4_tests.nv    # the three lengths, fragments, and the option walk
novo test tests/pktipv6_tests.nv    # the extension chain and its three length rules
novo test tests/pkttcp_tests.nv     # segment length, the window, and the options
novo test tests/pktudp_tests.nv     # the length field and the optional checksum
novo test tests/pktsum_tests.nv     # four verdicts and the two pseudo-headers
novo test tests/pkticmp_tests.nv    # two registries and one header shape
novo test tests/pktcover_tests.nv   # every public function is reached
```

The normative sources are RFC 791 for IPv4 and its options, RFC 8200 for
IPv6 and its extension headers, RFC 9293 for TCP, RFC 7323 and RFC 2018
for the TCP options, RFC 768 for UDP, RFC 792 and RFC 4443 for the two
ICMPs, RFC 826 for ARP, RFC 1071 for the checksum, RFC 1624 for updating
one, and IEEE 802.3 and 802.1Q for Ethernet and its tags. The fixtures
are headers built by hand from those documents' field tables. The
reference implementations are the Rust crate `pnet`'s packet module,
whose per-packet accessors this package's spans replace, and Python's
`scapy`, whose layer list is the coverage target.

The suite asserts that Ethernet padding does not become a UDP payload,
that a zero total length is refused by one entry point and accepted by
another, that a fragment offset comes back in bytes, that an option
length of 1 is refused rather than looped on, that an IPv6 fragment
header is eight bytes whatever its reserved byte says, that a SYN
advances the sequence space by one with no payload, that a zero UDP
checksum over IPv4 is `PktChecksumAbsent` and over IPv6 is not, and that
an ARP packet naming a protocol other than IPv4 is refused rather than
read four bytes at a time.

The tests compile today and fail at run, each on the
`not implemented: packet-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| Every public struct and enum | the types are declared |
| `pktspan.span`, `.whole`, `.end`, `.fits`, `.is_empty`, `.same` | no |
| `pktspan.sub`, `.after`, `.narrowed` | no |
| `pktspan.u8_at`, `.u16_be_at`, `.u32_be_at`, `.bytes_of` | no |
| `pkterror.is_truncation`, `.is_malformed`, `.is_caller_fault`, `.code`, `PktError.message` | no |
| `pktproto.protocol_of`, `.protocol_number`, `.protocol_name`, `.is_extension_header`, `.needs_pseudo_header` | no |
| `pktsum.pseudo_ipv4`, `.pseudo_ipv6`, `.pseudo_bytes`, `.pseudo_length` | no |
| `pktsum.ones_complement_sum`, `.ones_complement_fold`, `.checksum_of`, `.checksum_with_pseudo` | no |
| `pktsum.compare`, `.compare_udp_ipv4`, `.unverifiable`, `.is_valid`, `.checksum_reason`, `.checksum_code` | no |
| `pktsum.udp_transmit_value`, `.zero_means_absent`, `.incremental_update` | no |
| `pktether.parse`, `.destination_span`, `.source_span`, `.payload`, `.tag_count` | no |
| `pktether.length_or_type`, `.is_ethertype`, `.ethertype`, `.ethertype_name` | no |
| `pktether.is_multicast`, `.is_broadcast`, `.address_text` | no |
| `pktether.vlan`, `.vlan_tpid`, `.vlan_id`, `.vlan_priority`, `.vlan_drop_eligible`, `.vlan_tag_of`, `.vlan_tag` | no |
| `pktether.build`, `.build_tagged` | no |
| `pktarp.parse`, `.hardware_type`, `.protocol_type`, `.hardware_length`, `.protocol_length` | no |
| `pktarp.operation`, `.operation_code`, `.operation_name` | no |
| `pktarp.sender_hardware_span`, `.sender_protocol_span`, `.target_hardware_span`, `.target_protocol_span` | no |
| `pktarp.sender_ipv4_span`, `.target_ipv4_span`, `.sender_ipv4`, `.target_ipv4` | no |
| `pktarp.is_gratuitous`, `.is_probe`, `.build` | no |
| `pktipv4.parse`, `.parse_offloaded`, `.parse_verified` | no |
| `pktipv4.version`, `.header_length`, `.dscp`, `.ecn`, `.total_length`, `.identification`, `.ttl`, `.protocol`, `.checksum` | no |
| `pktipv4.dont_fragment`, `.more_fragments`, `.fragment_offset`, `.is_fragment`, `.has_transport_header` | no |
| `pktipv4.source`, `.destination`, `.source_span`, `.destination_span`, `.payload` | no |
| `pktipv4.options`, `.option_kind`, `.option_number`, `.option_class`, `.option_is_copied`, `.option_length`, `.option_data_span`, `.option_name` | no |
| `pktipv4.verify_checksum`, `.set_checksum`, `.pseudo`, `.pad_options`, `.build` | no |
| `pktipv6.parse`, `.parse_offloaded`, `.version`, `.traffic_class`, `.dscp`, `.ecn`, `.flow_label` | no |
| `pktipv6.payload_length`, `.next_header`, `.hop_limit`, `.source`, `.destination`, `.source_span`, `.destination_span`, `.payload` | no |
| `pktipv6.extension_limit`, `.extensions`, `.upper_layer`, `.upper_layer_span` | no |
| `pktipv6.ext_kind`, `.ext_next_header`, `.ext_length`, `.ext_data_span`, `.ext_options` | no |
| `pktipv6.ipv6_option_type`, `.ipv6_option_action`, `.ipv6_option_length`, `.ipv6_option_data_span` | no |
| `pktipv6.fragment_offset`, `.fragment_more`, `.fragment_identification`, `.routing_type`, `.routing_segments_left` | no |
| `pktipv6.pseudo`, `.build` | no |
| `pkticmp.parse`, `.icmp_type`, `.icmp_code`, `.icmp_checksum`, `.icmp_rest`, `.icmp_identifier`, `.icmp_sequence` | no |
| `pkticmp.icmp_payload_span`, `.icmp_is_error`, `.icmp_type_name`, `.icmp_embedded_span`, `.verify_checksum`, `.build` | no |
| `pkticmp.parse_v6`, `.icmpv6_type`, `.icmpv6_code`, `.icmpv6_checksum`, `.icmpv6_rest`, `.icmpv6_payload_span` | no |
| `pkticmp.icmpv6_is_error`, `.icmpv6_type_name`, `.icmpv6_embedded_span`, `.verify_checksum_v6`, `.build_v6` | no |
| `pkttcp.parse`, `.source_port`, `.destination_port`, `.sequence`, `.acknowledgement`, `.data_offset` | no |
| `pkttcp.flags`, `.has`, `.flag_names`, `.window`, `.scaled_window`, `.checksum`, `.urgent_pointer`, `.payload`, `.segment_length` | no |
| `pkttcp.options`, `.tcp_option_kind`, `.tcp_option_length`, `.tcp_option_data_span`, `.tcp_option_name`, `.is_known_option` | no |
| `pkttcp.maximum_segment_size`, `.window_scale`, `.sack_permitted`, `.sack_blocks`, `.timestamp_value`, `.timestamp_echo` | no |
| `pkttcp.verify_checksum`, `.pad_options`, `.build` | no |
| `pktudp.parse`, `.source_port`, `.destination_port`, `.length`, `.checksum`, `.payload` | no |
| `pktudp.verify_checksum`, `.build`, `.build_ipv4_without_checksum` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
