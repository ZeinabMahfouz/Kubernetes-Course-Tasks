# Task 1: Multi-Container Pods (Init Container & Sidecar)

An Nginx "Hello" page running on a **3-node Minikube cluster**. The page shows the current timestamp and refreshes it every 5 seconds:

```text
hello in MLOPS MENA COMMUNITY : 2026-11-03 11:49:18
```

Each Pod has three containers, uses **no volumes**, and relies on `shareProcessNamespace: true` so the containers can see each other.

---

## 📁 Files

| File | Purpose |
|---|---|
| `deployment.yaml` | The working Deployment (3 replicas, init container + nginx + sidecar) |
| `deployment-broken.yaml` | Same Deployment, but `init-done` runs `exit 1` on purpose |
| `service.yaml` | NodePort Service that exposes the Pods on port 80 |
| `screenshots/` | Evidence for every step below |

---

## 📦 Pod Architecture

| Container | Type | What it does |
|---|---|---|
| `init-done` | Init container (`busybox:1.36`) | Prints `done` and exits `0`. The other containers start only after it succeeds. |
| `nginx` | Main container (`nginx:1.27-alpine`) | Writes the first `index.html` with the current time, then serves it on port `80` |
| `time-updater` | Sidecar (`busybox:1.36`) | Every 5 seconds, finds the nginx master PID and rewrites the page through `/proc/<pid>/root/usr/share/nginx/html/index.html` |

### How the sidecar edits nginx's file without a volume

1. `shareProcessNamespace: true` puts all containers in the Pod in **one process namespace**, so `time-updater` can see the nginx process.
2. `pgrep -f "nginx: master"` gets the nginx PID.
3. `/proc/<pid>/root` points to **that process's root filesystem**. Writing to `/proc/<pid>/root/usr/share/nginx/html/index.html` changes the file nginx is serving.
4. The `SYS_PTRACE` capability lets the sidecar access another container's `/proc/<pid>/root`.

### Zero-downtime settings

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # create at most 1 extra Pod during an update
    maxUnavailable: 0  # never remove an old Pod before a new one is Ready
```

A `readinessProbe` on nginx means a Pod receives traffic only once it actually answers HTTP. `topologySpreadConstraints` spread the 3 replicas across the 3 nodes.

---

## 🚀 How to Run

```bash
# 1. Start a 3-node cluster
minikube start --nodes 3 --driver=docker
kubectl get nodes

# 2. Deploy
kubectl apply -f Task-1/
kubectl get pods -o wide -w

