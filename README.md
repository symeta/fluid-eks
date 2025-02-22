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
- Create the KMS encryption key for encryption at rest
  ```sh
    #Create the key
    export KEY_ID=$(aws kms create-key \
    --description "EKS secrets encryption key" \
    --tags TagKey=Environment,TagValue=Test \
    --tags TagKey=Purpose,TagValue=EKSSecrets \
    --query 'KeyMetadata.KeyId' \
    --output text)

    #create the alias for easy reference
    aws kms create-alias \
    --alias-name alias/eks-cluster-fluid \
    --target-key-id $KEY_ID

    #Create the alias
    export KEY_ARN=$(aws kms describe-key \
    --key-id $KEY_ID \
    --query 'KeyMetadata.Arn' \
    --output text)


  ```

- provision eks cluster
  ```sh
 # eksctl create cluster -f f6.yaml
 # Create cluster using envsubst to substitute the variables
  envsubst < f6.yaml | eksctl create cluster -f -

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
   kubectl create namespace fluid-system-feb15
   kubectl create namespace fluid-system
   helm upgrade --install fluid fluid/fluid -n fluid-system --set csi.kubelet.kubeConfigFile="/var/lib/kubelet/kubeconfig" --set csi.kubelet.certDir="/etc/kubernetes/pki"
   ```

- check fluid has been successfully installed
  ```sh
  kubectl get pod -n=fluid-system #all pods shoud be running
  ```

## 3. Create your access key credentials
- Create the bucket
```sh
export BUCKET_NAME=alluxioruntime-$(date +%Y%m%d-%H%M%S)
aws s3 mb s3://${BUCKET_NAME} --region us-west-2
aws s3api put-object --bucket ${BUCKET_NAME} --key fluid/ --content-length 0

```

- For more information look at: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html

  ```sh
  #export the credentials so we can create the secret in kubernetes
  export AWS_ACESS_KEY_ID=
  export AWS_SECRET_KEY=
  ```

## 4.fluid alluxioruntime to cache data from s3 bucket

- create dataset, alluxioruntime pod, relevant pvc 
  ```sh
  envsubst < secret.yaml | kubectl apply -n fluid-system  -f - 

  envsubst < dataset.yaml | kubectl apply -n fluid-system -f -

  #Make sure the runtime is fully initialized before creating the pod
  kubectl wait --for=condition=Ready dataset/s3-dataset -n fluid-system

  kubectl apply -f alluxioruntime.yaml -n fluid-system
  
  # Wait for master initialization
  kubectl wait --for=condition=MasterInitialized alluxioruntime/s3-dataset -n fluid-system --timeout=5m

  # Wait for workers to be ready
  kubectl wait --for=condition=WorkersReady alluxioruntime/s3-dataset -n fluid-system --timeout=5m

  ```

- check the status of the above
  ```sh
  # Check Dataset status
  kubectl get dataset s3-dataset -n fluid-system
  
  # Check AlluxioRuntime status
  kubectl get alluxioruntime s3-dataset -n fluid-system
    
  # Check PV,PVC created by the Dataset automatically status  
   kubectl get pv,pvc -n fluid-system
  ```

## 4.create an app to read data from fluid alluxioruntime data cache
```sh
#run the pod
kubectl apply -f data-reader-pod.yaml -n fluid-system

kubectl describe pod data-reader-pod -n fluid-system
```


## 5 delete the resources:
```sh
kubectl delete -f data-reader-pod.yaml -n fluid-system
kubectl delete -f dataset-pvc.yaml -n fluid-system
kubectl delete -f alluxioruntime.yaml -n fluid-system
envsubst < dataset.yaml | kubectl delete -n fluid-system -f -
envsubst < secret.yaml | kubectl delete -n fluid-system  -f - 
```