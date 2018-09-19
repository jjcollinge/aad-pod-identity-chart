# Helm chart for Kubernetes AAD Pod Identity
A super simple helm chart for setting up the components to use [Azure Active Directory Pod Identities in Kubernetes](https://github.com/Azure/aad-pod-identity).

## Prerequisites
* Azure Subscription with elevated permission
* Azure Kubernetes Service (AKS) or ACS-Engine deployment
* `kubectl` authenticated to said Kubernetes cluster
* [Helm v1.10+](https://github.com/helm/helm)
* [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* [git](https://git-scm.com/downloads)

## Getting Started

1. Create a new Azure User Identity using the Azure CLI:
> __NOTE:__ It's simpler to use the same resource group as the nodes are deployed in. For AKS this is the MC_ resource group. If not you'll need to grant the cluster's service principal the "Managed Identity Operator" role.
```shell
az identity create -g <res-group> -n <id-name>
```
2. Assign this new identity the role of _Reader_ for the resource group:
```shell
az role assignment create --role Reader --assignee <principal-id> --scope /subscriptions/<subscriptionid>/resourcegroups/<resourcegroup>
```

3. Clone this repository and navigate to its root
```shell
git clone git@github.com:jjcollinge/aad-pod-identity-chart.git && cd aad-pod-identity-chart
```

4. Copy the Azure resource ID and client ID properties of the Identity into the `values.yaml` file.

5. Update the selector value to match the pods you wish to apply the identity to in the `values.yaml` file.

6. Install the Helm chart into your cluster
```shell
helm install --values values.yaml .
```
This chart will deploy the following:
* AzureIdentity `CustomResourceDefinition`
* AzureIdentityBinding `CustomResourceDefinition`
* AzureAssignedIdentity `CustomResourceDefinition`
* AzureIdentity instance
* AzureIdentityBinding instance
* Managed Identity Controller (MIC) `Deployment`
* Node Managed Identity (NMI) `DaemonSet`

7. Deploy your application which uses the MSI authentication endpoint via ADAL with the correct label selector.

    A demo application is available [here](https://github.com/Azure/aad-pod-identity#demo-app). If you do use the demo app, please update the `deployment.yaml` with the appropriate subscription ID, client ID and resource group name. Also make sure the selector you defined in your `AzureIdentityBinding` matches a label on this deployment.

8. Validate the MIC has detected your `AzureIdentityBinding` by viewing the logs
```shell
kubectl logs mic-768489d94-pjxqf
...
I0919 13:12:34.222107       1 event.go:218] Event(v1.ObjectReference{Kind:"AzureIdentityBinding", Namespace:"default", Name:"msi-binding", UID:"UID", APIVersion:"aadpodidentity.k8s.io/v1", ResourceVersion:"231329", FieldPath:""}): type: 'Normal' reason: 'binding applied' Binding msi-binding applied on node aks-agentpool-12603492-1 for pod demo-77d858d9f9-62d4r-default-msi
```

9. Check the MIC has created a new `AzureAssignedIdentity` for your deployment
```shell
kubectl get AzureAssignedIdentity
...
NAME                                CREATED AT
demo-77d858d9f9-62d4r-default-msi   1m
```

10. Check the NMI successfully retrieved a token for the app by viewing the NMI logs
```shell
kubectl logs nmi-cn4cc
...
time="2018-09-19T13:56:20Z" level=info msg="Status (200) took 37041685 ns" req.method=GET req.path=/metadata/identity/oauth2/token req.remote=10.244.0.12
```

11. Finally, validate the application is behaving and logging as expected. The demo application will log details about the MSI acquisition
```
kubectl logs demo-77d858d9f9-62d4r
...
time="2018-09-19T13:14:28Z" level=info msg="succesfully acquired a token using the MSI, msiEndpoint(http://169.254.169.254/metadata/identity/oauth2/token)" podip=10.244.0.12 podname=demo-77d858d9f9-62d4r podnamespace=demo-77d858d9f9-62d4r
```

> __NOTE:__ If you have previously installed the Helm chart you may get the following error:
> Error: object is being deleted: customresourcedefinitions.apiextensions.k8s.io ?
> "azureassignedidentities.aadpodidentity.k8s.io" already exists
> This is because Helm does not actively manage the CRDs, please delete them manually
> using `kubectl delete crds ...`


