# Kubernetes

The ten commands that cover most of a shift, in the order I reach for them.

## First, know where you are pointing

```bash
kubectl config current-context
kubectl get nodes -o wide
```

A surprising number of "the deploy did nothing" tickets are a context pointing at
the wrong cluster. Check the context before you check the workload.

## What is broken

```bash
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get events --sort-by=.lastTimestamp | tail -30
```

Events are namespaced and short lived: look at them early, because they age out.
`--field-selector` is cheaper than grepping a full listing on a busy cluster.

## Why is it broken

```bash
kubectl describe pod api-7d9f8c6b5-x2k4m
kubectl logs deploy/api --tail=100 --all-pods=true
kubectl logs api-7d9f8c6b5-x2k4m --previous
```

`describe` ends with the events for that pod, which is where `ImagePullBackOff` and
`CrashLoopBackOff` explain themselves. `--previous` reads the logs of the container
that died, which is the only place a startup crash leaves evidence.

## Rollouts

```bash
kubectl rollout status deploy/api --timeout=120s
kubectl rollout undo deploy/api
kubectl rollout history deploy/api
```

`rollout status` blocks until the new ReplicaSet is actually available, so it is the
right gate in a pipeline. `undo` is one revision back by default; `--to-revision`
for anything further.

## Services and reachability

```bash
kubectl get svc,endpoints api
kubectl port-forward svc/api 8080:80
kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash
```

An empty `endpoints` list means the Service selector does not match any ready pod.
That is a label problem, not a network problem. `netshoot` saves you from
installing `curl` into a distroless image.

## Apply, carefully

```bash
kubectl diff -f k8s/
kubectl apply -f k8s/
```

`diff` shows what will change before it changes. On a cluster with an admission
controller or a GitOps sync, `apply` by hand is how drift starts: check whether
Argo CD or Flux owns the resource first.
