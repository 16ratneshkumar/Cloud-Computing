# Network Virtualization

# PART 1: THE THREE FUNDAMENTAL PACKET TECHNIQUES

Every packet processing system uses one or more of these three core techniques:

1. **Packet Processing** — Examining and modifying packet headers/payloads
2. **Packet Bypass** — Avoiding normal kernel networking stack for performance
3. **Packet Filtering** — Allowing or denying packets based on rules

Let's explore each technique across all execution modes and machine types.

---

# TECHNIQUE 1: PACKET PROCESSING

## Definition
Packet processing means reading packet headers, making decisions, and potentially modifying the packet (rewriting headers, encapsulating, decapsulating, fragmenting, etc.).

---

## 1.1 Packet Processing in KERNEL MODE

### What is Kernel Mode?
- Code runs at CPU privilege level 0 (Ring 0)
- Direct hardware access
- No memory protection (crash = system crash)
- Context: interrupt handlers, softirq, kernel threads

### How Packet Processing Works in Kernel

```
Packet arrives at NIC → DMA to memory → Interrupt → Kernel takes over

Step-by-step in kernel:

1. HARDWARE INTERRUPT (hardirq context):
   - NIC raises interrupt line
   - CPU jumps to interrupt handler (NIC driver)
   - Driver acknowledges interrupt
   - Driver schedules NET_RX_SOFTIRQ
   - Returns from interrupt (~1-5µs)

2. SOFTIRQ CONTEXT (ksoftirqd or in-interrupt softirq):
   - net_rx_action() called
   - Driver's poll() function called (NAPI)
   - For each packet:
     a. Allocate sk_buff structure (kernel packet descriptor)
     b. DMA packet data into sk_buff buffer
     c. Set sk_buff metadata:
        - skb->dev = receiving interface
        - skb->protocol = ethertype (0x0800 for IPv4)
        - skb->mac_header, network_header, transport_header offsets
     d. Call netif_receive_skb(skb)

3. PACKET RECEIVE PROCESSING (netif_receive_skb):
   - __netif_receive_skb_core()
   - Check for rx_handler (OVS, bridge, bonding register handlers here)
   - Deliver to protocol handler based on skb->protocol:
     - ETH_P_IP (0x0800) → ip_rcv()
     - ETH_P_ARP (0x0806) → arp_rcv()
     - ETH_P_8021Q (0x8100) → vlan_handler

4. IP LAYER PROCESSING (ip_rcv):
   - Validate IP header checksum
   - Netfilter PREROUTING hook
     iptables rules in PREROUTING chain executed here
   - Routing decision: ip_route_input()
     a. Lookup destination IP in FIB (Forwarding Information Base)
        Routing table stored as LC-trie (level-compressed trie)
        Longest prefix match: O(log n) where n = number of routes
     b. Result: LOCAL (for this host) or FORWARD (route to another interface)

5a. IF LOCAL (packet destined for this host):
    - Netfilter LOCAL_IN hook
    - Deliver to L4 protocol:
      - ip->protocol == 6 (TCP) → tcp_v4_rcv()
      - ip->protocol == 17 (UDP) → udp_rcv()
    - L4 processing finds socket (4-tuple lookup: src_ip, dst_ip, src_port, dst_port)
    - Copy data to socket receive buffer
    - Wake up application waiting on recv() syscall

5b. IF FORWARD (packet being routed):
    - Netfilter FORWARD hook
    - Decrement TTL, recompute IP checksum
    - ARP resolution for next hop (if not in cache)
    - Netfilter POSTROUTING hook
    - Transmit: ip_finish_output() → qdisc (traffic control) → NIC driver

6. NETFILTER HOOKS (iptables/nftables rules):
   - Each hook point (PREROUTING, INPUT, FORWARD, POSTROUTING, OUTPUT)
   - For each rule in chain:
     - Match packet against rule criteria (source IP, dest port, etc.)
     - If match: execute action (ACCEPT, DROP, DNAT, SNAT, LOG, etc.)
     - If no match: continue to next rule
   - Connection tracking (conntrack):
     - Hash table lookup on 5-tuple
     - State: NEW, ESTABLISHED, RELATED, INVALID
     - For SNAT/DNAT: store translation in conntrack table

Data structures:
  struct sk_buff {
      struct sk_buff *next, *prev;       // Linked list
      struct net_device *dev;            // Interface
      unsigned char *head, *data, *tail, *end;  // Buffer pointers
      __u16 transport_header, network_header, mac_header;  // Offsets
      __be16 protocol;                   // Ethertype
      __u32 mark;                        // Netfilter mark
      struct dst_entry *dst;             // Routing decision
  };
```

### Kernel Mode Packet Processing on Different Machines

#### PHYSICAL MACHINE (Kernel Mode)
```
Flow: NIC → DMA → Kernel → Application

Hardware: Real NIC (Intel, Mellanox, Broadcom)
Driver: igb, ixgbe, mlx5_core (kernel modules)
Process:
  1. NIC DMA to RAM (physical address)
  2. Interrupt to CPU
  3. Kernel allocates sk_buff from slab allocator
  4. Full kernel network stack (as described above)
  5. Deliver to application socket buffer

Performance: ~10-50µs latency, 5-10 Gbps per core
```

#### VIRTUAL MACHINE (Kernel Mode)
```
Flow: Physical NIC → Hypervisor Kernel → vSwitch → VM Kernel → Application

TWO KERNELS INVOLVED:
  - Hypervisor kernel (host kernel, KVM/ESXi/Hyper-V)
  - Guest kernel (inside VM)

Example: KVM with virtio-net

HOST SIDE (Hypervisor Kernel):
  1. Physical NIC → DMA → host sk_buff
  2. Host kernel processes (as above)
  3. Packet reaches OVS or Linux bridge (running in host kernel)
  4. Bridge/OVS forwards to tap device or vhost-net
  5. vhost-net kernel module:
     - Writes packet to virtqueue (shared memory ring buffer)
     - Signals VM via eventfd or MSI-X interrupt injection

GUEST SIDE (VM Kernel):
  6. VM receives interrupt from hypervisor
  7. virtio-net driver in guest kernel reads virtqueue
  8. Allocates guest sk_buff
  9. Full guest kernel network stack (ip_rcv, netfilter in guest, etc.)
  10. Delivers to application in VM

Performance: ~50-200µs latency (includes VM exit/entry), 2-5 Gbps
Overhead: Double kernel processing (host + guest)
```

#### CONTAINER (Kernel Mode)
```
Flow: NIC → Kernel → veth → Container netns

SINGLE KERNEL, MULTIPLE NETWORK NAMESPACES:

Process:
  1. Physical NIC → DMA → host kernel sk_buff
  2. Host kernel processes (ip_rcv, routing)
  3. Routing decision: "send to 172.17.0.2" 
     Route lookup: 172.17.0.2/32 dev veth-abc (learned from container)
  4. Packet sent to veth pair:
     - veth is a virtual ethernet pair: veth-host <-> veth-container
     - Packet written to veth-host → appears on veth-container
  5. veth-container exists in container's network namespace
     - Container netns has separate:
       - Routing table
       - iptables rules
       - Interface list
       - Socket table
  6. Kernel processes packet IN THE CONTAINER NETNS:
     - ip_rcv() again (but using container's routing table)
     - iptables rules in container netns
     - Deliver to socket in container's socket table
  7. Application in container receives via normal recv() syscall

Performance: ~10-30µs latency (veth overhead ~5µs), 8-15 Gbps
Key insight: SAME KERNEL, just different namespace contexts
```

