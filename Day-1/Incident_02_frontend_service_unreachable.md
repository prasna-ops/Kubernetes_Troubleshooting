# Incident #2: frontend Service Unreachable After Label Cleanup

Oct 16, 2026 · Prasan Gupta

## Incident summary

The `frontend` Service in namespace `shop` sent no traffic to its pods because of two misconfigurations: its selector did not match the pod labels, and its `targetPort` pointed to 8080 while nginx listens on 80. The pods were healthy the whole time; the Service routing was broken.

| Field | Value |
| --- | --- |
| Service | `frontend` (namespace `shop`) |
| Severity | P1 |
| Trigger | Platform team "label cleanup" the day before |
| Symptom | `checkout-client` 100% failing to reach `http://frontend` |
| Customer impact | Full storefront outage |
| Root cause 1 | Service selector `app: frontend-web` matched no pods (pods are labeled `app: frontend`) |
| Root cause 2 | Service `targetPort: 8080`, but the nginx container listens on port 80 |
| Fix | Selector changed to `app: frontend`; `targetPort` changed to 80 |
| Environment | Lab: AWS EC2 (ap-south-1), kind cluster `lab`, Kubernetes v1.31.0 |

## The request path

Every hop has to work for traffic to reach a pod. This incident broke two hops at the same time.

```
checkout-client
   │  http://frontend
   ▼
DNS (CoreDNS)  ──►  Service ClusterIP (port 80)
                         │
                         │  selector: which pods?      <-- broken #1
                         ▼
                    Endpoints (podIP:targetPort)
                         │
                         │  targetPort: which port?    <-- broken #2
                         ▼
                    nginx pod listening on :80
```

| Hop | What to check | Command |
| --- | --- | --- |
| DNS | Does the name resolve? | `kubectl exec <pod> -- nslookup frontend` |
| Service | Does it exist, what selector and ports? | `kubectl describe svc frontend -n shop` |
| Endpoints | Did the selector find any pods? | `kubectl get endpoints frontend -n shop` |
| Pod labels | Do labels match the selector exactly? | `kubectl get pods -n shop --show-labels` |
| Container port | What does the app listen on? | `kubectl describe pod -n shop -l app=frontend` |
| Direct test | Does the pod answer on that port? | `kubectl exec <client> -- wget -qO- http://<podIP>:<port>` |

## Investigation, step by step

### Alert

`checkout-client` in namespace `shop` cannot reach the `frontend` service; 100% of requests failing. The app team said: "Our pods are Running and healthy, we didn't deploy anything, it must be the network." The only recent change was a platform team label cleanup.

### Step 1: Check the pods

```bash
kubectl get pods -n shop
kubectl describe pod -n shop <frontend-pod>
```

Result: both `frontend` pods were `1/1 Running`, zero restarts, readiness probe passing. The app team was right that the pods were healthy. The problem had to be between the client and the pods.

### Step 2: Check the Service

```bash
kubectl get svc -n shop
```

The Service existed with a ClusterIP and port 80, so DNS and the Service object were present. `get svc` does not show whether the Service actually reaches any pods.

### Step 3: Check the Endpoints and compare labels

```bash
kubectl get endpoints frontend -n shop
kubectl describe svc frontend -n shop
kubectl get pods -n shop --show-labels
```

Example output:

```
NAME       ENDPOINTS   AGE
frontend   <none>      5m

Selector:          app=frontend-web
TargetPort:        8080/TCP
Endpoints:         <none>

NAME                 READY  STATUS   LABELS
frontend-xxxx-aaaa   1/1    Running  app=frontend,pod-template-hash=...,tier=web
frontend-xxxx-bbbb   1/1    Running  app=frontend,pod-template-hash=...,tier=web
```

How to read it:

- **`Endpoints: <none>`** means the Service selector matched zero pods.
- **Selector `app=frontend-web`** versus pod label **`app=frontend`**: the values differ, so nothing matches.
- Labels must match **exactly**: same key, same value, case-sensitive.

### Step 4: Fix #1, the selector

```bash
kubectl edit svc frontend -n shop
```

```yaml
spec:
  selector:
    app: frontend        # was: frontend-web
```

```bash
kubectl get endpoints frontend -n shop
```

Endpoints now listed both pod IPs. The client logs **still showed errors**, so there was a second problem.

### Step 5: Check the port

```bash
kubectl get endpoints frontend -n shop
kubectl describe pod -n shop -l app=frontend | grep -i port
```

Example output:

```
NAME       ENDPOINTS                         AGE
frontend   10.244.0.5:8080,10.244.0.6:8080   8m

    Port:          80/TCP
    Readiness:     http-get http://:80/ ...
```

The Endpoints showed port **8080**, but the container listens on **80**. The client error was `Connection refused`: the request reached the pod IP, but nothing was listening on 8080.

Optional proof, testing the pod directly without the Service:

```bash
kubectl exec -n shop deploy/checkout-client -- wget -qO- -T 3 http://<pod-IP>:80     # returns HTML
kubectl exec -n shop deploy/checkout-client -- wget -qO- -T 3 http://<pod-IP>:8080   # connection refused
```

### Step 6: Fix #2, the targetPort

