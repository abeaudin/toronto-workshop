Toronto Linode Workshop
======================

# About

Package of template files, examples, and illustrations for the Toronto Linode Workshop.

# Contents

## Template Files
- Sample Terraform files for deploying an LKE cluster on Linode.
- Sample kubernetes deployment files and helm instructions for running the ELK stack on an LKE cluster.

## Step by Step Instructions

### Overview

![image](https://user-images.githubusercontent.com/7717493/194063848-45d8649a-8f31-41c0-b391-d585a13bd626.png)
The scenario is written to demonstrate the deployment of an ELK stack on an LKE Cluster

The workshop scenario builds the following components and steps-

1. A Secure Shell Linode (provisioned via the Linode Cloud Manager GUI) to serve as the command console for the environment setup.

2. Installing developer tools on the Secure Shell (terraform and kubectl) for use in environment setup.

3. One Linode Kubernetes Engine (LKE) Cluster, deployed to a the Toronto Linode region, provisioned via terraform.

4. Deploying the ELK stack to the LKE cluster.


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
![image](https://user-images.githubusercontent.com/7717493/194547712-d1d8b728-05f1-44dc-b342-16113878f2f0.png)

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
![image](https://user-images.githubusercontent.com/7717493/194547571-64489701-7191-4140-a57b-6d3ed73f838c.png)

Next step is to use kubectl to deploy the elastic stack to the LKE cluster. 

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
```
chmod 400 lke-cluster-config.yaml
```
3. Define the yaml file output from the prior step as the kubeconfig.
```
export KUBECONFIG=lke-cluster-config.yaml
```
4. Install helm
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
```
```
chmod 700 get_helm.sh
```
```
./get_helm.sh
```

5. Configure helm with the elastic repo
```
helm repo add elastic https://helm.elastic.co
```
```
helm repo update
```

6. Install ELK stack on the LKE Cluster
```
helm install elasticsearch elastic/elasticsearch
```
```
helm install filebeat elastic/filebeat
```
```
helm install kibana elastic/kibana
```

Once these command have been executed, watch the deployment with 
```
kubectl get pods -w
```
until you see output like
```
/home/workshop# kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
elasticsearch-master-0           0/1     Running   0          107s
elasticsearch-master-1           0/1     Running   0          107s
elasticsearch-master-2           0/1     Running   0          107s
filebeat-filebeat-7vx2t          0/1     Running   0          104s
filebeat-filebeat-lj8s5          0/1     Running   0          105s
filebeat-filebeat-n6vlv          0/1     Running   0          104s
kibana-kibana-864bccc7f5-47dmn   0/1     Running   0          65s
```

7. Next, we need to set our certificate and private key values as kubeconfig secrets. This will allow us to enable TLS on our LKE clusters. 

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
** Note: If your service shows as pending and it won't change, check to make sure that you're not at your nodebalancer limit.

### Summary of Linode Provisioning 

With the work above done, you've successfully setup a cluster in the Toronto region, and deployed the ELK stack to an LKE Cluster in it.
