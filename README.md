# fluid-eks
The objective of this repo is to test the performance and scalability of fluid on amazon eks

## amazon eks cluster provision

- pre-requisite
  - provision an ec2
  - install eksctl on the ec2
    ```sh
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv -v /tmp/eksctl /usr/local/bin
    eksctl version
    eksctl create cluster -f fluidemo1.yaml
    ``` 
- 
