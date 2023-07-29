# hcn-hcs


## Host Compute Services
Host Compute Network (HCN) service API is a public-facing Win32 API that provides platform-level access to manage the virtual networks, virtual network endpoints, and associated policies. Together this provides connectivity and security for virtual machines (VMs) and containers running on a Windows host.

### Host Compute Network
An HCN Network is an entity that's used to represent a host compute network and its associated system resources and policies. An HCN network typically can include:

* A set of metadata (ID, name, type)
* A virtual switch
* A host virtual network adapter (acting as a default gateway for the network)
* A NAT instance (if required by the network type)
* A set of subnet and MAC pools
* Any network-wide policies to be applied (for example, ACLs)


### Host Compute Endpoint
An HCN Endpoint is an entity that's used to represent an IP endpoint on an HCN network and its associated system resources and policies. An HCN endpoint typically consists of:

* A set of metadata (ID, name, parent network ID)
* Its network identity (IP address, MAC address)
* Any endpoint specific policies to be applied (ACLs, routes)

### Host Compute LoadBalancer
An HCN load balancer is an entity that's used to represent a host compute network load balancer. Load balancers allow you to have load balanced host compute network endpoints.
