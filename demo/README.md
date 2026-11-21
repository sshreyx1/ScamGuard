# Kubernetes demo (minikube)

This runs the whole backend (gateway + aggregator + 6 detectors + BERT + Redis + Envoy) on Kubernetes, on
your own laptop, for free, using [minikube](https://minikube.sigs.k8s.io/) (a one-node Kubernetes cluster
that runs inside Docker — no cloud account needed). `docker compose up` (see the root [README](../README.md))
is the easiest way to run the backend day to day; this `demo/` folder is the version that shows the
distributed-systems behaviour: multiple replicas, autoscaling under load, pod-kill resilience, rolling
updates with no downtime.

Load balancing across pods is handled by [Envoy](https://www.envoyproxy.io/), which sits in front of every
scalable gRPC service — plain Kubernetes Services balance connections, not requests, which does not spread
load across replicas for long-lived gRPC connections.

## One-time setup

Install, if you don't have them already:

- **Docker Desktop**: https://www.docker.com/products/docker-desktop/ — open it once and make sure it says "running".
- **minikube**: https://minikube.sigs.k8s.io/docs/start/ (Windows: download `minikube-windows-amd64.exe`,
  rename to `minikube.exe`, put it on your `PATH`).
- **kubectl**: https://kubernetes.io/docs/tasks/tools/#kubectl (Docker Desktop on Windows/Mac already ships
  one — check with `kubectl version --client`).

`demo/up.sh` (Mac/Linux/WSL/Git-Bash) and `demo/up.ps1` (Windows PowerShell) check for all three and tell
you exactly what's missing, so you can just try running one of them and follow what it says.

### Laptop size

`up.sh`/`up.ps1` read what Docker reports (`docker info`) and pick sensible minikube/autoscaler settings on
their own:

- **6+ CPUs, 10+ GiB**: full settings (autoscaling up to 4 replicas, BERT 2 to 4).
- **Under 6 CPUs or under 10 GiB**: the script switches to a "lite" profile on its own (autoscaling up to
  2, BERT and the aggregator stay at 1 replica, fewer simulated load-test users) and says so.
- **Under 8 GiB total, on Windows**: raise WSL2's memory limit. Create/edit `%UserProfile%\.wslconfig`:
  ```
  [wsl2]
  memory=8GB
  processors=4
  ```
  then run `wsl --shutdown` and restart Docker Desktop. On Mac, raise it in Docker Desktop → Settings → Resources.

## Build and deploy

```bash
./demo/up.sh          # Mac/Linux/WSL/Git-Bash
```
```powershell
.\demo\up.ps1          # Windows PowerShell
```

This builds all 11 images (gateway, aggregator, 6 detectors, BERT, review, retrain), starts minikube, loads
the images into it, and deploys everything. The BERT image needs internet the first time: it downloads the
trained model from Hugging Face at build time. Re-running `up` is safe — it rebuilds only what changed and
re-applies the manifests.

When it finishes, `kubectl get pods,hpa` should show everything `Running` (aggregator and the English
impersonation detector start with 2 pods on purpose).

## Expose it to a phone or emulator

In its own terminal (leave it running):

```bash
./demo/expose.sh        # or .\demo\expose.ps1
```

This prints the HTTPS address to save in the app's Settings (`https://10.0.2.2:8443/api/` for the Android
emulator, `https://<Wi-Fi IP>:8443/api/` for a real phone). It goes through the Ingress, not a specific pod,
so the address keeps working across restarts. Install `demo/tls/ca.crt` on the device once. There is also a
plain-HTTP fallback, `./demo/port-forward.sh` (or `.ps1`), which forwards straight to one gateway pod and
reconnects on its own if that pod restarts.

Check it is alive:

```bash
curl http://127.0.0.1:8080/healthz             # {"status":"ok",...}
python3 demo/tools/demo_flow.py                 # plays 4 scripted calls, prints the verdicts
```

On Windows, use `127.0.0.1` rather than `localhost` — resolving `localhost` can add several hundred
milliseconds to every request while it tries IPv6 first.

## Showing the distributed-systems behaviour

- **Live traffic and scaling**: `./demo/watch.sh` (or `.ps1`) shows pods and autoscalers refreshing every
  2s; `python3 demo/tools/load_sim.py --users 40 --duration 70 --pace 25` generates load and pushes
  autoscaling to kick in within 10–20 seconds.
- **Even load spread**: `./demo/spread.sh` (or `.ps1`) shows the request count landing on each pod — this is
  what Envoy's per-request load balancing buys over a plain Kubernetes Service.
- **Fault tolerance**: `./demo/chaos.sh impersonation-en` (or `.ps1`) deletes a pod while load is running;
  Kubernetes replaces it and in-flight calls keep their verdicts.
- **Fail closed on purpose**: `kubectl scale deployment redis -n scamguard --replicas=0`, send one request
  (it fails, because the aggregator will not judge a call without its history), then scale Redis back up.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `docker pull`/build fails with `429 Too Many Requests` | Docker Hub's anonymous rate limit | `docker login` with a free account, then retry. |
| Pods stuck `Pending` | Not enough CPU/memory on the node | `kubectl describe pod <name> -n scamguard` will say "Insufficient cpu/memory". Free up RAM, raise Docker Desktop's memory limit, or re-run `up` (it picks a smaller profile automatically). |
| `bert` pod never becomes `Ready` | Model still loading, or a bad/partial download | Give it ~60s. If still not ready, `kubectl logs -n scamguard deploy/bert`; see [`backend/workers/bert/README.md`](../backend/workers/bert/README.md). |
| Gateway returns `502` | The aggregator or Redis is unreachable, or every detector timed out | `kubectl get pods -n scamguard`, then `kubectl logs -n scamguard deploy/aggregator`. |
| Phone/emulator can't connect | Usually the network, not the backend | Run `demo/tools/phone_check.sh`/`.ps1` first. Common causes: `port-forward` not running, a firewall, venue Wi-Fi blocking device-to-device traffic (use a phone hotspot instead), or a Settings URL that doesn't end in `/`. |
| `kubectl get hpa` shows `<unknown>` under TARGETS | metrics-server hasn't reported yet | Wait ~30–60s; `minikube addons enable metrics-server` if it's still stuck. |
| `minikube start` fails mentioning memory/resources | Docker Desktop's resource limit is too low | Docker Desktop → Settings → Resources → raise CPU/Memory, restart, retry. |
| Requests to `localhost:8080` go to the wrong backend | `docker compose` and this demo's `port-forward` both default to host port 8080 | Don't run both at once, or forward to a different port: `kubectl port-forward -n scamguard svc/gateway 18080:8080`. |
| Everything feels slow (every request ~0.5–2s) on Windows | Windows resolves `localhost` over IPv6 first and times out before falling back | Use `http://127.0.0.1:8080` instead of `http://localhost:8080` everywhere. |

## Tearing down

```bash
./demo/down.sh            # deletes the 'scamguard' namespace, keeps minikube running
./demo/down.sh --all       # also deletes minikube itself
```
