# Windows Networking

Welcome to our site about windows networking! This is a community effort, outside of the CNCF, and is written as a collection
of technical articles about the details of the kube proxy on windows which we discovered, while porting it over to KPNG.

The main focus of this site is biased around the kube-proxy, but we will try to add more and more content over time, as its 
not possible to understand the kube proxy w/o also thinking about things like network policies, CNIs, L2 and L3 networks, DRS, OVS, 
and so on.

As a first example, its interesting to consider the metadata associated with a kube proxy service in windows:

```
// todo @daman replace this !
type serviceInfo struct {
	*proxy.BaseServicePortInfo
	targetPort             int
	externalIPs            []*externalIPInfo
	loadBalancerIngressIPs []*loadBalancerIngressInfo
	hnsID                  string
	nodePorthnsID          string
	policyApplied          bool
	remoteEndpoint         *endpointsInfo
	hns                    HostNetworkService
	preserveDIP            bool
	localTrafficDSR        bool
	internalTrafficLocal   bool
	winProxyOptimization   bool
}
```

TODO add detailed definitions of these fields here... and how they're used.

## Contributors !

Please make PRs to https://github.com/aroradaman/windows-networking/ if you want to help us with this site, or file issues.
Our goal is to make windows on kubernetes networking problems fun to work on and easy to document.  We dont have a particularly
high bar for PRs! 

## CNI

All Kubernetes pods require a CNI, CNIs give pods an IP address that is routable within a cluster.  Before the Kube proxy can be usable,
thus, we need a CNI.

## Kube Proxy

On windows, or linux, the kube proxys purpose in life is to look at pods, services, and endpoints, and write low level networking rules that make it so that, when you hit a service,
you are forwarded to the underlying pod backing it: 

pods
```
kubo@uOFLhGS9YBJ3y:~$ kubectl get pods -A | grep dns                                                                                                                                                                                                                                                          
kube-system                         coredns-5656df985f-ltb5t                                              1/1     Running     0             14d                                                                                                                                                               
kube-system                         coredns-5656df985f-sl9nv                                              1/1     Running     0             14d                                                              ``
services
```                                                                                                 
kubo@uOFLhGS9YBJ3y:~$ kubectl get services -A | grep dns                                                                                                                                                                                                                                                      
kube-system                         kube-dns                                             ClusterIP   100.64.0.10      <none>        53/UDP,53/TCP,9153/TCP   14d
```
endpoints 
```
kubo@uOFLhGS9YBJ3y:~$ kubectl get endpoints -A | grep dns                                                                                                                                                                                                                                                    
kube-system                         kube-dns                                             100.96.1.3:53,100.96.1.8:53,100.96.1.3:53 + 3 more...   14d                                                                                                                                                          
```

## HostNetworking



## Host Process containers






