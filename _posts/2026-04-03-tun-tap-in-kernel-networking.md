---
title: "TUN/TAP in kernel networking"
date: 2026-04-03
tags: [linux, networking, kernel]
---

Notes from a long chat I had walking up the TUN/TAP abstraction in Linux. I'm leaving the Q&A chain intact — each section is one follow-up — because the order matters: each question is a confusion the previous answer surfaced.

## What is TUN/TAP?

A physical NIC sits between the kernel's network stack (software) and the wire (hardware). TUN and TAP are *virtual* network drivers. Instead of being backed by a wire, they're backed by a user-space program. They let the kernel "trick" applications into thinking they're sending data over a wire, while the data is actually being redirected to another piece of software.

The primary distinction:

| | TUN | TAP |
| --- | --- | --- |
| OSI Layer | L3 (IP) | L2 (Ethernet) |
| Data unit | IP packets | Ethernet frames |
| Behaves like | Point-to-point link | Bridge / switch port |
| Use case | VPNs, routing, NAT | VMs, virtual switches |

The data flow:

1. Application A sends a packet to some IP.
2. The routing table sees that IP belongs to `tun0` / `tap0`.
3. Kernel puts the packet in a queue.
4. Application B (the VPN or VM software) has a file descriptor open to `/dev/net/tun` and `read()`s the packet from that queue.
5. Application B can then encrypt it, wrap it in another protocol, or send it across the internet.

## Can you create a netdev on demand?

Yes — provided the kernel module is loaded and you have the right privileges.

Manually via CLI:

```bash
sudo ip tuntap add mode tun dev tun0
sudo ip tuntap add mode tap dev tap0
sudo ip link set dev tun0 up
```

Programmatically (the way OpenVPN, WireGuard, QEMU do it):

1. Open `/dev/net/tun`.
2. Issue `ioctl(fd, TUNSETIFF, &ifr)`.
3. Kernel creates a new virtual netdev in memory instantly.
4. By default, the device is destroyed when the file descriptor closes (unless marked persistent).

## Are netdevs different from "Linux devices"? And what's the difference between an interface and a netdev?

Yes — the biggest difference is that a netdev can exist entirely as a software construct in the kernel without any hardware backing it. Most Linux devices show up as files in `/dev/` (e.g. `/dev/sda`); **netdevs do not have entries in `/dev/`**. They live in the networking subsystem as instances of `struct net_device` — a struct in memory the kernel can conjure out of thin air. A loopback (`lo`) interface, like a TUN/TAP device, has no hardware behind it.

"Interface" and "netdev" are usually used interchangeably, but technically they're two views of the same thing:

- **netdev** (`struct net_device`): the low-level kernel object. Owns the MAC address, MTU, DMA buffers, driver code. Concerned with *how* to send a packet.
- **interface**: the L3 attachment point — IP address, subnet, routing rules. Concerned with *where* the packet is going.

You can have a netdev that is "up" but has no IP, in which case it can see L2 traffic but cannot participate in IP networking.

## How does the networking subsystem bridge with the device subsystem?

Through a generic networking layer. Three pieces:

1. **`struct net_device`** — every interface (physical or virtual) is one of these. The driver registers it via `register_netdev()`, telling the networking subsystem "I'm `eth0`; route packets for this MAC to me."
2. **`sk_buff`** (socket buffer) — the data carrier. Instead of copying packet bytes between layers, the kernel passes pointers to `sk_buff`s. As an `skb` goes down the stack, headers get prepended; as it goes up, headers are stripped.
3. **`netdev_ops`** — function pointer table. The networking subsystem calls `ndo_start_xmit(skb, dev)` to transmit; the driver-specific implementation is what actually pushes the `skb` to hardware (or to user space, for TUN/TAP).

When packets arrive: hardware fires an IRQ, the driver acknowledges and schedules a softirq, then NAPI takes over and polls the driver's ring buffer in bulk to avoid drowning the CPU in interrupts.

## What syscall does `ip` use to create a netdev?

`ip` (from `iproute2`) talks to the kernel via **Netlink**, not `ioctl` like the older `ifconfig`. The flow is:

1. `socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)`
2. `sendmsg(fd, …)` with a Netlink message of type `RTM_NEWLINK`, with an `IFLA_LINKINFO` attribute describing the device kind ("vlan", "veth", "bridge", "dummy", …).
3. `recvmsg(fd, …)` to read the kernel's ack.

