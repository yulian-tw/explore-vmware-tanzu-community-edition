# Exploring VMware Tanzu CLI with local k8s (docker)

## Installation Steps:

Following https://tanzucommunityedition.io/docs/latest/getting-started .

### Notes:
1. Default Kubernetes Settings (local docker)
    - CNI provider: Antrea
    - Cluster Service CIDR: (populated: 100.64.0.0./13 , hinted: /13) - use default
    - Cluster Pod CIDR:     (populated: 100.96.0.0/11 ,  hinted: 192.168.0.0/16 ) - use 100.96.0.0/16
2. When running `tanzu management-cluster create` , make sure the terminal session is having kubectl tool. Otherwise, run it at the right terminal. (The command has memory for previous config).
3. There is problem following the guide to create a workload cluster.
   ```sh
   Error: required config variable 'CLUSTER_PLAN' not set
   ```
4. Workload cluster created successfully with cluster configuration file. See:
   - https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-tanzu-k8s-clusters-index.html#create-a-tanzu-kubernetes-cluster-configuration-file-1.
   - https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-tanzu-config-reference.html
   ```sh
   cp ~/.config/tanzu/tkg/clusterconfigs/owlffnk3hh.yaml first-tanzu-mgmt.yaml.copy
   cp first-tanzu-mgmt.yaml.copy first-tanzu-workload.yaml
   # modify the contents for workload.yaml
   # add CLUSTER_PLAN to the configuration file
   ```
5. The default configuration deployed the following into the 2 types of cluster.
   - `*` : denotes deployment in `tkg-system` namespace, in master and worker nodes in both clusters
   - additional namespace in management cluster:
     - `capd-system`, `capi-system` : just one controller manager in worker node
     - `capi-kubeadm-*` and `capi-webhook-system`

        | Components       | Management Cluster | Workload Cluster | Behaviour
        | :--------------- | :----------------: | :--------------: | :--------------- |
        | antrea (CNI)     | Y <br/> controller on master node | Y <br/>controller on worker node | antrea-agents on master and worker nodes |
        | coredns          | Y | Y | on worker node only |
        | metrics-server   | Y | Y | on worker node only |
        | kapp-controller* | on worker node only | on master node only | - |
        | tanzu-cap <br/> controller-manager* | on master nodes only | on worker nodes only | - |
        | tkr-controller-manager* | on worker node only <br/> with another addons pod | N | - |
        | cert-manager            | on worker node with a <br/> webhook pod on master node | N | - |
6. How to verify if the workload cluster is managed by the management cluster?
   - To read on how management cluster works:
     - [ ] https://cluster-api.sigs.k8s.io/
     - [ ] https://kind.sigs.k8s.io/

