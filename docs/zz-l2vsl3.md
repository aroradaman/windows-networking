# l2l3

TLDR if you don't want to read all of this:

- There are L2 CNI options in both linux and windows.
- Calico is a L3 CNI, but antrea is an L2 cni.
- Clouds use L2 CNIs under the hood and glue them together via networky magic for x-node communication. 

On Windows, 
- the L2 bridge CNI assigns MAC address -> container in same L2 bridge network.
- SIMPLE, less stress on network switches, NO NEED for learning MAC addresses of short-lived containers.
- SAME NODE comms = direct using MAC addresses,
- CROSS NODE comms -> IP-based routing and NAT mechanisms.
- NO NETWORK NAMESPACE: win containers do not have network namespaces like Linux containers,
- BECAUSE of that the CNI logic replaces the need for Linux network programming with calls to the Host Network Service (HNS) on Windows.

On Linux:
- L2 bridge networking is typically achieved using CNI plugins
- PLUGINS: MacVLan / Bridge... each container receives its unique MAC address IN network namespace.
- Network Namespace: UNIQUE per pod unlike windows
- SAME NODE node can communicate directly using unique MAC addresses like windows
- SAME HOST L2 containers are isolated by netNS.  NOT the case in windows.

## Why bother learning about l2 and l3 networks ?

L2 and L3 cnis are confusing for people.  For example antrea is L2 (it uses a switch) vs calico is a l3 cni (IPIP).
Kind net is an example of a popular bridge  CNI https://www.tkng.io/cni/kindnet/ , that relies on layer2 network connectivity.
Its important to know the difference between layer2 connectivity (like what kind or l2bridge relies on) vs l3 (which is what calico relies on).

L3 is obviously more flexible.  I think... l2 tech is based on older stuff that works well, but probably isnt as reliable in the cloud world, bc
you cant as easily make gaurantees about the low level network connectivity of things.

That said things like OVS and L2 bridge are still used in the cloud, l2 bridge makes up the local part of windows CNIs in the cloud, which rely on otherthings
for higher level routing..... 

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
 
## l2bridge cni

In the old docker days of windows networking for k8s, people would make layer 2 bridges like so
```
docker network create -d l2bridge --subnet=10.244.3.0/24 -o com.docker.network.windowsshim.dnsservers=10.127.130.7,10.127.130.8 --gateway=10.244.3.1 -o com.docker.network.windowsshim.enable_outboundnat=true -o com.docker.network.windowsshim.outboundnat_exceptions=10.244.0.0/16,10.10.0.0/24,10.127.130.36/30 winl2bridge
```

See https://techcommunity.microsoft.com/t5/networking-blog/l2bridge-container-networking/ba-p/1180923 for details... 

##  win l2bridge CNI

So theres an real "l2 bridge" cni at https://github.com/containernetworking/plugins/blob/main/plugins/main/windows/win-bridge/win-bridge_windows.go that you can study.

L2 bridge CNI is a good example of a windows specific networking technology, so we should learn about it. 

Cloud CNIs can take advantage of the l2 bridge CNI as a building block in their CNI offering, but the l2bridge CNI isnt a "kubernete networking plugin" in the truest sense bc it
doesnt allow traffic to flow between nodes - it only forwards local (node) traffic from a CNI IP to the mac address of a windows pod. 

A "layer 2" bridge-based networking approach for container networking ON WINDOWS.

In a layer 2 bridge networking setup,
- containers are connected to the same layer 2 (data link layer) network,
- they are in the same broadcast domain
- the containers have unique MAC addresses, which map to their IPs.

So, whats an l2 container network look like ? 

- When a container is created in a Kubernetes cluster, the container runtime (e.g., Docker) delegates networking to the CNI plugin, in this case, the l2bridge CNI.
- The l2bridge CNI plugin creates a virtual bridge interface on the host.
- This bridge acts as a virtual switch connecting the containers to each other. Antreas OVS does a similar thing.
- Interface Attachment: new containers are created....  their network interfaces are attached to this bridge.
- MAC Addresses = The l2bridge CNI keeps track of the MAC addresses associated with each container's network interface,
- L2 bridge builds a table for the bridge of CNI IP -> Mac address. 
- When a container wants to communicate with another container within the same broadcast domain (bridge), it can send packets directly to the other container's MAC address.  This happens by the container
sending traffic to the IP, which gets routed via the switch to the mac address
- When outside containers need to talk to containers on a diff node, they get sent to the node first, then traffic is NAT'd to the underlying container.

