# Troubleshooting Notes

## 1. `unknown field "spec.containers[...].nfs"`

**Cause:** The NFS volume source was placed under `spec.containers` instead of `spec.volumes`.

**Fix:** Define the mount under the relevant container's `volumeMounts`, and define the NFS source under `spec.volumes`.

## 2. `pod updates may not change fields` when adding resources or probes

**Cause:** Most Pod spec fields are immutable after Pod creation. Adding resource settings or probes to an existing Pod is rejected.

**Fix for this standalone lab Pod:**

```bash
kubectl delete pod sidecar-demo
kubectl apply -f manifests/sidecar-demo.yaml
```

For managed workloads, use a Deployment and update its Pod template so Kubernetes performs a rollout.

## 3. NFS mount fails

Check the NFS server and export:

```bash
sudo systemctl status nfs-kernel-server
sudo exportfs -v
showmount -e localhost
```

Check that the Kubernetes node can reach the NFS server IP and TCP port 2049. Confirm the IP in the manifest is current and reachable from the Kubernetes node; a WSL IP can change.

## 4. Readiness becomes false while liveness restarts the container

A readiness failure removes the container from service endpoints; it does not itself restart the container. A liveness failure, after the configured failure threshold, causes the kubelet to restart the container.

## 5. `Restart Count` differs between containers

Inspect each container separately:

```bash
kubectl get pod sidecar-demo -o jsonpath='{range .status.containerStatuses[*]}{.name}{" | Restarts: "}{.restartCount}{" | Ready: "}{.ready}{"\\n"}{end}'
```

The Nginx container was observed restarting while the BusyBox sidecar continued running.