#### KUBERNETES POD (Kernel Mode)
```
Flow: NIC → Kernel → CNI plugin logic → veth → Pod netns

K8s pod = container(s) sharing one network namespace

Example: Calico CNI in BGP mode

Process:
  1. Physical NIC → host kernel
  2. Routing lookup: "destination is pod IP 192.168.1.5"
     Route installed by Calico: 192.168.1.5/32 dev cali1234567
  3. cali1234567 is veth pair: cali1234567 (host) <-> eth0 (in pod netns)
  4. Before packet goes to veth, iptables processing:
     - Felix (Calico agent) has installed iptables chains:
       iptables -A cali-tw-cali1234567 -m set --match-set cali-allowed-sources src -j ACCEPT
       iptables -A cali-tw-cali1234567 -j DROP
     - "cali-tw" = "to workload" (ingress to pod)
     - Policy enforcement happens HERE in kernel iptables
  5. If ACCEPT: packet sent to veth pair → enters pod netns
  6. In pod netns:
     - Interface eth0 (the veth end in pod)
     - IP 192.168.1.5/32 assigned to eth0
     - Default route: 169.254.1.1 dev eth0 (magic proxy ARP route)
     - Kernel delivers to application socket

Performance: ~20-100µs (iptables overhead + veth), 5-15 Gbps
Overhead: NetworkPolicy = iptables rule evaluation
```

---

## 1.2 Packet Processing in USER MODE

### What is User Mode?
- Code runs at CPU privilege level 3 (Ring 3)
- Memory protection (segfault instead of kernel panic)
- Cannot directly access hardware
- Must use syscalls to interact with kernel

### How Packet Processing Works in User Mode

```
Application needs to send/receive packets → syscall → kernel

RECEIVING (Application perspective):

1. Application calls recv() or read():
   int sockfd = socket(AF_INET, SOCK_STREAM, 0);
   bind(sockfd, ...);
   listen(sockfd, ...);
   int conn = accept(sockfd, ...);
   recv(conn, buffer, size, 0);  ← SYSCALL

2. SYSCALL MECHANISM:
   - CPU executes syscall instruction
   - Switch from Ring 3 (user) to Ring 0 (kernel)
   - Save user context (registers, stack pointer)
   - Jump to kernel syscall handler
   - Syscall overhead: ~100-200ns

3. IN KERNEL (during syscall):
   - Look up socket by file descriptor
   - Check socket receive buffer
   - If data available:
     - Copy from kernel socket buffer to user buffer
     - Copy overhead: ~1-3µs for small packets
     - Return to userspace with byte count
   - If no data:
     - Put process to sleep (TASK_INTERRUPTIBLE)
     - Scheduler switches to another process
     - When packet arrives: kernel wakes process
     - Return to userspace

4. Return to user mode:
   - Restore user context
   - Application continues with data in buffer

SENDING (Application perspective):

1. Application calls send() or write():
   send(sockfd, buffer, size, 0);  ← SYSCALL

2. IN KERNEL (during syscall):
   - Allocate sk_buff
   - Copy data from user buffer to sk_buff
   - Build TCP/IP headers
   - Pass through network stack (routing, netfilter, qdisc)
   - Hand to NIC driver
   - Return to userspace (may return before actual transmission)

ZERO-COPY OPTIMIZATIONS (reduce user-kernel copying):

sendfile():
  sendfile(socket_fd, file_fd, offset, count);
  - Kernel reads file directly to socket buffer
  - No copy to userspace
  - Used by nginx, Apache for static files

splice():
  - Move data between file descriptors via kernel pipe
  - No userspace involvement

io_uring (modern):
  - Shared ring buffers between kernel and userspace
  - Application writes I/O requests to submission queue
  - Kernel processes, writes results to completion queue
  - Batches syscalls: one syscall for many operations
  - Latency: ~1-2µs vs ~5-10µs for traditional syscalls
```

### User Mode Packet Processing on Different Machines

#### PHYSICAL MACHINE (User Mode)
```
Application runs directly on physical OS

Example: nginx web server

Process:
  1. nginx listens on port 80:
     socket(), bind(), listen(), accept() → kernel creates socket
  2. Client connects → kernel TCP handshake (SYN, SYN-ACK, ACK)
     All in kernel, no nginx involvement yet
  3. Kernel accepts connection → returns new socket to nginx
  4. nginx calls epoll_wait() → blocks waiting for data
  5. Packet arrives → kernel processes (NIC → kernel stack → socket buffer)
  6. Kernel wakes nginx (epoll event)
  7. nginx calls recv() → syscall → kernel copies data to nginx buffer
  8. nginx processes HTTP request (all in userspace)
  9. nginx calls sendfile(socket, file_fd) → kernel sends file
  10. Kernel transmits response → NIC → wire

Performance: ~50-100µs application latency
Benefit: No virtualization overhead
Drawback: Application can crash entire system if it has kernel bug
```

#### VIRTUAL MACHINE (User Mode)
```
Application runs in userspace INSIDE the VM

Example: Application in Ubuntu VM on KVM hypervisor

Process:
  1. Application in VM does socket(), bind(), listen()
     → Syscall to GUEST KERNEL
  2. Guest kernel creates socket (in guest kernel space)
  3. Packet arrives:
     Physical NIC → Host kernel → OVS → vhost-net → virtqueue
  4. Guest kernel interrupt → virtio-net driver → guest sk_buff
  5. Guest kernel processes → delivers to socket buffer (in guest kernel)
  6. Application calls recv() → syscall to guest kernel
     Guest kernel copies from guest socket buffer to application buffer
  7. Application processes in VM userspace

Performance: ~100-300µs application latency
Overhead: 
  - Guest userspace → guest kernel (syscall ~100ns)
  - Guest kernel → host kernel (VM exit ~1-5µs)
  - Host kernel → guest kernel (VM entry ~1-5µs)

Key: Application doesn't know it's in a VM!
```

#### CONTAINER (User Mode)
```
Application runs in userspace in container's namespace

Example: nginx in Docker container

Process:
  1. Container starts:
     docker run -p 8080:80 nginx
     Docker creates network namespace for container
  2. nginx in container does socket(), bind(80), listen()
     → Syscall to HOST KERNEL (same kernel as host!)
     But syscall executes in CONTAINER'S NETWORK NAMESPACE context
  3. Socket created in container's namespace
     Port 80 binding is isolated to container namespace
     (Host can also have something on port 80 - different namespace)
  4. Packet arrives:
     NIC → host kernel → docker0 bridge → veth pair → container netns
  5. Host kernel delivers to socket in container namespace
  6. nginx calls recv() → syscall (to same host kernel)
     Kernel copies to nginx buffer
  7. nginx processes in container userspace (same as any process)

Performance: ~50-150µs application latency
Key difference from physical:
  - Same kernel
  - Just different namespace context
  - Minimal overhead (~5-20µs for veth crossing)

Docker port mapping (8080:80):
  - iptables DNAT rule on host:
    -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
  - External client → host:8080 → DNAT → 172.17.0.2:80 (container)
```

#### KUBERNETES POD (User Mode)
```
Application runs in container(s) in pod

Example: Python Flask app in K8s pod

Process:
  1. Pod starts: kubectl run flask --image=flask-app
     Kubelet calls CNI plugin (e.g., Calico)
     CNI creates veth pair, assigns IP (192.168.1.5)
  2. Application in pod:
     app.run(host='0.0.0.0', port=5000)
     → Syscall to host kernel (in pod's netns context)
  3. Socket bound to 192.168.1.5:5000 (pod IP)
  4. Service created:
     apiVersion: v1
     kind: Service
     spec:
       selector: {app: flask}
       ports: [{port: 80, targetPort: 5000}]
       clusterIP: 10.96.0.100
  5. kube-proxy installs iptables rules:
     -A KUBE-SERVICES -d 10.96.0.100 -p tcp --dport 80 -j KUBE-SVC-XXX
     -A KUBE-SVC-XXX -j DNAT --to-destination 192.168.1.5:5000
  6. Client pod sends to 10.96.0.100:80
     → Host kernel iptables DNAT → 192.168.1.5:5000
     → Routing → cali1234567 veth
     → Delivers to socket in flask pod's netns
  7. Flask app recv() → syscall → data copy
  8. Flask processes request in userspace

Performance: ~100-200µs (iptables + veth + policy)
Components:
  - kube-proxy (userspace daemon managing iptables)
  - CNI plugin (userspace daemon managing routes/policy)
  - Application (userspace in container)
All userspace components talking to kernel via syscalls
```

---

## 1.3 Packet Processing in BYPASS MODE

### What is Bypass Mode?
Techniques that avoid the standard kernel network stack for performance:
- DPDK (userspace poll-mode drivers)
- AF_XDP (XDP sockets)
- Hardware offload (NIC does processing)

### DPDK (Data Plane Development Kit)