Netlink is a specialized IPC mechanism for the networking subsystem. It's asynchronous, multicast-capable (kernel can broadcast "new interface!" to listeners), and supports nested attributes — all things `ioctl` is bad at.

You can watch this with `strace -e socket,sendmsg,recvmsg ip addr show`.

For TUN/TAP specifically, though, creation goes through `ioctl(fd, TUNSETIFF, &ifr)` on a file descriptor opened against `/dev/net/tun`. That's because TUN/TAP devices straddle the line between a `/dev` character device and a netdev — you need an `ioctl` to bind the file descriptor to a freshly-created netdev.

| | Syscall | Flag |
| --- | --- | --- |
| General netdev (bridge, veth) | `sendmsg` | `RTM_NEWLINK` |
| TUN/TAP | `ioctl` | `TUNSETIFF` |
| Old `ifconfig` | `ioctl` | `SIOCSIFADDR` |

## Going deeper on TUN: does it just do TCP/IP processing and hand off to a Layer-2 backend?

No — **TUN doesn't have an L2 backend at all**. The kernel treats a TUN device as a point-to-point link with no Ethernet, no MAC, no ARP. When an application reads from a TUN file descriptor, the very first byte it sees is the IP header (e.g. `0x45` for IPv4). There is no 14-byte Ethernet header.

TAP, by contrast, mimics a physical Ethernet card — it speaks L2. Reading from a TAP gives you a full Ethernet frame: dest MAC, src MAC, EtherType, then payload. The kernel will issue ARP requests on a TAP device, expecting some peer to respond.

That's also why you can put a TAP into a Linux bridge but not a TUN: bridges switch on MAC, and TUN doesn't speak Ethernet.

One small detail: when reading from `/dev/net/tun`, the kernel can optionally prepend a 4-byte "Packet Information" (PI) header — metadata, not a network header.

## Wait — TUN/TAP aren't netdevs? They're devices? Apps usually don't write to interfaces directly; they use sockets and let the routing table pick.

Both, simultaneously. A TUN/TAP instance is *at the same time* a character device file (`/dev/net/tun`) and a netdev (`tun0` / `tap0`).

- To the application using it (the VPN, the VM): it's a device file. The program does `read()` and `write()` on a fd, like a serial port.
- To the kernel networking stack: it's a netdev that shows up in the routing table.

You're right that normal apps don't write to interfaces — they open a socket and the kernel handles the rest. But a TUN/TAP application is acting as the *virtual wire*, not as a normal app. The standard flow is:

```
app → socket → TCP/IP → routing → physical NIC driver → wire
```

The TUN/TAP flow is:

```
app A → socket → TCP/IP → routing → tun0 → TUN driver → fd queue
                                                            ↓
                                                         read()
                                                            ↓
                                                 VPN app B (encrypts, wraps)
                                                            ↓
                                                socket → real NIC → wire
```

If the VPN tried to use a socket to talk to the kernel, you'd get an infinite loop. The `read`/`write` interface to `/dev/net/tun` is a backdoor that lets a user-space program sit *below* the IP stack of the virtual interface. When it writes to the device, the kernel perceives that write as a packet *arriving from the outside* on that interface.

In the source: `net_device` is the netdev structure (what `ip link` shows). `miscdevice` is the character device registration for `/dev/net/tun`.

## What the kernel sends to the NIC is an Ethernet frame, right? That's what sits in the Tx/Rx queues?

Yes — almost always an Ethernet frame.

The Tx/Rx queues themselves don't actually hold the frame bytes. They hold **descriptors** — pointers to where the frame data lives in RAM. The kernel writes a descriptor into the Tx ring; the NIC then DMAs the actual frame out of system RAM.

The frame on the wire has: dest/src MAC, EtherType (e.g. `0x0800` for IPv4), the IP+TCP+payload, and a frame check sequence (FCS). The NIC hardware usually computes the FCS at the last moment to save CPU.

Two exceptions:

- **Loopback**: traffic to `127.0.0.1` doesn't hit a NIC; the kernel can skip framing.
- **TSO/GSO**: the kernel can hand a "super-packet" (e.g. 64KB) to the NIC, and the NIC chops it into MTU-sized frames itself.

## So that's why a VM's host-side backend has to be a TAP, not a TUN — virtio-net data includes Ethernet frames?

