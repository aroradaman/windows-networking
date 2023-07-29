# cloud-cni

Local CNIs like antrea use OVS, and calico uses vxlan (local) or IPIP for production clusters.  

- containers in the cloud have cloud IPs, no need for local IPAM at the CNI level
- containers in the cloud have network devices attached to them from the cloud at their VM, so theyre faster.

local CNIs like calico are different than azure and eks cnis.

Lets look at some diffs in a table.  

| Feature                            | Azure CNI (Windows)                | AWS VPC CNI (EKS)                | Calico CNI for Windows         |
|------------------------------------|-----------------------------------|----------------------------------|---------------------------------|
| Networking                         | Azure CNI assigns an IP address from the subnet to every pod, making them fully accessible with their own IP address in the network. | EKS CNI plugin assigns each pod a valid VPC IP address, enabling direct communication with services like RDS and ElastiCache. | Calico CNI assigns IP addresses to pods and provides network policy management. |
| Node Isolation                     | Not inherently using Hyper-V isolation. | N/A  | Not inherently using Hyper-V isolation. |
| Security                           | Applies Azure Network Policies at the subnet level. | Applies AWS Security Group at the instance level. | Provides network policy management for fine-grained security control. |
| Scalability                        | Supports up to 4000 nodes.        | Supports up to 5000 nodes.       | Scales to large clusters with thousands of nodes. |
| IP Address Management              | Delegated to Azure.               | Managed by CNI plugin (IPAMD).   | Calico IPAM handles IP address management. |
| IP Exhaustion Mitigation           | Azure platform handles automatic IP address assignment and reclamation. | Manages a "warm pool" of IPs, and automatically allocates new ENIs and IPs as needed. | Calico IPAM ensures efficient IP utilization and allocation. |
| IP for Pods                        | Every pod gets an IP from the subnet. | Every pod gets an IP from the VPC. | Every pod gets an IP from the network CIDR. |
| Inter-Pod Communication            | Uses overlay network for inter-pod communication. | Directly uses VPC networking for inter-pod communication. | Uses BGP-based networking for inter-pod communication. |
| Control Plane                      | Azure-managed.                    | AWS-managed.                     | Self-managed or third-party orchestrators like Kubernetes. |
| Maximum Pods per Node              | Varies by the VM size.            | Determined by formula: (Number of network interfaces for the instance type × (the number of IP addresses per network interface - 1)) + 2 | Varies based on node resources and IP allocation settings. |
| Warm Pool Management               | N/A                               | Configurable via environment variables: WARM_ENI_TARGET, WARM_IP_TARGET, MINIMUM_IP_TARGET | Supports advanced configuration for warm pool management. |
| Primary Networking Component       | Azure CNI Plugin.                 | CNI Binary and IPAMD.            | Calico CNI Plugin. |
| IP Address Reuse Strategy          | Delegated to Azure.               | Uses a cool-down cache to manage and reuse IP addresses. | Calico CNI efficiently reuses IP addresses for optimal utilization. |
| Configurable Network Policy        | Yes                               | Yes | Yes |
| Container Isolation                | Uses process isolation by default. Hyper-V isolation can be enabled for stronger security boundaries. | All containers in a pod share a network namespace. | Uses Windows process-level isolation for containers. |



## Examples

How woudl you create a CNI for a windows system ? stealing from calicos implementation: 

- Define a network
```
$network = @{
  "Subnet" = "10.0.0.0/16";
  "GatewayAddress" = "10.0.0.1";
  "Name" = "calico-network";
  "Type" = "Overlay";
}
```
Create the network
```
$networkId = (docker network create -d l2bridge --subnet=10.0.0.0/16 --gateway=10.0.0.1 -o com.docker.network.windowsshim.networkname=calico-network).Trim()
```
 Define an endpoint for a pod
```
$endpoint = @{
  "Name" = "endpoint1";
  "IPAddress" = "10.0.0.2";
  "Network" = $networkId;
}
```

Create an endpoint
```
  Add-HnsEndpoint -Endpoint $endpoint
```