```
KEY CONCEPT: Application directly accesses NIC, kernel is OUT of data path

Architecture:
  Application → DPDK libraries → PMD (Poll Mode Driver) → NIC registers → NIC

Setup:
  1. Reserve hugepages (2MB or 1GB pages):
     echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
     Why: Reduces TLB misses, DMA works better with large contiguous memory
  
  2. Unbind NIC from kernel driver:
     echo 0000:01:00.0 > /sys/bus/pci/drivers/ixgbe/unbind
     Why: Kernel can't interfere with DPDK control of NIC
  
  3. Bind NIC to VFIO (Virtual Function I/O) or UIO (Userspace I/O):
     echo vfio-pci > /sys/bus/pci/devices/0000:01:00.0/driver_override
     echo 0000:01:00.0 > /sys/bus/pci/drivers/vfio-pci/bind
     Why: Allows userspace to access PCI device safely
  
  4. Application initializes DPDK EAL (Environment Abstraction Layer):
     rte_eal_init(argc, argv);
     Discovers NICs, maps NIC registers to userspace virtual memory

Packet Processing Flow:
  1. NIC receives packet → DMA to hugepage memory
  2. NIC updates RX descriptor ring (in hugepage memory)
  3. DPDK application POLLS RX ring (while(1) loop, CPU 100% busy):
     uint16_t nb_rx = rte_eth_rx_burst(port_id, queue_id, mbufs, BURST_SIZE);
     - No interrupt
     - No kernel
     - No syscall
     - Just reads memory that NIC updated
  
  4. Application gets rte_mbuf (DPDK packet descriptor):
     struct rte_mbuf {
         void *buf_addr;          // Virtual address of packet data
         rte_iova_t buf_iova;     // Physical/IOVA address (for DMA)
         uint16_t data_off;       // Offset to packet start
         uint16_t pkt_len;        // Packet length
         uint64_t ol_flags;       // Offload flags
         ...
     };
  
  5. Application processes packet (parse headers, classify, modify):
     struct ether_hdr *eth = rte_pktmbuf_mtod(mbuf, struct ether_hdr *);
     struct ipv4_hdr *ip = (struct ipv4_hdr *)(eth + 1);
     // Modify IP destination
     ip->dst_addr = new_dst;
     // Recompute checksum (or use NIC offload)
     
  6. Application transmits:
     rte_eth_tx_burst(port_id, queue_id, &mbuf, 1);
     - Writes to TX descriptor ring
     - NIC DMAs from hugepage
     - Transmits
  
  7. No return to kernel at any point!

Performance:
  - Latency: 1-5µs (application to application)
  - Throughput: 10-24 Mpps per core (64-byte packets)
  - CPU: Core is 100% busy polling (acceptable for dedicated appliances)

Data structures:
  - rte_mbuf: Pre-allocated from mempool (no malloc during processing)
  - rte_mempool: Memory pool of rte_mbufs in hugepage memory
  - Descriptor rings: Circular buffers shared between app and NIC
```

### DPDK on Different Machines

#### PHYSICAL MACHINE (DPDK)
```
Dedicated physical server running DPDK application

Example: VNF router using DPDK

Setup:
  - Physical server with 2x 25G NICs
  - Ubuntu with DPDK packages
  - CPU cores isolated: isolcpus=2-9 (for DPDK)
  
Flow:
  1. Boot: kernel loads on core 0,1
  2. DPDK app starts on cores 2-9:
     ./router -l 2-9 -n 4 -- -p 0x3
     (-l: logical cores, -n: memory channels, -p: port mask)
  3. DPDK binds NIC0 (port 0) and NIC1 (port 1) to vfio-pci
  4. Kernel can no longer see these NICs
  5. Packet arrives at NIC0 → DMA to hugepage → DPDK app polls
  6. DPDK app routes: lookup in hash table (rte_hash)
     struct ipv4_hdr *ip = ...;
     uint32_t next_hop = rte_hash_lookup(routing_table, &ip->dst_addr);
  7. Transmit out NIC1: rte_eth_tx_burst(1, 0, &mbuf, 1);

Performance: 40+ Mpps forwarding, <3µs latency
Use case: NFV routers, firewalls, load balancers (no general-purpose OS needed)
```

#### VIRTUAL MACHINE (DPDK in VM)
```
VM running DPDK application, hypervisor uses vhost-user

Example: vRouter VNF in OpenStack

Setup:
  - Host runs OVS-DPDK (host DPDK process)
  - VM configured with vhost-user interface (not virtio-net):
    <interface type='vhostuser'>
      <source type='unix' path='/var/run/openvswitch/vhost-user-1' mode='client'/>
    </interface>
  - Hugepages allocated on host and guest
  
Flow:
  1. Packet arrives at physical NIC
  2. Host OVS-DPDK processes (PMD thread polling physical NIC)
  3. OVS flow lookup → forward to VM
  4. OVS writes to vhost-user ring buffer (shared hugepage memory):
     - vhost-user = userspace (not kernel vhost-net)
     - Shared memory between OVS-DPDK process and QEMU
     - QEMU maps guest memory
  5. VM's DPDK application polls vhost-user ring (via virtio-pmd in DPDK):
     rte_eth_rx_burst(virtio_port_id, 0, mbufs, 32);
     - No VM exit!
     - Polling shared memory
  6. VM DPDK processes packet
  7. VM transmits: rte_eth_tx_burst() → writes to vhost-user TX ring
  8. OVS-DPDK PMD polls vhost-user TX ring → transmits out physical NIC

Performance: 8-15 Mpps per VM, ~10-30µs latency
Benefit: Much faster than virtio-net (no kernel, no VM exits for data)
Limitation: VM must run DPDK (not standard kernel networking)
```

#### CONTAINER (DPDK in Container)
```
DPDK running inside container

Example: Containerized VNF (CNF - Cloud-Native Network Function)

Setup:
  - Container needs:
    - Access to hugepages: -v /dev/hugepages:/dev/hugepages
    - Privileged mode OR specific capabilities: --cap-add=IPC_LOCK,SYS_ADMIN
    - CPU pinning: --cpuset-cpus=2-5
  - NIC bound to vfio-pci on host (or SR-IOV VF passed to container)
  
Flow:
  1. Container starts with DPDK app
  2. DPDK app initializes EAL inside container
  3. App mmaps hugepages (shared with host via mount)
  4. App accesses NIC via VFIO:
     Option A: Entire physical NIC (exclusive, not shareable)
     Option B: SR-IOV VF assigned to container namespace
  5. Polling loop runs (CPU 100% on cores 2-5)
  6. Process same as physical DPDK

Performance: Same as physical (~40 Mpps)
Container overhead: Nearly zero (it's just a namespace, same kernel)
Challenge: Resource isolation (DPDK is greedy with CPU and memory)
```

#### KUBERNETES (DPDK in Pod)
```
DPDK VNF as a Kubernetes pod (SR-IOV device plugin)

Example: DPDK-based vRouter in K8s

Setup:
  - Node has SR-IOV NIC
  - SR-IOV device plugin running (DaemonSet):
    Discovers VFs, advertises to kubelet as resources
  - Pod spec requests SR-IOV VF:
    resources:
      requests:
        intel.com/sriov_netdevice: '1'
      limits:
        intel.com/sriov_netdevice: '1'
        hugepages-1Gi: 2Gi
  
Flow:
  1. Kubelet sees pod needs SR-IOV VF
  2. SR-IOV device plugin:
     - Allocates VF from pool
     - Moves VF into pod's network namespace:
       ip link set <vf> netns <pod-netns>
  3. Pod's DPDK app starts:
     - Binds VF to vfio-pci (inside pod)
     - DPDK EAL discovers VF
  4. DPDK app polls VF directly (bypass kernel entirely)
  5. Process same as physical

Performance: Same as physical (VF has dedicated hardware queue)
Benefit: K8s orchestration + DPDK performance
Drawback: Complex (needs SR-IOV, device plugin, hugepages, CPU manager)
```

---

# TECHNIQUE 2: PACKET BYPASS

## Definition
Bypass means avoiding the kernel network stack to achieve higher performance by:
- Userspace I/O (DPDK, covered above)
- Kernel bypass at driver level (XDP)
- Hardware offload (NIC does the work)

