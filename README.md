# Kubernetes Pods & Deployments with Minikube

## Student Information

**Name:** Egwu Chidiebere Agha

-----

## 1. Minikube Setup

I started a local Kubernetes cluster using Minikube, which spins up a lightweight, single-node cluster on my own machine so I could practice real Kubernetes workflows without needing cloud infrastructure.

To start the cluster, I ran:

```bash
minikube start
```

This provisions the cluster components (API server, kubelet, etc.) inside a local VM/container. Once it finished, I verified everything came up correctly with:

```bash
minikube status
```

This confirmed that the host, kubelet, and API server were all reported as `Running`.

Next, I verified the underlying Kubernetes node itself — separate from Minikube’s own status — using:

```bash
kubectl get nodes
kubectl cluster-info
```

`kubectl get nodes` showed a single node named `minikube` in a `Ready` state, running Kubernetes version `[fill in your version here]`. `kubectl cluster-info` confirmed the address of the control plane, proving `kubectl` was correctly talking to the cluster.

**Screenshots:** `screenshots/minikube-status.png`, `screenshots/nodes.png`

-----

## 2. Pod

Before introducing a Deployment, I created a single standalone Pod to understand the most basic building block in Kubernetes. A Pod wraps one or more containers together with shared networking and storage — in this case, just one Nginx container.

I created it with:

```bash
kubectl run nginx-pod --image=nginx:latest
```

This told Kubernetes to schedule a Pod named `nginx-pod` running the `nginx:latest` image directly, with no Deployment managing it.

I checked that it was created and running with:

```bash
kubectl get pods
kubectl get pods -o wide
```

The `-o wide` flag showed additional detail, including the Pod’s IP address and which node it landed on (`minikube`, since that’s the only node available).

To inspect the Pod in more depth, I ran:

```bash
kubectl describe pod nginx-pod
kubectl get pod nginx-pod -o yaml
```

`describe` gave a human-readable summary of the Pod’s status, container info, and recent events. The `-o yaml` output showed the full underlying object definition Kubernetes stores for it. From this I confirmed the Pod was using the `nginx:latest` image, was in a `Running` state, was running on the `minikube` node, contained exactly one container, and had an assigned Pod IP (visible in the screenshot).

**Screenshot:** `screenshots/pod.png`

-----

## 3. Deployment

The company wanted Kubernetes to manage the application instead of me running a Pod directly, so I created a Deployment — a controller that manages a set of identical Pods on my behalf, automatically keeping the desired number running.

I wrote the Deployment definition in `manifests/deployment.yaml`, specifying:

- Deployment name: `web-app`
- Container name: `nginx`
- Image: `nginx:latest`
- Initial replicas: `3`

I chose to name the Deployment `web-app` rather than `nginx` deliberately — the Deployment name describes the *role* of the application (a web app), while the container name (`nginx`) describes the specific technology running inside it. This separation is standard practice, since the underlying image could change later without needing to rename the Deployment.

I applied it with:

```bash
kubectl apply -f manifests/deployment.yaml
```

Unlike `kubectl run`, `apply` reads a YAML file describing the *desired state* and lets Kubernetes reconcile the cluster to match it — this is the standard, repeatable, version-controllable way to manage resources.

I then verified it came up correctly:

```bash
kubectl get deployments
kubectl get pods
kubectl get pods -o wide
```

This confirmed the `web-app` Deployment existed and had created 3 Pods, all running the `nginx:latest` image.

To understand what was happening underneath the Deployment, I also ran:

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

This revealed the actual object chain: the Deployment doesn’t create Pods directly — it creates a **ReplicaSet**, and the ReplicaSet is what actually creates and continuously watches the Pods. The Deployment mainly manages ReplicaSet versions (useful for rollouts), while the ReplicaSet’s only job is to keep exactly the desired number of Pods running.

The key difference from the standalone Pod in Part 2: that Pod was unmanaged, so if it were deleted, nothing would replace it. The Deployment’s Pods are managed by a ReplicaSet, which automatically detects and corrects any mismatch between the desired and actual Pod count. This is exactly why a Deployment is preferred over creating individual Pods manually in any real system — it provides automatic self-healing, easy scaling, and rolling updates instead of requiring manual tracking and recreation of each Pod.

**Screenshot:** `screenshots/deployment.png`

-----

## 4. Scaling

With `web-app` running at 3 replicas, I simulated the application becoming more popular by scaling it up, then simulated demand dropping by scaling it back down: **3 → 5 → 2**.

