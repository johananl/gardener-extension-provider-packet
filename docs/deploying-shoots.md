# Gardener - Deploying a shoot cluster on Equinix Metal (Packet)

This guide explains how to deploy a Gardener shoot cluster on Equinix Metal. The guide assumes
there is already a running garden cluster with at least one seed cluster registered which can serve
Equinix Metal shoot clusters.

The following instructions assume a Linux workstation. The instructions for macOS should be very
similar.

## Requirements

- A running garden cluster with at least one seed registered
- `kubectl` installed
- An Equinix Metal project ID and API key

## Instructions

**Important!** Gardener uses a special k8s API server which doesn't have any nodes or pods and
which deals with Gardener resources only. In this guide we refer to this API server as the
"Gardener API server". The kubeconfig file for accessing this API server is generated when creating
a garden cluster into `export/kube-apiserver/kubeconfig` in the "landscape" directory. The
"regular" API server that is used by the underlying k8s cluster is a *different* API server.

### Register the Packet extension controller

Gardener is [extensible](https://gardener.cloud/documentation/concepts/extensions/). In order for
us to be able to deploy workload clusters on Equinix Metal, we need to register the
[Packet provider extension](https://github.com/gardener/gardener-extension-provider-packet) with
the Gardener API server.

Set the `KUBECONFIG` environment variable to point at the kubeconfig file belonging to the Gardener
API server and test the connectivity:

```
cd my-landscape
export KUBECONFIG=$(pwd)/export/kube-apiserver/kubeconfig
kubectl get seeds
```

Sample output:

```
NAME   STATUS   PROVIDER   REGION         AGE    VERSION   K8S VERSION
aws    Ready    aws        eu-central-1   104m   v1.14.0   v1.18.12
```

Register the latest version of the Packet provider extension with the Gardener API server (`v1.8.0`
is used here):

```
export VERSION=v1.8.0
kubectl apply -f "https://raw.githubusercontent.com/gardener/gardener-extension-provider-packet/${VERSION}/example/controller-registration.yaml"
```

Verify the extension was registered successfully:

```
kubectl get controllerregistration provider-packet
```

Sample output:

```
NAME              RESOURCES                                                   AGE
provider-packet   ControlPlane/packet, Infrastructure/packet, Worker/packet   92s
```

Register the latest version of the CoreOS/Flatcar OS extension with the Gardener API server
(`v1.6.0` is used here):

```
export VERSION=v1.6.0
kubectl apply -f "https://raw.githubusercontent.com/gardener/gardener-extension-os-coreos/${VERSION}/example/controller-registration.yaml"
```

Verify the controller was registered successfully:

```
kubectl get controllerregistration os-coreos
```

Sample output:

```
NAME        RESOURCES                                                     AGE
os-coreos   OperatingSystemConfig/coreos, OperatingSystemConfig/flatcar   6s
```

### Create a CloudProfile

Gardener uses a k8s custom resource called `CloudProfile` for specifying the type of machines which
can be created for a given cloud provider.

Create a file called `cloudprofile.yaml`:

```
cat <<EOF >cloudprofile.yaml
---
apiVersion: core.gardener.cloud/v1beta1
kind: CloudProfile
metadata:
  name: packet-flatcar
spec:
  type: packet
  kubernetes:
    versions:
    - version: 1.20.0
    - version: 1.19.3
    - version: 1.18.12
  machineImages:
  - name: flatcar
    versions:
    - version: 0.0.0-stable
  machineTypes:
  - name: t1.small.x86
    cpu: "4"
    gpu: "0"
    memory: 8Gi
    usable: true
  - name: c2.medium.x86
    cpu: "24"
    gpu: "0"
    memory: 64Gi
    usable: true
  - name: c3.medium.x86
    cpu: "24"
    gpu: "0"
    memory: 64Gi
    usable: true
  - name: x1.small.x86
    cpu: "4"
    gpu: "0"
    memory: 32Gi
    usable: true
  regions:
  - name: ams1
  - name: ewr1
  - name: am6
  providerConfig:
    apiVersion: packet.provider.extensions.gardener.cloud/v1alpha1
    kind: CloudProfileConfig
    machineImages:
    - name: flatcar
      versions:
      - version: 0.0.0-stable
        id: flatcar_stable
EOF
```

Apply the `CloudProfile` to the Gardener API server:

```
kubectl apply -f cloudprofile.yaml
```

### Create and bind a secret

Gardener needs provider-specific secrets in order to be able to manage machines on behalf of the
user.

Create a secret containing the Equinix Metal API token and project ID on the Gardener API server:

```
export TOKEN=xxxxxxxx
export PROJECT_ID=xxxxxxxx

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: packet-shoots
  namespace: garden
type: Opaque
data:
  apiToken: $(echo -n $TOKEN | base64)
  projectID: $(echo -n $PROJECT_ID | base64)
EOF
```

Gardener uses a custom resource called `SecretBinding` to designate a user-specified secret as the
secret to use when talking to the cloud provider.

Create a `SecretBinding` on the Gardener API server:

```
cat <<EOF | kubectl apply -f -
---
apiVersion: core.gardener.cloud/v1beta1
kind: SecretBinding
metadata:
  name: packet-shoots
  namespace: garden
secretRef:
  name: packet-shoots
quotas: []
EOF
```

### Create a shoot cluster

Create a file called `shoot.yaml`:

```
cat <<EOF >shoot.yaml
---
apiVersion: core.gardener.cloud/v1alpha1
kind: Shoot
metadata:
  name: my-em-shoot
  namespace: garden
spec:
  cloudProfileName: packet-flatcar
  region: ams1
  secretBindingName: packet-shoots
  provider:
    type: packet
    infrastructureConfig:
      apiVersion: packet.provider.extensions.gardener.cloud/v1alpha1
      kind: InfrastructureConfig
    controlPlaneConfig:
      apiVersion: packet.provider.extensions.gardener.cloud/v1alpha1
      kind: ControlPlaneConfig
    workers:
    - name: worker
      machine:
        type: t1.small.x86
      minimum: 2
      maximum: 2
  networking:
    nodes: 10.80.128.0/25 # change me
    type: calico
  kubernetes:
    version: 1.20.0
  maintenance:
    autoUpdate:
      kubernetesVersion: true
      machineImageVersion: true
  addons:
    kubernetes-dashboard:
      enabled: true
    nginx-ingress:
      enabled: true
EOF
```

Edit `shoot.yaml` and modify `spec.networking.nodes`, which must match the project's **private
CIDR** in the relevant facility. Modify any other options as necessary.

Apply the Shoot to the Gardener API server:

```
kubectl apply -f shoot.yaml
```

At this point Gardener should take care of creating the shoot cluster on Equinix Metal. To monitor
the progress, either check the Gardener web UI or run the following command:

```
kubectl -n garden describe shoots my-em-cluster
```