---

## 2.1 XDP (eXpress Data Path) — Kernel Bypass at Driver Level

### What is XDP?
- eBPF programs attached at NIC driver receive path
- Runs BEFORE sk_buff allocation
- Can drop, redirect, or pass packets with minimal overhead
- Kernel code, but executes at the earliest possible point

```
Normal kernel: NIC → DMA → sk_buff alloc → netif_receive_skb → ip_rcv → ...
                     ↑
                     ~100ns+ just for sk_buff allocation

XDP path:      NIC → DMA → XDP program → decision (100-500ns total)
                     ↑
                     XDP runs here, before ANY kernel overhead
```

### XDP Actions

```c
XDP_DROP:       Drop packet immediately (DDoS mitigation)
XDP_PASS:       Pass to kernel (normal path)
XDP_TX:         Reflect packet back out same interface (simple switch)
XDP_REDIRECT:   Send to different interface or AF_XDP socket
XDP_ABORTED:    Error condition (malformed packet)
```

### XDP Program Example (DDoS Protection)

```c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>

// BPF map: blacklist of source IPs
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1000000);
    __type(key, __u32);       // Source IP
    __type(value, __u8);      // Dummy value
} blacklist SEC(".maps");

SEC("xdp")
int xdp_drop_blacklist(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;
    
    // Parse Ethernet header
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)  // Bounds check (verifier requirement)
        return XDP_ABORTED;
    
    // Check if IP packet
    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_PASS;
    
    // Parse IP header
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_ABORTED;
    
    // Check blacklist
    __u32 src_ip = ip->saddr;
    if (bpf_map_lookup_elem(&blacklist, &src_ip)) {
        return XDP_DROP;  // Drop at driver level, no sk_buff allocated
    }
    
    return XDP_PASS;  // Not blacklisted, continue to kernel
}

char _license[] SEC("license") = "GPL";
```

```bash
# Compile XDP program
clang -O2 -target bpf -c xdp_drop.c -o xdp_drop.o

# Load and attach to interface
ip link set dev eth0 xdp obj xdp_drop.o sec xdp

# Add IP to blacklist (from userspace)
bpftool map update name blacklist key 0x01020304 value 0x00
# This blocks 4.3.2.1 (hex is little-endian)

# Packet from 4.3.2.1 arrives:
# NIC → DMA → XDP program sees src_ip=0x01020304
# Map lookup finds it → return XDP_DROP
# Packet dropped at driver, ~50ns processing, 0 kernel overhead
```

### XDP on Different Machines

#### PHYSICAL MACHINE (XDP)
```
XDP program attached to physical NIC

Example: DDoS mitigation at edge router

Setup:
  - Physical server with XDP-capable NIC (most modern NICs)
  - Load XDP program: ip link set eth0 xdp obj program.o

Flow:
  1. Packet arrives at NIC
  2. NIC DMAs to memory (RX ring buffer)
  3. Driver calls XDP hook BEFORE netif_receive_skb():
     act = bpf_prog_run_xdp(xdp_prog, xdp_md);
  4. XDP program executes (eBPF JIT-compiled to native code):
     - Has access to xdp_md->data (packet bytes)
     - Can read/modify packet
     - Makes decision (DROP, PASS, TX, REDIRECT)
  5. Based on action:
     - XDP_DROP: Driver frees buffer immediately, no kernel processing
     - XDP_PASS: Driver continues normal path (allocate sk_buff, etc.)
     - XDP_TX: Driver queues packet for TX on same interface
     - XDP_REDIRECT: Packet goes to specified target

Performance: 
  - DROP: ~50-100ns per packet, 20-24 Mpps per core
  - PASS: Same as normal kernel
  - TX: ~100-200ns, 10-20 Mpps per core

Use case: DDoS protection, fast packet filtering, load balancing (Facebook's Katran)
```

#### VIRTUAL MACHINE (XDP in guest or host)
```
XDP can run in TWO places in VM scenario:

Option A: XDP in GUEST (on virtio-net interface inside VM)
  - VM has virtio-net interface (eth0)
  - Attach XDP to eth0 inside VM:
    (inside VM) ip link set eth0 xdp obj program.o
  - Packets arriving at VM's virtio-net trigger XDP
  - Performance: Moderate (still has vhost overhead before XDP runs)
  - Use: Guest wants to filter its own traffic

Option B: XDP on HOST (on tap/vhost interface)
  - Attach XDP to tap device or physical NIC on host
  - Filters before packet even reaches VM
  - Performance: Best for dropping unwanted VM traffic
  - Use: Hypervisor-level filtering

Example (XDP on host tap):
  VM's tap device: tap0
  (on host) ip link set tap0 xdp obj filter.o
  Packet to VM → host NIC → OVS → tap0 → XDP (DROP or PASS) → vhost → VM
  
If XDP drops: VM never sees packet, zero VM overhead
```

#### CONTAINER (XDP on veth)
```
XDP can attach to veth interfaces

Example: Per-container XDP firewall

Setup:
  - Container has veth pair: veth-abc (host) <-> eth0 (container)
  - Attach XDP to host-side veth:
    ip link set veth-abc xdp obj filter.o
    
Flow:
  1. Packet to container → routed to veth-abc
  2. Before veth transmits to container, XDP runs:
     XDP program checks packet, DROP or PASS
  3. If PASS: veth transmits to container
  
Performance: ~200-500ns overhead (veth + XDP)
Benefit: Per-container filtering without iptables overhead
Limitation: XDP on veth is "generic XDP" (software, not driver-level)
  - Generic XDP runs after sk_buff allocation
  - Slower than native XDP (but faster than iptables)
```

#### KUBERNETES (XDP with Cilium)
```
Cilium uses XDP for high-performance filtering

Setup:
  - Cilium agent loads XDP programs on:
    - Physical host interface (for node-level filtering)
    - veth pairs (for pod-level filtering)
    
Flow (NodePort service with XDP acceleration):
  1. External packet arrives at node IP:port
  2. XDP program on physical NIC sees packet
  3. XDP checks: Is this a NodePort service?
     - Lookup in BPF map: (dst_ip, dst_port) → Service
  4. If yes: XDP REDIRECTS to backend pod veth directly
     - No iptables
     - No kube-proxy
     - Packet goes: NIC → XDP → redirect → veth → pod
  5. If no: XDP_PASS to kernel
  
Performance: ~40-60 Gbps service load balancing (vs 10-20 Gbps with iptables)
Benefit: Bypasses entire host network stack for forwarded traffic
```

---

## 2.2 Hardware Offload — Ultimate Bypass

### SmartNIC / Hardware OVS

```
Packet processing done entirely in NIC hardware (ASIC or FPGA)

Example: Mellanox ConnectX-5/6 with OVS offload

Capability:
  - NIC has embedded switch ASIC (eSwitch)
  - Supports TC (Traffic Control) flower offload
  - OVS rules installed in NIC hardware

Setup:
  ethtool -K eth0 hw-tc-offload on
  ovs-vsctl set Open_vSwitch . other_config:hw-offload=true
  
Flow:
  1. OVS adds flow:
     ovs-ofctl add-flow br0 "ip,nw_src=10.0.0.1,nw_dst=10.0.0.2,actions=output:2"
  2. OVS kernel module tries to offload to NIC:
     - Translates OpenFlow to TC flower
     - Sends to NIC driver via tc filter API
  3. NIC driver programs hardware:
     - Match: src_ip=10.0.0.1, dst_ip=10.0.0.2
     - Action: forward to port 2 (VF or physical port)
  4. Packet arrives at NIC:
     - NIC ASIC matches in hardware TCAM (ns latency)
     - NIC forwards directly (CPU sees nothing!)
  5. Host CPU load: ~0% for forwarded traffic

Performance: Line rate (100 Gbps), <1µs latency
CPU: Freed up for application work
Limitation: 
  - TCAM size limited (4K-32K flows)
  - Not all OVS actions supported (vendor-specific)
  - Complex flows fall back to software
```

### Hardware OVS on Different Machines

