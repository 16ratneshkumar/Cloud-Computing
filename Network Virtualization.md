# Network Virtualization:

---

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
  8. OVS-DPDK PMD p