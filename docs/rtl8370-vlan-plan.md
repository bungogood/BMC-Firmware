# RTL8370 VLAN Enablement Plan

## Goal

Enable hardware-enforced 802.1Q VLAN membership and multicast containment on
the Turing Pi 2 RTL8370MB-CG without disrupting BMC or RK1 management.

## Current State

The official Turing Pi 2 driver provides DSA port isolation and bridge offload
but does not expose VLAN filtering, PVID, or VLAN membership callbacks. The
patch series now adds those callbacks and the required table support. The
switch is managed through Turing Pi's non-standard RTK-I2C transport, not
ordinary I2C register reads.

The upstream RTL8365MB VLAN code required a TP2-specific transport correction:
RTK-I2C supports one 16-bit register per transaction and cannot use regmap bulk
operations. The corrected image uses individual register operations for table
and VLAN membership configuration entries.

## Safety Rules

- Keep the official BMC OTA image and SD recovery image available.
- Use the BMC serial console for every custom firmware boot.
- Do not modify forwarding, VLAN, or port-control registers until their reset
  and post-init values have been captured. The chip-ID selector sequence is an
  exception: it is already used by the production probe and is always cleared.
- Do not add DSA VLAN callbacks until a single table entry can be written,
  read back, and restored on target hardware.
- Validate normal four-node bridging after every firmware boot.

## Phase 1: Read-Only Diagnostics

Add a debugfs report to the existing RTL8370 DSA driver. It must use the
existing RTK-I2C regmap. Apart from the production driver's temporary chip-ID
selector sequence, it must perform reads only.

Capture the following state:

- Chip ID, revision, and detected port topology.
- Global VLAN control, ingress filtering, and accepted-frame registers.
- All PVID selector registers.
- VLAN membership configuration entries.
- VLAN4K table entries for selected VIDs.
- Port isolation and external-interface state.

Record the output after official firmware initialization. Compare it with the
RTL8370 vendor-family register references before writing any values.

## Phase 2: Reversible Hardware Probe

Identify an unused membership-table entry from the read-only dump. With serial
recovery available:

1. Save the entry and affected PVID registers.
2. Write a test entry that does not affect any live port.
3. Read it back through RTK-I2C.
4. Restore the exact saved values.
5. Confirm all RK1 management links remain reachable.

This phase establishes whether the table layout and access semantics match the
RTL8370 hardware on Turing Pi 2.

## Phase 3: RTL8370-Specific VLAN Driver

Implement a separate RTL8370 VLAN path rather than enabling RTL8365MB VLAN
callbacks for chip ID 0x6368. The implementation must:

- Preserve firmware-reserved membership entries.
- Use validated PVID, membership, VLAN4K, ingress, and egress semantics.
- Keep VLAN filtering disabled by default.
- Program only VLAN 1 initially, then one tagged test VLAN.
- Provide explicit rollback for every register changed.

## Phase 4: Network Validation

Validate in this order:

1. Normal management bridge on all four RK1 nodes.
2. One tagged unicast VLAN.
3. Tagged broadcast containment.
4. Low-rate multicast containment.
5. Incremental multicast rate tests with router health and switch-port counters.

High-rate multicast is out of scope until each preceding check succeeds.

## Findings

### 2026-07-26: Userspace RTK-I2C Probe

The official BMC detects the switch twice during boot as
`RTL8370MB-CG` at I2C address `0x5c` on `/dev/i2c-0`. The existing kernel
driver is bound as `realtek-smi`.

A read-only userspace probe reproduced the driver's three-message RTK-I2C read
shape using `I2C_RDWR`: a zero-length read address, two no-start register
bytes, then a no-start two-byte read. Reads of chip ID and revision returned
`0x0000`, which conflicts with successful in-kernel detection.

No switch registers were written and all node connectivity remained intact.
The adapter capability report does not expose the required no-start behavior
through standard userspace tooling. Therefore userspace I2C access is not a
reliable diagnostic transport for this board.

The next diagnostic must run inside the existing `realtek-smi` kernel driver
and use its proven RTK-I2C regmap path. A read-only debugfs report is the
appropriate implementation.

