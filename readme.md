# Cilium installation on OpenShift (Google Cloud)

- [Cilium installation on OpenShift (Google Cloud)](#cilium-installation-on-openshift-google-cloud)
  - [Prerequisites](#prerequisites)
  - [Deploying the cluster](#deploying-the-cluster)
    - [Creating the installation configuration file](#creating-the-installation-configuration-file)
    - [Changing network configuration](#changing-network-configuration)
    - [Creating manifests](#creating-manifests)
    - [Preparing Cilium OLM manifests](#preparing-cilium-olm-manifests)
      - [Version 11](#version-11)
      - [Version 12](#version-12)
    - [Preparing CiliumConfig](#preparing-ciliumconfig)
      - [CiliumConfig version 11](#ciliumconfig-version-11)
      - [CiliumConfig version 12](#ciliumconfig-version-12)
    - [Cluster installation](#cluster-installation)
  - [Cilium deployment status](#cilium-deployment-status)
  - [Cilium connectivity test](#cilium-connectivity-test)
    - [Creating SCC](#creating-scc)
    - [Deploying the connectivity test](#deploying-the-connectivity-test)
    - [Cleanup after connectivity test](#cleanup-after-connectivity-test)

## Prerequisites

This guide describe the installation of Cilium `v1.11.7` and `v1.12.0`.

- OpenShift installation program - v4.10.28

```shell
curl -O https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.10.28/openshift-install-linux-4.10.28.tar.gz
sudo tar -xf openshift-install-linux-4.10.28.tar.gz -C /usr/local/bin/
```

- Cilium CLI

```shell
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
```

- SSH key pair for cluster node SSH access

```shell
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519_cilium
chmod 400 ~/.ssh/id_ed25519_cilium
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_cilium
```

- GCP service account

[Reference](https://docs.openshift.com/container-platform/4.10/installing/installing_gcp/installing-gcp-account.html#installation-gcp-service-account_installing-gcp-account)

Set the `GOOGLE_APPLICATION_CREDENTIALS` environment variable to the full path to your service account private key file.

Replace `account.json` to correct service account private key file

```shell
CREDENTIALS=$(readlink -e account.json)
export GOOGLE_APPLICATION_CREDENTIALS=$CREDENTIALS
```

- Pull secret from [Red Hat Console](https://console.redhat.com/openshift/install/pull-secret)

## Deploying the cluster

### Creating the installation configuration file

```shell
CLUSTER_NAME="cilium-00"
openshift-install create install-config --dir "${CLUSTER_NAME}"
```

### Changing network configuration

```shell
sed -i "s/networkType: .*/networkType: Cilium/" "${CLUSTER_NAME}/install-config.yaml"
```

### Creating manifests

```shell
openshift-install create manifests --dir "${CLUSTER_NAME}"
```

### Preparing Cilium OLM manifests

#### Version 11

```shell
cilium_olm_rev="master"
cilium_version="1.11.7"
curl --silent --location --fail --show-error "https://github.com/cilium/cilium-olm/archive/${cilium_olm_rev}.tar.gz" --output /tmp/cilium-olm.tgz
tar -C /tmp -xf /tmp/cilium-olm.tgz
cp /tmp/cilium-olm-${cilium_olm_rev}/manifests/cilium.v${cilium_version}/* "${CLUSTER_NAME}/manifests"
rm -rf -- /tmp/cilium-olm.tgz "/tmp/cilium-olm-${cilium_olm_rev}"
ls ${CLUSTER_NAME}/manifests/cluster-network-*-cilium-*
```

#### Version 12

```shell
cilium_olm_rev="master"
cilium_version="1.12.0"
curl --silent --location --fail --show-error "https://github.com/cilium/cilium-olm/archive/${cilium_olm_rev}.tar.gz" --output /tmp/cilium-olm.tgz
tar -C /tmp -xf /tmp/cilium-olm.tgz
cp /tmp/cilium-olm-${cilium_olm_rev}/manifests/cilium.v${cilium_version}/* "${CLUSTER_NAME}/manifests"
rm -rf -- /tmp/cilium-olm.tgz "/tmp/cilium-olm-${cilium_olm_rev}"
ls ${CLUSTER_NAME}/manifests/cluster-network-*-cilium-*
```

### Preparing CiliumConfig

Replace default `CiliumConfig` with following YAML definition file

#### CiliumConfig version 11

```shell
cat <<EOF > ${CLUSTER_NAME}/manifests/cluster-network-07-cilium-ciliumconfig.yaml
apiVersion: cilium.io/v1alpha1
kind: CiliumConfig
metadata:
  name: cilium
  namespace: cilium
spec:
  ipam:
    mode: "cluster-pool"
    operator:
      clusterPoolIPv4PodCIDR: "10.128.0.0/14"
      clusterPoolIPv4MaskSize: "23"
  nativeRoutingCIDR: "10.128.0.0/14"
  endpointRoutes: {enabled: true}
  kubeProxyReplacement: "probe"
  clusterHealthPort: 9940
  tunnelPort: 4789
  cni:
    binPath: "/var/lib/cni/bin"
    confPath: "/var/run/multus/cni/net.d"
  prometheus:
    serviceMonitor: {enabled: false}
  hubble:
    tls: {enabled: false}
    relay: {enabled: true}
    ui: {enabled: true}
EOF
```

#### CiliumConfig version 12

```shell
cat <<EOF > ${CLUSTER_NAME}/manifests/cluster-network-07-cilium-ciliumconfig.yaml
apiVersion: cilium.io/v1alpha1
kind: CiliumConfig
metadata:
  name: cilium
  namespace: cilium
spec:
  ipam:
    mode: "cluster-pool"
    operator:
      clusterPoolIPv4PodCIDR: "10.128.0.0/14"
      clusterPoolIPv4MaskSize: "23"
  tunnelPort: 4789
  cni:
    binPath: "/var/lib/cni/bin"
    confPath: "/var/run/multus/cni/net.d"
  prometheus:
    serviceMonitor: {enabled: false}
  hubble:
    tls: {enabled: false}
    relay: {enabled: true}
    ui: {enabled: true}
EOF
```

### Cluster installation

```shell
openshift-install create cluster --dir "${CLUSTER_NAME}"
```

When the cluster deployment completes, directions for accessing your cluster, including a link to its web console and credentials for the kubeadmin user, display in your terminal.

## Cilium deployment status

```shell
cilium status -n cilium
```

Output:

```text
    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         OK
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/

Deployment        cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet         cilium             Desired: 6, Ready: 6/6, Available: 6/6
Deployment        hubble-relay       Desired: 1, Ready: 1/1, Available: 1/1
Deployment        hubble-ui          Desired: 1, Ready: 1/1, Available: 1/1
Containers:       cilium             Running: 6
                  cilium-operator    Running: 2
                  hubble-relay       Running: 1
                  hubble-ui          Running: 1
Cluster Pods:     119/163 managed by Cilium
Image versions    hubble-relay       quay.io/cilium/
```

## Cilium connectivity test

### Creating SCC

```shell
oc apply -f - <<EOF
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: cilium-test
allowHostPorts: true
allowHostNetwork: true
users:
  - system:serviceaccount:cilium-test:default
priority: null
readOnlyRootFilesystem: false
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
volumes: null
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostPID: false
allowPrivilegeEscalation: false
allowPrivilegedContainer: false
allowedCapabilities: null
defaultAddCapabilities: null
requiredDropCapabilities: null
groups: null
EOF
```

### Deploying the connectivity test

```shell
oc new-project cilium-test
oc apply -n cilium-test -f https://raw.githubusercontent.com/cilium/cilium/v1.11/examples/kubernetes/connectivity-check/connectivity-check.yaml
oc get pods -n cilium-test
```

### Cleanup after connectivity test

```shell
oc delete ns cilium-test
oc delete scc cilium-test
```
