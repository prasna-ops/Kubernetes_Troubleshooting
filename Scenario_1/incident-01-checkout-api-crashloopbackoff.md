# Incident #1: checkout-api CrashLoopBackOff After v2.4.1 Release

Oct 16, 2026 · Prasan Gupta

## Incident summary

Release v2.4.1 of `checkout-api` crash-looped on startup because the code expected an environment variable named `DATABASE_HOST`, while the ConfigMap still provided `DB_HOST`. The rolling update kept the old v2.4.0 pods serving, so customers saw reduced capacity, not a full outage.

| Field | Value |
| --- | --- |
| Service | `checkout-api` (namespace `prod`) |
| Severity | P2 |
| Trigger | Deploy of v2.4.1, change-cause "new payment gateway config" |
| Symptom | New pods in `CrashLoopBackOff`, rollout stuck, alert: deployment not progressing |
| Customer impact | Partial: old v2.4.0 pods kept serving traffic at reduced capacity |
| Root cause | Env var renamed in code (`DB_HOST` to `DATABASE_HOST`) without updating the ConfigMap |
| Mitigation | `kubectl rollout undo` back to v2.4.0 |
| Permanent fix | Add `DATABASE_HOST` to the ConfigMap (keeping `DB_HOST`), then re-ship v2.4.1 |
| Environment | Lab: AWS EC2 (ap-south-1), Ubuntu 24.04, `t3.medium`, kind cluster `lab`, Kubernetes v1.31.0 |

## Lab environment setup

The lab runs on a single EC2 instance with Docker and a kind cluster, so nothing is installed on the Windows laptop.

1. Set a billing budget alert (Billing > Budgets) before launching anything.
2. Launch EC2: ap-south-1, Ubuntu 24.04 LTS, `t3.medium` (2 vCPU, 4 GB), 20 GB gp3, security group allowing SSH (22) from My IP only.
3. Connect from Windows using the built-in OpenSSH client.
4. Install Docker, kubectl and kind, then create the cluster.
5. Stop the instance after every session to avoid compute charges.

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER && newgrp docker

curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl

curl -Lo kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x kind && sudo mv kind /usr/local/bin/

kind create cluster --name lab
kubectl get nodes
```

The node showed `NotReady` for about 20 seconds, then `Ready`. That is normal: the CNI (pod networking) starts after the node registers. A node stuck in `NotReady` points to the CNI or kubelet.

### Side issue solved: SSH key rejected on Windows

The first SSH attempt failed with `WARNING: UNPROTECTED PRIVATE KEY FILE!` and `Permission denied (publickey)`.

- **Diagnosis:** The host fingerprint prompt appeared, so the network and security group were fine. SSH ignored the key because "Authenticated Users" could read the `.pem` file, inherited from the D: drive permissions.
- **Fix:** Restrict the file to the current user only.

```
icacls D:\Prasan_machine.pem /inheritance:r
icacls D:\Prasan_machine.pem /remove "NT AUTHORITY\Authenticated Users" "BUILTIN\Users" "Everyone"
icacls D:\Prasan_machine.pem /grant:r "%USERNAME%":R
ssh -i D:\Prasan_machine.pem ubuntu@<EC2-public-IP>
```

| SSH symptom | Likely cause |
| --- | --- |
| `Permission denied (publickey)` + unprotected key warning | Key file permissions too open |
| `Permission denied (publickey)` only | Wrong username (`ubuntu` vs `ec2-user`) or wrong key pair |
| Connection timed out | Security group, NACL, route table, or no public IP |

Best practice: store keys in `C:\Users\<you>\.ssh\`, which already has tight permissions.

## Investigation, step by step

Five commands took the investigation from alert to root cause: pod status, pod description, crashed container logs, the ConfigMap, and the rollout history.

### Alert

`checkout-api` deployment in namespace `prod` has not progressed for 10 minutes. Some pods are not Ready. The dev team had just shipped a new release.

### Step 1: Check pod status

```bash
kubectl get pods -n prod
```

```
NAME                            READY   STATUS             RESTARTS      AGE
checkout-api-6b8444b654-6nhf2   1/1     Running            0             94s
checkout-api-6b8444b654-72vtq   1/1     Running            0             94s
checkout-api-86d7f58446-47cck   0/1     CrashLoopBackOff   2 (17s ago)   37s
checkout-api-86d7f58446-rs6pz   0/1     CrashLoopBackOff   3 (37s ago)   86s
```

How to read it:

- **Two hashes in the pod names** (`6b8444b654` and `86d7f58446`) mean two ReplicaSets, which means two versions are running and a rollout is in progress.
- **`0/1` READY** means the container is not ready to receive traffic.
- **`CrashLoopBackOff`** means the container starts, dies, and Kubernetes restarts it with growing delays.
- **RESTARTS climbing** confirms repeated crashes.
- **Conclusion:** the new version crashes; the old version still serves.

### Step 2: Describe a crashing pod

Mistake made first: `kubectl describe -n prod <pod-name>` failed with `the server doesn't have a resource type`. The resource type is required: `kubectl describe pod <pod-name>`.

