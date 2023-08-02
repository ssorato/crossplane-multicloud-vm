# Crossplane multicloud VM

Create an AWS or Azure VM using [Crossplane](https://www.crossplane.io/)

## Microk8s

We'll use microk8s to install Crossplane.

```bash
$ sudo snap install microk8s --classic

$ microk8s status --wait-ready

$ microk8s config | tee ~/.kube/microk8s
$ chmod 0600 ~/.kube/microk8s

$ export KUBECONFIG=~/.kube/microk8s 
$ kubectl get nodes
```

optional:

```bash
$ sudo microk8s enable hostpath-storage
$ sudo microk8s enable registry
```

Cleanup

```bash
$ sudo snap remove microk8s
```

## Install Crossplane

```bash
$ helm repo add crossplane-stable https://charts.crossplane.io/stable
$ helm repo update
$ helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-stable/crossplane

$ kubectl get pods -n crossplane-system
```

## Install the Crossplane providers

### Azure

sample `azure-credentials.json`

```json
{
  "clientId": "<client id>",
  "clientSecret": "<client secret>",
  "subscriptionId": "<subscription id>",
  "tenantId": "<tenant id>",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```

```bash
$ kubectl apply -f providers/upbound-azure.yaml 

$ kubectl create secret generic azure-secret -n crossplane-system --from-file=creds=./providers/azure-credentials.json

$ kubectl get providers -w
```

wait until _healty_ is _True_

``` bash
$ kubectl  apply -f providers/upbound-config-azure.yaml
```

### AWS

sample `aws-credentials.txt`

```
[default]
aws_access_key_id = <access key>
aws_secret_access_key = <secret key>
```

```bash
$ kubectl apply -f providers/upbound-aws.yaml 

$ kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./providers/aws-credentials.txt

$ kubectl get providers -w
```

wait until _healty_ is _True_

``` bash
$ kubectl  apply -f providers/upbound-config-aws.yaml
```

## Create the VM using Claim

Before apply configure the 

* `sshPublicKey` with the ssh public key
* `myPublicIP` with source public ip ( this will be enable to access to the vm )

### Azure

```bash
$ yq --inplace ".spec.parameters.sshPublicKey = \"<my public ssh key>\"" azure-upbound-vm.yaml
$ yq --inplace ".spec.parameters.myPublicIP = \"<my source ip>/32\"" azure-upbound-vm.yaml
$ kubectl apply -f config-multicloud-vm.yaml
$ kubectl apply -f azure-upbound-vm.yaml
$ kubectl get virtualmachineclaims -w
```

### AWS

```bash
$ yq --inplace ".spec.parameters.sshPublicKey = \"<my public ssh key>\"" aws-upbound-vm.yaml
$ yq --inplace ".spec.parameters.myPublicIP = \"<my source ip>/32\"" aws-upbound-vm.yaml
$ kubectl apply -f config-multicloud-vm.yaml
$ kubectl apply -f aws-upbound-vm.yaml
$ kubectl get virtualmachineclaims -w
```

## Access to the VM

### Azure

```bash
$ ssh -i <private key> sandbox@<public ip>
```

### AWS

```bash
$ ssh -i <private key> ec2-user@<public ip>
```

## Cleanup

### Azure

```bash
$ kubectl delete -f azure-upbound-vm.yaml
```

### AWS

```bash
$ kubectl delete -f aws-upbound-vm.yaml
```

## Crossplane Packages

Install Crossplane cli

```bash
$ curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
$ sudo mv kubectl-crossplane /usr/local/bin
```

Build the package

```bash
$ cd package
$ kubectl crossplane build configuration
```

Send the package to the Docker Hub

```bash
$ docker login
$ kubectl crossplane push configuration ssorato/crossplane-multicloud-vm:v1.0.0
```
