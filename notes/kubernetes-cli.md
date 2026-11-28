# Kubernetes CLI for DevOps

The wider reference behind [`kubernetes.md`](kubernetes.md), which is the triage order
for a bad shift. This one is the command surface: contexts, reading the cluster,
debugging, applying, capacity, access and output shaping.

## 1. Contexts, before anything else

```bash
kubectl config get-contexts
kubectl config use-context prod-cluster
kubectl config current-context
kubectl config view --minify --flatten
```

Every command below acts on the current context. Half of the "the deploy did nothing"
tickets are a context pointing at the wrong cluster, so check it first and check it
again before anything that writes.

```bash
kubectl config set-context --current --namespace=payments
export KUBECONFIG=~/.kube/prod:${KUBECONFIG}
```

`set-context --current --namespace` saves typing `-n payments` on every command.
`KUBECONFIG` accepts a colon separated list, which is how you keep a prod and a dev
file side by side without merging them by hand.

## 2. Reading the cluster

```bash
kubectl get all -n payments
kubectl get pods -o wide --show-labels
kubectl get pods -l app=api,tier=backend
kubectl get pods -A --field-selector status.phase!=Running
```

`get all` is a lie worth knowing: it covers pods, services, deployments, replicasets
and statefulsets, not configmaps, secrets, jobs, ingress or CRDs. Use `api-resources`
to see what else exists on this cluster.

```bash
kubectl api-resources --namespaced=true | grep -i monitoring
kubectl explain deployment.spec.strategy --recursive | head -40
```

`explain` reads the API schema from the server, so it is right for this version, not
for a blog post you found. Faster than guessing a field name in a YAML file.

```bash
kubectl describe pod api-7d9f8c6b5-x2k4m
kubectl events --for pod/api-7d9f8c6b5-x2k4m --sort-by=.lastTimestamp
```

`describe` ends with that object's events, which is where `ImagePullBackOff` and
`CrashLoopBackOff` explain themselves. `kubectl events --for` is the newer, less noisy
way to read them.

## 3. Logs, shells and debugging

```bash
kubectl logs deploy/api --tail=100 -f
kubectl logs api-7d9f8c6b5-x2k4m -c istio-proxy --since=10m
kubectl logs api-7d9f8c6b5-x2k4m --previous
kubectl logs -l app=api --all-containers --prefix
```

`--previous` is the only place a container that crashed on startup leaves evidence.
`-l` with `--prefix` reads every pod of a deployment at once, which is what you want
when only one replica is broken.

```bash
kubectl exec -it api-7d9f8c6b5-x2k4m -- sh
kubectl exec api-7d9f8c6b5-x2k4m -- env | sort | grep -i proxy
kubectl cp api-7d9f8c6b5-x2k4m:/tmp/dump.txt ./dump.txt
```

If `bash` is missing the image is probably distroless or Alpine: use `sh`. On a
distroless image there is no shell at all, so reach for `kubectl debug` instead.

```bash
kubectl debug -it api-7d9f8c6b5-x2k4m --image=busybox:1.36 --target=api
kubectl debug node/worker-1 -it --image=busybox:1.36
```

The first command adds an ephemeral container that shares the pod's network and
process namespace, so you can poke at the app without rebuilding the image. The
second drops you into the node's host namespaces through a pod.

```bash
kubectl port-forward svc/api 8080:80
kubectl top pod -n payments --containers
```

`port-forward` is a debugging tool: it holds a tunnel for the life of the command and
is not a way to expose a service. `top` needs metrics-server; if it errors, that is
the missing piece, not a permission problem.

## 4. Applying and rolling back

```bash
kubectl diff -f k8s/            # what would change
kubectl apply -f k8s/
kubectl apply -k overlays/prod/ # kustomize
kubectl delete -f k8s/old-job.yaml
```

`diff` before `apply` is the habit that prevents surprise deletions. `apply -k` runs
kustomize first, so the overlay's patches and name prefixes are part of what you see.

```bash
kubectl create deployment api --image=ghcr.io/heinsitt/api:1.4.0 --dry-run=client -o yaml > api.yaml
kubectl create configmap app-config --from-file=config/ --dry-run=client -o yaml
```

`--dry-run=client -o yaml` turns the imperative command into a manifest you can edit
and commit. It is the quickest way to get a correct skeleton for an unfamiliar
resource.

```bash
kubectl rollout status deploy/api --timeout=120s
kubectl rollout restart deploy/api
kubectl rollout history deploy/api
kubectl rollout undo deploy/api --to-revision=3
kubectl scale deploy/api --replicas=6
kubectl autoscale deploy/api --min=2 --max=10 --cpu-percent=70
```