#### PHYSICAL MACHINE (SmartNIC)
```
Server with Mellanox ConnectX-6 (100G SmartNIC)

Setup:
  - Physical ports: 2x 100G QSFP
  - SR-IOV enabled: 128 VFs
  - OVS-kernel with hw-offload
  
Use case: NFV infrastructure node

Flow (VM1 to VM2 on same host):
  1. Both VMs have SR-IOV VFs
  2. OVS rule: VF1 → VF2 (installed in NIC hardware)
  3. VM1 sends packet → VF1 → NIC eSwitch (hardware)
  4. NIC eSwitch forwards to VF2 → VM2
  5. Host CPU: Not involved at all (0% load)
  
VM-to-external:
  1. OVS rule: VF1 → VXLAN encap → physical port
  2. NIC hardware does VXLAN encapsulation
  3. Packet sent out 100G port
  4. CPU: Still not involved

Performance: 100 Gbps VM-to-VM, <500ns latency
```

#### VIRTUAL MACHINE (VM on SmartNIC host)
```
VM benefits from host's SmartNIC offload

Scenario: OpenStack compute node with SmartNIC

Setup:
  - Host: Ubuntu with OVS, SmartNIC
  - VM: Ubuntu with virtio-net
  - Neutron creates OVS flows for VM
  
Flow:
  1. VM1 sends to VM2 (different compute node)
  2. VM1 virtio-net → vhost-user → OVS (host)
  3. OVS flow: encap VXLAN, send to remote host
  4. This flow is OFFLOADED to SmartNIC
  5. vhost-user hands packet to NIC → NIC encaps → wire
  6. Host CPU: Only vhost-user overhead (~10% CPU)
     vs. 60-80% CPU without offload
  
Benefit: VMs get line-rate networking without host CPU cost
Drawback: VM doesn't control offload (host does)
```

---

# TECHNIQUE 3: PACKET FILTERING

## Definition
Filtering is deciding whether to allow or deny packets based on rules.

---

## 3.1 Packet Filtering in KERNEL MODE

### iptables / Netfilter

```
Netfilter is the kernel framework, iptables is the userspace tool

Architecture:
  - 5 hook points in IP stack: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING
  - Tables: filter (default), nat, mangle, raw, security
  - Chains: built-in (INPUT, FORWARD, OUTPUT) or custom
  - Rules: match criteria + action (ACCEPT, DROP, REJECT, LOG, etc.)

Rule evaluation:
  - Linear chain walk: O(n) where n = number of rules
  - For each packet, each rule checked sequentially
  - First matching rule wins (unless rule is non-terminating like LOG)

Example firewall:
  iptables -A INPUT -p tcp --dport 22 -j ACCEPT         # Allow SSH
  iptables -A INPUT -p tcp --dport 80 -j ACCEPT         # Allow HTTP
  iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  iptables -A INPUT -j DROP                              # Default deny
  
Packet processing:
  1. Packet arrives → Netfilter PREROUTING hook
  2. For each rule in PREROUTING chain:
     - Match protocol, port, state, etc.
     - If match: execute action (jump to chain, ACCEPT, DROP, etc.)
     - If no match: continue to next rule
  3. Routing decision
  4. Netfilter INPUT hook (if local) or FORWARD (if routing)
  5. Same rule walk for INPUT/FORWARD
  
Performance:
  - 10 rules: ~1µs overhead
  - 100 rules: ~5-10µs overhead
  - 1000 rules: ~50-100µs overhead (degraded badly)
  - 10,000 rules: Minutes to load, seconds per packet (unusable)
```

### Connection Tracking (conntrack)

```
Stateful firewall magic: remember connection state

How it works:
  1. First packet of connection (SYN):
     - Netfilter creates conntrack entry:
       (src_ip, src_port, dst_ip, dst_port, proto) → state=NEW
     - Hash table: 5-tuple hashed, entry stored
  
  2. Reply packet (SYN-ACK):
     - Conntrack lookup by 5-tuple (reversed)
     - State updated: NEW → ESTABLISHED
  
  3. Subsequent packets:
     - Conntrack lookup: ESTABLISHED
     - iptables rule: -m conntrack --ctstate ESTABLISHED -j ACCEPT
     - Single rule accepts all ESTABLISHED traffic!
  
  4. FIN or timeout:
     - Connection closed or idle timeout
     - Conntrack entry removed

Performance cost:
  - Hash table lookup: ~500ns
  - State update: ~200ns
  - Total: ~1-2µs per packet
  
Table exhaustion:
  - Default max: 65,536 entries (tunable to 1M+)
  - High-churn workloads (microservices): millions of connections
  - Symptom: "nf_conntrack: table full, dropping packet"
  - Solution: Increase nf_conntrack_max, reduce timeouts, or bypass with eBPF
```

### iptables Filtering on Different Machines

#### PHYSICAL MACHINE (iptables)
```
Firewall on physical server

Example: Web server with iptables firewall

Rules:
  iptables -P INPUT DROP         # Default deny
  iptables -A INPUT -i lo -j ACCEPT                      # Loopback
  iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT  # SSH from LAN
  iptables -A INPUT -p tcp --dport 80 -j ACCEPT         # HTTP
  iptables -A INPUT -p tcp --dport 443 -j ACCEPT        # HTTPS
  iptables -A INPUT -j LOG --log-prefix "DROP: "        # Log drops
  iptables -A INPUT -j DROP

Flow:
  1. External client → server:443
  2. Packet arrives → NIC → sk_buff → ip_rcv → Netfilter INPUT
  3. Rule walk:
     - Rule 1 (lo): No match (not loopback)
     - Rule 2 (ESTABLISHED): No match (NEW connection)
     - Rule 3 (SSH): No match (dest port 443, not 22)
     - Rule 4 (HTTP): No match (dest port 443, not 80)
     - Rule 5 (HTTPS): MATCH! Action: ACCEPT
  4. Packet continues to nginx application

Performance: ~2-5µs filtering overhead (5 rules checked)
```

#### VIRTUAL MACHINE (iptables in guest)
```
VM runs its own iptables

Scenario: Ubuntu VM on KVM host

Flow:
  1. Packet arrives: Physical NIC → host OVS → vhost-net → VM
  2. Inside VM guest kernel:
     - virtio-net driver receives → guest sk_buff
     - Guest ip_rcv → Guest Netfilter INPUT hook
     - Guest iptables rules evaluated (VM's own firewall)
  3. If guest iptables ACCEPTS: deliver to application in VM
  
Layering:
  - Host can ALSO have iptables rules on tap device
  - Packet filtered TWICE: host iptables, then guest iptables
  
Example (two-layer filtering):
  Host: iptables -A FORWARD -o tap0 -p tcp --dport 3389 -j DROP  # Block RDP to VM
  Guest: iptables -A INPUT -p tcp --dport 22 -j ACCEPT           # Allow SSH in VM
  
Result: RDP blocked at host (never reaches VM), SSH allowed if reaches VM
```

#### CONTAINER (host iptables)
```
Container traffic filtered by HOST iptables

Docker creates iptables rules automatically:

When you run: docker run -p 8080:80 nginx

Docker creates:
  iptables -t nat -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
  iptables -A DOCKER -d 172.17.0.2 -p tcp --dport 80 -j ACCEPT
  
Per-container chains:
  iptables -N DOCKER-USER      # For user-added rules
  iptables -N DOCKER           # For Docker-managed rules
  
Flow:
  1. External → host:8080
  2. PREROUTING: DNAT to 172.17.0.2:80
  3. FORWARD chain: Check DOCKER chain
  4. DOCKER chain: Allow to 172.17.0.2:80
  5. Packet sent to docker0 bridge → veth pair → container
  
User firewall:
  iptables -I DOCKER-USER -s 192.168.1.0/24 -j DROP  # Block subnet
  
Performance degradation with many containers:
  - 10 containers: ~10µs
  - 100 containers: ~50-100µs (100 chains to check)
  - Solution: Use nftables or Cilium (eBPF)
```

