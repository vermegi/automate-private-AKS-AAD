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


```
