# Gatekeeper playground


Based on article https://medium.com/@LachlanEvenson/mutating-kubernetes-resources-with-gatekeeper-3e5585d49ead

Gatekeeper can now mutate resources https://open-policy-agent.github.io/gatekeeper/website/docs/mutation/

## Create cluster

```bash
K8S_VERSION=${k8sVersion:-v1.22.0@sha256:b8bda84bb3a190e6e028b1760d277454a72267a5454b57db34437c34a588d047}
CLUSTER_NAME=${CLUSTER_NAME:-gatekeeper}
cat <<EOF | kind create cluster --image kindest/node:${K8S_VERSION} --name ${CLUSTER_NAME} --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
nodes:
- role: control-plane
- role: worker
EOF
echo "Waiting on cluster to be ready..."
sleep 10
kubectl wait pod --timeout=-1s --for=condition=Ready -l '!job-name' -n kube-system > /dev/null
```

## Install Gatekeeper

Download gatekeeper-mutation
```bash
GATEKEEPER_VERSION=${GATEKEEPER_VERSION:-v3.5.2}
curl -fLO https://raw.githubusercontent.com/open-policy-agent/gatekeeper/$GATEKEEPER_VERSION/deploy/experimental/gatekeeper-mutation.yaml
```

```bash
kubectl apply -f gatekeeper-mutation.yaml
```

### Verify Gatekeeper installation

```bash
kubectl get pods -n gatekeeper-system
```

## Examples

### Example AssignMetadata


Create an `AssingMetadata` resource to add a `location` label

```yaml
apiVersion: mutations.gatekeeper.sh/v1alpha1
kind: AssignMetadata
metadata:
  name: label-location
spec:
  match:
    scope: Namespaced
    namespaces: ["default"]
  location: "metadata.labels.location"
  parameters:
    assign:
      value: "puerto-rico"
```

```bash
kubectl apply -f assign-metadata.yaml
```

Create a pod

```bash
kubectl run nginx --image quay.io/bitnami/nginx
```

Check the labels

```bash
kubectl get pods --show-labels
```

Expected output:
```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
nginx   1/1     Running   0          27s   location=puerto-rico,run=nginx
```

Delete the pod
```
kubectl delete pod nginx
```

### Example Assign

Create a `Assign` resource to set priviledge `false`

```yaml
apiVersion: mutations.gatekeeper.sh/v1alpha1
kind: Assign
metadata:
  name: set-privileged-false
spec:
  applyTo:
  - groups: [""]
    kinds: ["Pod"]
    versions: ["v1"]
  match:
    scope: Namespaced
    namespaces: ["default"]
  location: "spec.containers[name:*].securityContext.privileged"
  parameters:
    assign:
      value: false
```

```bash
kubectl apply -f assign.yaml
```

create a pod with priviledge `true`

```bash
kubectl run nginx --image quay.io/bitnami/nginx --privileged=true
```

Verify priviledge on the new pod
```bash
kubectl get pod nginx -o yaml | grep privileged
```

## Clean up

Uninstall gatekeeper

```bash
kubectl delete -f gatekeeper-mutation.yaml
```

Delete cluster

```bash
kind delete cluster --name ${CLUSTER_NAME}
```