### Known issue:
1. If the Docker host machine is rebooted, the cluster will need to be re-created. Support for clusters surviving a host reboot is tracked in [GitHub issue](https://github.com/vmware-tanzu/community-edition/issues/832).
   - If using local docker, need to stop all Tanzu containers.
      ```sh
      # will get an error
      tanzu management-cluster get

      # confirm and stop Tanzu containers
      docker ps -a
      docker stop $(docker ps -qa)

      # this still keeps some reference to be cleaned up
      tanzu config server list
      kind get clusters
      # clean up commands
      tanzu config server delete first-tanzu-mgmt
      kind delete clusters ...

      # create the management and workload clusters again
      tanzu management-cluster create --ui
      ...
      # need to get the new cluster admin credentials to access the cluster
      tanzu cluster kubeconfig get first-tanzu-workload --admin
      kubectl get pods -A
      ```

### Actual commands in terminal (MacOS):

```sh
# Install Tanzu CLI CE
brew install vmware-tanzu/tanzu/tansu-community-edition
/usr/local/Cellar/tanzu-community-edition/v0.9.1/libexec/configure-tce.sh

# Through UI, create a bootstrap cluster and generate cluster config for management cluster
tanzu management-cluster create --ui
# equivalent to
# tanzu management-cluster create --file ~/.config/tanzu/tkg/clusterconfigs/owlffnk3hh.yaml -v 6
cp ~/.config/tanzu/tkg/clusterconfigs/owlffnk3hh.yaml ./first-tanzu-mgmt.yaml

# Validate mgmt cluster creation
tanzu management-cluster get

# Validate access to the mgmt cluster
# MGMT-CLUSTER-NAME=first-tanzu-mgmt
tanzu management-cluster kubeconfig get first-tanzu-mgmt --admin # Get credential into kubectl
kubectl config use-context first-tanzu-mgmt-admin@first-tanzu-mgmt
kubectl get nodes
kubectl get pods -A # checkout what were deployed =)

# Create workload cluster. The guide is not working, see Notes #3.
# from installation output, run ???tanzu cluster create [name] -f [file]???
# from guide, run ???tanzu cluster create <WORKLOAD-CLUSTER-NAME> --plan dev???
# WORKLOAD-CLUSTER-NAME=first-tanzu-workload
tanzu cluster create --file ./first-tanzu-workload.yaml

# Verify workload cluster creation
tanzu cluster list
tanzu cluster get first-tanzu-workload

# Validate access to the workload cluster
tanzu cluster kubeconfig get first-tanzu-workload --admin
kubectl config use-context first-tanzu-workload-admin@first-tanzu-workload
kubectl get nodes
kubectl get pods -A # checkout what were deployed =)

# How to confirm if the workload cluster is actually managed by management cluster?
tanzu management-cluster get # doesn't mention on workload clusters
```

### Useful references:
- https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-tanzu-cli-reference.html
- https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-tanzu-config-reference.html

???--

## Package Management

### Installing package in Workload Cluster

Try with: https://tanzucommunityedition.io/docs/latest/package-readme-fluent-bit-1.7.5/

```sh
# Install Tanzu Community Edition package repo into global namespace in the cluster
tanzu package repository add tce-repo --url projects.registry.vmware.com/tce/main:0.9.1 --namespace tanzu-package-repo-global
# Check status of repo and list available packages
tanzu package repository list --namespace tanzu-package-repo-global
tanzu package available list

# List available versions for a specific package
tanzu package available list fluent-bit.community.tanzu.vmware.com

# Install
tanzu package install fluent-bit --package-name fluent-bit.community.tanzu.vmware.com --version 1.7.5
# Verify
tanzu package installed list
kubectl get pods -A
kubectl get daemonset -A
kubectl logs -f daemonset/fluent-bit -n fluent-bit # see logs from all pods

# Remove a package from the cluster
tanzu package installed delete fluent-bit
# Verify
kubectl get daemonset -A
```
### More to explore
- [ ] [Deploy monitoring stack on Tanzu Community Edition](https://tanzucommunityedition.io/docs/latest/docker-monitoring-stack/#deploying-grafana--prometheus--contour--local-path-storage--cert-manager-on-tanzu-community-edition)
- [ ] [Package Creation](https://tanzucommunityedition.io/docs/latest/package-creation-step-by-step/)
  - required to use [Carvel suite of tools](https://carvel.dev/), a VMware-backed project
---

## Planning for clean up:

#### Clean up clusters ?

References at the end of page: https://tanzucommunityedition.io/docs/latest/getting-started/#creating-clusters

```
# delete any deployed workload clusters
tanzu cluster delete <WORKLOAD-CLUSTER-NAME>
tanzu cluster list

# delete mgmt clusters
tanzu management-cluster get
tanzu management-cluster delete <MGMT-CLUSTER-NAME>
```

#### To uninstall Tanzu CLI:

Extracted from installation outputs.

```
==> Thanks for installing Tanzu Community Edition!
==> The Tanzu CLI has been installed on your system
==>

==> ******************************************************************************
==> * To initialize all plugins required by Tanzu Community Edition, an additional
==> * step is required. To complete the installation, please run the following
==> * shell script:
==> *
==> * /usr/local/Cellar/tanzu-community-edition/v0.9.1/libexec/configure-tce.sh
==> *
==> ******************************************************************************
==>

==> * To cleanup and remove Tanzu Community Edition from your system, run the
==> * following script:
==> /usr/local/Cellar/tanzu-community-edition/v0.9.1/libexec/uninstall.sh
==>
```
