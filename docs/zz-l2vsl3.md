# l2l3

L2 and L3 cnis are confusing for people.  For example antrea is L2 (it uses a switch) vs calico is a l3 cni (IPIP).

## Why windows CNIs are different

Windows CNIs are different then windows ones because:
- theress no "network namespace" for windows CNIs, so the pause container, has less relevance
- the cni plugins logic in linux , and commands that need to be called for network programming are replaced w/ calls to hns
- the windows CNI community came long after the linux CNI community (for example, EKS support for windows is newer)
- things like hostnetwork containers on windows in containerd arent as straightforward -0 you need things like hostProcess containers to do them, which is a new k8s feature.

Thus the linux networking features you get for free on containers cant be taken for granted to exist in windows containers.

## comparing some common local CNIs for windows


Heres a table contrasting different windows CNI implementations

| CNI Implementation    | NetworkPolicy Support | L2/L3 Networking       | Scalability                               | Pod-to-Pod Encryption | Network Security | IPAM (IP Address Management)      | Additional Features                                       |
|-----------------------|----------------------|-------------------------|-------------------------------------------|----------------------|------------------|----------------------------------|-----------------------------------------------------------|
| Flannel for Windows   | no                  | L3 (VXLAN encapsulation)| more for prototypes         | No                   | Yes              | Various backends (e.g., etcd)     |   on-premises and cloud-based clusters,  multiple cloud providers .|
| Calico for Windows    | Yes                  | L3 (BGP routing)        | scalable          | No                   | Yes              | Automatic IPAM (IP Address Management) options | Support for IPv4 and IPv6 dual-stack networking, Network segmentation through BGP route propagation,  Networkpolicies,  |
| Antrea for Windows    | Yes                  | L2   (Open vSwitch)| Scalable  | No                   | Yes              | Yes+nsx integration              | Layer 2 and Layer 3 networking with Open vSwitch, Implements Network Policies for security and isolation, Offers Kubernetes native policy support  |
 
