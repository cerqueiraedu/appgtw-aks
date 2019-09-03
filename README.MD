# Sources: 
- https://azure.github.io/application-gateway-kubernetes-ingress/setup/install-existing/
- https://github.com/Azure/aad-pod-identity#deploy-the-azure-aad-identity-infra
- https://github.com/Azure/application-gateway-kubernetes-ingress

## Prerequisite:
* As listed on Brownfield Deployment: this documents assumes you already have the following tools and infrastructure installed: 
* Helm installed on your local machine
* AKS with Advanced Networking enabled
* App Gateway v2 in the same virtual network as AKS
* AAD Pod Identity installed on your AKS cluster

## Steps
- enable k8s dashboard
```bash
$ kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
```

- create basic pfx certificate to create app gateway  

```powershell
New-SelfSignedCertificate `
  -certstorelocation cert:\localmachine\my `
  -dnsname www.aikido-app.com

$pwd = ConvertTo-SecureString -String "Azure123456!" -Force -AsPlainText
Export-PfxCertificate `
  -cert cert:\localMachine\my\E1E81C23B3AD33F9B4D1717B20AB65DBB91AC630 `
  -FilePath c:\appgwcert.pfx `
  -Password $pwd
```
- necessary rbac and agents for aad-pod-identity. this is the component that will allow the comunicatation with ARM to setup app gateway rules

```bash
$ kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml
```
  
- create new managed identity to k8s vms agent pool

```
az identity create -g MC_poc-appgtw-k8s_appgtw-k8s_centralus -n appgtw-identity -o json
az identity show -g MC_poc-appgtw-k8s_appgtw-k8s_centralus -n appgtw-identity 
```

-- IMPORTANT: added appgtw-identity to app-gtw resoure group as contributor as well,  but this might not be necessary.
-- the step below should be enough, but keep it in mind.

- query the get complete app gtw id:
```
az network application-gateway list --query [].id
```

- assign the managed identity to app gateway as well	
```
az role assignment create --role Contributor --assignee {managed_identity_id} --scope /subscriptions/{id}/resourceGroups/{app_gtw_rg}/providers/Microsoft.Network/applicationGateways/{app_gtw_name}
```
   
- assign AzureIdentity and AzureIdentityBinding   
```bash
$ kubectl apply -f aadpodidentity.yaml
```
   
-- IMPORTANT: it's not required to use a Service Principal
- Using a Service Principal - not done yet - not necessary?
- az ad sp create-for-rbac --subscription {id} --sdk-auth | base64 -w0 - not done yet - not necessary?
- It is also possible to provide AGIC access to ARM via a Kubernetes secret. - not done yet - not necessary?

# HELM:

- https://docs.microsoft.com/en-us/azure/aks/kubernetes-helm#create-a-service-account

```bash
$ kubectl create serviceaccount --namespace kube-system tiller-sa
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller-sa
$ helm init --tiller-namespace kube-system --service-account tiller-sa
$ helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
$ helm repo update
```

## Install AGIC into cluster using helm chart:
```bash
$ helm install -f agic-helm-config.yaml application-gateway-kubernetes-ingress/ingress-azure --name demoappgtw
```

You can now deploy guestbook.yaml and ingress-guestbook.yaml as well for testing purposes.

If everything was corretly setup, you should be able to see 3 new targets in application gateway backend pool.
