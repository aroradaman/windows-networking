# kpng-diffstore

Anyone building a kube proxy impl needs to diff endpoints/services periodically.

- When an endpoint is created, you need to updated LB rules to Loadbalkance to that endpoint
- When an endpoint is deleted, you need to delete a LB endpoint
- When a service is created, you need to add forwarding rule to a node to not reject that IP:port combination (and also add endpoints in the bullets above, likely)

... and so on.

So you need to know:

- when things were added (so you can update things) 
- what changed (so you can avoid rewriting already current forwarding rules on the node) 

KPNG has built an abstract tool for this called *DIFFSTORE* lets learn about it.

DiffStore is a powerful tool provided by KPNG `sigs.k8s.io/kpng/diffstore` for tracking incremental changes and computing deltas after processing a full state.

## Defining diff-store
```go
// loadBalancerStore
var loadBalancerStore = diffstore.NewAnyStore[string, *hcn.LoadBalancer](func(a, b *hcn.LoadBalancer) bool {
    return a.Equal(b)
})
```

## Adding items to diff-store
```go
loadBalancerStore.Get(loadBalancer.Key()).Set(loadBalancer)
```

## Computing incremental changes
```go
# Done() computes all the incremental changes internally
loadBalancerStore.Done()

# Reset() resets the diffstore for next fullstate
loadBalancerStore.Reset()
```
## Iterating over incremental changes
```go
for _, item := range loadBalancerStore.Deleted() {
    loadBalancer := item.Value().Get()
    // delete existing loadbalancer
}
	
for _, item := range loadBalancerStore.Changed() {
    loadBalancer := item.Value().Get()
    if item.Created() {
        // create new loadBalancer
    }else{
        // update existing loadBalancer
    }
}
```

