# Kubernetes Multi-Container Pod, NFS Storage & Health Probes Lab

A hands-on learning lab using a two-container Pod (Nginx + BusyBox), shared `emptyDir` storage, an external NFS export, container resource requests/limits, and Kubernetes startup, liveness, and readiness probes.

## Lab environment

- Kubernetes: Docker Desktop Kubernetes
- Host environment: Windows 11 + Ubuntu on WSL2
- Containers: `nginx:alpine` and `busybox:1.36`
- NFS server: Ubuntu WSL2
- Workload: Pod named `sidecar-demo`

> **Environment-specific setting:** The manifest currently contains the NFS server IP used during this lab (`172.29.20.211`). WSL IP addresses can change. Before deploying, update `spec.volumes[].nfs.server` to the current IP reachable from your Kubernetes node.

## What this lab demonstrates

### 1. Multi-container Pod and Sidecar Pattern

- Nginx serves content from `/usr/share/nginx/html`.
- BusyBox periodically writes the current date to `/shared/index.html`.
- Both containers mount the same `emptyDir` volume, so Nginx serves the file written by BusyBox.

### 2. NFS storage

- An NFS server exports `/mnt/shared`.
- The NFS volume is mounted in the Nginx container at `/mnt/nfs`.
- Read/write behavior was verified from the Pod and from the NFS server.

### 3. Resource requests and limits

| Container | CPU request | CPU limit | Memory request | Memory limit |
|---|---:|---:|---:|---:|
| `main-container` (Nginx) | `100m` | `250m` | `64Mi` | `128Mi` |
| `sidecar-container` (BusyBox) | `50m` | `100m` | `32Mi` | `64Mi` |

The Pod was observed with the `Burstable` QoS class.

### 4. Health probes

- **Startup probe:** allows time for the application to start before liveness and readiness checks take over.
- **Liveness probe:** detects an unhealthy container and can cause the kubelet to restart it after the configured failure threshold.
- **Readiness probe:** determines whether the container is ready to receive traffic. A readiness failure alone does not restart the container.

## Repository layout

```text
.
├── README.md
├── manifests/
│   └── sidecar-demo.yaml
├── screenshots/
│   ├── pod-status.png
│   ├── pod-resources-probes.png
│   ├── nfs-read-write.png
│   ├── liveness-recovery-watch.png
│   └── restart-counts.png
└── docs/
    └── troubleshooting.md
```

## NFS server setup (lab reference)

On the Ubuntu WSL2 machine used as the NFS server:

```bash
sudo apt update
sudo apt install nfs-kernel-server -y
sudo mkdir -p /mnt/shared
sudo chown nobody:nogroup /mnt/shared
sudo chmod 777 /mnt/shared
```

For this local lab, the export was configured in `/etc/exports` as:

```text
/mnt/shared *(rw,sync,no_subtree_check,insecure)
```

Then apply and verify:

```bash
sudo exportfs -a
sudo systemctl enable --now nfs-kernel-server
sudo exportfs -v
showmount -e localhost
```

**Security warning:** This broad export and `chmod 777` are for an isolated learning lab only. Do not copy them into production. Restrict exports to trusted client addresses, use least-privilege filesystem permissions, and review NFS security settings. The lab server used `root_squash` (the default).

## Deploy

First update the NFS server IP in `manifests/sidecar-demo.yaml`, then validate and apply:

```bash
kubectl apply --dry-run=client -f manifests/sidecar-demo.yaml
kubectl apply -f manifests/sidecar-demo.yaml
kubectl get pods
kubectl describe pod sidecar-demo
```

Expected Pod status after successful startup:

```text
sidecar-demo   2/2   Running
```

## Verify shared `emptyDir`

```bash
kubectl exec sidecar-demo -c main-container -- wget -qO- localhost
```

The response should show the date periodically written by the BusyBox sidecar.

## Verify NFS read/write

From the Nginx container:

```bash
kubectl exec sidecar-demo -c main-container -- sh -c 'echo "NFS from Kubernetes" > /mnt/nfs/linkedin-test.txt && cat /mnt/nfs/linkedin-test.txt'
```

From the Ubuntu NFS server:

```bash
cat /mnt/shared/linkedin-test.txt
```

The same text should be visible from both locations.

## Verify probes and container restart

Inspect probe configuration:

```bash
kubectl describe pod sidecar-demo
```

The output should list Startup, Liveness, and Readiness probes for `main-container`.

For a controlled lab test, stop Nginx:

```bash
kubectl exec sidecar-demo -c main-container -- nginx -s stop
kubectl get pod sidecar-demo -w
```

After the liveness check fails enough times, Kubernetes restarts the Nginx container. Confirm per-container restart counts:

```bash
kubectl get pod sidecar-demo -o jsonpath='{range .status.containerStatuses[*]}{.name}{" | Restarts: "}{.restartCount}{" | Ready: "}{.ready}{"\\n"}{end}'
```

Expected behavior: the Nginx container's restart count increases; the BusyBox sidecar should remain running.

## Cleanup

Delete the lab Pod:

```bash
kubectl delete pod sidecar-demo
```

Deleting the Pod does **not** delete files from the external NFS server. Remove NFS data or stop the NFS service separately only if you intentionally want to clean up the server.

## Screenshots

See the `screenshots/` directory for terminal evidence captured during the lab.

## Learning outcomes

- Multi-container Pods and the Sidecar Pattern
- Shared `emptyDir` volumes
- NFS volume integration
- CPU and memory requests/limits
- Kubernetes QoS classification
- Startup, liveness, and readiness probes
- Troubleshooting mount failures and immutable Pod fields
- Observing automatic container recovery

This is an educational local lab, not a production-ready deployment.
