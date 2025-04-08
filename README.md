# fluid-eks
The objective of this repo is to test the performance and scalability of fluid on amazon eks

## 1.amazon eks cluster provision

- pre-requisite
  - provision an ec2
  - install eksctl on the ec2
    ```sh
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv -v /tmp/eksctl /usr/local/bin
    eksctl version
    ```
  - install kubectl on the ec2
    ```sh
    aws sts get-caller-identity
    curl --silent --location -o /usr/local/bin/kubectl \
    https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.3/2024-12-12/bin/linux/amd64/kubectl
    chmod +x /usr/local/bin/kubectl
    kubectl version --client
    ```
- provision eks cluster
  ```sh
  eksctl create cluster -f f6.yaml

  #wait till the cluster been provisioned, takes around 15 minutes

  aws eks update-kubeconfig --region us-west-2 --name f6
  ```

## 2.fluid installation

-  install helm
   ```sh
   curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
   chmod 700 get_helm.sh
   ./get_helm.sh
   ```
-  install fluid
   ```sh
   helm repo add fluid https://fluid-cloudnative.github.io/charts
   helm repo update
   helm upgrade --install fluid fluid/fluid -n fluid-system --set csi.kubelet.kubeConfigFile="/var/lib/kubelet/kubeconfig" --set csi.kubelet.certDir="/etc/kubernetes/pki"
   ```

- check fluid has been successfully installed
  ```sh
  kubectl get pod -n=fluid-system #all pods shoud be running
  ```
## 3.fluid alluxioruntime to cache data from s3 bucket
- create secret data-set-secret
  ```sh
  export AWS_ACCESS_KEY_ID=<your ak>
  export AWS_SECRET_ACCESS_KEY=<your sk>
  kubectl apply -f secret.yaml -n fluid-system
  ```
- create dataset, alluxioruntime pod. pvc will be auto-created. 
  ```sh
  kubectl apply -f dataset.yaml -n fluid-system
  kubectl apply -f alluxioruntime.yaml -n fluid-system
  ```
- check the status of the above
  ```sh
  # Check secret status
  kubectl get secret data-set-secret -n fluid-system

  # Check Dataset status
  kubectl get dataset s3-dataset -n fluid-system
  
  # Check AlluxioRuntime status
  kubectl get alluxioruntime s3-dataset -n fluid-system
  
  # Check PVC status
  kubectl get pvc s3-dataset -n fluid-system
  ```

## 4.create an app to read data from fluid alluxioruntime data cache
- make sure that the plugins container of the csi-nodeplugin-fluid pod has aws-cli installed
```sh
kubectl exec -it csi-nodeplugin-fluid-xxxxx -c plugins -n fluid-system -- /bin/sh
apk add --no-cache aws-cli
aws --version
```
- create data-reader-pod
```sh
#run the pod
kubectl apply -f data-reader-pod.yaml -n fluid-system

#check the status of the pod
kubectl describe pod data-reader-pod -n fluid-system
```

## delete resources
```sh
#delete the app pod
kubectl delete pod data-reader-pod -n fluid-system

# Delete PVC first
kubectl delete pvc s3-dataset -n fluid-system

# Delete AlluxioRuntime if it exists
kubectl delete alluxioruntime s3-dataset -n fluid-system

# Delete Dataset
kubectl delete dataset s3-dataset -n fluid-system
```


