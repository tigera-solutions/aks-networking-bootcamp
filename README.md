# aks-networking-bootcamp

This guide provides examples of different networking options for AKS cluster with Calico.

## networking and policy options for AKS cluster with Calico

AKS provides several different networking and policy options. The list below mostly focuses on Calico related options.

- the simplest and most basic AKS networking option is [kubenet](https://learn.microsoft.com/en-us/azure/aks/configure-kubenet) that allows you to configure AKS cluster with private POD CIDR which will not exhaust IP addresses of your Azure VNET. However, this option has a limitation of 400 nodes in the cluster which may not be suitable for large AKS clusters. You can configure Calico as a policy engine with `kubenet` networking by passing `--network-policy calico` setting into `az aks create` command to create AKS cluster via Azure CLI.
- more advance networking option is [Azure CNI](https://learn.microsoft.com/en-us/azure/aks/concepts-network#azure-cni-advanced-networking) that allows users to choose from several CNI configuration options:
  - [Azure CNI networking with routable POD IPs](https://learn.microsoft.com/en-us/azure/aks/configure-azure-cni?tabs=configure-networking-portal). This option allows each POD to get an IP from subnets associated with the AKS cluster which makes the PODs directly addressable within AKS cluster VNET. The downside of this networking option could be IP exhaustion if VNET is shared with other Azure resources or AKS cluster scaling reaches the capacity of the VNET. You can configure Calico as the policy engine with this networking option by passing the `--network-policy calico` setting into `az aks create` command to create AKS cluster via Azure CLI.
  - [Azure CNI overlay networking option with private POD IPs](https://learn.microsoft.com/en-us/azure/aks/azure-cni-overlay?tabs=kubectl). This option allows each POD to get an IP from a private CIDR configured for the Azure CNI overlay network and does not require PODs to get an IP from the VNET. This allows for more flexibility in AKS cluster networking and mitigates the risk of getting VNET IP exhaustion issue. You can configure Calico as the policy engine with this networking option by passing the `--network-policy calico` setting into `az aks create` command to create AKS cluster via Azure CLI.
  - [Bring your own CNI plugin with AKS](https://learn.microsoft.com/en-us/azure/aks/use-byo-cni?tabs=azure-cli). This option allows you to choose any third-party CNI solution that works with AKS. This option decouples AKS cluster provisioning process from the CNI installation. You install the CNI after the AKS cluster is provisioned. Since the CNI is installed after AKS cluster provisioning, the lifecycle management of the CNI is outside of the scope of AKS cluster lifecycle management. Hence, any updates or upgrades to the CNI should be scheduled and performed by the infrastructure team.

## Calico networking options for AKS

Calico is designed to allow users to choose the best dataplane option that suits their requirements. Calico supports several dataplanes:

- Standard Linux (Iptables)
- [eBPF](https://docs.tigera.io/calico/latest/operations/ebpf/)
- [Windows HNS](https://docs.tigera.io/calico/latest/getting-started/kubernetes/windows-calico/)
- [VPP](https://docs.tigera.io/calico/latest/getting-started/kubernetes/vpp/)

## create AKS cluster with Calico

This section provides a few examples to install Calico in AKS and configure different dataplane options.

>If you're interested in `kubenet` networking option, see example in [install-calico-on-aks](https://github.com/tigera-solutions/install-calico-on-aks?tab=readme-ov-file#using-az-aks-to-install-kubenet-network-plugin-and-calico-for-network-policy-on-aks) repo.

### prepare Azure Cloud Shell environment

This guide uses [Azure Cloud Shell](https://learn.microsoft.com/en-us/azure/cloud-shell/overview) to create and interact with AKS clusters. Make sure to login into Azure portal to access Azure Cloud Shell.

```bash
# see available AKS versions in your region
LOCATION='westus2'
az aks get-versions -l $LOCATION --output table

# create resource group for AKS cluster
RG='demo'
az group create --name $RG --location $LOCATION
```

If you want to minimize typing of `kubectl` commands, consider configuring [kubectl autocomplete](https://kubernetes.io/docs/reference/kubectl/quick-reference/#kubectl-autocomplete).

```bash
# exec these commands in your Azure Cloud Shell
source <(kubectl completion bash)
echo "alias k=kubectl" >> ~/.bashrc
alias k=kubectl
```

### AKS cluster with Azure CNI and Calico plugin for policy

In this example AKS cluster is provisioned with [Azure CNI](https://learn.microsoft.com/en-us/azure/aks/configure-azure-cni?tabs=configure-networking-portal) option where each pod gets a routable IP assigned from the Azure VNET.

```bash
# set vars
RG='demo'
LOCATION='westus2'
CLUSTER_NAME='azcni-cali'
K8S_VERSION=1.29

# create AKS cluster
az aks create \
  --resource-group $RG \
  --name $CLUSTER_NAME \
  --kubernetes-version $K8S_VERSION \
  --nodepool-name 'nix' \
  --node-count 3 \
  --network-plugin azure \
  --network-policy calico \
  --node-vm-size Standard_B2ms \
  --max-pods 70 \
  --generate-ssh-keys \
  --enable-managed-identity \
  --output table

# get cluster credentials which are saved into ~/.kube dir
az aks get-credentials --resource-group $RG --name $CLUSTER_NAME
```

Once the AKS cluster is provisioned, the PODs will be networked using Azure CNI plugin with routable IPs and Calico plugin for policies. If you want to take advantage of advanced Calico security and observability capabilities, you can upgrade Calico plugin to Calico Cloud or Calico Enterprise.

Refer to [connect AKS cluster to Calico Cloud](#connect-aks-cluster-to-calico-cloud) to upgrade Calico plugin to Calico Cloud version.

### AKS cluster with Azure CNI overlay and Calico Cloud for policy

In this example AKS cluster is provisioned with [Azure CNI overlay](https://learn.microsoft.com/en-us/azure/aks/azure-cni-overlay) option where each pod gets IP from a private POD CIDR network and therefore is not routable by default within the Azure VNET. Also, this example does not configure Calico plugin for network policy at the cluster provisioning (i.e. no `--network-policy calico` flag) but installs it afterwards as a third-party CNI plugin for network policy.

```bash
# set vars
RG='demo'
LOCATION='westus2'
CLUSTER_NAME='azcni-overlay-cali'
K8S_VERSION=1.29
POD_CIDR='192.168.0.0/16'

# create AKS cluster
az aks create \
  --resource-group $RG \
  --name $CLUSTER_NAME \
  --kubernetes-version $K8S_VERSION \
  --nodepool-name 'nix' \
  --node-count 3 \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr $POD_CIDR \
  --node-osdisk-size 50 \
  --node-vm-size Standard_B2ms \
  --max-pods 70 \
  --generate-ssh-keys \
  --enable-managed-identity \
  --output table

# get cluster credentials which are saved into ~/.kube dir
az aks get-credentials --resource-group $RG --name $CLUSTER_NAME
```

Refer to [connect AKS cluster to Calico Cloud](#connect-aks-cluster-to-calico-cloud) to install Calico plugin and connect the cluster to Calico Cloud management plane.

### AKS cluster with Azure CNI overlay and Windows nodepool and Calico Cloud plugin

In this example AKS cluster is provisioned with [Azure CNI](https://learn.microsoft.com/en-us/azure/aks/configure-azure-cni?tabs=configure-networking-portal) option where each pod gets a routable IP assigned from the Azure VNET. The cluster is provisioned with a few Windows specific settings to allow for Windows nodepools. The Calico plugin is installed after cluster provisioning as a third-party CNI plugin for network policy.

```bash
# set vars
RG='demo'
LOCATION='westus2'
CLUSTER_NAME='azcni-cali-win'
K8S_VERSION=1.29
WIN_NP_NAME=calwin
WIN_PASSWORD="Pa22w0rd${RANDOM}-${RANDOM}"

# create AKS cluster
az aks create \
  --resource-group $RG \
  --name $CLUSTER_NAME \
  --kubernetes-version $K8S_VERSION \
  --nodepool-name 'nix' \
  --node-count 4 \
  --network-plugin azure \
  --node-osdisk-size 50 \
  --node-vm-size Standard_B2ms \
  --max-pods 70 \
  --generate-ssh-keys \
  --enable-managed-identity \
  --windows-admin-username azure \
  --windows-admin-password $WIN_PASSWORD \
  --output table

# get cluster credentials which are saved into ~/.kube dir
az aks get-credentials --resource-group $RG --name $CLUSTER_NAME
```

[Connect AKS cluster to Calico Cloud](#connect-aks-cluster-to-calico-cloud) to install Calico plugin and connect the cluster to Calico Cloud management plane.

Configure Windows HNS networking in Calico

```bash
# get config values from AKS cluster settings
KUBERNETES_SERVICE_HOST=$(az aks show -g $RG -n $CLUSTER_NAME --query 'fqdn' --output tsv)
KUBERNETES_SERVICE_PORT=$(kubectl get endpoints kubernetes -ojsonpath='{.subsets[0].ports[0].port}')
SERVICE_CIDR=$(az aks show -g $RG -n $CLUSTER_NAME --query 'networkProfile.serviceCidr' --output tsv)

# configure kubernetes-services-endpoint configmap
kubectl apply -f - << EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "${KUBERNETES_SERVICE_HOST}"
  KUBERNETES_SERVICE_PORT: "${KUBERNETES_SERVICE_PORT}"
EOF

# configure service CIDR in the Installation CR
kubectl patch installation default --type merge --patch="{\"spec\": {\"serviceCIDRs\": [\"${SERVICE_CIDR}\"], \"calicoNetwork\": {\"windowsDataplane\": \"HNS\"}}}"

# enable firewall rules for Prometheus
kubectl patch felixConfiguration default --type merge --patch '{"spec": {"windowsManageFirewallRules": "Enabled"}}'
```

Add Windows nodepool to the cluster

```bash
az aks nodepool add \
  --resource-group $RG \
  --cluster-name $CLUSTER_NAME \
  --os-type Windows \
  --node-count 1 \
  --name $WIN_NP_NAME
```

### AKS cluster with BYO Calico CNI in eBPF mode

In this example AKS cluster is provisioned with [Bring your own CNI plugin](https://learn.microsoft.com/en-us/azure/aks/use-byo-cni?tabs=azure-cli) option where each pod gets networked by a third-party CNI getting a private IP from configured POD CIDR. The Calico CNI in this example is installed using eBPF mode.

```bash
# set vars
RG='demo'
LOCATION='westus2'
CLUSTER_NAME='calico-ebpf'
K8S_VERSION=1.29
POD_CIDR='10.244.0.0/16'

# create AKS cluster
az aks create \
  --resource-group $RG \
  --name $CLUSTER_NAME \
  --kubernetes-version $K8S_VERSION \
  --nodepool-name 'nix' \
  --node-count 3 \
  --network-plugin none \
  --pod-cidr $POD_CIDR \
  --node-osdisk-size 50 \
  --node-vm-size Standard_B2ms \
  --max-pods 70 \
  --generate-ssh-keys \
  --enable-managed-identity \
  --output table

# get cluster credentials which are saved into ~/.kube dir
az aks get-credentials --resource-group $RG --name $CLUSTER_NAME
```

Install Calico CNI in eBPF mode

```bash
# configure Calico Helm repo
helm repo add projectcalico https://docs.tigera.io/calico/charts

# configure Helm values
cat > values.yaml <<EOF
installation:
  kubernetesProvider: AKS
  cni:
    type: Calico
    ipam:
      type: Calico
  calicoNetwork:
    linuxDataplane: BPF
    hostPorts: Disabled
    bgp: Disabled
    ipPools:
    - cidr: $POD_CIDR
      encapsulation: VXLAN
EOF

# install Calico
kubectl create namespace tigera-operator
helm install calico projectcalico/tigera-operator --version v3.27.0 -f values.yaml --namespace tigera-operator

# get config values from AKS cluster settings
KUBERNETES_SERVICE_HOST=$(az aks show -g $RG -n $CLUSTER_NAME --query 'fqdn' --output tsv)
KUBERNETES_SERVICE_PORT=$(kubectl get endpoints kubernetes -ojsonpath='{.subsets[0].ports[0].port}')

# configure kubernetes-services-endpoint configmap
kubectl apply -f - << EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "${KUBERNETES_SERVICE_HOST}"
  KUBERNETES_SERVICE_PORT: "${KUBERNETES_SERVICE_PORT}"
EOF

# disable kube-proxy
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
```

## connect AKS cluster to Calico Cloud

> [!NOTE]
> if you installed Calico open source using Helm method, make sure to use Helm method to upgrade Calico to Calico Cloud or Calico Enterprise.

Follow instructions in the official [Calico Cloud documentation](https://docs.tigera.io/calico-cloud/about) to [connect your AKS cluster to Calico Cloud](https://docs.tigera.io/calico-cloud/get-started/connect/) management plane.

## clone workshop repo

```bash
git clone https://github.com/tigera-solutions/aks-networking-bootcamp.git && cd aks-networking-bootcamp
```

## configure Calico Cloud/Enterprise

Configure Calico Cloud/Enterprise

>Note that some base layer configuration is specific to Calico commercial version such as Calico Cloud or Calico Enterprise and will not work on Calico open source.

```bash
# configure FelixConfiguration resource
kubectl patch felixconfiguration default --type merge --patch-file demo/setup/felix.yaml
```

## deploy sample application and test policies

Deploy `hipstershop` demo application and a few utility pods

```bash
kubectl apply -f demo/app/hipstershop.yaml
kubectl apply -f demo/app/centos.yaml
kubectl apply -f demo/app/netshoot.yaml
```

Test connectivity to `hipstershop` services

```bash
# connect to hipstershop/frontend service from default/netshoot pod
kubectl exec -t netshoot -- sh -c 'curl -m2 -sI frontend.hipstershop | grep -i http'
kubectl exec -t netshoot -- sh -c 'nc -zv -w2 paymentservice.hipstershop 50051'
# connect to hipstershop/frontend service from a local hipstershop/centos pod
kubectl -n hipstershop exec -t centos -- sh -c 'curl -m2 -sI frontend.hipstershop | grep -i http'
```

Deploy security base layer for the cluster

```bash
kubectl apply -f demo/00-tiers/
kubectl apply -f demo/01-base/
```

Deploy permissive global `default-deny` policy

```bash
kubectl apply -f demo/10-zero-trust-security/calico.staged.default-deny.yaml
```

Deploy policies for `hipstershop` application to only allow required communications between the services

```bash
kubectl apply -f demo/10-zero-trust-security/hipstershop.policies.yaml
```

Deploy `iis` application for Windows nodes

>Note, this only works if your cluster has Windows nodepool

```bash
kubectl apply -f demo/app/iis.yaml

# test connection to IIS service
kubectl exec -t netshoot -- sh -c 'curl -m2 -sI iis-svc | grep -i http'

# secure IIS app
kubectl apply -f demo/10-zero-trust-security/iis-policy.yaml
```

Use network sets

```bash
kubectl apply -f demo/10-zero-trust-security/embargo.networkset.yaml
kubectl apply -f demo/10-zero-trust-security/calico.embargo-policy-egress.yaml
```

Configure egress controls via DNS policies

```bash
kubectl apply -f demo/20-egress-controls/allowed-domains-netset.yaml
kubectl apply -f demo/20-egress-controls/azure-services-netset.yaml
kubectl apply -f demo/20-egress-controls/azure-services.dns-policy.yaml
kubectl apply -f demo/20-egress-controls/calico.global.dns-policy.yaml

# test access to external domains
kubectl -n hipstershop exec -t centos -- sh -c 'curl -m2 -sI google.com | grep -i http'
kubectl -n hipstershop exec -t centos -- sh -c 'curl -m2 -sI myaccount.blob.core.windows.net 80'
```

Configure security alerts

```bash
kubectl apply -f demo/30-alerts
```

Configure threat intelligence feeds

```bash
kubectl apply -f demo/40-threatfeeds/feodo-threatfeed.yaml
kubectl apply -f demo/40-threatfeeds/feodo-block-policy.yaml

# test access to IP from the threatfeed list
IP=$(kubectl get globalnetworkset -l threatfeed=feodo -ojson | jq '.items[] | .spec.nets[0]' | sed -e 's/^"//' -e 's/"$//' -e 's/\/32//')
kubectl exec -t netshoot -- sh -c "nc -zv -w2 $IP 22"
```

## clean up demo and delete AKS clsuter

```bash
# delete apps
kubectl delete -f demo/app

# delete AKS cluster and resource group
az aks delete -n $CLUSTER_NAME -g $RG --yes --no-wait
az group delete --resource-group $RG --yes --no-wait
```