#### KUBERNETES (iptables NetworkPolicy via Calico)
```
NetworkPolicy implemented via iptables

Example NetworkPolicy:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: deny-all-allow-http
  spec:
    podSelector: {app: web}
    policyTypes: [Ingress]
    ingress:
    - from:
      - podSelector: {app: frontend}
      ports:
      - protocol: TCP
        port: 80

Calico Felix translates to iptables:
  # Create IPSet for frontend pods
  ipset create cali-frontend-pods hash:ip
  ipset add cali-frontend-pods 192.168.1.5   # frontend pod IPs
  ipset add cali-frontend-pods 192.168.1.6
  
  # Create iptables rules
  iptables -N cali-tw-web-pod     # "to-workload" chain
  iptables -A cali-tw-web-pod -p tcp --dport 80 -m set --match-set cali-frontend-pods src -j ACCEPT
  iptables -A cali-tw-web-pod -j DROP
  
  # Jump to per-pod chain
  iptables -A FORWARD -o cali-web-pod -j cali-tw-web-pod

Flow (frontend pod → web pod):
  1. Frontend pod sends to web pod IP
  2. Host kernel routes to web pod's veth
  3. Netfilter FORWARD hook
  4. Jump to cali-tw-web-pod chain
  5. Check: src IP in cali-frontend-pods IPSet? Yes → ACCEPT
  6. Packet sent to web pod
  
Flow (other pod → web pod):
  1. Other pod sends to web pod
  2-4. Same
  5. Check: src IP in IPSet? No → DROP
  6. Packet dropped
  
Performance with 1000 pods:
  - IPSet lookup: ~O(1) (hash table)
  - Total overhead: ~5-10µs per packet
  - Much better than 1000 separate IP rules (which would be ~500µs)
```

---

## 3.2 Packet Filtering in USER MODE

### User-mode Firewalls (Application-level)

```
Filtering done by userspace application, not kernel

Examples:
  - Application-level: nginx with geo-blocking, rate limiting
  - Proxy-based: Squid, HAProxy with ACLs
  - WAF (Web Application Firewall): ModSecurity, AWS WAF
```

#### nginx geo-blocking (User Mode)
```
nginx.conf:
  geo $block_country {
      default 0;
      91.0.0.0/8 1;    # Block range (example: Russia)
      112.0.0.0/8 1;   # Block range
  }
  
  server {
      listen 80;
      if ($block_country) {
          return 403;
      }
      ...
  }

Flow:
  1. TCP connection established (kernel accepts, no filtering yet)
  2. nginx accepts connection (accept() syscall)
  3. nginx receives HTTP request (recv() syscall)
  4. nginx reads client IP from socket: getpeername()
  5. nginx checks geo database: Is IP in blocked range?
  6. If yes: nginx sends "HTTP/1.1 403 Forbidden", closes socket
  7. If no: nginx processes request normally
  
Key: Filtering happens AFTER TCP connection established
      CPU/memory used for every connection (even blocked ones)
      
Performance: ~100µs per request overhead (geo lookup)
vs. kernel iptables: ~2µs
Why use it: Layer 7 awareness (HTTP headers, URL, etc.)
```

### User-mode Filtering on Different Machines

#### PHYSICAL MACHINE (User-mode Firewall)
```
Example: Squid proxy with ACLs

squid.conf:
  acl internal_network src 192.168.0.0/16
  acl blocked_sites dstdomain .facebook.com .twitter.com
  http_access deny blocked_sites
  http_access allow internal_network
  http_access deny all

Flow:
  1. Client connects to Squid (TCP)
  2. Squid accepts (kernel allows, no filtering)
  3. Client sends HTTP CONNECT or GET
  4. Squid parses HTTP request (userspace)
  5. Squid checks ACLs:
     - Is destination facebook.com? Yes → deny (send 403)
  6. Squid closes client connection
  
All in userspace, full protocol parsing
```

#### VIRTUAL MACHINE (User-mode in VM)
```
Example: Application in VM with built-in rate limiting

Scenario: API server in VM with request rate limiting

Code (Python Flask):
  from flask import Flask
  from flask_limiter import Limiter
  
  app = Flask(__name__)
  limiter = Limiter(app, key_func=lambda: request.remote_addr)
  
  @app.route('/api')
  @limiter.limit("10 per minute")
  def api():
      return "OK"

Flow:
  1. Client → VM → Flask receives request
  2. Flask limiter checks Redis/memory:
     - How many requests from this IP in last 60s?
     - If >= 10: return HTTP 429 (Too Many Requests)
     - If < 10: Process request, increment counter
  
All in application userspace (inside VM)
Performance: ~500µs overhead (Redis lookup)
```

#### CONTAINER (User-mode Filter in Container)
```
Example: Envoy proxy sidecar in pod

Envoy config:
  rate_limits:
  - actions:
    - request_headers:
        header_name: "x-user-id"
        descriptor_key: "user_id"
    limits:
    - per_header_limit: 100
      time_unit: MINUTE

Flow:
  1. Request → pod → Envoy sidecar (localhost:15001)
  2. Envoy receives HTTP request
  3. Envoy checks rate limit:
     - Extracts x-user-id header
     - Checks local cache: Has user_id made 100 requests in last minute?
     - If yes: Return 429
     - If no: Proxy to application container
  
All in Envoy userspace (inside pod's container)
```

---

## 3.3 Packet Filtering in BYPASS MODE

### eBPF/XDP Filtering

```
Filtering at driver level or TC hook using eBPF

Advantages over iptables:
  - O(1) map lookups vs O(n) rule chains
  - Runs at driver (XDP) or TC hook
  - Programmable: can implement custom logic

Example: XDP blacklist (shown earlier)
  - Hash map of blocked IPs
  - XDP program: lookup in map, drop if found
  - ~50ns per packet vs ~5µs for iptables
```

#### PHYSICAL MACHINE (XDP Filtering)
```
Example: DDoS protection

XDP program attached to physical NIC
  - Blocks 1M malicious IPs
  - Hash map: 1M entries
  - Performance: 20 Mpps drop rate (vs 2 Mpps with iptables)
```

#### KUBERNETES (Cilium eBPF NetworkPolicy)
```
Cilium replaces iptables with eBPF

NetworkPolicy → Cilium translates to eBPF program

BPF program on veth:
  SEC("tc/ingress")
  int policy_check(struct __sk_buff *skb) {
      __u32 src_identity = skb->cb[CB_SRC_ID];  # Set by Cilium agent
      __u32 dst_identity = skb->cb[CB_DST_ID];
      
      struct policy_key key = {src_identity, dst_identity, dst_port, proto};
      struct policy_val *val = bpf_map_lookup_elem(&policy_map, &key);
      
      if (!val || val->action == DROP)
          return TC_ACT_SHOT;
      
      return TC_ACT_OK;
  }

Performance:
  - Map lookup: O(1), ~100ns
  - vs iptables: O(n), ~5µs for 100 rules
  - 50x faster
```

---

# PART 2: ANOMALY DEEP-DIVES

Now we solve the two complex packet journey scenarios you requested.

---

# ANOMALY 1: Windows Host → VirtualBox → Ubuntu VM → K8s Pod

```
SCENARIO:
Physical machine: Windows (kernel0, user0)
  └─ VirtualBox hypervisor
      └─ Ubuntu VM (kernel1, user1)
          └─ Kubernetes (running in VM)
              └─ Pod (container)
                  └─ Application

QUESTION: Packet arrives at physical NIC → how does it reach the application?
```

## Complete Packet Journey with Techniques & Modes

### Stage 1: Physical NIC → Windows Kernel (KERNEL0 MODE)

```
1. PACKET ARRIVES AT PHYSICAL NIC:
   Hardware: Intel Ethernet NIC (e.g., e1000, I219)
   
2. NIC DMA:
   - NIC reads packet from wire
   - NIC DMAs packet data to Windows memory (physical address)
   - NIC raises interrupt (MSI-X)
   
3. WINDOWS KERNEL INTERRUPT HANDLER (KERNEL0, Ring 0):
   - CPU jumps to NDIS (Network Driver Interface Specification) driver
   - Driver acknowledges interrupt
   - Driver schedules DPC (Deferred Procedure Call) - Windows equivalent of softirq
   
   Technique: PACKET PROCESSING (Windows TCP/IP stack)
   Mode: KERNEL0 (Windows kernel)
   
4. WINDOWS DPC (still KERNEL0):
   - NDIS miniport driver processes packet
   - Creates NET_BUFFER structure (Windows equivalent of sk_buff)
   - Passes to Windows TCP/IP stack (tcpip.sys)
   
5. WINDOWS TCP/IP STACK:
   - Checks if packet is for local host or for routing
   - Destination: VirtualBox VM's IP (assigned to VirtualBox NAT/bridged interface)
   - Routing decision: Forward to VirtualBox virtual switch
   
6. WINDOWS FIREWALL (KERNEL0):
   - Windows Filtering Platform (WFP) checks rules
   - If allowed: Continue
   
   Technique: PACKET FILTERING (WFP in kernel)
   Mode: KERNEL0
```