```bash
kubectl describe pod -n prod checkout-api-86d7f58446-47cck
```

Key lines from the output:

```
Labels:           app=checkout-api
                  version=v2.4.1
    State:          Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Fri, 16 Oct 2026 07:34:39 +0000
      Finished:     Fri, 16 Oct 2026 07:34:39 +0000
    Restart Count:  4
    Environment Variables from:
      checkout-config  ConfigMap  Optional: false
Events:
  Normal   Pulled     kubelet  Container image "busybox:1.36" already present on machine
  Normal   Created    kubelet  Created container app
  Normal   Started    kubelet  Started container app
  Warning  BackOff    kubelet  Back-off restarting failed container app
```

| Line | Meaning |
| --- | --- |
| `version=v2.4.1` | The crash came with the new release |
| `Reason: Error`, `Exit Code: 1` | The application exited on its own with an error; not a Kubernetes failure |
| Started and Finished in the same second | Dies instantly at startup, which points to config, env vars or arguments |
| Events: Pulled, Created, Started all fine | Not an image or registry problem |
| `Environment Variables from: checkout-config` | The app reads its config from this ConfigMap |

### Step 3: Read logs from the crashed container

```bash
kubectl logs -n prod checkout-api-86d7f58446-47cck --previous
```

```
checkout-api v2.4.1 starting
loading payment gateway config...
FATAL: required env DATABASE_HOST is not set, refusing to start
```

`--previous` shows the logs of the last crashed attempt. Without it, the output is often empty because a fresh container just restarted. Describe tells you **that** it crashed; logs tell you **why**.

### Step 4: Inspect the ConfigMap

```bash
kubectl get configmap checkout-config -n prod -o yaml
```

```
data:
  DB_HOST: orders-db.prod.svc.cluster.local
  DB_PORT: "5432"
```

The ConfigMap provides `DB_HOST`. There is no `DATABASE_HOST`.

### Step 5: Check rollout history

```bash
kubectl rollout history deployment/checkout-api -n prod
```

```
REVISION  CHANGE-CAUSE
1         release v2.4.0
2         release v2.4.1 - new payment gateway config
```

Revision 1 (v2.4.0) is the known-good rollback target. Revision 2 (v2.4.1) introduced the failure.

### Customer impact check

The rolling update does not remove old pods until new pods become Ready, so v2.4.0 kept serving. Confirm which pods receive traffic:

```bash
kubectl get endpoints -n prod checkout-api
```

## Root cause analysis

The v2.4.1 code renamed its database host variable from `DB_HOST` to `DATABASE_HOST`, but the ConfigMap in `prod` was never updated, so the app found no value and exited with code 1 at startup.

|  | v2.4.0 (working) | v2.4.1 (broken) |
| --- | --- | --- |
| Env var the code reads | `DB_HOST` | `DATABASE_HOST` |
| Provided by `checkout-config` | Yes | No |
| Result | Starts and serves | Exits with code 1, `CrashLoopBackOff` |

**Why it worked on the developer's machine:** `DATABASE_HOST` was set locally, so the missing key in the cluster config never showed up before deploy.

**Why the impact stayed partial:** The default RollingUpdate strategy keeps old pods until new ones pass readiness. The new pods never became Ready, so the rollout stalled and v2.4.0 kept serving.

**Five whys:**

1. Why did checkout degrade? New pods kept crashing and the rollout stalled.
2. Why did the pods crash? The app exited because `DATABASE_HOST` was not set.
3. Why was it not set? The ConfigMap only had `DB_HOST`.
4. Why was the ConfigMap not updated? The code change and the config change were not shipped together.
5. Why was that not caught? No check validates required env vars against the target environment's config before deploy.

## Resolution

The resolution follows the rule "restore first, fix second": roll back to v2.4.0, add the new config key alongside the old one, re-ship v2.4.1, then verify.

