# VLAN 20 Application Tuning Guide

## Purpose

This guide covers application-side UDP multicast configuration and performance
tuning for the private VLAN 20 network on the Turing Pi 2 RK1 nodes.

The switch and VLAN path have been validated through 950 Mbps. Application
throughput still depends on packet size, syscall rate, socket buffering, CPU
placement, NIC descriptor rings, and how the application batches packets.

## Network Layout

| Node | VLAN interface | VLAN address | Physical interface |
| --- | --- | --- | --- |
| `tpn1` | `end0.20` | `10.20.0.1/24` | `end0` |
| `tpn2` | `end1.20` | `10.20.0.2/24` | `end1` |
| `tpn3` | `end1.20` | `10.20.0.3/24` | `end1` |
| `tpn4` | `end1.20` | `10.20.0.4/24` | `end1` |

The standard multicast endpoint used during validation was:

```text
Group: 239.192.0.1
Port:  9400
TTL:   1
```

VLAN 20 is present only on the four node-facing switch ports. The BT Hub
uplink is not a VLAN 20 member.

## Application Model

Applications do not manually attach an 802.1Q header when using a normal UDP
socket. They select the VLAN interface or its IP address, and Linux adds VLAN
tag 20 through `end0.20` or `end1.20`.

The address and VLAN ID are independent. `10.20.0.1` uses VLAN 20 because that
address belongs to an interface created with `vlan id 20`, not because the IP
address contains the number 20.

## Confirm The Interface

Before starting an application, verify the local configuration:

```sh
ip -d link show end1.20
ip -4 addr show end1.20
ip route get 10.20.0.3 from 10.20.0.2
```

Use `end0.20` on `tpn1` and `end1.20` on the other nodes.

The VLAN address must be present before the application starts:

```sh
ip -4 -br addr show end1.20
```

## Python Sender

This is an appropriate baseline for control traffic and moderate data rates:

```python
import socket
import struct

GROUP = "239.192.0.1"
PORT = 9400
VLAN_IP = "10.20.0.1"

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Select the VLAN 20 interface for outgoing multicast.
sock.setsockopt(
    socket.IPPROTO_IP,
    socket.IP_MULTICAST_IF,
    socket.inet_aton(VLAN_IP),
)

# The switch provides containment; TTL 1 also prevents routed propagation.
sock.setsockopt(
    socket.IPPROTO_IP,
    socket.IP_MULTICAST_TTL,
    struct.pack("B", 1),
)

# Request a larger send buffer. Linux may cap this at net.core.wmem_max.
sock.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, 4 * 1024 * 1024)

sock.sendto(b"hello over VLAN 20", (GROUP, PORT))
```

To force the interface by name on Linux:

```python
sock.setsockopt(
    socket.SOL_SOCKET,
    socket.SO_BINDTODEVICE,
    b"end0.20\0",
)
```

`SO_BINDTODEVICE` may require additional privileges. For multicast,
`IP_MULTICAST_IF` with the VLAN IP is normally sufficient.

## Python Receiver

Join the multicast group on the VLAN address, not the management address:

```python
import socket

GROUP = "239.192.0.1"
PORT = 9400
VLAN_IP = "10.20.0.2"

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 16 * 1024 * 1024)
sock.bind(("", PORT))

membership = socket.inet_aton(GROUP) + socket.inet_aton(VLAN_IP)
sock.setsockopt(
    socket.IPPROTO_IP,
    socket.IP_ADD_MEMBERSHIP,
    membership,
)

def process(payload, sender):
    # Enqueue or process the datagram without per-packet console logging.
    pass

while True:
    payload, sender = sock.recvfrom(65535)
    process(payload, sender)
```

Binding to `0.0.0.0` allows the multicast socket to receive group traffic.
`IP_ADD_MEMBERSHIP` determines the group and local interface.

## C Socket Setup

The equivalent sender configuration in C is:

```c
int fd = socket(AF_INET, SOCK_DGRAM, 0);
struct in_addr vlan_address;
unsigned char ttl = 1;
int send_buffer = 16 * 1024 * 1024;

inet_aton("10.20.0.1", &vlan_address);

setsockopt(fd, IPPROTO_IP, IP_MULTICAST_IF,
           &vlan_address, sizeof(vlan_address));
setsockopt(fd, IPPROTO_IP, IP_MULTICAST_TTL,
           &ttl, sizeof(ttl));
setsockopt(fd, SOL_SOCKET, SO_SNDBUF,
           &send_buffer, sizeof(send_buffer));
```