Exactly. The `virtio-net` driver inside the guest is built to look like a real Ethernet card to the guest OS. The guest expects to have a MAC address, do ARP, and emit Ethernet frames. So whatever sits on the host side of that virtual wire has to accept full frames — which is what TAP does. A TUN device would see the Ethernet header, not know what to do with it, and drop the packet.

That also makes TAPs easy to plug into a Linux bridge (`br0`), which switches on MAC. The VM ends up looking, to the rest of the network, like a physical machine plugged into a switch.

## What's a bond device?

A virtual interface that aggregates multiple physical NICs into one logical pipe (`bond0`). Two reasons to use it: redundancy (if one cable dies, traffic shifts to another) and throughput (combine multiple links).

The bond interface sits *on top* of the physical interfaces (`eth0`, `eth1`, …). The kernel hands an Ethernet frame to `bond0`; the bonding driver picks which physical NIC to ship it out, depending on the mode:

- **balance-rr (mode 0)**: round-robin across NICs.
- **active-backup (mode 1)**: only one active, others standby. Most common for reliability.
- **802.3ad / LACP (mode 4)**: requires a switch that supports LACP; negotiates link aggregation with it.
- **balance-alb (mode 6)**: load-balancing without switch support, by playing tricks with ARP replies.

A bond is *not* the same as a bridge. A bond makes multiple links look like one link to a single next hop. A bridge is a virtual switch connecting different devices. Common topology in production: physical NICs → `bond0` → `br0` → VM TAP devices.

## What does writing to a TAP device do? Where does it pop out?

It pops out on the netdev side — the kernel sees it as an *incoming* packet on `tap0`.

You write an Ethernet frame into the fd; the TAP driver wraps it into an `sk_buff`; the kernel netstack treats `tap0` as having received a frame. If `tap0` is in a bridge, the bridge then switches it.

Two directions:

- **Userspace → kernel**: app `write()`s a frame to the fd; kernel sees it as RX on `tap0`.
- **Kernel → userspace**: a process pings the VM; the bridge forwards the frame to `tap0`; the TAP driver queues it for the fd; the app's `read()` unblocks.

That's exactly how QEMU works: the guest's `virtio-net` thinks it's talking to real hardware; QEMU sits in host userspace, `read()`ing frames the guest sent and `write()`ing in frames from the outside.

## So this is how a VPN works: the VPN program creates a TUN device. App sends data; kernel adds TCP/IP; routing sends to TUN netdev; the IP packet pops out of `/dev/net/tun` to the VPN daemon; the daemon wraps it in another IP header destined for the VPN server, then sends it over a regular socket — and this time the routing table doesn't intercept it because the destination isn't covered by the TUN route.

That's almost exactly right. The only refinement: the kernel doesn't "intercept" or "not intercept" — it just consults the routing table both times. The routing table is the magic.

The VPN daemon modifies the routing table so that the **default gateway** (`0.0.0.0/0`) points at the TUN device. With one critical exception: a more specific route must exist that sends the **VPN server's IP** through the *physical* gateway. Without that exception, the encrypted-outer packet would itself try to go through the VPN, infinitely.

So the round trip is:

1. App sends an HTTP request → kernel wraps in TCP/IP → routing says "default goes to `tun0`" → packet falls into the TUN queue.
2. VPN daemon `read()`s the packet, encrypts it, wraps it in a new IP+UDP packet addressed to the VPN server.
3. Daemon `write()`s that outer packet via a fresh socket → kernel routes it → routing table hits the *specific* route for the VPN server IP → goes out the physical NIC.

| Layer | Content |
| --- | --- |
| Outer IP | dst = VPN server |
| Transport | usually UDP |
| VPN | keys, IV, etc. |
| Encrypted payload | original IP packet + payload |

## Isn't the overhead high? Instead of one syscall trap (the `send()`), now there are three.

Yes — that's the classical user-space VPN bottleneck. You pay:

1. **Multiple context switches**: app → kernel → VPN daemon → kernel.
2. **Memory copies**: app buffer → kernel → daemon buffer → kernel again after encryption.
3. **Wakeup latency**: every time the TUN queue has data, the daemon has to be scheduled.

This is why modern VPNs avoid it:

