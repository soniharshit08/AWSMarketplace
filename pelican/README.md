## **Pelican on AWS marketplace**

### Overview
Datametica intends to bring its suite of products starting with the Pelican Data Verification product to the cloud marketplace. We have implemented support to launch and use Pelican over Kubernetes and can be integrated with AWS's EKS as a Kubernetes app from the marketplace.

### EKS Deployment architecture
Pelican can be deployed using helm charts on EKS. There are two offerings as shown in below architectural diagrams. In the case of UBB (Usage Based Billing) - the custom metered billing is applicable in the 'pay as you go model' - billed by AWS monthly. In the case of BYOL - the customer has to connect to DM support and ask for a Pelican product license.

#### Pelican deployed with Usage based billing offer:
![alt text](resources/pelican-ubb.png)

#### Pelican deployed with BYOL offer:
![alt text](resources/pelican-byol.png)

## **Prerequisites**
 
Make sure you have installed the latest version of [AWS CLI](https://aws.amazon.com/cli/), [Docker](https://docs.docker.com/get-docker) & [Helm3](https://helm.sh/) 
 
**For linux systems, use the aws CLI:**
  
**Pelican Application images**

Following are the latest Pelican Images available at marketplace registry:

Option-1. With Pelican-UBB offer   

- [709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-ubb/pelican-web:1.0.10](http://709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-ubb/pelican-web:1.0.10)
- [709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-ubb/pelican-ubb:1.0.2](http://709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-ubb/pelican-ubb:1.0.2)
- [709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-ubb/pelican-db:5.7](http://709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-ubb/pelican-db:5.7)

Option-2. With Pelican-BYOL offer

- [709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-byol/pelican-web:1.0.1](http://709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-byol/pelican-web:1.0.1)
- [709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-byol/pelican-db:5.7](http://709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-byol/pelican-db:5.7)


##
### Steps to deploy pelican on EKS
**Create a new EKS cluster :**

Creation of an EKS cluster can be simplified using eksctl commands (document URL :  https://eksctl.io/ ).

If you choose to use a separate EKS environment solely to host the pelican, then it is recommended that you create a *private nodegroup* in your EKS cluster and use a NAT gateway for communication.

**Note** : You will have to create an EC2 Keypair if SSH access is desired for the nodes. Ref: [EC2 key pairs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)

```
eksctl create cluster --name pelican-cluster \
  --region us-east-1 \
  --zones us-east-1a,us-east-1b \
  --nodegroup-name private-ng1 \
  --nodes 1 \
  --ssh-public-key <EC2keypair> \
  --node-private-networking \
  --vpc-nat-mode HighlyAvailable
```


**Step 1** : Configure Your EKS cluster

Pelican can easily be launched into an existing EKS environment or you can [spin up a new one](https://github.com/aquasecurity/marketplaces/blob/5.3/aws/docs/eks/pages/aqua-in-a-box.md#create-a-new-EKS-cluster). using [<https://eksctl.io/>].

Configure the kubeconfig file for your EKS cluster on your local machine.

```
eksctl utils write-kubeconfig --cluster=<name> 

optional: [--kubeconfig=<path>][--profile=<profile>][--set-kubeconfig-context=<bool>]
```

Verify the node status

```
Kubectl get nodes
```

##
**Step 2** : Create a Service account with EKS IAM permissions

This command helps you set up the required IAM permissions required by Pelican Platform to run smoothly on Amazon EKS.

```
eksctl utils associate-iam-oidc-provider --cluster=<clustername> --approve [--profile=<profile>]
```

```
eksctl create iamserviceaccount --name pelican \
   --namespace default \ 
   --cluster <clustername> \
   --attach-policy-arn arn:aws:iam::aws:policy/AWSMarketplaceMeteringRegisterUsage \
   --approve

optional: [--profile <profile>]
```


##
**Step 3** : Install the Pelican Helm chart (Choose below option as per the offer you opted for)

Option-1: helm install command fot pelican-ubb

```
helm install \
      --set-string pelican.service.loadBalancerSourceRanges=<ip-range> \
      --set-string pelicandb.password=<password> \
      --set-string pelicandb.image.repo=<pelicandb-image-uri> \
      --set-string pelicandb.image.tag=<pelicandb-image-tag> \
      --set-string pelican.image.repo=<pelican-image-uri> \
      --set-string pelican.image.tag=<pelican-image-tag> \
      --set-string ubb.image.repo=<pelicanubb-image-uri> \
      --set-string ubb.image.tag=<pelicanubb-image-tag> <release-name> charts/pelican-ubb-chart/
```
Option-2: helm install command for pelican-byol

```
helm install \
    --set-string pelican.service.loadBalancerSourceRanges=<ip-range> \
    --set-string pelicandb.password=<password> \
    --set-string pelicandb.image.repo=<pelicandb-image-uri> \
    --set-string pelicandb.image.tag=<pelicandb-image-tag> \
    --set-string pelican.image.repo=<pelican-image-uri> \
    --set-string pelican.image.tag=<pelican-image-tag> <release-name>  charts/pelican-byol-chart
```

Example: Deploy UBB with latest images
```
helm install \
      --set-string pelican.service.loadBalancerSourceRanges=0.0.0.0/0 \
      --set-string pelicandb.password='DbRoot@312!' \
      --set-string pelicandb.image.repo='709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-ubb/pelican-db' \
      --set-string pelicandb.image.tag=5.7 \
      --set-string pelican.image.repo=709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-ubb/pelican-web \
      --set-string pelican.image.tag=1.0.10 \
      --set-string ubb.image.repo=709825985650.dkr.ecr.us-east-1.amazonaws.com/datametica/pelican-ubb/pelican-ubb \
      --set-string ubb.image.tag=1.0.2 dm-pelican charts/pelican-ubb-chart/
```
##
**Step 4** : Launch Pelican console

Obtain the Pelican console URL by running the following command.
```
kubectl get svc
```

After running the above command, you will get the LoadBalancer IP address and port number to launch the Pelican application from a web browser with appropriate network configurations (in case of VPN).

## Common configuration overrides
Refer to the `values.yaml` file for a full list of available values to override; some common keys are listed here:

| Key                                          | Default value                                 | Description                                                                                                                                                                                                                                                                 |
| -------------------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pelican.service.loadBalancerSourceRanges`   | 0.0.0.0/0                                     |  Restricts traffic through the load balancer to the IPs specified in this field. Provide comma separated CIRDs  |
| `pelicandb.password`                         | None                                          | The Pelican DB (MySql) pelican user password.                   |
| `pelicandb.image.repo`                       | None                                          | The Pelican DB docker image repository path.                    |
| `pelicandb.image.tag`                        | None                                          | The Pelican DB docker image tag.                                |
| `pelican.image.repo`                         | None                                          | The Pelican docker image repositiry path.                       |
| `pelican.image.tag`                          | None                                          | The Pelican docker image tag.                                   |
| `ubb.image.repo`                             | None                                          | The Pelican UBB agent image repository path.                    |
| `ubb.image.tag`                              | None                                          | The Pelican UBB agent image tag.                                |
| `pelican.service.port`                       | 8080                                          | The Pelican web service port number.                            |
| `pelicandb.port`                             | None                                          | The Pelican DB service port number.                             |

</tbody>
</table>
