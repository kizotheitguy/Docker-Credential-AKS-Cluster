# Docker-Credential-AKS-Cluster

# Creating Docker credentials for Azure Kubernetes Service Cluster

This document is a method to resolving Kubernetes deployment with docker image pull request error sometimes encountered from Azure Container Registry (Azure ACR) or other private image registry.

## Requirements
```
az login --service-principal -u $AZURE_SERVICEPRINCIPAL -p $AZURE_PASSWORD --tenant $AZURE_AD_TENANT
az account set --subscription $AZURE_SUBSCRIPTION
az aks install-cli
az aks get-credentials -g myResourceGroup -n myClusterName
```

## Create the secrets in the Kubernetes cluster
```
kubectl create secret docker-registry mysecrets \
  --docker-server=$CONTAINER_REG_NAME.azurecr.io \
	--docker-username=$DOCKER_ADMIN \
	--docker-password=$DOCKER_PASS \
	--docker-email=$DOCKER_EMAIL
```

## Run the command to get the secrets created
```
kubectl get secret mysecrets -o yaml
```

## The file will look like this
```
apiVersion: v1
data:
  .dockercfg: <<some key values>>
kind: Secret
metadata:
  creationTimestamp: <<>A Time Stamp>
  name: mysecrets
  namespace: default
  resourceVersion: "123456"
  selfLink: /api/v1/namespaces/default/secrets/mysecrets
  uid: xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxxx
type: kubernetes.io/dockercfg
```

## Export the mysecrets to a file
```
kubectl get secret mySecrets -o yaml > mySecrets.yml
```

## Then replace "dockercfg" with "dockerconfigjson"
```
sed -i 's/dockercfg/dockerconfigjson/g' mySecrets.yml
```

## Delete and recreate the secrets with the .yml file
```
kubectl delete secret mySecrets
kubectl apply -f mySecrets.yml
```

If you run the command "kubectl get secret mySecrets -o yaml" again you will find that the file now looks different

You can now add the secrets name to your object file
```
kind: Deployment
metadata:
  name: myApp
spec:
  template:
    metadata:
      labels:
        app: Prod
    spec:
      containers:
      - name: myApp
        image: myPrivateRegistry/myCustomImage:lates
		imagePullPolicy: Always
        ports:
        - containerPort: 80
	  imagePullSecrets:
      - name: mySecrets
```

## Automate the Step 
Here is a simple bash script to automate the process mentioned above.

```
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'


echo "Logon to Azure Portal"
az login --service-principal -u $AZURE_SERVICEPRINCIPAL -p $AZURE_PASSWORD --tenant $AZURE_AD_TENANT
az account set --subscription $AZURE_SUBSCRIPTION
az aks install-cli
az aks get-credentials -g myResourceGroup -n myClusterName

echo "Creating Initial Docker ImagePullSecret"
kubectl create secret docker-registry $K8s_SECRET_NAME \
    --docker-server=$CONTAINER_REG_NAME.azurecr.io \
	--docker-username=$DOCKER_ADMIN \
	--docker-password=$DOCKER_PASS \
	--docker-email=$DOCKER_EMAIL

echo "Get Docker ImagePullSecret file"
kubectl get secret $K8s_SECRET_NAME -o yaml > $K8s_SECRET_NAME.yml

echo "Editing Docker ImagePullSecret file"
sed -i 's/dockercfg/dockerconfigjson/g' $K8s_SECRET_NAME.yml

echo "Deleting Docker Credentials in Kube-system"
kubectl delete secret $K8s_SECRET_NAME

echo "Creating New Docker ImagePullSecret"
kubectl apply -f $K8s_SECRET_NAME.yml

kubectl get secret $CONTAINER_REG_NAME
```