# 3. Open the page (on Windows, keep this terminal open)
minikube service hello-nginx --url
```

---

## ✅ Part 1: Normal Operation

### 3-node cluster ready
![Nodes](screenshots/01-nodes.png)

### Pods running, one per node
All 3 Pods are `Running` with `READY 2/2`. The init container doesn't count in READY because it already finished. The Pods are spread over `minikube`, `minikube-m02`, and `minikube-m03`.

![Pods running](screenshots/02-pods-running.png)

### Init container output and sidecar updates
```bash
kubectl logs <pod> -c init-done              # -> done
kubectl logs <pod> -c time-updater --tail=5  # -> one update every 5 seconds
```
![Sidecar logs](screenshots/03-sidecar-logs.png)

### Timestamp updates in the browser
The same page, refreshed a few seconds apart, shows a new timestamp:

| First load | After refresh |
|---|---|
| ![Browser 1](screenshots/04-browser-1.png) | ![Browser 2](screenshots/04-browser-2.png) |

> Timestamps are in UTC (container default), which is 3 hours behind Riyadh time (UTC+3).

---

## 💥 Part 2: Forcing the Init Container to Fail

In `deployment-broken.yaml`, the only change is the init container command:

```yaml
# before
command: ["sh", "-c", "echo done"]
# after
command: ["sh", "-c", "echo done; exit 1"]
```

```bash
kubectl apply -f Task-1/deployment-broken.yaml
kubectl get pods -o wide -w
```

### Observed: `Init:CrashLoopBackOff`
A new Pod (`hello-nginx-745649c95b-snj88`) is created. It cycles through `Init:0/1` → `Init:Error` → `Init:CrashLoopBackOff`, and its restart count keeps rising. Kubernetes waits longer between retries each time (exponential back-off).

The **3 original Pods remain `Running 2/2` with 0 restarts.**

![CrashLoopBackOff](screenshots/05-crashloop.png)

---

## 🔍 Part 3: Failure Diagnosis

```bash
kubectl describe pod hello-nginx-745649c95b-snj88
kubectl logs hello-nginx-745649c95b-snj88 -c init-done
kubectl get deployment hello-nginx
kubectl get replicasets
```

### `kubectl describe pod`: the init container

![Describe 1](screenshots/06-describe1.png)

![Describe 2](screenshots/07-describe2.png)

| Evidence | Value | Meaning |
|---|---|---|
| `init-done` → State | `Terminated`, `Reason: Error`, **`Exit Code: 1`** | The init container fails on every run |
| `init-done` → Restart Count | `5` | Kubernetes keeps retrying it |
| `nginx` / `time-updater` → State | `Waiting`, `Reason: PodInitializing`, empty Container ID | Never created, because the init gate blocked them |
| Conditions → `Initialized` | `False` | The Pod is stuck in the init phase |
| Events | `Warning BackOff ... Back-off restarting failed container init-done` | Source of the `CrashLoopBackOff` status |
| `kubectl logs -c init-done` | `done` | The command did run. The failure comes from the exit code, not from the echo. |

### Deployment and ReplicaSets during the failure

![Deployment status](screenshots/08-deployment-status.png)

```text
NAME          READY   UP-TO-DATE   AVAILABLE
hello-nginx   3/3     1            3

NAME                     DESIRED   CURRENT   READY
hello-nginx-745649c95b   1         1         0     <- new (broken) ReplicaSet
hello-nginx-d75ff8dfb    3         3         3     <- old (working) ReplicaSet
```

- `AVAILABLE 3`: all 3 old Pods still serve traffic.
- `UP-TO-DATE 1`: only 1 Pod has the new spec, and it is not Ready.
- Because of `maxUnavailable: 0`, the rollout **pauses**. No old Pod is removed while the new one is unhealthy.

### Zero downtime: the page keeps working during the failure

![Browser during failure](screenshots/09-browser-during-failure.png)

---

## 🔧 Part 4: Fix and Recovery

```bash
kubectl apply -f Task-1/deployment.yaml
kubectl rollout status deployment/hello-nginx
kubectl get pods -o wide
kubectl logs hello-nginx-d75ff8dfb-2nz8v -c time-updater --tail=3
```

![Recovered](screenshots/10-recovered.png)

### Verification checks

| Check | Result |
|---|---|
| Rollout | `deployment "hello-nginx" successfully rolled out` |
| Broken Pod | Removed |
| Original Pods | The same 3 Pods (`d75ff8dfb-*`), age **79m**, **0 restarts**. They were never interrupted. |
| Pod spread | One Pod on each node: `minikube`, `minikube-m02`, `minikube-m03` |
| Sidecar | Still updating every 5 seconds (`12:58:53 → 12:58:58 → 12:59:03`) |

**Why the old Pods were kept:** re-applying the original spec makes it match the old ReplicaSet (`d75ff8dfb`) again. Kubernetes scales the broken ReplicaSet to `0` and keeps the healthy Pods running. Nothing restarts.

---

## 📚 Key Takeaways

1. **Init containers are a gate.** If one exits non-zero, the main containers never start and the Pod shows `Init:Error` / `Init:CrashLoopBackOff`.
2. **`kubectl describe pod`** is the main diagnosis tool: exit code, restart count, container states, and events.
3. **`maxUnavailable: 0` + a readiness probe** give zero downtime during a bad rollout, because old Pods are kept until new ones are Ready.
4. **`shareProcessNamespace` + `/proc/<pid>/root`** let a sidecar reach another container's filesystem without a shared volume.
5. **Recovery** is just applying the good config again (or `kubectl rollout undo deployment/hello-nginx`).

---

## 🧹 Cleanup

```bash
kubectl delete -f Task-1/
minikube stop
```
