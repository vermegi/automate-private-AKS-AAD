# Notes on AKS AAD RBAC integration

> Note: this writeup was done in Jan of 2022 using a Kubectl version of: 1.22.

Getting the kubeconfig file happens like usual:

~~~sh
az aks get-credentials -n [aks-name] -g [rg-name]
~~~

The `~/.kube/config` file looks then like:

~~~yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS...LS0K (this is base64 encoded certificate data for the cluster)
    server: https://xstofaks-79f2f498.hcp.westeurope.azmk8s.io:443
  name: xstof-aks
contexts:
- context:
    cluster: xstof-aks
    user: clusterUser_xstof-aks_xstof-aks
  name: xstof-aks
current-context: xstof-aks
kind: Config
preferences: {}
users:
- name: clusterUser_xstof-aks_xstof-aks
  user:
    auth-provider:
      config:
        apiserver-id: 6dae42f8-4368-4678-94ff-3960e28e3630
        client-id: 80faf920-1908-4b52-b5ef-a8e7bedfc67a
        config-mode: '1'
        environment: AzurePublicCloud
        tenant-id: 72f988bf-86f1-41af-91ab-2d7cd011db47
      name: azure
~~~

We can inspect what happens by proxying requests through `mitmproxy`:

~~~sh
mitmproxy -k

export HTTPS_PROXY=localhost:8080
kubectl get pods --insecure-skip-tls-verify
~~~

When executing the kubectl command, a login prompt will appear.  This will fetch an access_token with an audience pointing to a Service Principal called "Azure Kubernetes Service AAD Server".  (Originating from a multitenant app registration.)

Then a call is made to the Kubernetes API on `https://xstofaks-79f2f498.hcp.westeurope.azmk8s.io/api/v1/namespaces/default/pods`.  In this call the access_token is provided in the `Authorization` header as a bearer token.

The kubeconfig file now looks like this:

~~~yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0...0tLS0K
    server: https://xstofaks-79f2f498.hcp.westeurope.azmk8s.io:443
  name: xstof-aks
contexts:
- context:
    cluster: xstof-aks
    user: clusterUser_xstof-aks_xstof-aks
  name: xstof-aks
current-context: xstof-aks
kind: Config
preferences: {}
users:
- name: clusterUser_xstof-aks_xstof-aks
  user:
    auth-provider:
      config:
        access-token: eyJ0e...zQ
        apiserver-id: 6dae42f8-4368-4678-94ff-3960e28e3630
        client-id: 80faf920-1908-4b52-b5ef-a8e7bedfc67a
        config-mode: "1"
        environment: AzurePublicCloud
        expires-in: "4835"
        expires-on: "1642940689"
        refresh-token: 0.ARoA...jzgZQtv
        tenant-id: 72f988bf-86f1-41af-91ab-2d7cd011db47
      name: azure
~~~

Note how the `auth-provider` section for the user specifies the `azure` auth provider and provides the arguments for it.  This indicates the use of the "in-tree" [Kubernetes auth provider for Azure](https://docs.k8spharos.dev/cloud_providers/kube-in-tree.html#azure-cloud-provider).  This auth provider [is implemented here](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/client-go/plugin/pkg/client/auth/azure/README.md) and [marked for deprecation](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/client-go/plugin/pkg/client/auth/azure/azure.go#L93) in favor for `kubelogin` instead, which follows the "exec" model.

Kubelogin [is implemented here](https://github.com/Azure/kubelogin).  Let's install it:

~~~sh
wget https://github.com/Azure/kubelogin/releases/download/v0.0.10/kubelogin-linux-amd64.zip
unzip kubelogin-linux-amd64.zip
sudo mv bin/linux_amd64/kubelogin /usr/bin
~~~

To use the kubelogin exec provider, we first need to indicate this by transforming our kubelogin file.  We can do so like this:

~~~sh
kubelogin convert-kubeconfig --login azurecli # --login indicates we want to use the auth context from the azure cli (other options exist)
~~~

Our kubeconfi now looks like this:

~~~yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0t...LS0K
    server: https://xstofaks-79f2f498.hcp.westeurope.azmk8s.io:443
  name: xstof-aks
contexts:
- context:
    cluster: xstof-aks
    user: clusterUser_xstof-aks_xstof-aks
  name: xstof-aks
current-context: xstof-aks
kind: Config
preferences: {}
users:
- name: clusterUser_xstof-aks_xstof-aks
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - get-token
      - --server-id
      - 6dae42f8-4368-4678-94ff-3960e28e3630
      - --login
      - azurecli
      command: kubelogin
      env: null
      provideClusterInfo: false
~~~

Instead of an `auth-provider` we now find `exec` for our user.  Reading the configuration, it instructs kubectl to execute the following command and use the result for authentication:

~~~sh
kubelogin get-token --server-id 6dae42f8-4368-4678-94ff-3960e28e3630 --login azurecli
~~~

This provides us with the following json as a result:

~~~json
{
    "kind": "ExecCredential",
    "apiVersion": "client.authentication.k8s.io/v1beta1",
    "spec": {},
    "status": {
        "expirationTimestamp": "2022-01-23T17:04:20Z",
        "token": "eyJ0eXAiOiJKV1QiLC...vKbzlsxz_W1hrA"
    }
}
~~~