### 2026-07-26: In-Kernel Diagnostic Build

`tp2bmc/patches/linux/rtl8370-debugfs.patch` adds the debugfs file
`/sys/kernel/debug/rtl8370_identity`. The corrected version performs the same
bounded selector sequence as the production probe: write `0x0249` to `0x13c2`,
read `0x1300` and `0x1301`, then clear `0x13c2`. It reports the values as
`chip_id` and `chip_revision` and does not access forwarding or VLAN controls.

The patched Linux 6.8.12 kernel built successfully with:

```sh
make O=/work/rtl8370-debug-output BR2_EXTERNAL=/mnt/tp2bmc linux
```

The diagnostic firmware was subsequently assembled, flashed, and run on the
BMC. The validation command is:

```sh
cat /sys/kernel/debug/rtl8370_identity
```

### 2026-07-26: Live In-Kernel Probe

The diagnostic image was built, checksum-verified, applied through `tpi
firmware`, and activated with a BMC reboot. The BMC returned with build time
`2026-07-26 17:40:05-00:00`; all four RK1 nodes again responded to ICMP.

The first image exposed an invalid assumption about debugfs private data and
caused a recoverable NULL-pointer oops in the reader process. It did not alter
switch state or interrupt node connectivity. The replacement image uses an
explicit TP2 switch context, validates its regmap before reading, and completed
without an oops.

The live read-only report was:

```text
chip_id=0x0000
chip_revision=0x0000
```

During the same boot the driver independently reported `found an RTL8370MB-CG`
twice. Review of `rtl8365mb_get_chip_id_and_ver()` explains the apparent
conflict: the driver first writes `0x0249` to selector register `0x13c2`, reads
`0x1300` and `0x1301`, and then clears `0x13c2`. The diagnostic omitted that
sequence. The production chip table identifies RTL8370MB-CG as chip ID `0x6368`
and revision `0x0010`. A corrected report must reuse the exact bounded sequence;
the zero values do not invalidate the identity registers or RTK-I2C transport.

### 2026-07-26: RTL8370MB-CG Datasheet

`RTL8370MB.pdf` is the exact RTL8370MB draft datasheet, revision 0.2 dated
2015-08-11, and its ordering table names the `RTL8370MB-CG` package. This is
direct evidence for the target rather than an RTL8365 or RTL8370N family proxy.

The datasheet verifies:

- Eight integrated copper ports 0-7 and extension GMAC ports 8-9.
- External register access via RTK-I2C when strap bits `SMI_SEL_1:0` are `11`.
- A 16-bit register address and 16-bit register value over the management bus.
- A 4096-entry VLAN table with per-VLAN membership and untag definitions.
- Port-based and 802.1Q tag-aware VLAN classification, per-port PVID, tagged-only
  admission, and member-set ingress filtering.
- Independent controls to discard leaky VLAN frames and multicast VLAN frames
  between VLAN domains.
- IGMPv1/v2/v3 and MLDv1/v2 snooping, with the CPU responsible for installing
  multicast lookup entries.

The datasheet does not publish numeric addresses or bit layouts for VLAN,
PVID, ingress-filtering, multicast-leakage, or table-access registers. Those
values still require a matching public SDK source or carefully bounded live
read validation. No VLAN register write is justified by the datasheet alone.

### 2026-07-26: Public RTL8370MB SDK Sources

Two public source trees contain Realtek SDK code selected or packaged for the
RTL8370MB:

- `MirrShad/RTL8370SmallDemo` has a complete `RTL8370MB/API` directory,
  including the generated register map and VLAN implementation.
- `kelemvor4/er8411` selects `CONFIG_RTL8370MB` and builds the same
  `rtl8367c_*` API. Its configuration names this path `FORCE_PROBE_RTL8370B`,
  and its switch probe maps chip ID `0x6368` to `CHIP_RTL8370B`.

The generated maps in both trees agree on the following direct registers and
the VLAN implementation uses them consistently:

