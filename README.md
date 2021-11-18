# Exploring VMware Tanzu CLI with local k8s (docker)

## Installation Steps:

Following https://tanzucommunityedition.io/docs/latest/getting-started/#creating-clusters .

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
4. Need to set up cluster configuration file. See:
   - https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-tanzu-k8s-clusters-index.html#create-a-tanzu-kubernetes-cluster-configuration-file-1.
   - https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.4/vmware-tanzu-kubernetes-grid-14/GUID-tanzu-config-reference.html
   ```sh
   cp ~/.config/tanzu/tkg/clusterconfigs/owlffnk3hh.yaml first-tanzu-mgmt.yaml.copy
   cp first-tanzu-mgmt.yaml.copy first-tanzu-workload.yaml
   # modify the contents for workload.yaml
   # add CLUSTER_PLAN to the configuration file
   ```
5. xxx

### Actual commands in terminal (MacOS):

```sh
# Install Tanzu CLI CE
brew install vmware-tanzu/tanzu/tansu-community-edition
/usr/local/Cellar/tanzu-community-edition/v0.9.1/libexec/configure-tce.sh

tanzu management-cluster create --file ~/.config/tanzu/tkg/clusterconfigs/owlffnk3hh.yaml -v 6

# Validate mgmt cluster creation
tanzu management-cluster get

# Get credential into kubectl
# tanzu management-cluster kubeconfig get <MGMT-CLUSTER-NAME> --admin
# MGMT-CLUSTER-NAME=first-tanzu-mgmt
tanzu management-cluster kubeconfig get first-tanzu-mgmt --admin

# Create workload cluster. The guide is not working, see Notes #3.
# from installation output, run “tanzu cluster create [name] -f [file]”
# from guide, run “tanzu cluster create <WORKLOAD-CLUSTER-NAME> --plan dev”
# WORKLOAD-CLUSTER-NAME=first-tanzu-workload


```

—

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