### Stage 2: Windows Kernel → VirtualBox Hypervisor (USER0 MODE)

```
7. VIRTUALBOX VIRTUAL SWITCH:
   - VirtualBox runs as userspace process on Windows (VirtualBox.exe)
   - BUT: Packet forwarding happens via kernel driver (VBoxDrv.sys)
   
   VBoxDrv.sys is a Windows kernel driver that:
   - Creates virtual network adapters
   - Bridges traffic between Windows host network and VM
   
8. PACKET FORWARDED TO VBOXDRV.SYS (KERNEL0):
   - Windows routing sends packet to VirtualBox virtual adapter
   - VBoxDrv.sys receives packet (kernel driver)
   
   Technique: PACKET PROCESSING (forwarding)
   Mode: KERNEL0 (Windows kernel driver)
   
9. VBOXDRV → VIRTUALBOX.EXE NOTIFICATION:
   - VBoxDrv.sys signals VirtualBox.exe (userspace) that packet arrived
   - VirtualBox.exe reads packet from shared memory
   
   Technique: USER0 MODE receives notification
   Mode: USER0 (VirtualBox hypervisor process)
```

### Stage 3: VirtualBox Hypervisor → Ubuntu VM (KERNEL1 MODE)

```
10. VIRTUALBOX VIRTUAL NIC EMULATION (USER0):
    - VirtualBox.exe emulates network card for VM (e.g., Intel PRO/1000 MT Desktop)
    - Packet placed in emulated NIC's receive buffer
    - VirtualBox injects virtual interrupt into Ubuntu VM
    
    Technique: PACKET PROCESSING (NIC emulation)
    Mode: USER0 (VirtualBox process)
    
11. UBUNTU VM RECEIVES INTERRUPT (KERNEL1 MODE):
    - VM's kernel (kernel1) receives virtual interrupt
    - VM's NIC driver (e.g., e1000) interrupt handler runs
    - Driver reads from virtual NIC's register (actually VirtualBox memory)
    
    Mode: KERNEL1 (Ubuntu VM kernel, Ring 0 IN THE VM)
    
12. UBUNTU VM NETWORK STACK (KERNEL1):
    - VM kernel allocates sk_buff
    - Copies packet data from VirtualBox shared memory
    - Processes through Linux network stack:
      - netif_receive_skb()
      - ip_rcv()
      - Netfilter PREROUTING
      - Routing decision
    
    Technique: PACKET PROCESSING (Linux kernel network stack)
    Mode: KERNEL1
    
13. ROUTING IN UBUNTU VM:
    - Destination IP: Pod IP (e.g., 192.168.1.5)
    - Routing table lookup: Installed by Kubernetes CNI
    - Route: 192.168.1.5/32 dev cali1234567 (Calico veth interface)
```

### Stage 4: Ubuntu VM Kernel → Kubernetes Pod (KERNEL1 + USER1)

```
14. CNI VETH INTERFACE (KERNEL1):
    - Packet routed to cali1234567 (host-side veth)
    - This is still in KERNEL1 (Ubuntu VM's kernel)
    
    Technique: PACKET PROCESSING (veth forwarding)
    Mode: KERNEL1
    
15. IPTABLES NETWORKPOLICY CHECK (KERNEL1):
    - Before packet goes to veth, Netfilter FORWARD hook
    - Calico iptables rules:
      iptables -A cali-tw-cali1234567 -m set --match-set cali-allowed src -j ACCEPT
      iptables -A cali-tw-cali1234567 -j DROP
    - Checks source IP against IPSet
    - If ACCEPT: Continue
    
    Technique: PACKET FILTERING (iptables)
    Mode: KERNEL1
    
16. VETH PAIR CROSSING:
    - Packet written to cali1234567 (host side of veth)
    - Appears on eth0 (pod side of veth)
    - eth0 exists in pod's network namespace (different routing table, different socket table)
    - Still same KERNEL1, just different namespace context
    
    Technique: PACKET PROCESSING (namespace traversal)
    Mode: KERNEL1 (in pod netns context)
    
17. POD NETWORK NAMESPACE (KERNEL1):
    - Packet arrives on eth0 in pod
    - Pod's routing table used:
      ip route in pod: default via 169.254.1.1 dev eth0
    - Destination IP matches pod IP (192.168.1.5) assigned to eth0
    - Kernel delivers to socket
    
18. APPLICATION RECEIVES (USER1):
    - Application in pod called recv() earlier (blocked waiting)
    - Kernel wakes application
    - Kernel copies data from socket buffer to application buffer (user1 memory)
    - recv() returns to application
    
    Technique: PACKET PROCESSING (final delivery)
    Mode: USER1 (application in container)
```

## Summary of Anomaly 1 Journey

```
┌──────────────────────────────────────────────────────────────────────┐
│ PACKET JOURNEY: Physical NIC → Windows → VirtualBox → Ubuntu → K8s │
└──────────────────────────────────────────────────────────────────────┘

NIC → [KERNEL0: Windows TCP/IP + WFP filtering] 
    → [KERNEL0: VBoxDrv.sys forwarding] 
    → [USER0: VirtualBox.exe NIC emulation] 
    → [KERNEL1: Ubuntu VM network stack + iptables filtering] 
    → [KERNEL1: CNI veth + NetworkPolicy iptables] 
    → [KERNEL1: Pod netns delivery] 
    → [USER1: Application recv()]

Techniques used:
  - PACKET PROCESSING: At every kernel layer (Windows, VBox, Ubuntu, CNI)
  - PACKET FILTERING: Windows WFP (kernel0), Ubuntu iptables (kernel1), NetworkPolicy (kernel1)
  - NO BYPASS: All standard kernel paths

Modes:
  - KERNEL0: Windows kernel (TCP/IP, WFP, VBoxDrv.sys)
  - USER0: VirtualBox hypervisor (NIC emulation, VM management)
  - KERNEL1: Ubuntu VM kernel (full Linux network stack, iptables, CNI)
  - USER1: Application in Kubernetes pod

Why no bypass:
  - Windows/VirtualBox don't support DPDK
  - Nested virtualization (VM in VM via Windows) adds overhead
  - Standard kernel path is only option

Performance:
  - Latency: ~500µs - 2ms (very high due to nested virtualization)
  - Throughput: ~1-2 Gbps (Windows → VBox → VM overhead)
  - Bottleneck: VirtualBox NIC emulation (USER0) and double kernel (kernel0 + kernel1)
```

---

# ANOMALY 2: Ubuntu Physical → K8s Pod → VM in Pod → Application

```
SCENARIO:
Physical machine: Ubuntu (kernel0, user0)
  └─ Kubernetes (running directly on Ubuntu)
      └─ Pod (container)
          └─ KubeVirt VM (virtualized inside pod)
              └─ Application

QUESTION: Packet arrives at physical NIC → how does it reach application in VM-in-pod?
```

This is a **VM running INSIDE a container** — kubevirt/kata containers architecture.

## Complete Packet Journey

### Stage 1: Physical NIC → Ubuntu Kernel (KERNEL0 MODE)

