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