Receiver membership in C:

```c
struct ip_mreq membership;
int receive_buffer = 16 * 1024 * 1024;

inet_aton("239.192.0.1", &membership.imr_multiaddr);
inet_aton("10.20.0.2", &membership.imr_interface);

setsockopt(fd, SOL_SOCKET, SO_RCVBUF,
           &receive_buffer, sizeof(receive_buffer));
setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP,
           &membership, sizeof(membership));
```

Check every socket call and fail immediately on errors. Also read the buffer
size back with `getsockopt()`: Linux doubles the requested value internally and
caps it according to the host sysctl.

## Packet Size

Use payloads around 1,400 bytes unless the application has a strong reason to
use smaller datagrams.

Advantages of larger packets:

- Fewer syscalls per gigabit.
- Fewer SKBs and DMA descriptors.
- Lower interrupt and packet-processing overhead.
- Better useful-data-to-header ratio.

Keep the complete IPv4 packet within the VLAN interface MTU. For a normal
1,500-byte IPv4 MTU, the theoretical maximum UDP payload is 1,472 bytes:

```text
1500 - 20-byte IPv4 header - 8-byte UDP header = 1472 bytes
```

Using 1,400 bytes leaves room for application headers and avoids edge cases.
Do not depend on IP fragmentation for a high-rate stream.

## Packet Rate Matters

The cost is often packets per second rather than bits per second.

At approximately 950 Mbps with 1,400-byte test packets, the sender processes
roughly 85,000 packets per second. A one-syscall-per-packet design therefore
performs roughly 85,000 sends per second before accounting for receiver work.

Small packets can hit the host packet-rate ceiling long before reaching one
gigabit of payload throughput.

## Socket Buffer Tuning

Inspect the current limits:

```sh
sysctl net.core.rmem_max
sysctl net.core.wmem_max
sysctl net.core.netdev_max_backlog
```

A reasonable starting point for a dedicated high-rate application is:

```sh
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl -w net.core.wmem_max=16777216
sudo sysctl -w net.core.netdev_max_backlog=8192
```

Then request matching socket buffers in the application.

For persistent settings, use a dedicated file such as:

```text
/etc/sysctl.d/60-vlan20-udp.conf
```

Example contents:

```ini
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 8192
```

Apply it with:

```sh
sudo sysctl --system
```

Large buffers absorb temporary scheduling delays. They do not fix a sender
that cannot produce packets quickly enough or a receiver that cannot process
them before the buffer fills.

## Batch System Calls

For C and C++ applications, prefer batching:

- Use `sendmmsg()` instead of one `sendto()` per packet.
- Use `recvmmsg()` instead of one `recvfrom()` per packet.
- Process several datagrams before returning to unrelated work.
- Preallocate packet buffers and message structures.
- Avoid allocation, formatting, logging, and locking in the packet loop.

Example sending pattern:

```c
struct mmsghdr messages[BATCH_SIZE];

/* Initialize msg_name, msg_iov, and msg_iovlen once. */

int sent = sendmmsg(fd, messages, batch_count, 0);
if (sent < 0)
    perror("sendmmsg");
```

Batching reduces syscall overhead, but the current RK1 kernel/NIC path still
showed a roughly 619 Mbps ceiling for ordinary userspace UDP during testing.
Do not assume batching alone will reach line rate.

## UDP Segmentation Offload

The NIC reports `tx-udp-segmentation: on`, but multicast `UDP_SEGMENT` sends
returned `EIO` during validation. Treat multicast UDP GSO as unavailable on the
current RK1 kernel and stmmac driver until a targeted driver fix is validated.

Applications must gracefully fall back instead of assuming `UDP_SEGMENT` works.

## NIC Descriptor Ring

The default RK1 TX ring has 512 descriptors. High-rate validation became
reliably lossless after increasing it to 1,024:

```sh
sudo ethtool -g end1
sudo ethtool -G end1 tx 1024
```

Use `end0` on `tpn1` and `end1` on the other nodes.

This setting is runtime-only unless applied by a boot service. Verify support
before applying it because ring limits are driver-specific.

Restore the default used during testing with:

```sh
sudo ethtool -G end1 tx 512
```

## CPU And IRQ Placement

At high packet rates, keep packet generation and NIC interrupt processing on
different fast cores.

