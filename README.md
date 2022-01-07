# automate-private-AKS-AAD

## Deploy AKS cluster

Script created by using [AKS Deploy Helper](https://azure.github.io/Aks-Construction/)

```bash
# Create Resource Group 
az group create -l WestEurope -n az-k8s-6460-rg 

# Deploy template with in-line parameters 
az deployment group create -g az-k8s-6460-rg  --template-uri https://github.com/Azure/Aks-Construction/releases/download/0.4.0-preview/main.json --parameters \
	resourceName=az-k8s-6460 \
	upgradeChannel=stable \
	agentCountMax=20 \
	custom_vnet=true \
	bastion=true \
	enable_aad=true \
	AksDisableLocalAccounts=true \
	enableAzureRBAC=true \
	adminprincipleid=$(az ad signed-in-user show --query objectId --out tsv) \
	registries_sku=Basic \
	acrPushRolePrincipalId=$(az ad signed-in-user show --query objectId --out tsv) \
	omsagent=true \
	retentionInDays=30 \
	networkPolicy=calico \
	azurepolicy=audit \
	enablePrivateCluster=true
```

## DevOps Agent

```bash
# Add subnet for the agent VM
az network vnet subnet create -g az-k8s-6460-rg --vnet-name vnet-az-k8s-6460 -n AgentSubnet --address-prefixes 10.240.4.0/28![image](https://user-images.githubusercontent.com/1042817/148532727-55d3b917-d8b0-48ee-9f69-f6cb0e743e7c.png)

# create the VM
az vm create -n DevOpsAgentVM -g  az-k8s-6460-rg --image UbuntuLTS -l westeurope --admin-username $USERNAME  --admin-password $PASSWORD --authentication-type password --public-ip-address "" --vnet-name vnet-az-k8s-6460 --subnet AgentSubnet

# add VM extension 
az vm extension set -n customScript --publisher Microsoft.Azure.Extensions  --vm-name DevOpsAgentVM --resource-group az-k8s-6460-rg  --protected-settings '{"fileUris": ["https://raw.githubusercontent.com/vermegi/automate-private-AKS-AAD/main/install_tools.sh"],"commandToExecute": "./install_tools.sh --AGENT_USER $AGENTUSER --AGENT_POOL $AGENTPOOL --AGENT_TOKEN $AGENTTOKEN --AZDO_URL $DEVOPSURL"}'
```

For now env values for devops agent are still in the script, however, install of agent is commented out and I did that part by hand, since the original script acted up on me ...
The script does install all other dependencies like helm and kubectl.

Install devops agent: follow guidance in docs.

## In the portal

- Enable managed identity (can also be done on VM creation)
- Assign [Azure Kubernetes Service Cluster User Role](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#azure-kubernetes-service-cluster-user-role) to MI
- Take not of the MI object ID

## On the agent machine

- Log in on your agent machine through Bastion

```bash
# Login with your azure credentials
az login 
az aks get-credentials -n <name of the cluser> -g <rg-name>
```
This will create a kubeconfig file for you

- Install [kubelogin](https://github.com/Azure/kubelogin). I did this through homebrew
- Once installed, you can start using it: 

```bash
export KUBECONFIG=/path/to/kubeconfig
kubelogin convert-kubeconfig -l azurecli
kubectl get nodes
# this will not ask you to log in again, it will reuse your CLI creds.
```

- Create a rolebinding for your MI: 

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: msi-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: <MI object ID>
EOF
```

- log in as your MI and update kubeconfig

```bash
az login --identity
az aks get-credentials -n <name of the cluser> -g <rg-name>
```

When asked override the existing kubeconfig file.

- Issue kubectl statements through the MI: 

```bash
kubelogin convert-kubeconfig -l msi
kubectl get nodes
# this should work
```
