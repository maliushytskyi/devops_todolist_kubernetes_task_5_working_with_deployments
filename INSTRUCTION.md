# Deploying the ToDo app to Kubernetes

## Prerequisites
- A running cluster (e.g. `minikube start`) and `kubectl` configured for it.
- The HPA needs the metrics server. On minikube: `minikube addons enable metrics-server`.
- In `deployment.yml`, set `image:` to your own image on Docker Hub (e.g. `<username>/todoapp:2.0.0`).

## Deploy
```bash
kubectl create namespace mateapp
kubectl apply -f deployment.yml
kubectl apply -f hpa.yml
```

Check the result:
```bash
kubectl get deployment,pods,hpa -n mateapp
```
In the idle state you should see 2 running pods.

## Access the app
No Service is required by the task, so use port-forwarding:
```bash
kubectl port-forward -n mateapp deployment/todoapp 8000:8000
```
Then open http://localhost:8000/ (landing page) or http://localhost:8000/api/ (API).

## Resource requests and limits
| | CPU | Memory |
|---|---|---|
| requests | 100m | 128Mi |
| limits | 250m | 256Mi |

- Django with the dev server is a small, mostly idle process, so a small request (a tenth of a core, 128Mi) is enough to schedule 2 pods on a modest cluster such as minikube.
- Requests are what the scheduler reserves and what the HPA uses as the 100% baseline for utilization.
- Limits are 2-2.5x the requests: this leaves headroom for bursts (e.g. API load) while preventing one pod from starving the node. A pod above the memory limit is OOM-killed and restarted; above the CPU limit it is throttled.

## HPA configuration
- `minReplicas: 2` keeps the app available if one pod dies or is being updated; `maxReplicas: 5` caps resource usage on a small cluster.
- CPU target 70%: scales out before pods saturate while avoiding constant scaling on small spikes.
- Memory target 80%: memory is not released quickly, so a slightly higher threshold avoids needless scale-ups, but still reacts before the 256Mi limit is hit.
- Both metrics are set; the HPA scales on whichever one demands more replicas.

## Update strategy
`RollingUpdate` with `maxSurge: 1` and `maxUnavailable: 0`:
- With 2 replicas, a new pod is started first (3 pods briefly), and an old one is removed only after the new one is Ready. So there are always at least 2 ready pods: zero downtime.
- `maxSurge: 1` limits the extra resource usage during a rollout to a single pod.
- The readiness probe (TCP 8000) makes sure traffic goes only to pods that have started.
