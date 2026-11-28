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