```bash
kubectl edit svc frontend -n shop
```

```yaml
spec:
  ports:
  - name: http
    port: 80             # unchanged: the port clients call on the Service
    targetPort: 80       # was: 8080, now matches the container's port
```

`targetPort` was the field to change because it is the port on the **pod**. `port` is the port clients use on the **Service**; clients call `http://frontend` (port 80), so that was already correct.

### Step 7: Verify

```bash
kubectl get endpoints frontend -n shop
kubectl logs -n shop deployment/checkout-client --tail=5
```

- [x] Endpoints list both pod IPs on port 80
- [x] Client logs show `OK frontend responded (<title>Welcome to nginx!</title>)`

## Root cause analysis

The label cleanup changed the Service selector to `app: frontend-web` without changing the pod labels, and the Service also had `targetPort: 8080` while nginx serves on 80.

**Five whys:**

1. Why was the storefront down? The client could not reach any frontend pod through the Service.
2. Why not? The Service had no endpoints, and after that was fixed, it sent traffic to the wrong port.
3. Why no endpoints? The selector `app: frontend-web` did not match the pod label `app: frontend`.
4. Why was the selector wrong? The label cleanup changed the Service but not the Deployment's pod template.
5. Why was it not caught? No check verifies that a Service has endpoints after a change, and there was no alert on empty endpoints.

**Why "pods are healthy" was true but misleading:** a pod's readiness probe checks the pod itself. It says nothing about whether any Service routes to it.

## Key concepts

### port vs targetPort vs containerPort

| Field | Where it lives | Meaning |
| --- | --- | --- |
| `port` | Service | Port clients connect to on the Service (`frontend:80`) |
| `targetPort` | Service | Port on the pod that the Service forwards to |
| `containerPort` | Pod spec | Port the container declares; informational only |
| `nodePort` | Service (NodePort type) | Port opened on every node, range 30000-32767 |

`containerPort` does not open or block anything. The app listens wherever its own config says. `targetPort` must match the port the app **actually** listens on.

Tip: use a **named port**. Declare `ports: - name: http, containerPort: 80` in the pod, then set `targetPort: http` in the Service. If the container port changes, the Service follows automatically.

### Reading connection errors

| Error | What it means | Usual cause |
| --- | --- | --- |
| `Connection refused` | Reached the IP, nothing listening on that port | Wrong `targetPort`, app not started, or Service with no endpoints |
| `timed out` | No reply at all | NetworkPolicy, security group, NACL, routing |
| `bad address` / `could not resolve` | DNS lookup failed | Wrong name or namespace, CoreDNS problem |
| HTTP 503 from an ingress or proxy | Proxy has no healthy backend | No endpoints or all pods unready |

## Prevention and lessons learned

The strongest prevention is treating Service selectors and pod labels as one unit: change them together, in the same manifest or Helm chart, and verify endpoints after every change.

| Prevention | How it helps |
| --- | --- |
| Keep Service and Deployment in the same Helm chart or Kustomize base, with labels from one shared value | Selector and labels cannot drift apart |
| Use named ports (`targetPort: http`) | Port changes in the pod do not break the Service |
| Post-deploy check: fail the pipeline if `kubectl get endpoints` is empty | Catches selector mistakes in seconds |
| Alert on Services with zero ready endpoints (kube-state-metrics) | Detection before customers notice |
| Synthetic checks calling the Service from inside the cluster | Measures the real request path, not just pod health |
| GitOps (Argo CD or Flux) instead of `kubectl edit` | Changes are reviewed, and the next sync will not undo the fix |
| Treat "cleanup" changes as real changes: PR, review, staging first | Low-risk-looking changes still cause outages |

### Lessons learned

- Healthy pods do not mean a working Service. Always check **Endpoints**.
- `kubectl get svc` is not enough; use `kubectl describe svc` to see the selector, target port and endpoints.
- After fixing one problem, test again; a second problem can hide behind the first.
- `Connection refused` points to a port or listener problem, not a firewall.
- `kubectl edit` on a live cluster creates drift; in production, fix the manifest in Git.

### Troubleshooting order for "service unreachable"

1. Pods healthy? `kubectl get pods -n <ns>`
2. Service exists, right selector and ports? `kubectl describe svc <name> -n <ns>`
3. Endpoints populated? `kubectl get endpoints <name> -n <ns>`
4. Labels match selector? `kubectl get pods -n <ns> --show-labels`
5. Port matches app? Compare `targetPort` with what the container listens on.
6. Direct pod test, bypassing the Service: `wget http://<podIP>:<port>`
7. DNS works? `nslookup <svc>` from a pod.
8. NetworkPolicy blocking? `kubectl get networkpolicy -n <ns>`

### Interview questions this scenario covers

- A Service exists and the pods are Running, but requests fail. How do you debug it?
- What are Endpoints (and EndpointSlices), and how are they created?
- What is the difference between `port`, `targetPort`, `containerPort` and `nodePort`?
- What does `Connection refused` tell you compared to a timeout?
- Why can a readiness probe pass while the Service is still broken?
- How would you detect a Service with no endpoints automatically?

## Cleanup

```bash
kubectl delete namespace shop
kubectl get ns
```