`rollout status` blocks until the new ReplicaSet is available, so it is the correct
gate in a pipeline. `restart` is a rolling restart, not a rebuild: it re-pulls only
if the image tag changed and the pull policy allows it.

## 5. Jobs, cron jobs, config and secrets

```bash
kubectl create job --from=cronjob/nightly-backup backup-manual
kubectl get jobs -n payments --sort-by=.metadata.creationTimestamp
kubectl logs job/backup-manual
kubectl delete job backup-manual
```

`--from=cronjob` runs the schedule now, which is how you test a backup job without
waiting for 02:00. Finished jobs stay in the namespace until something deletes them,
so a nightly job leaves a year of dead objects unless `ttlSecondsAfterFinished` or a
cleanup policy exists.

```bash
kubectl create secret generic app-secrets --from-literal=DB_PASS=change-me \
  --dry-run=client -o yaml > secrets.yaml
kubectl create secret docker-registry ghcr \
  --docker-server=ghcr.io --docker-username=heinsitt --docker-password=...
kubectl get secret app-secrets -o jsonpath='{.data.DB_PASS}' | base64 -d
```

A Secret is base64, not encryption: anyone with `get secrets` in the namespace reads
it. Use `--dry-run=client -o yaml` to get a manifest, then keep it out of git or seal
it with Sealed Secrets or SOPS.

```bash
kubectl get cm app-config -o yaml
kubectl rollout restart deploy/api
```

Environment variables from a ConfigMap are read once, at container start. Editing the
ConfigMap does not change the running pod: the restart is the deploy. Mounted files do
update eventually, but most applications never re-read them.

## 6. Nodes, scheduling and capacity

```bash
kubectl get nodes -o wide
kubectl top node
kubectl describe node worker-1 | sed -n '/Allocated resources/,/Events/p'
```

The `Allocated resources` block compares requests against allocatable, which is the
honest answer to "why is this pod Pending". Real usage from `top` is a different
number: requests are what the scheduler reserved.

```bash
kubectl get pods -A -o wide --field-selector spec.nodeName=worker-1
kubectl get pods -A --field-selector status.phase=Pending
kubectl describe pod api-7d9f8c6b5-x2k4m | grep -A6 Events
```

A Pending pod is one of four things: not enough CPU or memory, a taint with no
toleration, a nodeSelector or affinity that matches nothing, or an unbound PVC. The
events at the bottom of `describe` say which.

```bash
kubectl cordon worker-1
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon worker-1
```

`drain` is `cordon` plus eviction. `--ignore-daemonsets` is required on almost every
real node, and `--delete-emptydir-data` is the flag that acknowledges emptyDir data
will be lost. Check PodDisruptionBudgets before draining anything that serves traffic.

```bash
kubectl taint nodes worker-2 dedicated=db:NoSchedule
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

## 7. Access and RBAC

```bash
kubectl auth can-i --list -n payments
kubectl auth can-i get secrets --as=system.serviceaccount:payments:api
kubectl get sa,role,rolebinding -n payments
kubectl create token api -n payments --duration=1h
```

`--as` impersonates, so you can answer "what can this ServiceAccount do" without
holding its token. `create token` prints a short lived JWT, which is the quickest way
to test the API from a pod's point of view.

```bash
kubectl get clusterrolebinding -o wide | grep -i cluster-admin
```

Every broad binding is a future incident. If a workload only needs to read pods in one
namespace, it should have a Role, not a ClusterRole.

## 8. Output shaping and housekeeping

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
kubectl get pods -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount' | tail -5
kubectl wait --for=condition=available deploy/api --timeout=120s
```

`jsonpath` is for scripts, `custom-columns` is for humans, and `--sort-by` is how you
find the one pod that has restarted forty times. `wait` turns "applied" into
"actually serving", which is what a pipeline should gate on.

```bash
kubectl get pods -A -o json | jq -r '.items[] | select(.status.containerStatuses[]?.restartCount > 5) | "\(.metadata.namespace)/\(.metadata.name)"'
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -20
```

The first line is the crashloop report nobody writes until an incident. The second is
the cluster's recent history in order, which is often enough to correlate a deploy
with a failure without opening a dashboard.

## One rule about writing

Imperative commands are for reading and for debugging. Anything that should still be
true tomorrow belongs in a manifest, a Helm chart or a kustomize overlay, applied by
Argo CD or Flux. A `kubectl edit` that fixes an outage is fine; a `kubectl edit` that
stays is drift, and the next sync will quietly undo it.
