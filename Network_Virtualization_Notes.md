# Network Virtualization Notes

## 1. Core Architectural Concepts

### Plane Separation (SDN Pattern)
Network virtualization often relies on **Software-Defined Networking (SDN)** principles:
- **Control Plane**: The "brain" of the network. It makes decisions about where traffic should be sent (Routing tables, forwarding logic). In virtualization, this is often a software controller (e.g., OVN, NSX).
- **Data Plane (Forwarding Plane)**: The "muscles". It performs the actual movement of packets based on the control plane's instructions. Techniques like **eBPF**, **DPDK**, and **XDP** live here.
- **Management Plane**: The interface for administrators to configure the network (APIs, CLI).

### Overlay vs. Underlay Networks
- **Underlay**: The physical network infrastructure (Switches, Routers, Cables). It provides IP connectivity between physical hosts.
- **Overlay**: The virtual network built on top of the underlay. It allows virtual machines/containers to communicate as if they were on the same L2/L3 segment, regardless of physical location.
    - **Encapsulation**: Used to wrap overlay packets inside underlay packets. Protocols include **VXLAN**, **Geneve**, and **NVGRE**.

---

## 2. Packet Handling Techniques: Modes & Mechanisms

### A. Packet Filtering
Deciding the fate of a packet (Accept, Drop, Reject, Log) based on header criteria.

- **IPTables / Netfilter**:
    - **Mode**: **Kernel Mode**.
    - **Mechanism**: Hooks into the Linux kernel networking stack at various points.
    - **Pros**: Built-in, reliable. **Cons**: High latency with large rulesets.
- **eBPF (Extended Berkeley Packet Filter)**:
    - **Mode**: **Kernel Mode** (Execution), **User Mode** (Control).
    - **Mechanism**: Secure kernel sandbox running JIT-compiled bytecode.
    - **Pros**: Extremely fast, programmable. **Cons**: Steep learning curve.

### B. Packet Bypass (Kernel Bypass)
Completely skipping the OS kernel to handle packets directly in an application.

- **DPDK (Data Plane Development Kit)**:
    - **Mode**: **User Mode**.
    - **Mechanism**: Uses **Poll Mode Drivers (PMD)** to scan the NIC, bypassing interrupts.
    - **Pros**: Highest throughput (Zero-copy). **Cons**: 100% CPU polling.

### C. Packet Processing (Offloading & Acceleration)
Transforming or routing packets with high efficiency.

- **XDP (eXpress Data Path)**:
    - **Mode**: **Kernel Mode** (Driver Level).
    - **Mechanism**: Runs eBPF programs before the `sk_buff` data structure is allocated.
    - **Pros**: Ideal for DDoS protection. **Cons**: Driver-dependent.
- **SR-IOV (Single Root I/O Virtualization)**:
    - **Mode**: **Hardware/Kernel Level**.
    - **Mechanism**: Physical NIC presents multiple Virtual Functions (VF) directly to VMs.
    - **Pros**: Near-native hardware performance. **Cons**: Breaks live migration.

---

## 3. Kernel Space vs. User Space Networking

| Feature | Kernel Space (standard) | User Space (bypass/DPDK) |
| :--- | :--- | :--- |
| **Data Movement** | Context switching & data copying | Zero-copy (Direct Memory Access) |
| **Interrupts** | Event-driven (IRQ) | Polling (Always On) |
| **Stack** | Standard OS Stack | Application-specific Stack |
| **Stability** | High (handled by OS) | App-dependent |

---

## 4. Network Path Across Environments

1. **Physical Machine**: Hardware NIC -> Driver -> Kernel Stack -> App Socket.
2. **Virtual Machine (VM)**: NIC -> Host Bridge -> Hypervisor -> Virtual NIC -> Guest Kernel -> App.
3. **Containers**: Host NIC -> Host Namespace -> Virtual Bridge -> **Veth Pair** -> Container Namespace -> App.
4. **Kubernetes (K8s)**: Extends container model using **CNI** (VXLAN/Geneve encapsulation in Kernel).

---

## 5. Anomaly Walkthroughs (Techniques & Modes)

### Anomaly 1: Window -> VirtualBox -> Ubuntu -> Container -> K8s -> App
*Scenario: Packets entering a nested stack (VM inside Host, K8s inside VM).*

| Step | Mode | System Level | Technique/Technology Applied |
| :--- | :--- | :--- | :--- |
| **1. Physical Entry** | Hardware | NIC | Physical Signal Reception |
| **2. Host OS (Win)** | Kernel | Network Stack | **NDIS Driver Interception** |
| **3. Hypervisor Layer** | Kernel/User | VirtualBox | **Bridged Networking & Software Switching** |
| **4. VM Entrance** | Kernel | Guest (Ubuntu) | **VirtIO-net Driver (Hardware Abstraction)** |
| **5. VM Networking** | Kernel | Linux Kernel | **Netfilter/IPTables (Service NAT/Filtering)** |
| **6. Pod Isolation** | Kernel | Namespace | **Linux Bridge & Veth Pair (Inter-namespace routing)** |
| **7. Final Destination**| User | K8s App | **Socket API (Buffer Read)** |

### Anomaly 2: Window -> K8s -> Linux Container -> App
*Scenario: K8s running directly on Windows (via WSL2 Utility VM).*

| Step | Mode | System Level | Technique/Technology Applied |
| :--- | :--- | :--- | :--- |
| **1. Physical Entry** | Hardware | NIC | Physical Signal Reception |
| **2. Host OS (Win)** | Kernel | Network Stack | **Hyper-V Virtual Switch (L2 Switching)** |
| **3. WSL2 Boundary** | Kernel | Internal Gateway | **Internal Bridge/IP Forwarding** |
| **4. WSL2 Linux** | Kernel | Linux Kernel | **eBPF/IPTables (Kube-proxy Rule Processing)** |
| **5. Container Access** | Kernel | Namespace | **Veth Pair (Namespace crossover)** |
| **6. Final Destination**| User | Container App | **Socket API (User Space processing)** |
