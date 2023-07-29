## diffstore
DiffStore is a powerful tool provided by KPNG `sigs.k8s.io/kpng/diffstore` for tracking incremental changes and computing deltas after processing a full state.

#### Defining diff-store
```go
// loadBalancerStore
var loadBalancerStore = diffstore.NewAnyStore[string, *hcn.LoadBalancer](func(a, b *hcn.LoadBalancer) bool {
    return a.Equal(b)
})
```

#### Adding items to diff-store
```go
loadBalancerStore.Get(loadBalancer.Key()).Set(loadBalancer)
```

#### Computing incremental changes
```go
# Done() computes all the incremental changes internally
loadBalancerStore.Done()

# Reset() resets the diffstore for next fullstate
loadBalancerStore.Reset()
```
#### Iterating over incremental changes
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

