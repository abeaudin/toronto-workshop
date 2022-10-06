Toronto Linode Workshop
======================

# About

Package of template files, examples, and illustrations for the Chicago Linode Workshop.

# Contents

## Template Files
- Sample Terraform files for deploying an LKE cluster on Linode.
- Sample kubernetes deployment files for running CockroachDB on an LKE cluster.

## Step by Step Instructions

### Overview

![image](https://user-images.githubusercontent.com/7717493/194063848-45d8649a-8f31-41c0-b391-d585a13bd626.png)
The scenario is written to demonstrate the deployment of cockroachdb on an LKE Cluster

The workshop scenario builds the following components and steps-

1. A Secure Shell Linode (provisioned via the Linode Cloud Manager GUI) to serve as the command console for the environment setup.

2. Installing developer tools on the Secure Shell (git, terraform, and kubectl) for use in environment setup.

3. One Linode Kubernetes Engine (LKE) Clusters, deployed to a the Toronto Linode region, provisioned via terraform.

4. Deploying the cockroach db container to the LKE cluster.


### Build a Secure Shell Linode
![image](https://user-images.githubusercontent.com/7717493/194062204-f8389c14-30b9-4c64-b005-e4bf66e069b3.png)

We'll first create a Linode using the "Secure Your Server" Marketplace image. This will give us a hardened, consistent environment to run our subsequent commands from. 

1. Login to Linode Cloud Manager, Select "Create Linode," and choose the "Secure Your Server" Marketplace image. 
2. Within the setup template for "Secure Your Server," select the Ubuntu 20.04 LTS image type. 
3. Once your Linode is running, login to it's shell (either using the web-based LISH console from Linode Cloud Manager, or via your SSH client of choice).

### Install and Run git 

Next step is to init git, and pull this repository to the Secure Shell Linode. The repository includes terraform and kubernetes configuration files that we'll need for subsequent steps.

1. Pull down this repository to the Linode machine-

```
git init && git pull https://github.com/abeaudin/toronto-workshop-public
```

### Install Terraform 

Next step is to install Terraform. Run the below commands from the Linode shell-
```
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
```
```
 wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
 ```
 NOTE: This command will results in garbage on the screen, this is ok.
 ```
 echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
 ```
  ```
  sudo apt update && sudo apt-get install terraform
  ```

### Provision LKE Cluster using Terraform
![image](https://user-images.githubusercontent.com/7717493/194061886-5dc2dee1-8611-4ab3-9200-b5cde6aea221.png)

Next, we build the LKE cluster, with the terraform files that are included in this repository, and pulled into the Linode Shell from the prior git command.

1. From the Linode Cloud Manager, create an API token and copy it's value (NOTE- the Token should have full read-write access to all Linode components in order to work properly with terraform).

2. From the Linode shell, set the TF_VAR_token env variable to the API token value. This will allow terraform to use the Linode API for infrastructure provisioning.
```
export TF_VAR_token=[api token value]
```
3. Initialize the Linode terraform provider-
```
terraform init 
```
4. Next, we'll use the supplied terraform files to provision the LKE cluster. First, run the "terraform plan" command to view the plan prior to deployment-
```
terraform plan \
 -var-file="terraform.tfvars"
 ```
 5. Run "terraform apply" to deploy the plan to Linode and build your LKE cluster-
 ```
 terraform apply \
 -var-file="terraform.tfvars"
 ```
Once deployment is complete, you should see an LKE cluster within the "Kubernetes" section of your Linode Cloud Manager account.

### Deploy Containers to LKE 
![image](https://user-images.githubusercontent.com/7717493/194061837-e8846d2c-6669-49cd-9d45-395126442fe4.png)

Next step is to use kubectl to deploy the cockroach db container to the LKE cluster. 

1. Install kubectl via the below commands from the Linode shell-
```
sudo apt-get update && sudo apt-get install -y ca-certificates curl && sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
sudo apt-get update && sudo apt-get install -y kubectl
```
2. Extract the needed kubeconfig from each cluster into a yaml file from the terraform output.
```
 export KUBE_VAR=`terraform output kubeconfig_1` && echo $KUBE_VAR | base64 -di > lke-cluster-config.yaml
```
3. Define the yaml file output from the prior step as the kubeconfig.
```
export KUBECONFIG=lke-cluster-config.yaml
```
4. You can now use kubectl to manage the first LKE cluster. Enter the below command to view a list of clusters, and view which cluster is currently being managed.
```
kubectl config get-contexts
```
5. Deploy an application to the first LKE cluster, using the deployment.yaml file included in this repository.
```
kubectl create -f deployment.yaml
```
6. Next, we need to set our certificate and private key values as kubeconfig secrets. This will allow us to enable TLS on our LKE clusters. 

NOTE: For ease of the workshop, the certificate and key are included in the repository. This is not a recommended practice.
```
kubectl create secret tls mqtttest --cert cert.pem --key key.pem
```
7. Deploy the service.yaml included in the repository via kubectl to allow inbound traffic.
```
kubectl create -f service.yaml
```
8. Validate that the service is running, and obtain it's external IP address.
```
kubectl get services -A
```
### Summary of Linode Provisioning 

With the work above done, you've successfully setup a cluster in the Toronto regions, and deployed the cockroach db application to it. 
