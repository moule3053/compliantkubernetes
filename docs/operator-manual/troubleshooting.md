# Troubleshooting Tools

Help! Something is wrong with my Compliant Kubernetes cluster. Fear no more, this guide will help you make sense.

This guide assumes that:

* You have [pre-requisites](operator-manual/getting-started/) installed.
* Your environment variables, in particular `CK8S_CONFIG_PATH` is set.
* Your config folder (e.g. [for OpenStack](/operator-manual/openstack/#initialize-configuration-folder)) is available.
* `compliantkubernetes-apps` and `compliantkubernetes-kubespray` is available.

## I have no clue where to start

If you get lost, start checking from the "physical layer" and up.

### Are the Nodes still accessible via SSH?

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    ansible -i $CK8S_CONFIG_PATH/inventory/$CLUSTER/inventory.ini all -m ping
done
```

### Are the Nodes "doing fine"?

Dmesg should not display unexpected messages. [OOM](https://en.wikipedia.org/wiki/Out_of_memory) will show up here.

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    ansible -i $CK8S_CONFIG_PATH/inventory/$CLUSTER/inventory.ini all -m shell -a 'echo; hostname; dmesg | tail -n 10'
done
```

Uptime should show high uptime (e.g., days) and low load (e.g., less than 3):

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    ansible -i $CK8S_CONFIG_PATH/inventory/$CLUSTER/inventory.ini all -m shell -a 'echo; hostname; uptime'
done
```

Any process that uses too much CPU?

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    ansible -i $CK8S_CONFIG_PATH/inventory/$CLUSTER/inventory.ini all -m shell -a 'echo; hostname; ps -Ao user,uid,comm,pid,pcpu,tty --sort=-pcpu | head -n 6'
done
```

Is there enough disk space? All writeable file-systems should have at least 30% free.

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    ansible -i $CK8S_CONFIG_PATH/inventory/$CLUSTER/inventory.ini all -m shell -a 'echo; hostname; df -h'
done
```

Is there enough available memory? There should be at least a few GB of available memory.

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    ansible -i $CK8S_CONFIG_PATH/inventory/$CLUSTER/inventory.ini all -m shell -a 'echo; hostname; cat /proc/meminfo | grep Available'
done
```

Can Nodes access the Internet?

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    ansible -i $CK8S_CONFIG_PATH/inventory/$CLUSTER/inventory.ini all -m shell -a 'echo; hostname; curl --silent  https://checkip.amazonaws.com'
done
```

### Are the Kubernetes clusters doing fine?

Are the Nodes reporting in on Kubernetes? All Kubernetes Nodes, both control-plane and workers, should be `Ready`:

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    sops exec-file ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml \
        'kubectl --kubeconfig {} get nodes'
done
```

### Is Rook doing fine?

If Rook is installed, is Rook doing fine? You should see `HEALTH_OK`.

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    sops exec-file ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml \
        'kubectl --kubeconfig {} -n rook-ceph apply -f ./compliantkubernetes-kubespray/rook/toolbox-deploy.yaml'
done

for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    CEPH_TOOLS_POD=$(
        sops exec-file ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml \
            'kubectl --kubeconfig {} -n rook-ceph get pod -l "app=rook-ceph-tools" -o name')
    echo $CEPH_TOOLS_POD
    sops exec-file ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml \
        'kubectl --kubeconfig {} -n rook-ceph exec '$CEPH_TOOLS_POD' -- ceph status'
done
```

### Are Kubernetes resources doing fine?

Are all Pod fine? Pods should be `Running` or `Completed`, and fully `Ready` (e.g., `1/1` or `6/6`)?

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    sops exec-file ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml \
        'kubectl --kubeconfig {} get --all-namespaces pods'
done
```

Are all Deployments fine? Deployments should show all Pods Ready, Up-to-date and Available (e.g., `2/2 2 2`).

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    sops exec-file ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml \
        'kubectl --kubeconfig {} get --all-namespaces deployments'
done
```

Are all DaemonSets fine? DaemonSets should show as many Pods Desired, Current, Ready and Up-to-date, as Desired.

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    sops exec-file ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml \
        'kubectl --kubeconfig {} get --all-namespaces ds'
done
```

Are Helm Releases fine? All Releases should be `deployed`.

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    export KUBECONFIG=kube_config_$CLUSTER.yaml
    sops -d ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml > $KUBECONFIG
    helm list --all --all-namespaces
    shred $KUBECONFIG
done
```

Are (Cluster)Issuers fine? All Resources should have `Ready` equal `TRUE`.

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    export KUBECONFIG=kube_config_$CLUSTER.yaml
    sops -d ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml > $KUBECONFIG
    kubectl get clusterissuers,issuers,certificates --all-namespaces
    shred $KUBECONFIG
done
```

## Where do I find the Nodes public and private IP?

```bash
find . -name inventory.ini
```

or

```bash
for CLUSTER in ${SERVICE_CLUSTER} "${WORKLOAD_CLUSTERS[@]}"; do
    ansible-inventory -i $CK8S_CONFIG_PATH/inventory/$CLUSTER/inventory.ini --list all
done
```

`ansible_host` is usually the public IP, while `ip` is usually the private IP.

## Node cannot be access via SSH

!!!important
    Make sure it is "not you". Are you well connected to the VPN? Is this the only Node which lost SSH access?

!!!important
    If you are using Rook, it is usually set up with replication 2, which means it can tolerate **one** restarting Node. Make sure that, either Rook is healthy or that you are really sure you are restarting the right Node.

Try connecting to the unhealthy Node via a different Node and internal IP:

```bash
UNHEALTHY_NODE=172.0.10.205  # You lost access to this one
JUMP_NODE=89.145.xxx.yyy  # You have access to this one

ssh -J ubuntu@$JUMP_NODE ubuntu@$UNHEALTHY_NODE
```

Try rebooting the Node via cloud provider specific CLI:

```bash
UNHEALTHY_NODE=cksc-worker-2

# Example for ExoScale
exo vm reboot --force $UNHEALTHY_NODE
```

If using Rook make sure its health goes back to `HEALTH_OK`.

## Node seems not fine

!!!important
    If you are using Rook, it is usually set up with replication 2, which means it can tolerate **one** restarting Node. Make sure that, either Rook is healthy or that you are really sure you are restarting the right Node.

Try rebooting the Node:

```bash
UNHEALTHY_NODE=89.145.xxx.yyy

ssh ubuntu@$UNHEALTHY_NODE sudo reboot
```

If using Rook make sure its health goes back to `HEALTH_OK`.

## Pod seems not fine

Before starting, set up a handy environment:

```bash
CLUSTER=cksc  # Cluster containing the unhealthy Pod

export KUBECONFIG=kube_config_$CLUSTER.yaml
sops -d ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml > $KUBECONFIG
```

Check that you are on the **right** cluster:

```bash
kubectl get nodes
```

Find the name of the Pod which is not fine:

```bash
kubectl get pod -A

# Copy-paste the Pod and Pod namespace below
UNHEALTHY_POD=prometheus-kube-prometheus-stack-prometheus-0
UNHEALTHY_POD_NAMESPACE=monitoring
```

Try to kill  and check if the underlying Deployment, StatefulSet or DaemonSet will restart it:

```bash
kubectl delete pod -n $UNHEALTHY_POD_NAMESPACE $UNHEALTHY_POD
kubectl get pod -A --watch
```

## Helm Release is `failed`

Before starting, set up a handy environment:

```bash
CLUSTER=cksc  # Cluster containing the failed Release

export KUBECONFIG=kube_config_$CLUSTER.yaml
sops -d ${CK8S_CONFIG_PATH}/.state/kube_config_$CLUSTER.yaml > $KUBECONFIG
```

Check that you are on the **right** cluster:

```bash
kubectl get nodes
```

Find the failed Release:
```bash
helm ls --all-namespaces --all

FAILED_RELEASE=user-rbac
FAILED_RELEASE_NAMESPACE=kube-system
```

Just to make sure, do a drift check (see the other section).

Remove the failed Release:
```bash
helm uninstall -n $FAILED_RELEASE_NAMESPACE $FAILED_RELEASE
```

Re-apply `apps` according to documentation.

## How do I check if `apps` drifted due to manual intervention?

```bash
# For service cluster
ln -sf $CK8S_CONFIG_PATH/.state/kube_config_${SERVICE_CLUSTER}.yaml $CK8S_CONFIG_PATH/.state/kube_config_sc.yaml
./compliantkubernetes-apps/bin/ck8s ops helmfile sc diff  # Respond "n" if you get WARN
```

```bash
# For the workload clusters
for CLUSTER in "${WORKLOAD_CLUSTERS[@]}"; do
    ln -sf $CK8S_CONFIG_PATH/.state/kube_config_${CLUSTER}.yaml $CK8S_CONFIG_PATH/.state/kube_config_wc.yaml
    ./compliantkubernetes-apps/bin/ck8s ops helmfile wc diff  # Respond "n" if you get WARN
done
```