- **WireGuard** lives entirely *in the kernel*. The packet enters the virtual interface, gets encrypted right there in the network stack, and goes straight out the physical interface. No userspace round trip — hence why WireGuard benchmarks so much faster than OpenVPN.
- **AES-NI** (CPU instructions) makes the encryption math cheap, leaving context-switching as the dominant cost.
- **Zero-copy techniques** (`mmap`, `vmsplice`) let a userspace daemon see kernel packet memory without copying the bytes — though the context switch still happens.

For a 100 Mbps home connection, you don't notice. For a 10 Gbps backbone, the extra trips are the difference between line rate and meltdown.

## Does Windows have TUN/TAP?

Not natively in the same form, but functionally equivalent third-party drivers are the de facto standard. Windows networking goes through NDIS (Network Driver Interface Specification), so virtual adapters install as NDIS miniports.

- **TAP-Windows6**: the legacy driver every OpenVPN install used to ship. Slow because it carries Ethernet emulation overhead even when used purely for L3.
- **Wintun**: built by the WireGuard project as a modern, minimal alternative. Designed purely as a pipe for IP packets, no Ethernet baggage. Significantly lower overhead — uses shared-memory ring buffers between the driver and the daemon instead of IRP-based reads.

| | Linux | Windows legacy | Windows modern |
| --- | --- | --- | --- |
| Tool | TUN/TAP module | TAP-Windows6 | Wintun |
| Interface | `/dev/net/tun` | NDIS miniport | NDIS miniport |
| Data path | file descriptor | I/O Request Packets | shared-memory rings |

## Does NordVPN use OpenVPN or WireGuard?

Both, but WireGuard is the default. NordVPN's primary protocol is **NordLynx**, which is WireGuard with a Double-NAT layer on top — WireGuard alone has a privacy quirk where the server stores a stable mapping from public key to internal IP, and Double-NAT erases that. They still ship OpenVPN as a fallback for restrictive networks (OpenVPN-over-TCP-port-443 looks like HTTPS, which gets through firewalls that block UDP).

## What does proxy / socket redirection mean? When the program opens a socket, does the kernel see it's a foreign IP and swap it for the proxy server?

That's roughly the kernel-mode (transparent-proxy) version. There are actually two distinct mechanisms:

**Library-level hooking** (`LD_PRELOAD`, eBPF on Linux; LSP / WFP on Windows): the proxy intercepts the standard library's `connect()`. When the app calls `connect(socket, "1.2.3.4")`, the hook redirects to `connect(socket, "127.0.0.1:1080")` — the local proxy. The app *thinks* it's talking to the original destination; it's really talking to a local proxy that then relays.

**Transparent proxying via the kernel**: an `iptables`/`nftables` rule says "if dst matches this set, redirect to local port 1080". The app actually opens a socket to the original IP; the kernel rewrites the destination silently as the SYN goes out. TPROXY is the Linux kernel feature that powers this.

This is "cheaper" than a TUN VPN for two reasons:

- Selective: doesn't touch traffic to non-matching destinations.
- No IP-in-IP wrapping; data is just streamed over an existing encrypted connection at L5, so per-packet header overhead is lower.

| | TUN/TAP VPN | Socket redirection |
| --- | --- | --- |
| Intercept point | network layer | session layer |
| Tool | virtual netdev + routing | iptables / LD_PRELOAD / WFP |
| Overhead | moderate (encapsulation) | low (stream forwarding) |
| App's view | "I'm on a new network" | "I'm talking to the origin directly" |

The mental model: VPNs *route envelopes*; proxies *have someone else rewrite the address before it's mailed*.

## When Linux boots with multiple NICs, how does it pick names like `eth0`?

Historically, the kernel just assigned `eth0`, `eth1`, … in driver-initialization order. Which is unstable on modern hardware — drivers init in parallel, so two identical NICs could swap names across reboots.

So now `udev` (managed by systemd) renames interfaces immediately after the kernel discovers them, using metadata that doesn't change unless you physically move the card. Predictable name prefixes:

- `eno1` — onboard, integrated into firmware (ACPI).
- `ens1` — hotplug PCI-E slot.
- `enp3s0` — by physical PCI bus path. `en` = Ethernet, `p3` = bus 3, `s0` = slot 0.
- `enx78e7d1...` — by MAC address.

The "path" form (`enp3s0`) is the most deterministic: the physical traces don't move, so the NIC at bus 3 / slot 0 always has that name regardless of init order.

If you want the old style back (common on embedded systems and the Pi), set `net.ifnames=0` in the kernel command line.
