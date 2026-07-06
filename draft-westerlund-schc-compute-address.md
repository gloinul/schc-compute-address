---
title: "SCHC Compute-Address Compression/Decompression Actions for Dynamically Assigned IP Addresses"
abbrev: "SCHC Compute-Address CDAs"
docname: draft-westerlund-schc-compute-address-latest
category: std
ipr: trust200902
area: "Internet"
workgroup: "Static Context Header Compression"
submissiontype: IETF
keyword:
  - SCHC
  - header compression
  - dynamic address
  - LPWAN
  - IPv6
  - compute CDA

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Magnus Westerlund
    organization: Ericsson
    city: Kista
    country: Sweden
    email: magnus.westerlund@ericsson.com
  -
    fullname: Lorenzo Corneo
    organization: Ericsson
    city: Jorvas
    country: Finland
    email: lorenzo.corneo@ericsson.com

contributor:
  -
    fullname: Edgar Ramos
    organization: Ericsson
    city: Jorvas
    country: Finland
    email: edgar.ramos@ericsson.com
  -
    fullname: Ari Keränen
    organization: Ericsson
    city: Jorvas
    country: Finland
    email: ari.keranen@ericsson.com

normative:
  RFC2119:
  RFC8174:
  RFC8724:
  RFC8981:
  RFC4291:
  RFC6234:
  RFC5869:

informative:
  RFC7217:
  RFC4862:
  RFC8200:
  RFC8376:

--- abstract

This document defines new Matching Operators (MOs) and
Compression/Decompression Actions (CDAs) for the Static Context
Header Compression and fragmentation (SCHC) framework defined in
RFC 8724. These extensions enable efficient compression of
dynamically assigned IP addresses, including IPv4 addresses
assigned via DHCP, IPv6 prefixes learned through SLAAC or
DHCPv6, IPv6 Interface Identifiers (IIDs), and IPv6 temporary
privacy addresses generated per RFC 8981.

The mechanism relies on both the compressor and decompressor
sharing knowledge of the set of addresses assigned to the
device. Addresses are organized into deterministically sorted
tables, allowing compression to a small index value. For
temporary IPv6 addresses, a synchronized generation algorithm
enables compression to an epoch and counter value.

--- middle

# Introduction

Static Context Header Compression and fragmentation (SCHC)
{{RFC8724}} provides a mechanism for compressing protocol headers
over constrained links. SCHC relies on a shared static context (a
set of Rules) between the compressor and decompressor. Each Rule
describes how individual header fields are matched and compressed
using Matching Operators (MOs) and Compression/Decompression Actions
(CDAs).

The CDAs defined in {{RFC8724}} assume that address field values are
either known at Rule provisioning time (enabling the use of "not-sent"
or "mapping-sent" CDAs) or can be derived from Layer 2 information
(using "DevIID" or "AppIID" CDAs). However, in many modern network
deployments, IP addresses are assigned dynamically and may change over
time. Examples include:

- IPv4 addresses assigned via DHCP
- IPv6 prefixes learned through Router Advertisements (SLAAC
  {{RFC4862}}) or DHCPv6 Prefix Delegation
- IPv6 Interface Identifiers generated per RFC 7217
- IPv6 temporary privacy addresses generated per {{RFC8981}}

Furthermore, in mobile network architectures (e.g., 3GPP), there are
no Layer 2 addresses available at the packet level, rendering the
DevIID and AppIID CDAs from {{RFC8724}} inapplicable.

This document defines new MOs and CDAs that enable efficient
compression of dynamically assigned addresses by leveraging shared
knowledge of the device's assigned address set between the compressor
and decompressor. The new CDAs are:

- comp-addr-v4: for IPv4 addresses
- comp-addr-prefix: for IPv6 address prefixes
- comp-addr-iid: for IPv6 Interface Identifiers
- comp-temp-iid: for IPv6 temporary privacy IIDs

# Terminology

{::boilerplate bcp14-tagged}

This document uses the terminology defined in {{RFC8724}}. The
following additional terms are used:

Address Table:
: A deterministically sorted list of address information elements of a
  given category, derived from the set of addresses assigned to the
  device. The Address Table is maintained per-context and shared
  between compressor and decompressor.

Address Category:
: A classification of address information. This document defines three
  categories: IPv4 full address, IPv6 prefix, and IPv6 Interface
  Identifier (IID).

Address Index:
: A zero-based integer identifying an entry's position in an Address
  Table.

Epoch:
: A time interval identifier used in the generation of temporary IPv6
  IIDs. The epoch counter identifies which time interval was used as
  input to the IID generation function.

