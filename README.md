# hello-newapp

Flask demo app with Prometheus metrics, built by Jenkins and deployed with GitOps:

```
Jenkins ─► docker build (dind) ─► push docker.io/<user>/helloworld:<BUILD_NUMBER> + :latest
        └► helm template ./chart ─► commit to elevy99927-demos/gitops
                                    systems/<DEPARTMENT>/<ENV>/<user>.yaml ─► Argo CD
```

## Jenkins parameters

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `ENV` | `dev` | Environment folder in gitops. The folder must already exist. |
| `DEPARTMENT` | `devops` | Department folder in gitops: `hr`, `finance`, `legal`, `digital`, `devops` |
| `SERVICE_MONITOR` | off | Render the ServiceMonitor (needs the Prometheus Operator CRDs) |
| `DIND_STORAGE_DRIVER` | `overlay2` | dind storage. Use `vfs` only if `overlay2` fails on your nodes |

The first run of a new job uses the defaults; after it, Jenkins shows the parameters.

## Jenkins credentials

| ID | Kind | Value |
|----|------|-------|
| `docker-cred` | Username with password | `<user>` + a Docker Hub access token |
| `github-token` | Username with password | `elevy99927-demos` + a GitHub token with write access to the gitops repo |

## Build agent requirements

The build runs **docker-in-docker**: the agent pod starts its own `dockerd` in a **privileged** container.
It doesn't use the node's container runtime, so it works the same on containerd, CRI-O or Docker nodes.
The only requirement is that the agent namespace allows privileged pods.

| Platform | Works? | What to do |
|----------|--------|-----------|
| **minikube / kind** | ✅ | Nothing. |
| **EKS** (EC2 node groups) | ✅ | If Pod Security is enforced: `kubectl label ns <jenkins-ns> pod-security.kubernetes.io/enforce=privileged` |
| **EKS Fargate** | ❌ | Fargate doesn't allow privileged pods. Run the agents on an EC2 node group. |
| **Rancher** (RKE2 / k3s) | ✅ | CIS-hardened RKE2 defaults to `restricted`: label the namespace as for EKS. |
| **OpenShift** | ✅ | Allow the privileged SCC for the agent's service account: `oc adm policy add-scc-to-user privileged -z default -n <jenkins-ns>` |

Clusters with Kyverno/Gatekeeper policies may also block privileged pods; ask for an exception for the agent namespace.

### Built-in fixes for different clusters

- **Storage:** Docker's data is on an `emptyDir` (`/var/lib/docker`) on the node's disk, so `overlay2` works.
  If it still fails (old kernel, unusual filesystem), run with `DIND_STORAGE_DRIVER=vfs`.
- **MTU:** `docker build --network=host` runs the build steps in the pod's own network, so it works on CNIs
  with an MTU below 1500 (OpenShift OVN-Kubernetes, Calico/Canal VXLAN, Cilium) without tuning.
- **Docker Hub rate limit:** `docker login` happens before the build, so base images are pulled as `<user>`.
- **CPU architecture:** the image is built for the node's CPU. On arm64 nodes (Graviton) it is arm64 only.