### Phase 1: Restore service

```bash
kubectl rollout undo deployment/checkout-api -n prod
kubectl rollout status deployment/checkout-api -n prod
kubectl get pods -n prod
```

Expected: 3 pods, all on the old hash `6b8444b654`, all `1/1 Running`. Status update to post: "Mitigated: rolled back to v2.4.0, investigating."

Rollback comes first because it is fast and proven safe. Debugging while customers are affected is slow and risky.

### Phase 2: Fix the config, backward compatible

```bash
kubectl patch configmap checkout-config -n prod --type merge \
  -p '{"data":{"DATABASE_HOST":"orders-db.prod.svc.cluster.local"}}'

kubectl get configmap checkout-config -n prod -o yaml
```

Keep `DB_HOST`: v2.4.0 still needs it, and removing it would break the rollback path. Config changes should be additive; remove the old key only after v2.4.1 is stable.

### Phase 3: Re-ship v2.4.1

```bash
kubectl rollout history deployment/checkout-api -n prod
kubectl rollout undo deployment/checkout-api -n prod --to-revision=2
kubectl rollout status deployment/checkout-api -n prod
```

After a rollback, revision numbers shift: old revision 1 becomes revision 3. Revision 2 is still v2.4.1.

Pods read env vars from a ConfigMap **only when the container starts**. Updating a ConfigMap does not change running pods, so a new rollout is required. (Volume-mounted ConfigMaps do update eventually, but env vars never do.)

### Phase 4: Verify

```bash
kubectl get pods -n prod -L version
kubectl logs -n prod deployment/checkout-api
kubectl get endpoints -n prod checkout-api
```

- [ ] 3 pods running with `version=v2.4.1`
- [ ] Logs show `connected to orders-db.prod.svc.cluster.local:5432`
- [ ] Endpoints list 3 pod IPs

### Lab vs real production

The ConfigMap was patched live because this is a lab. In production, change the YAML or Helm values in Git and deploy through the pipeline. A live `kubectl patch` creates drift, and the next deploy from Git would silently remove the fix.

## Prevention and lessons learned

The most effective prevention is shipping config and code together, so a renamed variable cannot reach prod without its value.

| Prevention | How it helps |
| --- | --- |
| Keep config with the app (Helm values or Kustomize in the same repo and PR) | Code change and config change ship together and are reviewed together |
| Deploy to staging with prod-like config first | The crash appears before prod |
| Startup check in CI (run the container with the target env's config) | Catches missing env vars before deploy |
| Additive, backward-compatible config changes | Old and new versions both run, so rollback stays safe |
| Readiness probes on every deployment | Broken pods never receive traffic |
| Alert on stalled rollouts (`progressDeadlineSeconds`, default 600s) | You hear about it in minutes |
| Auto-rollback with Argo Rollouts or Flagger | A failing canary rolls back without a human |
| GitOps (Argo CD or Flux) instead of `kubectl patch` | No drift between Git and the cluster |

### Lessons learned

- Always add the resource type to `kubectl describe`: `kubectl describe pod <name>`.
- Read logs with `--previous` for crash-looping containers.
- Exit code 1 plus an instant crash points to config, not Kubernetes.
- Restore first, fix second.
- ConfigMap changes need a new rollout when consumed as env vars.

### Exit code cheat sheet

| Exit code | Meaning |
| --- | --- |
| 0 | Exited normally (a long-running app should not exit) |
| 1 | Application error; read the logs |
| 126 | Command found but not executable |
| 127 | Command not found; wrong entrypoint or command |
| 137 | Killed by SIGKILL, usually OOMKilled |
| 143 | SIGTERM; normal graceful shutdown |

### Troubleshooting order for a crashing pod

1. `kubectl get pods -n <ns>`: status, restarts, which version.
2. `kubectl describe pod <name> -n <ns>`: exit code, reason, events.
3. `kubectl logs <name> -n <ns> --previous`: why it crashed.
4. Check config sources: ConfigMaps, Secrets, env.
5. `kubectl rollout history`: what changed and what to roll back to.
6. Mitigate (rollback), then fix, then verify.

### Interview questions this scenario covers

- What does `CrashLoopBackOff` mean, and how do you debug it?
- Why were old pods still running during a failed rollout?
- Do pods pick up ConfigMap changes automatically?
- How do you roll back a deployment, and to a specific revision?
- What is the difference between exit codes 1, 137 and 143?
- How would you stop this happening again?
