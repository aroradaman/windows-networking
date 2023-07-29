# history

HCN and HCS are the backbone of K8s networking.  We'll contrast them w/ WMI and Netsh in a second for you old school windows admins.
But first lets get straight to the point.

- Containerd relies heavily on HCS API
- CNIs and Kube proxy rely heavily on the HCN API

| Operation                | HCN API                                   | HCS API                                    |
|--------------------------|-------------------------------------------|--------------------------------------------|
| Network Creation         | Yes, create container networks            | No                                         |
| Network Deletion         | Yes, delete container networks            | No                                         |
| Network Enumeration      | Yes, list all container networks          | No                                         |
| Network Details          | Yes, get details of container networks    | No                                         |
| Container Creation       | No                                        | Yes, create containers                     |
| Container Deletion       | No                                        | Yes, delete containers                     |
| Container Enumeration    | No                                        | Yes, list all containers                   |
| Container Details        | No                                        | Yes, get details of containers             |
| Container Start/Stop     | No                                        | Yes, start/stop containers                 |
| Container Pause/Resume   | No                                        | Yes, pause/resume containers               |
| Network Endpoint Creation| Yes, create endpoints for networks        | No                                         |
| Network Endpoint Deletion| Yes, delete endpoints for networks        | No                                         |
| Network Policy Management| Yes, manage network policies              | No                                         |
| Container Resource Management | No                                  | Yes, manage container resources like CPU and memory  |

## HyperV

A long time ago, Hyper-V networking APIs were designed for:
- VMs and reply to Hyper-V VM needs. 

VMs existed BEFORE containers, so.. well... the use of these APIs for container networking is a little awkward at times.

So, later on, MSFT made the HNS APIs
- these were designed for containers and Docker/ContainerD/Kubernetes/... needs. 
- These two APIs are not quite functionally interchangeable. 

## Examples of HyperV WMI vs HNS

Now, alot of people think about WMI for windows networking.  WMI's networking commands call out to HyperV commands. Thus, you can imagine, we
probably shouldnt try to use WMI for things related to container networks!  And you'd be right.  

2 examples of things you CANNOT DO with the hyperV APIS, taken from jocelyn B at MSFT.

- create / attach endpoints to Docker containers (whether these containers are Windows Server Containers, or Hyper-V isolated containers)
- attach network endpoints to traditional Hyper-V VMs

Examples of things that overlap between HNS and Hyperv
- VMSwitches: when HNS creates a vmswitch, the vmswitch is also visible in Hyper-V.
- Hypervisor for windows

Thus for windows networking in general, the rule for "container" vs "VM" networking programming is:

- containers = HNS API. 
- traditional Hyper-V VMs =  Hyper-V WMI API.

Per jocelyn at microsoft: Dont use the word "Hyper-v" to refer to container networks in MSFT, theyre really intended to be treated as separate networking data models.

## WMI

Although not relevant to containers, WMI comes up alot in the windows networking world, and understanding what it DOES do helps us to understand the HNS APIs specific niche
better:

- WMI can be used to monitor hard drive space, retrieve a list of installed applications, or even trigger actions based on specific system events.
- WMI is for gathering an inventory of software installed on a machine
- WMI can for security ~ detecting changes in FS, changing sec. settings
- wMI for networking ~ and you can do firewall'y tpy things w it.

BUT , again we dont use it for k8s networking ! Instead we use the HNS API.

## netsh

Theres a great article on how to do TCP dumps of AKS clusters on windows: https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/capture-tcp-dump-windows-node-aks?tabs=ssh. You'll notice it uses a tool called `netsh`.    So, just like WMI, we have another old school windows networking tool, netsh.  You could imagine building a user space kubernetes
windows proxy w/ netsh.  i.e. doing things like this:

```
# make a new interface
netsh interface ipv4 set interface "Interface Name" forwarding=enabled
# add a route from service ip 10.0..... to pod 100.1.2.3 
route ADD 100.1.2.3 MASK 255.255.255.255 10.0.96.10
```

In fact, before host process containers existed, this was a way, for example, to forward traffic over to prometheus containers.

## HNS

Now given that `netsh` and `winRM` appear to have similarities w/ iptables, you might assume they are the caulk at the bottom
of the windows networking tooling for k8s, right? wrong! Actually, we use *HNS*.  So the `iptables` rules that we make in the 
linux kube proxy, for example:

```
-A KUBE-SVC-SAL3JMSY3XQSA64S -m comment --comment "tkg-system/tkr-resolver-cluster-webhook-service -> 100.96.1.15:9443" -j KUBE-SEP-APPCGWWBW4GQN2HK                                                                                                                 
-A KUBE-SVC-SZBZVVMNBX2D3VFK ! -s 100.96.0.0/11 -d 100.68.215.151/32 -p tcp -m comment --comment "capi-kubeadm-bootstrap-system/capi-kubeadm-bootstrap-webhook-service cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ                                              -A KUBE-SVC-SZBZVVMNBX2D3VFK -m comment --comment "capi-kubeadm-bootstrap-system/capi-kubeadm-bootstrap-webhook-service -> 100.96.1.8:9443" -j KUBE-SEP-Z5LDPDXWJ7G5IJIQ                                                                                             -A KUBE-SVC-TCOU7JCQXEZGVUNU ! -s 100.96.0.0/11 -d 100.64.0.10/32 -p udp -m comment --comment "kube-system/kube-dns:dns cluster IP" -m udp --dport 53 -j KUBE-MARK-MASQ                                                                                              
```

Are actually implemented using the HNS api,  which means you can (kinda) forget all of this netsh and WMI stuff.  For k8s, we care about HNS..  

But for the sake of intuition, heres a table comparint these 3 networking APIs for the windows kernel.

| Feature             | Netsh                           | HNS                              | WinRM                         |
|---------------------|---------------------------------|----------------------------------|-------------------------------|
| Application focus   | Traditional network configurations | Container network configurations | Remote management           |
| Networking aspects  | Wide range                       | Focused on container networking  | Not directly involved       |
| Scriptability       | Yes                             | Via PowerShell/REST API          | Yes                         |
| Remote Execution    | No                              | No                               | Yes                         |
| Usage with Kubernetes | Not directly                  | Yes                              | Not directly                |
| Port forwarding     | Yes                             | Yes                              | No                          |
| Firewall management | Yes                             | Indirectly (via system)          | No                          |
| Routing             | Yes                             | Yes                              | No                          |
| Security Management | Yes (IPSec etc.)                | Indirectly (via system)          | Yes (Kerberos, certificates)  |

## WMI Provider  for HNS

But wait, theres a caveat here:  there is a WMI provider for HNS APIs.
However, it's recommended to use the direct JSON REST APIs for HNS. 

- The JSON REST APIs for HNS are better (more up-to-date). 
- New ambiguities in HNS API and documentation, which dont translate all the way back to legacy WinRM constructs.