DAD Counter:
: A counter used to resolve Duplicate Address Detection collisions
  during temporary IID generation, as defined in {{RFC8981}}.

# Problem Statement

SCHC was originally designed for Low-Power Wide Area Networks
(LPWANs) {{RFC8376}} where devices typically have fixed, pre-configured IP
addresses. In
such environments, the address can be stored as a Target Value (TV) in
the Rule and compressed using the "not-sent" CDA, achieving maximum
compression efficiency.

However, when SCHC is applied to networks where addresses are
dynamically assigned, several problems arise:

1. **Dynamic assignment:** Networks using DHCP, DHCPv6, or SLAAC
   assign addresses that may change each time the device connects.
   Storing these in the Rule's TV requires updating the Rule set on
   every address change, which conflicts with SCHC's static context
   design.

2. **Multiple addresses:** A device may be assigned multiple IPv4
   addresses, multiple IPv6 prefixes (e.g., via DHCPv6 Prefix
   Delegation for tethering), and multiple IIDs. The existing
   "mapping-sent" CDA could handle a fixed list, but the list itself
   is dynamic.

3. **Temporary privacy addresses:** RFC 8981 defines temporary
   addresses with randomized IIDs that change periodically. Sending
   the full 64-bit IID as a residual negates the benefit of
   compression.

4. **No Layer 2 addresses:** In mobile networks (e.g., 3GPP), there
   are no Layer 2 addresses in the packet headers that could be used
   to derive the IPv6 IID. The DevIID and AppIID CDAs from {{RFC8724}}
   are therefore not applicable in these environments.

Without the extensions defined in this document, the only option for
compressing dynamic addresses in SCHC is to use the "value-sent" CDA,
which transmits the full address field (32 bits for IPv4, 64 bits for
an IPv6 prefix or IID, or 128 bits for a full IPv6 address
{{RFC8200}}) as the compression residual. This results in poor compression efficiency for
the address fields.

# Architecture and Assumptions

The mechanisms defined in this document rely on the following
architectural assumptions:

1. **Shared address knowledge:** Both the compressor and decompressor
   have access to the same set of addresses assigned to the device.
   How this knowledge is obtained is deployment-specific and discussed
   informatively in {{provisioning}}.

2. **Per-context Address Tables:** The Address Tables are maintained
   per SCHC context (i.e., per device). Multiple Rules within the same
   context MAY reference the same Address Table. The table SHOULD be
   cached and only rebuilt when the underlying address set changes.

3. **Synchronized state:** Both endpoints MUST maintain synchronized
   address sets. When the address set changes (e.g., new address
   assigned, address expired), both sides MUST rebuild their Address
   Tables. Mechanisms for ensuring this synchronization are outside the
   scope of this document and may require further extensions.

4. **Clock synchronization (for comp-temp-iid):** For the temporary
   IID mechanism, both endpoints MUST have roughly synchronized wall
   clocks. The required accuracy depends on the configured time
   interval length but is typically in the order of seconds.

{{fig-architecture}} illustrates the relationship between the address
assignment infrastructure and the SCHC compressor/decompressor.

~~~
  +--------+          +------------+         +-----------+
  | Device |<-------->|   Network  |<------->|   SCHC    |
  | (UE)   | Address  | Address    | Address | Network   |
  |        | Assign.  | Assignment | State   | Compressor|
  | SCHC   |          | Function   |         | /Decompr. |
  | Compr/ |          | (DHCP/SMF/ |         |           |
  | Decomp |          |  SLAAC)    |         |           |
  +--------+          +------------+         +-----------+
      |                                           |
      |   Both maintain same Address Tables       |
      |<==========================================>|