Find the NIC IRQs dynamically because IRQ numbers can change after a reboot:

```sh
grep -i end1 /proc/interrupts
```

During validation, the relevant IRQs were moved to big core 6 and the generator
ran on big core 4:

```sh
echo 6 | sudo tee /proc/irq/IRQ_NUMBER/smp_affinity_list
taskset -c 4 ./your-sender
```

Do not hard-code the observed IRQ numbers into an application. Discover them
from `/proc/interrupts` or use a host-specific startup script.

Check effective placement:

```sh
cat /proc/irq/IRQ_NUMBER/effective_affinity_list
```

Receiver applications should similarly use a dedicated core that is separate
from the NIC IRQ core when possible.

## CPU Frequency

For controlled throughput tests or dedicated appliances, confirm that the CPU
is not using a low-frequency power-saving governor:

```sh
grep . /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
grep . /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq
```

If appropriate for the deployment, select the performance governor through the
system's normal CPU-frequency management tooling. Monitor temperature before
making this persistent.

## Receiver Architecture

A high-rate receiver should separate packet ingestion from expensive work.

Recommended flow:

```text
recvmmsg() batch
    -> validate fixed header
    -> append to bounded lock-free/SPSC queue
    -> worker thread performs decoding, storage, or rendering
```

Do not perform these operations in the receive loop:

- Per-packet console logging.
- Dynamic memory allocation.
- JSON encoding or decoding.
- Filesystem writes.
- Blocking RPC calls.
- Unbounded queue growth.

If the worker cannot keep up, define an explicit drop policy and expose a
counter. Silent latency growth is generally worse than controlled packet loss
for a real-time multicast stream.

## Sequence Numbers

Include a monotonic 64-bit sequence number in every datagram. This allows the
receiver to distinguish:

- Network or sender loss.
- Receiver buffer overflow.
- Reordering.
- Duplicate packets.
- Missing packets at stream startup or shutdown.

Also include a stream identifier when multiple senders or logical streams use
the same multicast group.

## Pacing

Sending as fast as possible creates large bursts and can overrun a descriptor
ring even when the average rate appears safe.

Preferred strategies:

- Pace batches against `CLOCK_MONOTONIC`.
- Use a bounded token bucket.
- Avoid long sleeps followed by large catch-up bursts.
- Record actual bytes and elapsed monotonic time.
- Report requested and achieved rates separately.

The application should never claim an 800 Mbps test when it requested 800 Mbps
but only generated 619 Mbps.

## Kernel-Bypass Options

The validated switch path reaches 950 Mbps, but ordinary userspace UDP sockets
did not reach that rate on the current RK1 software stack.

For applications that genuinely require near-line-rate multicast, evaluate:

- AF_XDP with a dedicated queue and preallocated UMEM.
- `PACKET_TX_RING`/`PACKET_RX_RING` for raw packet applications.
- DPDK where operational complexity is acceptable.
- A kernel producer for fixed-format test or appliance traffic.

These approaches require more ownership of Ethernet, VLAN, IP, UDP, checksums,
permissions, and lifecycle management. Use them only when optimized normal
sockets cannot meet the measured requirement.

## Performance Profiles

### Baseline Profile

Use this first:

- VLAN interface selected with `IP_MULTICAST_IF`.
- TTL set to 1.
- Payload around 1,400 bytes.
- Sequence number in every packet.
- 4-16 MiB requested socket buffers.
- No per-packet allocation or logging.

This profile is suitable for moderate rates and simple applications.

### High-Rate Socket Profile

Add these changes:

- `sendmmsg()` and `recvmmsg()` batching.
- 16 MiB host socket-buffer ceilings.
- Dedicated sender and receiver CPU cores.
- NIC IRQ on a separate core.
- TX ring increased to 1,024 descriptors.
- Continuous loss, backlog, and achieved-rate metrics.

Measure the actual application. The test environment observed an ordinary UDP
ceiling near 619 Mbps even after userspace batching.

### Near-Line-Rate Profile

Use when the requirement is close to 950 Mbps:

- Kernel pktgen for synthetic validation.
- AF_XDP, packet mmap, DPDK, or another optimized production data path.
- 1,024-entry TX ring.
- Explicit big-core and IRQ placement.
- Large, non-fragmented packets.
- A dedicated soak test under production-like background load.

## Measured Results