| Register range | SDK meaning |
| --- | --- |
| `0x0700-0x0705` | Two 5-bit VLAN member-configuration indexes per port |
| `0x0728-0x07a7` | 32 VLAN member configurations, four words per entry |
| `0x07a8` | Global CVLAN filtering enable, bit 0 |
| `0x07a9` | Per-port ingress filtering enable, bits 0-10 |
| `0x07aa-0x07ab` | Two-bit accepted-frame mode per port |
| `0x0500` | Indirect table command and table type |
| `0x0501` | Indirect table address |
| `0x0502` | Indirect table status |
| `0x0510-0x0519` | Indirect table write data |
| `0x0520-0x0529` | Indirect table read data |

The member-configuration reader performs ordinary reads across
`0x0728 + index * 4` through `+3`. In contrast, reading a VLAN4K entry requires
writes to the indirect address and command registers before reading its data.
Therefore the next VLAN diagnostic may read only `0x0700-0x07ab`; indirect
table access remains deferred until the corrected identity image has been
validated and serial recovery is confirmed.

These sources substantially improve confidence in the register addresses, but
they do not eliminate the need for live read validation: both APIs retain the
`rtl8367c` naming and classify `0x6368` internally as RTL8370B rather than
providing a separately generated RTL8370MB register model.

### 2026-07-26: Live Identity and Reversible VLAN4K Probe

The corrected identity sequence returned chip ID `0x6368` and revision
`0x0010`. Direct reset-state inspection found zeroed VLAN4K entries and VLAN
membership configurations before DSA bridge setup. A reversible write to
unused VID 4094 wrote `0000 4000 0000`, read it back, restored the original
`0000 0000 0000`, and left all four nodes reachable.

### 2026-07-26: VLAN Driver and RTK-I2C Transport Fix

The VLAN implementation is based on the upstream RTL8365MB series and retains
the TP2 RTK-I2C, CPU-tagging, and bridge-isolation changes. Linux 6.8
compatibility required explicit mask/shift handling and a composite driver
object. `CONFIG_VLAN_8021Q` and `CONFIG_BRIDGE_VLAN_FILTERING` are enabled.

The first VLAN-enabled boot consistently failed to attach `node3` and `ge0`.
VLANMC entry 1 contained malformed repeated words (`0007 0003 0003 0003`).
The upstream implementation used `regmap_bulk_read()` and
`regmap_bulk_write()`, but TP2's RTK-I2C transport accepts only one register per
transaction. `rtl8370-vlan-transport-fix.patch` replaces all five bulk table
and VLANMC accesses with individual reads and writes. The VLAN mutex is also
initialized in the TP2 I2C probe.

After deploying the transport fix, all `node1` through `node4`, `ge0`, and
`ge1` ports attached to `br0` without errors. All four RK1 management addresses
recovered within eight seconds. VLANMC entry 1 then read back correctly as
`006f 0000 0000 0001`, representing VLAN 1 membership on all six external
ports.

The first VLAN4K diagnostic incorrectly used `regmap_update_bits()` to issue
the indirect table command. Regmap suppresses the bus write when the register
already contains the requested value, but each command-register write is the
trigger for a new table operation. Consecutive reads therefore returned stale
data and made valid driver writes appear empty. The corrected diagnostic
preserves unrelated command-register bits but always performs a register
write.

With forced commands, the live table readback is:

```text
vlan4k[1]=7f7f 4000 0000
vlan4k[20]=000f 4000 0000
```

VID 1 contains ports 0-6 as members and untagged egress ports. VID 20 contains
only node ports 0-3 and uses tagged egress. The BT Hub uplink on port 6 is not
a VID 20 member.

### 2026-07-27: Filtering and Containment Validation

Filtering was first tested on disconnected `ge0`. With VID 20 active, the
hardware entry read `0020 4000 0000` and ingress filtering bit 5 was set. The
test rolled back to an empty VID 20 member mask and cleared the ingress bit.

The live `br0` bridge was then configured with tagged VID 20 on `node1` through
`node4`, leaving `ge0` and `ge1` in VID 1 only. Enabling bridge VLAN filtering
produced:

```text
vlan_ctrl[0x07a9]=0x006f
port_misc[0-3]=0x4880
port_misc[4]=0x48b0
port_misc[5-6]=0x4880
vlan4k[1]=7f7f 4000 0000
vlan4k[20]=000f 4000 0000
```