~~~
{: #fig-architecture title="Architecture Overview"}

# Address Table Construction {#address-table}

The compute-address CDAs defined in this document operate on Address
Tables that are derived from the set of IP addresses assigned to the
device's network interface(s). This section defines how these tables
are constructed.

## Address Categories

The assigned addresses are classified into three categories, each
producing a separate Address Table:

IPv4 Address Table:
: Contains full 32-bit IPv4 addresses assigned to the device.

IPv6 Prefix Table:
: Contains IPv6 prefixes (by default the first 64 bits of each
  assigned IPv6 address). The prefix length is configurable but
  defaults to 64 bits.

IPv6 IID Table:
: Contains IPv6 Interface Identifiers (by default bits 64-127 of each
  assigned IPv6 address, where bit 0 is the most significant bit). The
  offset and length are configurable but default to offset 64, length
  64 bits.

Each category's table is independent. A single IPv6 address
contributes one entry to the IPv6 Prefix Table and one entry to the
IPv6 IID Table.

## Input Filtering

Before constructing the Address Tables, the set of assigned addresses
MUST be filtered as follows:

- Loopback addresses (127.0.0.0/8 for IPv4, ::1/128 for IPv6) MUST
  be excluded.
- Link-local addresses (169.254.0.0/16 for IPv4, fe80::/10 for IPv6)
  SHOULD be excluded unless the Rule specifically targets link-local
  communication.
- Multicast addresses MUST be excluded.

The filtering criteria are determined by the address type filter
associated with the MO/CDA in the Rule. The default filter includes
only unicast global addresses.

## Sorting and Indexing

After filtering, the entries in each Address Table MUST be sorted in
ascending binary order (treating the address or address portion as an
unsigned integer). This produces a deterministic ordering that is
identical on both the compressor and decompressor, provided they have
the same input address set.

Each entry is assigned a zero-based index: the entry with the lowest
binary value receives index 0, the next receives index 1, and so on.

Duplicate entries (e.g., two IPv6 addresses sharing the same prefix)
MUST be deduplicated before indexing. Only unique values appear in the
table.

~~~
Device assigned addresses:
  IPv4: 198.51.100.34
  IPv6: 2001:db8::a38d:1841:2a6f:124b:763e
  IPv6: 2001:db8::1234:c84a:e8ff:fe45:ad7f

After filtering (loopback/link-local removed) and categorization:

IPv4 Address Table:
  +-------+----------------+
  | Index | Address        |
  +-------+----------------+
  |   0   | 198.51.100.34  |
  +-------+----------------+

IPv6 Prefix Table (first 64 bits, deduplicated):
  +-------+----------------+
  | Index | Prefix         |
  +-------+----------------+
  |   0   | 2001:db8::     |
  +-------+----------------+

IPv6 IID Table (bits 64-127, sorted ascending):
  +-------+---------------------------+
  | Index | IID                       |
  +-------+---------------------------+
  |   0   | 1234:c84a:e8ff:fe45:ad7f  |
  |   1   | a38d:1841:2a6f:124b:763e  |
  +-------+---------------------------+
~~~
{: #fig-table-example title="Example Address Table Construction"}

## Address Table Caching

Implementations SHOULD cache the constructed Address Tables to avoid
recomputation on every packet. The cached tables MUST be invalidated
and rebuilt when the underlying set of assigned addresses changes
(e.g., new address assigned, existing address expired or released).

Since the Address Tables are per-context, they are shared across all
Rules in the same context. A single table rebuild serves all Rules
that reference the same address category.

# Compute-Address MO and CDA {#compute-address}

This section defines three named Matching Operators and
Compression/Decompression Actions for dynamically assigned addresses.
Each operates on the corresponding Address Table defined in
{{address-table}}.

In all three cases, the MO and CDA share the same name. The Rule's
Field Descriptor specifies the number of bits allocated for the
residual, which determines the maximum number of entries that can be
indexed (2^bits entries).

## comp-addr-v4

The comp-addr-v4 MO and CDA operate on the IPv4 Address Table. They
are used to compress full 32-bit IPv4 addresses that have been
dynamically assigned to the device.

### Matching Operator

The comp-addr-v4 MO compares the packet's field value (a 32-bit IPv4
address) against all entries in the IPv4 Address Table.

The MO returns True if the field value matches any entry in the table
AND the matching entry's index is representable within the number of
residual bits allocated in the Rule.

The MO returns False otherwise (no match found, or the matching
entry's index exceeds the maximum representable value).

### Compression/Decompression Action

On compression: the CDA looks up the field value in the IPv4 Address
Table, obtains the matching entry's index, and encodes this index as
the compression residual using the number of bits specified in the
Rule.

On decompression: the CDA reads the residual bits, decodes the index
value, looks up the corresponding entry in the IPv4 Address Table, and
writes the 32-bit IPv4 address into the packet field.

### Rule Field Descriptor

A Field Descriptor using comp-addr-v4 has the following
characteristics:

- FID: The IPv4 address field (e.g., IPv4 Source Address or IPv4
  Destination Address)
- FL: 32 (the original field length)
- TV: Not used (left empty)
- MO: comp-addr-v4
- CDA: comp-addr-v4
- Sent bits: The number of bits for the index residual (e.g., 2 bits
  for up to 4 addresses)

## comp-addr-prefix

The comp-addr-prefix MO and CDA operate on the IPv6 Prefix Table.
They are used to compress IPv6 address prefixes that have been
dynamically assigned to the device (e.g., via SLAAC Router
Advertisements or DHCPv6 Prefix Delegation).

### Matching Operator

The comp-addr-prefix MO extracts the prefix portion (by default the
first 64 bits) from the packet's IPv6 address field and compares it
against all entries in the IPv6 Prefix Table.

The MO returns True if the extracted prefix matches any entry in the
table AND the matching entry's index is representable within the
allocated residual bits.

The MO returns False otherwise.

### Compression/Decompression Action

On compression: the CDA extracts the prefix from the field value,
looks it up in the IPv6 Prefix Table, and encodes the matching entry's
index as the residual.

On decompression: the CDA reads the residual, decodes the index, looks
up the prefix in the IPv6 Prefix Table, and writes it into the
appropriate portion of the IPv6 address field.

### Rule Field Descriptor

A Field Descriptor using comp-addr-prefix has the following
characteristics:

- FID: The IPv6 address prefix field (e.g., IPv6 Dev Prefix or IPv6
  App Prefix)
- FL: 64 (default prefix length)
- TV: Not used (left empty)
- MO: comp-addr-prefix
- CDA: comp-addr-prefix
- Sent bits: The number of bits for the index residual (e.g., 2 bits
  for up to 4 prefixes)

## comp-addr-iid

The comp-addr-iid MO and CDA operate on the IPv6 IID Table. They are
used to compress IPv6 Interface Identifiers that have been assigned to
the device, including stable IIDs generated per {{RFC7217}}.

### Matching Operator

The comp-addr-iid MO extracts the IID portion (by default bits 64-127)
from the packet's IPv6 address field and compares it against all
entries in the IPv6 IID Table.

The MO returns True if the extracted IID matches any entry in the
table AND the matching entry's index is representable within the
allocated residual bits.

The MO returns False otherwise.

### Compression/Decompression Action

On compression: the CDA extracts the IID from the field value, looks
it up in the IPv6 IID Table, and encodes the matching entry's index as
the residual.

On decompression: the CDA reads the residual, decodes the index, looks
up the IID in the IPv6 IID Table, and writes it into the appropriate
portion of the IPv6 address field.

### Rule Field Descriptor

A Field Descriptor using comp-addr-iid has the following
characteristics:

- FID: The IPv6 IID field (e.g., IPv6 Dev IID or IPv6 App IID)
- FL: 64 (default IID length)
- TV: Not used (left empty)
- MO: comp-addr-iid
- CDA: comp-addr-iid
- Sent bits: The number of bits for the index residual (e.g., 4 bits
  for up to 16 IIDs)

## Example Rule

The following example shows a compression Rule for an IPv6/UDP packet
where the device has dynamically assigned IPv6 addresses. The device
prefix and IID are compressed using comp-addr-prefix and comp-addr-iid
respectively.

~~~
+----------------+--+--+--+----------+---------------+---------------+------+
|       FID      |FL|FP|DI|    TV    |      MO       |      CDA      | Sent |
+----------------+--+--+--+----------+---------------+---------------+------+
|IPv6 Version    | 4|1 |Bi|6         | equal         | not-sent      |      |
|IPv6 DiffServ   | 8|1 |Bi|0         | equal         | not-sent      |      |
|IPv6 Flow Label |20|1 |Bi|0         | equal         | not-sent      |      |
|IPv6 Length     |16|1 |Bi|          | ignore        | compute-*     |      |
|IPv6 Next Header| 8|1 |Bi|17        | equal         | not-sent      |      |
|IPv6 Hop Limit  | 8|1 |Bi|255       | ignore        | not-sent      |      |
|IPv6 DevPrefix  |64|1 |Up|          | comp-addr-prf | comp-addr-prf |  2   |
|IPv6 DevIID     |64|1 |Up|          | comp-addr-iid | comp-addr-iid |  4   |
|IPv6 AppPrefix  |64|1 |Up|2001:db8::| equal         | not-sent      |      |
|IPv6 AppIID     |64|1 |Up|::1       | equal         | not-sent      |      |
+----------------+--+--+--+----------+---------------+---------------+------+
|UDP DevPort     |16|1 |Bi|5683      | equal         | not-sent      |      |
|UDP AppPort     |16|1 |Bi|5683      | equal         | not-sent      |      |
|UDP Length      |16|1 |Bi|          | ignore        | compute-*     |      |
|UDP Checksum    |16|1 |Bi|          | ignore        | compute-*     |      |
+----------------+--+--+--+----------+---------------+---------------+------+
~~~
{: #fig-rule-example title="Example Rule Using Compute-Address CDAs"}

In this example, the device's IPv6 source prefix is compressed to 2
bits (supporting up to 4 prefixes) and the IID to 4 bits (supporting
up to 16 IIDs). The total residual for the device's full 128-bit IPv6
address is only 6 bits.

# Compute Temporary IPv6 IID {#comp-temp-iid}

This section defines the comp-temp-iid MO and CDA for compressing IPv6
temporary privacy addresses generated per {{RFC8981}}. Unlike the
table-index approach of the compute-address CDAs in
{{compute-address}}, this mechanism uses a synchronized generation
algorithm that allows both endpoints to derive the same IID from a
small set of parameters transmitted as the compression residual.

## Overview

RFC 8981 defines a procedure for generating temporary IPv6 IIDs using
a pseudorandom function (PRF) with several inputs. The comp-temp-iid
mechanism constrains these inputs such that both the device and the
network-side compressor/decompressor can independently generate the
same IID given:

- Shared static context (secret key, interface identifier, network
  identifier)
- The IPv6 prefix (known via other means, e.g., comp-addr-prefix)
- A time epoch counter (transmitted in the residual)
- A DAD counter (transmitted in the residual)

The compression residual consists of the epoch counter (N bits) and the
DAD counter (M bits), where N and M are configured per-rule. This
typically results in fewer than 10 bits of residual instead of the full
64-bit IID.

## Shared State Requirements

The following information MUST be available to both the device-side and
network-side compressor/decompressor for the comp-temp-iid mechanism to
function:

secret_key:
: A device-specific secret key of at least 128 bits, used as input to
  the PRF. This key MUST be unique per device and MUST NOT be used for
  any other purpose. It MAY be derived from existing device credentials
  (e.g., from 3GPP security keys using a key derivation function such
  as HKDF {{RFC5869}}) or MAY be provisioned as part of the SCHC
  context setup.

Net_Iface:
: A network interface identifier. In networks without Layer 2
  addresses (e.g., 3GPP), this MUST be set to a deterministic value
  known to both endpoints, such as a value derived from the device's
  network identity.

Network_ID:
: A network-specific identifier for the subnet or attachment point.
  This SHOULD be employed if available (e.g., an SSID for Wi-Fi, or a
  cell/network identifier for mobile networks).

Time_Offset:
: A fixed time offset that defines when intervals begin, expressed as
  an offset within the interval period (e.g., minutes and seconds past
  the hour). Both compressor and decompressor MUST use the same offset
  value.

Interval_Length:
: The duration of each time interval, in seconds. This determines how
  frequently new temporary IIDs are generated. Typical values range
  from 900 seconds (15 minutes) to 3600 seconds (1 hour).

Max_Lifetime:
: The maximum lifetime of a temporary address, in seconds. This bounds
  how far back in time a valid epoch can refer.

Prefix:
: The IPv6 prefix used as input to the generation function. This is
  typically known through other compression mechanisms (e.g.,
  comp-addr-prefix) or network configuration.

## Epoch and Time Derivation

The epoch counter identifies which time interval was used to generate a
particular temporary IID. It is derived from the wall clock time as
follows:

1. Read the current wall clock time Tc (e.g., as a UNIX timestamp or
   NTP timestamp).

2. Compute the interval start time Tis by rounding Tc down to the most
   recent interval boundary. The interval boundaries are defined by
   Time_Offset and Interval_Length:

   Tis = Time_Offset + floor((Tc - Time_Offset) / Interval_Length) *
   Interval_Length

3. Compute the epoch counter as the number of complete intervals since
   a fixed reference point (e.g., UNIX epoch of 1970-01-01T00:00:00Z):

   epoch = floor((Tis - Time_Offset) / Interval_Length)

4. The N least significant bits of the epoch counter are used as the
   epoch value in the compression residual.

The number of bits N MUST be large enough that the epoch value does not
wrap around within the Max_Lifetime of any valid temporary address.
Specifically:

> N >= ceil(log2(Max_Lifetime / Interval_Length + 1))

For example, with Interval_Length = 1800 seconds (30 minutes) and
Max_Lifetime = 86400 seconds (24 hours), N >= ceil(log2(49)) = 6 bits.

## IID Generation Algorithm

The temporary IID is generated using the following algorithm, which is
a constrained application of the procedure in Section 3.3.2 of
{{RFC8981}}:

Step 1: Compute the Random Identifier (RID):

~~~
  RID = F(Prefix, Net_Iface, Network_ID, Tis, DAD_Counter, secret_key)
~~~

Where:

- F() is a pseudorandom function. This document RECOMMENDS
  HMAC-SHA-256 {{RFC6234}} with the secret_key as the HMAC key and the
  concatenation of the other parameters as the message. The output is
  truncated to 64 bits for use as the IID.
- Prefix is the IPv6 prefix (typically 64 bits).
- Net_Iface is the network interface identifier.
- Network_ID is the network identifier.
- Tis is the interval start time computed in {{comp-temp-iid}}.
- DAD_Counter starts at 0 and is incremented if the generated IID is
  found to be a duplicate or reserved.

Step 2: Derive the IID from RID. Take the least significant 64 bits of
the PRF output. Set bits 6 and 7 (the "u" and "g" bits in the Modified
EUI-64 format) to 0, as specified in {{RFC4291}} Section 2.5.1 for
non-globally-unique identifiers.

Step 3: Check the generated IID against reserved subnet anycast
addresses and other reserved IID values. If the IID is reserved, or if
Duplicate Address Detection (DAD) fails, increment DAD_Counter and
repeat from Step 1.

The PRF input SHOULD be constructed as the concatenation of the
parameters in a fixed, well-defined order:

~~~
  PRF_input = Prefix || Net_Iface || Network_ID || Tis || DAD_Counter
  IID = truncate_64(HMAC-SHA-256(secret_key, PRF_input))
  IID[6] = 0  (clear bit 6, the "u" bit)
  IID[7] = 0  (clear bit 7, the "g" bit)
~~~

Note: The choice of PRF is an open issue. Future versions of this
document may allow alternative PRFs or make the PRF selection
configurable via the SCHC profile.

## Matching Operator: comp-temp-iid

The comp-temp-iid MO extracts the IID portion (bits 64-127) from the
packet's IPv6 address field and attempts to match it against temporary
IIDs that could have been generated using the shared state.

The matching procedure is:

1. Check the IID cache (see {{caching}}). If the IID is found in the
   cache with a valid (epoch, DAD_Counter) pair, the MO returns True.

2. If not cached, iterate over recent epochs (starting from the
   current epoch and working backwards) and for each epoch iterate over
   DAD_Counter values (starting from 0), generating IIDs until a match
   is found or the search space is exhausted.

3. The search space is bounded by: epochs from current back to
   (current - 2^N + 1), and DAD_Counter from 0 to (2^M - 1), where N
   and M are the bit widths configured in the Rule.

4. If a match is found AND the epoch and DAD_Counter are representable
   in the allocated residual bits, the MO returns True.

5. Otherwise, the MO returns False.

## Compression/Decompression Action: comp-temp-iid

On compression: the CDA encodes the epoch counter (N bits) followed by
the DAD_Counter (M bits) as the compression residual. The N least
significant bits of the epoch counter value are placed in the most
significant N bits of the residual. The DAD_Counter is encoded as an
M-bit unsigned integer in the remaining bits.

~~~
  Residual format:
  +----------+-------------+
  |  Epoch   | DAD_Counter |
  | (N bits) |  (M bits)   |
  +----------+-------------+
~~~

On decompression: the CDA reads N + M bits from the residual, extracts
the epoch and DAD_Counter values, and either:

- Looks up the IID in the cache using (epoch, DAD_Counter) as the key,
  or
- Generates the IID using the algorithm in {{comp-temp-iid}} with the
  decoded epoch (converted back to Tis) and DAD_Counter.

The generated or cached IID is then written into the appropriate
portion of the IPv6 address field.

## Rule Field Descriptor

A Field Descriptor using comp-temp-iid has the following
characteristics:

- FID: The IPv6 IID field (e.g., IPv6 Dev IID)
- FL: 64
- TV: Not used (left empty)
- MO: comp-temp-iid
- CDA: comp-temp-iid
- Sent bits: N + M (e.g., 6 + 2 = 8 bits)

The values of N (epoch bits) and M (DAD counter bits) are configured
per-rule. They MUST be specified as part of the Rule definition.

## Temporary IID Caching {#caching}

Implementations SHOULD maintain a cache of generated temporary IIDs
with their associated (epoch, DAD_Counter) metadata. This avoids the
computational cost of iterating over the parameter space for every
packet.

On the device side, when a new temporary address is generated and
assigned to an interface, the device SHOULD store the epoch and
DAD_Counter used in its generation alongside the address.

On the network side, the compressor/decompressor SHOULD observe IIDs
used in uplink (device-to-network) packets and cache them with the
corresponding (epoch, DAD_Counter) values determined during the MO
matching process. This ensures that subsequent packets using the same
temporary address can be processed efficiently without re-iterating.

Cache entries SHOULD be expired when the corresponding temporary
address exceeds its Max_Lifetime.

# Address Table Updates

The Address Tables defined in {{address-table}} are derived from the
set of addresses currently assigned to the device. This set may change
over time due to:

- New address assignment (e.g., DHCP lease renewal with a new address,
  new prefix via Router Advertisement)
- Address expiration or release
- DHCPv6 Prefix Delegation changes
- Network handover in mobile networks

When the address set changes, both the compressor and decompressor MUST
rebuild the affected Address Table(s) using the procedure in
{{address-table}}. Since the table is sorted deterministically, both
sides will produce identical tables provided they have the same input
address set.

Note that rebuilding a table may change the indices of existing entries
(e.g., a new address with a lower binary value shifts all higher
entries up by one). Compressed packets in flight at the time of a table
rebuild may be decompressed incorrectly if the decompressor has already
updated its table while the compressor used the old table, or vice
versa.

Detailed mechanisms for synchronizing table updates between compressor
and decompressor, and for handling packets in flight during
transitions, are outside the scope of this document and are expected to
be addressed in future extensions.

# Address Set Provisioning {#provisioning}

This section provides informational guidance on how the network-side
compressor/decompressor can obtain knowledge of the device's assigned
addresses. The exact mechanisms are deployment-specific and outside the
normative scope of this document.

## General Mechanisms

In networks using standard IP address assignment protocols, the
network-side compressor/decompressor can learn the device's addresses
from:

- **DHCP/DHCPv6 server state:** The DHCP server maintains lease
  information that maps devices to their assigned IPv4 addresses or
  IPv6 prefixes. This information can be made available to the
  compressor.
- **RADIUS/Diameter attributes:** In networks using AAA
  infrastructure, address assignments may be communicated via RADIUS or
  Diameter attributes.
- **Router Advertisement monitoring:** For SLAAC, the network knows
  which prefixes have been advertised. Combined with knowledge of the
  device's IID generation method, the full address can be determined.

## Mobile Network Considerations

In 3GPP mobile networks (4G/5G), the following characteristics are
relevant:

- **No Layer 2 addresses:** Unlike Ethernet or LoRaWAN, 3GPP radio
  bearers do not carry Layer 2 addresses in the packet headers. The
  DevIID and AppIID CDAs from {{RFC8724}} are therefore not applicable.
  This is a primary motivation for the compute-address CDAs defined in
  this document.

- **Address assignment by SMF:** In 5G, the Session Management
  Function (SMF) is responsible for IP address allocation to the UE
  (User Equipment). The compressor, which may be located in the gNB
  (base station), CU (Central Unit), or UPF (User Plane Function),
  needs a mechanism to receive the assigned address information from
  the SMF.

- **Multiple PDU sessions:** A UE may have multiple PDU sessions, each
  with different IP addresses or prefixes. The compressor needs to know
  which addresses are associated with which session.

- **Prefix Delegation:** A UE may receive delegated prefixes (e.g.,
  for tethering). These additional prefixes must also be known to the
  compressor for efficient compression of traffic using those prefixes.

The specific 3GPP signaling extensions needed to convey address
information to the compressor are outside the scope of this document.

### Stable Privacy Addresses on Mobile Devices

Modern mobile operating systems (e.g., Android since version 10)
generate stable IPv6 IIDs using the algorithm defined in {{RFC7217}}
rather than the legacy EUI-64 derivation. The typical inputs are:

- A device-local secret key (persisted across reboots)
- The interface name (e.g., "rmnet0")
- The assigned IPv6 prefix
- A DAD counter

Since the secret key is generated and stored locally on the device, the
network-side compressor has no direct way to predict the resulting
stable IID. This IID therefore cannot be elided using the DevIID CDA
from {{RFC8724}}, nor can it be placed in the Rule's Target Value at
provisioning time (since it depends on the prefix, which is dynamic).

The comp-addr-iid CDA defined in this document addresses this case: the
network-side compressor learns the device's stable IID through the
address provisioning mechanism (see {{provisioning}}) and both sides
reference it via a small table index. This avoids transmitting the full
64-bit IID as a residual on every packet while requiring no changes to
the device's existing address generation behavior.

# Examples

## IPv4 Single Address (DHCP)

A device is assigned a single IPv4 address 198.51.100.34 via DHCP. The
IPv4 Address Table contains one entry:

~~~
  IPv4 Address Table:
    Index 0: 198.51.100.34

  Rule Field Descriptor:
    FID=IPv4 Src Addr, FL=32, MO=comp-addr-v4, CDA=comp-addr-v4,
    Sent=1 bit

  Compression: field value 198.51.100.34 matches index 0
               -> residual: 0b0 (1 bit)

  Decompression: residual 0b0 -> index 0 -> 198.51.100.34
~~~

## IPv6 Dual Prefix (SLAAC + DHCPv6-PD)

A device has two IPv6 prefixes: 2001:db8:1::/64 from SLAAC and
2001:db8:2::/64 from DHCPv6 Prefix Delegation.

~~~
  IPv6 Prefix Table (sorted ascending):
    Index 0: 2001:db8:1::  (2001:0db8:0001:0000)
    Index 1: 2001:db8:2::  (2001:0db8:0002:0000)

  Rule Field Descriptor:
    FID=IPv6 DevPrefix, FL=64, MO=comp-addr-prefix,
    CDA=comp-addr-prefix, Sent=1 bit

  Packet with source prefix 2001:db8:2::
  Compression: matches index 1 -> residual: 0b1 (1 bit)

  Decompression: residual 0b1 -> index 1 -> 2001:db8:2::
~~~

## Temporary Privacy Address

A device generates a temporary IID using the comp-temp-iid algorithm
with the following parameters:

~~~
  Shared state:
    secret_key:      0x4a7f...  (128 bits)
    Net_Iface:       0x00000001 (device-specific)
    Network_ID:      "mobile-net-1"
    Time_Offset:     0 seconds (intervals aligned to midnight)
    Interval_Length: 1800 seconds (30 minutes)
    Max_Lifetime:    86400 seconds (24 hours)

  Rule configuration:
    N = 6 bits (epoch), M = 2 bits (DAD counter)
    Total residual: 8 bits

  Current time: 2026-06-01T14:45:00Z
  Epoch calculation:
    Tis = 2026-06-01T14:30:00Z (rounded down to interval boundary)
    epoch = (number of 30-min intervals since reference) mod 64

  Suppose epoch = 42 (0b101010), DAD_Counter = 0 (0b00)

  Compression residual: 0b10101000 (8 bits)

  Decompression:
    epoch = 42, DAD_Counter = 0
    Tis = derive from epoch
    IID = truncate_64(HMAC-SHA-256(secret_key,
          Prefix || Net_Iface || Network_ID || Tis || 0))
    Write IID into packet field.
~~~

# Security Considerations

## Secret Key Management

The comp-temp-iid mechanism relies on a shared secret key between the
device and the network-side compressor/decompressor. Compromise of this
key would allow an attacker to predict all temporary IIDs generated by
the device, defeating the privacy protection intended by {{RFC8981}}.

The secret key MUST be stored securely on both endpoints. It SHOULD be
derived from existing security associations where possible (e.g., from
3GPP security keys using HKDF) rather than being transmitted in the
clear.

The secret key MUST NOT be reused for any purpose other than temporary
IID generation for SCHC compression.

## Clock Desynchronization

If the clocks of the compressor and decompressor drift apart beyond the
tolerance of the time interval, the decompressor may generate a
different IID than the one used by the device. This would result in
packet misdelivery or discard.

Implementations SHOULD monitor for decompression failures that may
indicate clock drift and trigger resynchronization. The Interval_Length
SHOULD be chosen to be significantly larger than the expected maximum
clock drift between synchronization events.

## Address Table Inconsistency

If the compressor and decompressor have different views of the device's
assigned address set, the Address Tables will differ, leading to
incorrect compression/decompression. This could result in packets being
delivered to the wrong address or being discarded.

Deployments MUST ensure that address assignment changes are propagated
to both endpoints before the new addresses are used in packet headers.
The exact mechanism for ensuring this is deployment-specific.

## Privacy Considerations

The compression residual for comp-addr-v4, comp-addr-prefix, and
comp-addr-iid reveals the index of the address in the sorted table. An
observer of the compressed traffic can determine:

- How many addresses the device has (from the maximum index value
  observed)
- Whether the device is using the same or different addresses across
  packets (same index = same address)

This information leakage is inherent to the compression mechanism and
should be considered acceptable given that SCHC is typically used over
links where the observer would already have access to the uncompressed
traffic (e.g., the radio link in a mobile network).

For comp-temp-iid, the epoch value in the residual reveals when the
temporary address was generated. This is less information than the full
IID but still provides some temporal correlation capability to an
observer.

# IANA Considerations

This document defines new Matching Operators and Compression/
Decompression Actions for the SCHC framework. When IANA registries are
established for SCHC MOs and CDAs, the following entries should be
registered:

New Matching Operators:

- comp-addr-v4
- comp-addr-prefix
- comp-addr-iid
- comp-temp-iid

New Compression/Decompression Actions:

- comp-addr-v4
- comp-addr-prefix
- comp-addr-iid
- comp-temp-iid

Note: At the time of writing, no IANA registry exists for SCHC MOs and
CDAs. If such registries are created, the above values should be
registered. Otherwise, this document serves as the definition of these
MO and CDA names.

--- back

# Acknowledgements
{:numbered="false"}

The authors would like to thank the members of the IETF SCHC Working
Group for their feedback and discussions.