```
1. PACKET ARRIVES AT PHYSICAL NIC:
   Hardware: Physical Ethernet NIC (e.g., Intel X710, Mellanox ConnectX)
   
2. NIC DMA + INTERRUPT (KERNEL0):
   - NIC DMAs packet to memory
   - Raises interrupt
   - Driver interrupt handler (ixgbe, mlx5_core)
   - Schedules NET_RX_SOFTIRQ
   
   Technique: PACKET PROCESSING (standard kernel RX)
   Mode: KERNEL0 (Ubuntu kernel)
   
3. SOFTIRQ PROCESSING (KERNEL0):
   - net_rx_action() → NAPI poll
   - Driver allocates sk_buff
   - netif_receive_skb()
   - Passes to IP layer
   
4. UBUNTU KERNEL IP PROCESSING (KERNEL0):
   - ip_rcv()
   - Netfilter PREROUTING
   - Routing decision
   
5. ROUTING LOOKUP (KERNEL0):
   - Destination IP: Pod IP (e.g., 10.244.1.5)
   - CNI has installed route:
     10.244.1.5/32 dev veth-abc
   - Decision: Forward to veth-abc
   
   Technique: PACKET PROCESSING (routing)
   Mode: KERNEL0
```

### Stage 2: Kernel → Pod Network Namespace (KERNEL0, different netns)

```
6. NETFILTER FORWARD + CNI POLICY (KERNEL0):
   - Packet going to pod → Netfilter FORWARD hook
   - CNI iptables rules (e.g., Calico):
     iptables -A cali-fw-veth-abc -m set --match-set allowed-sources src -j ACCEPT
   - NetworkPolicy enforcement happens here
   
   Technique: PACKET FILTERING (iptables NetworkPolicy)
   Mode: KERNEL0
   
7. VETH PAIR TRANSMISSION (KERNEL0):
   - Packet sent to veth-abc (host side)
   - Crosses veth boundary
   - Appears on eth0 (pod side, in pod's netns)
   
   Technique: PACKET PROCESSING (veth crossing)
   Mode: KERNEL0 (now in pod netns context)
   
8. POD NETWORK NAMESPACE (KERNEL0):
   - Interface: eth0 in pod netns
   - IP: 10.244.1.5 assigned to eth0
   - Routing in pod: default via 169.254.1.1 dev eth0
   - Packet destination matches eth0 IP
   - But... there's no application directly listening here!
   - Instead: kubevirt VM is running
```

### Stage 3: Pod → KubeVirt VM (Complex!)

```
9. KUBEVIRT ARCHITECTURE:
   Inside the pod, kubevirt runs:
   - virt-launcher process (USER0): QEMU/KVM wrapper
   - QEMU process (USER0): VM hypervisor
   - Linux KVM kernel module (KERNEL0): Hardware virtualization
   
10. PACKET INTERCEPTION IN POD (USER0):
    - virt-launcher has created TAP device in pod's netns:
      ip tuntap add tap0 mode tap
    - TAP device connected to VM
    - Packet routing in pod netns:
      Destination 10.244.1.5 → actually TAP device subnet
      OR: iptables DNAT to TAP device IP
    
    Example pod routing:
      10.244.1.100/32 dev tap0  (VM's IP inside pod)
      iptables -t nat -A PREROUTING -d 10.244.1.5 -j DNAT --to 10.244.1.100
    
    Technique: PACKET PROCESSING (DNAT + routing)
    Mode: KERNEL0 (in pod netns)
    
11. TAP DEVICE WRITE (KERNEL0 → USER0):
    - Packet routed to tap0
    - TAP device is character device (/dev/net/tun)
    - QEMU process has open file descriptor on tap0
    - Kernel writes packet to TAP device
    - QEMU reads from tap0 (read() syscall returns with packet)
    
    Technique: PACKET PROCESSING (kernel → user via TAP)
    Mode: KERNEL0 writes, USER0 reads
    
12. QEMU RECEIVES PACKET (USER0):
    - QEMU process running in pod
    - QEMU read packet from tap0
    - QEMU has emulated NIC for VM (virtio-net)
    - QEMU writes packet to virtqueue (shared memory with VM)
    - QEMU injects interrupt into VM (via KVM ioctl)
    
    Technique: PACKET PROCESSING (virtio-net emulation)
    Mode: USER0 (QEMU process in pod)
```

### Stage 4: KubeVirt VM Kernel → Application (KERNEL1 + USER1)

```
13. VM RECEIVES INTERRUPT (KERNEL1):
    - VM's kernel (kernel1) receives virtio interrupt
    - virtio-net driver processes
    - Reads packet from virtqueue
    - Allocates sk_buff in VM
    
    Technique: PACKET PROCESSING (virtio RX path)
    Mode: KERNEL1 (VM's kernel)
    
14. VM KERNEL NETWORK STACK (KERNEL1):
    - ip_rcv() in VM kernel
    - Netfilter in VM (if VM has firewall)
    - Routing in VM
    - Destination: Application IP/port
    
    Technique: PACKET PROCESSING (full network stack)
    Technique: PACKET FILTERING (if VM has iptables)
    Mode: KERNEL1
    
15. APPLICATION IN VM RECEIVES (USER1):
    - Application called recv() (blocked)
    - VM kernel wakes application
    - Copies from socket buffer to application buffer
    - recv() returns
    
    Mode: USER1 (application in VM)
```

## Summary of Anomaly 2 Journey

```
┌───────────────────────────────────────────────────────────────────────┐
│ PACKET JOURNEY: Physical NIC → Ubuntu → K8s Pod → VM → Application   │
└───────────────────────────────────────────────────────────────────────┘

NIC → [KERNEL0: Ubuntu network stack + Netfilter]
    → [KERNEL0: CNI veth + iptables NetworkPolicy] 
    → [KERNEL0: Pod netns → TAP device]
    → [USER0: QEMU reads TAP, virtio-net emulation]
    → [KERNEL1: VM kernel network stack]
    → [USER1: Application in VM]

Techniques:
  - PACKET PROCESSING: Every layer (host kernel, veth, TAP, QEMU, VM kernel)
  - PACKET FILTERING: NetworkPolicy (kernel0 iptables), VM firewall (kernel1 iptables)
  - NO BYPASS: Standard kernel paths (though kubevirt could use vhost-user for performance)

Modes:
  - KERNEL0: Ubuntu host kernel (network stack, CNI, pod netns, TAP device)
  - USER0: QEMU process in pod (TAP reader, virtio-net emulation)
  - KERNEL1: VM's kernel (full Linux network stack)
  - USER1: Application inside VM

Special notes:
  - TAP device is the bridge: KERNEL0 writes, USER0 (QEMU) reads
  - QEMU runs in USER0 (inside pod's namespace, but still userspace)
  - VM sees QEMU's virtio-net as real NIC (doesn't know it's nested)

Performance:
  - Latency: ~200-500µs (veth ~10µs + TAP ~50µs + virtio ~100µs + VM stack ~50µs)
  - Throughput: ~5-10 Gbps (limited by virtio and double network stack)
  - Bottleneck: TAP device read/write (crosses kernel-user boundary)

Possible optimization (BYPASS mode):
  - Use vhost-user instead of TAP:
    - QEMU and OVS share memory (hugepages)
    - Eliminates TAP device overhead
    - Performance: ~15-20 Gbps
    - Technique: BYPASS (vhost-user shared memory, no kernel)
```

## Visual Comparison of Both Anomalies

```
ANOMALY 1 (Windows → VirtualBox → Ubuntu VM → K8s Pod):
  NIC → [Win Kernel0] → [VBox User0] → [Ubuntu Kernel1] → [Pod User1]
        ↑                ↑                ↑                  ↑
        Standard        NIC               Full Linux         Application
        Windows         emulation         network stack      in container
        TCP/IP                           + CNI/iptables

ANOMALY 2 (Ubuntu → K8s Pod → KubeVirt VM → App):
  NIC → [Ubuntu Kernel0] → [Pod Kernel0 context] → [QEMU User0] → [VM Kernel1] → [App User1]
        ↑                   ↑                       ↑               ↑               ↑
        Standard            CNI veth +              TAP reader +    Full network    Application
        Linux stack         iptables                virtio-net      stack in VM

KEY DIFFERENCE:
  - Anomaly 1: Nested VM (Windows contains VirtualBox contains Ubuntu)
               Two separate kernels: Windows (kernel0) and Ubuntu (kernel1)
               Heavier: kernel0 + user0 + kernel1 + user1
  
  - Anomaly 2: VM-in-container (Ubuntu contains Pod contains VM)
               SAME KERNEL for host and pod (kernel0 in different netns)
               Lighter: kernel0 (host+pod) + user0 (QEMU) + kernel1 (VM) + user1 (app)
```

---