With kernel pktgen, a 1,024-entry TX ring, pktgen on big core 4, and NIC IRQs on
big core 6:

| Requested | Generated | Receiver result |
| --- | --- | --- |
| 800 Mbps | 799.92 Mbps | 714,281 of 714,286 packets per receiver |
| 900 Mbps | 900.02 Mbps | 803,571 of 803,571 packets per receiver |
| 925 Mbps | 924.91 Mbps | 825,892 of 825,892 packets per receiver |
| 950 Mbps | 949.94 Mbps | 848,214 of 848,214 packets per receiver |
| 955 Mbps | 954.96 Mbps | 852,621 of 852,678 packets per receiver |
| 960 Mbps | 959.85 Mbps | 857,048-857,049 of 857,142 packets |
| 975 Mbps | 969.96 Mbps | 868,211-868,212 of 870,535 packets |

A 30-second 950 Mbps soak generated 2,544,642 packets. Every receiver counted
2,544,639 packets, management remained reachable, and no VLAN 20 traffic was
observed on the BT Hub uplink.

The practical ceiling for the tested packet profile is 950 Mbps. Loss begins
above that point, and physical saturation occurs near 970 Mbps after Ethernet
framing overhead.

## Monitoring During A Test

Interface counters:

```sh
ip -s link show end1
ip -s link show end1.20
```

UDP stack counters:

```sh
nstat -az | grep -E 'Udp(InDatagrams|InErrors|RcvbufErrors|SndbufErrors)'
```

NIC details:

```sh
ethtool -S end1
ethtool -g end1
ethtool -k end1
```

Soft-network backlog indicators:

```sh
cat /proc/net/softnet_stat
```

Application metrics should include:

- Packets generated.
- Packets received.
- Missing sequence numbers.
- Reordered and duplicate packets.
- Requested rate.
- Actual rate based on bytes and monotonic elapsed time.
- Socket-buffer size actually granted.
- Queue depth and application-level drops.

Continue checking management connectivity while load is active.

## Acceptance Test

Validate a production application in stages:

1. Confirm management connectivity and VLAN 20 unicast.
2. Run multicast at 10 Mbps for at least 30 seconds.
3. Increase to 100 Mbps and verify sequence counters.
4. Increase to the expected production rate.
5. Run a sustained soak with normal background workloads.
6. Confirm `ge1` remains at background traffic.
7. Restart the application and verify clean group rejoin behavior.
8. Reboot one node and verify persistent interface configuration.

Stop increasing the rate when loss, queue growth, management degradation, or
unexpected uplink traffic appears. Diagnose that boundary before continuing.

## Troubleshooting

### No Packets Received

Check:

```sh
ip -4 addr show end1.20
ip maddr show dev end1.20
bridge vlan show
```

Verify that the receiver joined the group using its `10.20.0.x` address rather
than the `192.168.1.x` management address.

### Sender Plateaus Near 600 Mbps

This matched the observed one-datagram-per-syscall UDP ceiling.

Check packet size, batching, CPU usage, TX ring size, IRQ placement, and actual
socket buffer size. If those are already optimized, move to AF_XDP or another
lower-overhead data path rather than adding more application threads blindly.

### Receiver Loses Packets But NIC Counters Are Clean

Check:

- `UdpRcvbufErrors`.
- Granted `SO_RCVBUF` size.
- Receive-loop CPU utilization.
- Per-packet logging or allocation.
- Worker queue depth.
- Competing workloads on the same core.

### All Receivers Lose The Same Packets

The loss is likely before switch replication, usually in the sender, its TX
ring, or the sender-facing NIC path. Inspect sender errors and descriptor-ring
pressure before investigating each receiver independently.

### Management Is Reachable But VLAN 20 Is Not

Management uses VLAN 1, so this usually indicates a missing `.20` interface,
wrong group membership interface, or missing VLAN 20 switch membership.

## Safety And Rollback

Application tuning should not require changing the BMC switch configuration.

Restore temporary host tuning after experiments when it is not intended for
production:

```sh
sudo ethtool -G end1 tx 512
sudo sysctl -w net.core.rmem_max=212992
sudo sysctl -w net.core.wmem_max=212992
```

Restore IRQ affinity according to the host's normal policy or reboot the node.

Emergency switch-side rollback on the BMC remains:

```sh
ip link set br0 type bridge vlan_filtering 0
```

That command removes hardware VLAN containment and should be used only for
recovery, not as an application performance workaround.