All four management addresses remained reachable. Temporary node interfaces
were assigned `10.20.0.1/24` through `10.20.0.4/24`; bidirectional unicast
completed with zero loss.

Ten UDP multicast datagrams sent to `239.192.0.1:9400` from `10.20.0.1` were
received by each of the other three nodes. A subsequent 500-packet, 50-pps
VLAN 20 broadcast containment test increased the participating node-port
counters by approximately 500 packets while the `ge1` transmit counter
increased only by its normal background traffic. This agrees with the hardware
member mask, which excludes the BT Hub uplink.

### 2026-07-27: Throughput Validation

A sequence-numbered UDP multicast harness sent 1,400-byte payloads for ten
seconds per stage while three nodes received, the BMC continuously probed all
management addresses, and the `ge1` transmit counter was monitored.

| Rate | Sender | Receivers | Result |
| --- | --- | --- | --- |
| 10 Mbps | `tpn1` | `tpn2-4` | 8,644 packets each, zero loss |
| 50 Mbps | `tpn1` | `tpn2-4` | `tpn3-4` lossless; `tpn2` lost 7-8 of 43,222 packets |
| 100 Mbps | `tpn2` | `tpn1`, `tpn3-4` | 86,445 packets each, zero loss |
| 250 Mbps | `tpn2` | `tpn1`, `tpn3-4` | 216,113 packets each, zero loss |
| 500 Mbps | `tpn2` | `tpn1`, `tpn3-4` | 432,226 packets each, zero loss |

The small 50 Mbps loss was isolated to `tpn2` as a receiver and repeated after
raising its socket receive buffer from 416 KiB to 32 MiB. It had no UDP buffer,
NIC, or DSA error counters and was carrying substantially more unrelated host
traffic than the other receivers. Rotating `tpn2` to the sender produced
lossless reception through 500 Mbps, so this is a host receive-scheduling
effect rather than a switch forwarding limit.

Every stage had zero management probe failures. At 500 Mbps, `ge1` increased by
only 189 ambient packets rather than the 432,226 multicast packets. The switch
delivered about 1.5 Gbps of aggregate replicated node egress while maintaining
hardware containment. Temporary receive-buffer settings and test scripts were
removed after validation.

Attempts to request 800 Mbps exposed the RK1 traffic-generator ceiling. The
Python sender sustained 616 Mbps and a compiled C sender with batched
`sendmmsg()` calls sustained 619 Mbps; both completed the requested packet
count but required about 13 seconds instead of 10. A dual-source 400+400 Mbps
attempt reached approximately 617 Mbps aggregate because each source also had
to receive the other source's flooded multicast, and one management probe
timed out under that host load. The Hub uplink remained contained throughout.

A 900 Mbps stage was not run because the available RK1 sources cannot generate
that rate reliably. Establishing an 800-900 Mbps switch limit requires an
external line-rate generator or hardware traffic generator; labeling the
current approximately 619 Mbps source ceiling as an 800/900 Mbps test would be
incorrect.

### Persistent Configuration

`/etc/init.d/S41vlan` waits for `br0` and all DSA ports, adds tagged VID 20 to
the four node ports, and then enables bridge VLAN filtering. Any setup failure
leaves filtering disabled. Its stop path disables filtering before deleting
VID 20.

Each RK1 node has `/etc/netplan/60-vlan20.yaml`, generated from these settings:

| Node | Interface | Address |
| --- | --- | --- |
| `tpn1` | `end0.20` | `10.20.0.1/24` |
| `tpn2` | `end1.20` | `10.20.0.2/24` |
| `tpn3` | `end1.20` | `10.20.0.3/24` |
| `tpn4` | `end1.20` | `10.20.0.4/24` |

The final BMC image was reboot-tested. Filtering returned automatically,
hardware VIDs 1 and 20 matched the expected masks, all management nodes
recovered within eight seconds, and VLAN 20 unicast connectivity passed.

Emergency runtime rollback on the BMC is:

```sh
ip link set br0 type bridge vlan_filtering 0
```
