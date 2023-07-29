# l2l3

L2 and L3 cnis are confusing for people.  For example antrea is L2 (it uses a switch) vs calico is a l3 cni (IPIP).

Heres a table contrasting different windows CNI implementations

| CNI Implementation    | NetworkPolicy Support | L2/L3 Networking       | Scalability                               | Pod-to-Pod Encryption | Network Security | IPAM (IP Address Management)      | Additional Features                                       |
|-----------------------|----------------------|-------------------------|-------------------------------------------|----------------------|------------------|----------------------------------|-----------------------------------------------------------|
| Flannel for Windows   | Yes                  | L3 (VXLAN encapsulation)| Easily scalable and configurable         | No                   | Yes              | Various backends (e.g., etcd)     | Suitable for both on-premises and cloud-based clusters, Works across multiple cloud providers and environments, Compatible with CoreDNS for cluster DNS service, Supports both UDP and direct routing modes.|
| Calico for Windows    | Yes                  | L3 (BGP routing)        | Easily scalable and configurable         | No                   | Yes              | Automatic IPAM (IP Address Management) options | Support for IPv4 and IPv6 dual-stack networking, Network segmentation through BGP route propagation, Transparently handles pod and service IP assignments, Integration with Windows container networking stack. |
| Cilium for Windows    | Yes                  | L3 (eBPF)               | Scalable and performs well in large clusters | Yes                  | Yes              | Not specified                | Layer 3 and Layer 4 networking with eBPF (extended Berkeley Packet Filter) technology, Efficient load balancing and health checks for services, Distributed denial-of-service (DDoS) protection, Supports HTTP/gRPC/API-aware network policies, Deep visibility and troubleshooting capabilities. |
| Antrea for Windows    | Yes                  | L2 and L3 (Open vSwitch)| Scalable and performs well in large clusters | No                   | Yes              | Not specified                | Layer 2 and Layer 3 networking with Open vSwitch, Implements Network Policies for security and isolation, Offers Kubernetes native policy support, Support for both Windows and Linux nodes. |
 
