# HCN

Ok, now lets finally look at the Kubernetes networking bastion: HCN.  HCN has a few core objects that we use for the windows
kube proxy. 

### Host Compute Network

An HCN Network is an entity that's used to represent a host compute network and its associated system resources and policies. An HCN network typically can include:

* A set of metadata (ID, name, type)
* A virtual switch
* A host virtual network adapter (acting as a default gateway for the network)
* A NAT instance (if required by the network type)
* A set of subnet and MAC pools
* Any network-wide policies to be applied (for example, ACLs)

Example: Setup of a Network for Pods
```
Add-HcnNetwork -Network @{
    Name = "calico-network";
    Type = "Overlay";
    NetworkAdapterName = "Ethernet";
    Subnets = @(@{
        IPAddressPrefix = "192.168.0.0/16";
        Policies = @(@{
            Type = "VSID";
            VSID = 4096;
        })
    })
}
```
### Host Compute Endpoint

An HCN Endpoint is an entity that's used to represent an IP endpoint on an HCN network and its associated system resources and policies. An HCN endpoint typically consists of:

* A set of metadata (ID, name, parent network ID)
* Its network identity (IP address, MAC address)
* Any endpoint specific policies to be applied (ACLs, routes)

Lets look at how, hypothetically, you could create a pod network via an HCN Endpoint call:

Example: POD
```
Add-HcnEndpoint -Endpoint @{
    Name = "calico-endpoint";
    VirtualNetwork = "calico-network";
    IPAddress = "192.168.0.2";
    PrefixLength = 24;
    Policies = @(@{
        Type = "ACL";
        Settings = @{
            Rules = @( @{
                Protocol = 256;
                Action = "Allow";
                Direction = "Out";
                LocalAddresses = "0.0.0.0/0";
                RemoteAddresses = "0.0.0.0/0";
            })
        }
    })
}
```

### Host Compute LoadBalancer

The Kube Proxy uses a custom Loadbalancer API that the HCN defines.  You can see that it gets
activated if a service has session affinity
```
353   if flags.isIPv6 {
354     lbFlags |= LoadBalancerFlagsIPv6
355   }
356
357   lbDistributionType := hcn.LoadBalancerDistributionNone
358
```
Next, we check the sessionAffinity flag below... and CHANGE the *lbDistributionType*... to 
distribute VIA source IP.  This routes the incoming IP address consistently to the same backend.
```
359   if flags.sessionAffinity {
360     lbDistributionType = hcn.LoadBalancerDistributionSourceIP
361   }
362
363   loadBalancer := &hcn.HostComputeLoadBalancer{
364     SourceVIP: sourceVip,
365     PortMappings: []hcn.LoadBalancerPortMapping{
366       {
367         Protocol:         uint32(protocol),
368         InternalPort:     internalPort,
369         ExternalPort:     externalPort,
370         DistributionType: lbDistributionType,
371         Flags:            lbPortMappingFlags,
372       },
373     },
```

An HCN load balancer is an entity that's used to represent a host compute network load balancer. Load balancers allow you to have load balanced host compute network endpoints.
So, lets now look at an example of how this would work in a kube proxy. Note the other examples showed how to make pods / networks, which is more of a CNI thing - but the kube proxy, focuses on the loadbalancing part:

```
# Define the VIP (Virtual IP) of the service and the destination IP of the Pod
$VIP = "10.96.0.10"
$destinationIP = "100.1.2.3"

# Fetch the network 
$network = Get-HnsNetwork | Where-Object { $_.Name -eq "YourNetworkName" } 

# Create a load balancing policy
$policy = @{
    "Type" = "LOAD_BALANCING"
    "IP" = $VIP
    "Protocol" = "TCP"
    "InternalPort" = "YourServicePort"
    "ExternalPort" = "YourServicePort"
    "LoadBalancerDistribution" = "SourceIP"
    "Destinations" = @($destinationIP)
}

# Convert policy to JSON
$policyJSON = $policy | ConvertTo-Json

# Add the policy to the network
Add-HnsLoadBalancer $network.Id $policyJSON
```

## Customizing the loadbalancer: Affinity

Open question... can you program probabilities on the HCN loadbalancers ? Not sure. But, there are some customizations you can do.

Theres a LoadBalancerPortMapping struct in the HNS Api, that allopws you to , via the `DistributionType` choosed to have biases in loadbalancing, i.e.
- have all client IPs go to the same IP (2)
- have all traffic of protocol X on client IP Y go to the same IP (1)

The *HostComputeLoadBalancer* points to a *LoadBalancerPortMapping* object which has customizabliity. 

```
// LoadBalancerPortMapping is associated with HostComputeLoadBalancer
type *LoadBalancerPortMapping** struct {
	Protocol         uint32                       `json:",omitempty"` // EX: TCP = 6, UDP = 17
	InternalPort     uint16                       `json:",omitempty"`
	ExternalPort     uint16                       `json:",omitempty"`
	DistributionType LoadBalancerDistribution     `json:",omitempty"` // EX: Distribute per connection = 0, distribute traffic of the same protocol per client IP = 1, distribute per client IP = 2
	Flags            LoadBalancerPortMappingFlags `json:",omitempty"`
}

// HostComputeLoadBalancer represents software load balancer.
type HostComputeLoadBalancer struct {
	Id                   string                    `json:"ID,omitempty"`
	HostComputeEndpoints []string                  `json:",omitempty"`
	SourceVIP            string                    `json:",omitempty"`
	FrontendVIPs         []string                  `json:",omitempty"`
	PortMappings         []LoadBalancerPortMapping `json:",omitempty"`
	SchemaVersion        SchemaVersion             `json:",omitempty"`
	Flags                LoadBalancerFlags         `json:",omitempty"` // 0: None, 1: EnableDirectServerReturn
}
```

 
```
// LoadBalancerPortMapping is associated with HostComputeLoadBalancer
type LoadBalancerPortMapping struct {
	Protocol     uint32                       `json:",omitempty"` // EX: TCP = 6, UDP = 17
	InternalPort uint16                       `json:",omitempty"`
	ExternalPort uint16                       `json:",omitempty"`
	Flags        LoadBalancerPortMappingFlags `json:",omitempty"`
	Protocol         uint32                       `json:",omitempty"` // EX: TCP = 6, UDP = 17
	InternalPort     uint16                       `json:",omitempty"`
	ExternalPort     uint16                       `json:",omitempty"`
	DistributionType LoadBalancerDistribution     `json:",omitempty"` // EX: Distribute per connection = 0, distribute traffic of the same protocol per client IP = 1, distribute per client IP = 2
	Flags            LoadBalancerPortMappingFlags `json:",omitempty"`
}
```