To scale up to 5 replicas:

```bash
kubectl scale deployment web-app --replicas=5
kubectl get deployment
kubectl get pods
```

This confirmed 5 Pods were now running. Scaling doesn’t involve manually creating new Pods — I only changed the desired replica count, and the ReplicaSet controller automatically created 2 additional Pods to close the gap between desired (5) and actual (previously 3).

Then, as a challenge, I scaled back down to 2 replicas:

```bash
kubectl scale deployment web-app --replicas=2
kubectl get deployment
kubectl get pods
```

This confirmed the Pod count dropped from 5 to 2 — the ReplicaSet terminated 3 Pods to bring the actual count back in line with the new desired count.

This demonstrated why scaling a Deployment is so much easier than manually creating or deleting individual Pods: one command changes the desired count, and Kubernetes handles all the underlying creation, termination, and rescheduling automatically. Manually managing Pods would mean tracking every Pod’s name and status by hand and adjusting each one individually.

**Screenshot:** `screenshots/scaling.png`

-----

## 5. Self-Healing

This was the most important practical task, demonstrating Kubernetes’ core self-healing behavior. Starting from 2 Pods (the state left over from scaling down), I confirmed the current Pods with:

```bash
kubectl get pods
```

I then deliberately deleted one Pod by name:

```bash
kubectl delete pod <pod-name>
```

Immediately after, I re-checked the Pods:

```bash
kubectl get pods
```

I also watched the process live with:

```bash
kubectl get pods -w
```

**What happened:** the moment the Pod was deleted, the ReplicaSet detected that the actual Pod count (1) no longer matched the desired count (2), and it automatically created a brand-new Pod — with a new name and a new IP address — to replace it. I did not manually create this replacement Pod myself. Within a few seconds, the Deployment was back to exactly 2 Pods.

This happened because the ReplicaSet doesn’t just create Pods once and walk away — it continuously reconciles the actual state of the cluster against the desired state declared in the Deployment spec. Any mismatch, whether caused by a deliberate deletion, a crash, or a node failure, gets automatically corrected.

This demonstrates the Kubernetes concept of **self-healing / desired-state reconciliation** — the system constantly works to keep what’s actually running in line with what was declared, without any manual intervention.

**Screenshot:** `screenshots/self-healing.png`

-----

## 6. Troubleshooting

To simulate a real-world deployment mistake, I intentionally broke the Deployment by changing the container image to an invalid tag that doesn’t exist on Docker Hub:

```yaml
image: nginx:invalid
```

I applied the broken change:

```bash
kubectl apply -f manifests/deployment.yaml
```

Then checked the Pods:

```bash
kubectl get pods
```

The new Pods showed a status of `ErrImagePull`, which then progressed to `ImagePullBackOff` — Kubernetes’ way of indicating it tried to pull the image, failed, and was now backing off before retrying.

To investigate further, I ran:

```bash
kubectl describe pod <pod-name>
```

In the Pod’s Events section, the error clearly showed Kubernetes was unable to pull the image `nginx:invalid` because no image with that tag exists in the registry (a “manifest unknown” / “not found” style error).

**Why this happened:** Kubernetes doesn’t validate image tags when you write the YAML — it only discovers a tag is invalid when it actually tries to pull the image from the registry at Pod creation time. Since `nginx:invalid` doesn’t correspond to any real image, there was nothing for Kubernetes to download and run the container from.

**How I fixed it:** I edited `manifests/deployment.yaml` and restored the image to a valid tag:

```yaml
image: nginx:latest
```

Then re-applied the Deployment:

```bash
kubectl apply -f manifests/deployment.yaml
```

Kubernetes immediately began replacing the broken Pods with new ones using the valid image, and they transitioned to `Running` once the image was successfully pulled. This task reinforced that `kubectl describe pod` is the essential first step for diagnosing any Pod that isn’t behaving as expected — the Events section almost always reveals the root cause.

**Screenshot:** `screenshots/troubleshooting.png`

-----

## Clean Up

Once the assignment was complete, I removed all resources to avoid leaving anything running unnecessarily:

```bash
kubectl delete deployment web-app
kubectl delete pod nginx-pod
kubectl get pods
minikube stop
```

Deleting the Deployment automatically removed its ReplicaSet and all associated Pods. I then deleted the original standalone Pod separately, verified nothing remained with `kubectl get pods`, and finally stopped the Minikube cluster itself.

-----
