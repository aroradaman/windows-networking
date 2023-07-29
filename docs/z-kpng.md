# kpng-win

### KPNG
Kube-Proxy Next-Generation dec ouples the existing in-tree kube-proxy from proxy backends - userspace, iptabes and ipvs with option of running these backends in completely separate processes, or together with shared memory.
Along with that kpng also provides a set of tools like diff-store, a generics store for tracking deltas and generating incremental updates.  

#### Brain
Brain understands kubernetes constructs, connects to API Server, watches over nodes, services and endpoint slices and processes kubernetes specific logic like topology computation and provides the result of this computation to a client via gRPC watchable API.

### Backend
Backends are responsible for implementing the data-path for the packets. For any change in kubernetes object (addition of new endpoint), backend receives a full-state - a complete list of services with their endpoints and reconciles them. 

## Windows Service Proxy Implementation
This project is a fullstate implementation of a windows backend and to run this in kubernetes we consume the brain as library from `sigs.k8s.io/kpng`

### High Level Flow
To program a proxy in windows we will be dealing with HostComputeLoadBalancer and HostComputeEndpoints. We create two diff-stores, one for HostComputeEndpoint and one for HostComputeLoadBalancer. On receiving a fullstate callback we reset and populate diff-store with all the host compute objects we need program for the full state. After this we iterate over the incremental changes returned by diff-store and program the host compute objects in windows-kernel.
    
![flow.jpg](..%2Fimages%2Fflow.jpg)


### Low Level
#### Host Compute Endpoint
Host Compute Endpoints are virtual IP endpoints representing a virtual port in Hyper-V virtual switch. Unlike Kubernetes endpoints which belongs to a particular service, host compute endpoints are independent layer-2 objects and multiple host compute load balancers can use them.     

Our backend should only manage remote endpoints (endpoints not present on local machine) as local endpoints are owned and managed by CNIs.


```go
// Endpoint is a user-oriented definition of an HostComputeEndpoint in its entirety.
type Endpoint struct {
	// host compute identifier, returned after creation
	ID string

	IP         string
	IsLocal    bool
	MacAddress string
	ProviderIP string
}

```

#### Host Compute LoadBalancer
Host Compute load balancer is a fully configurable service which listens on provider VIP (Virtual IP) and load balances traffic to given host compute endpoints. Host compute load balancers and kubernetes services are almost identical.  

```go
// LoadBalancer is a user-oriented definition of an HostComputeEndpoint in its entirety.
type LoadBalancer struct {
	ID string
	// more documentation | these are subsets of global set of endpoints

	Endpoints []*Endpoint

	IP               string
	Flags            LoadBalancerFlags
	PortMappingFlags LoadBalancerPortMappingFlags
	SourceVip        string
	Protocol         uint32
	Port             int32
	TargetPort       int32
}

```

#### Callback
ServiceEndpoints pairs a Service with all its Endpoints.
```go
type ServiceEndpoints struct {
	Service   *localv1.Service
	Endpoints []*localv1.Endpoint
}
```
The flow and signature of KPNG callback: 
```go
func Callback(ch <-chan *client.ServiceEndpoints) {
    // iterate over service ports
        // cluster ip
        // iterate over endpoints
            // local endpoint
                // load HostCompute Endpoint from kernel
            // remote endpoint
                // create Host Compute Endpoint only if it doens't exists
                // add host compute endpoint to remote endpoints diffstore
            // crate HostCompute loadbalancer with all the host compute endpoints
            // add host compute loadbalancer to loadbalancer store 
    //
    
    programHostComputeObjects()
}

func programHostComputeObjects(){
    // iterate over deleted loadbalancers
        // delete loadbalancer
    
    // iterate over deleted endpoints
        // delete endpoints
        
    // iterate over updated endpoints
        // create/update endpoints
    
    // iterate over update loadbalancers
        // create/update loadbalancers
}

```
