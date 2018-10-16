# Install Solace Message Router HA deployment onto a Kubernetes running in AWS

## Purpose of this repository

This repository expands on [Solace Kubernetes Quickstart](https://github.com/SolaceProducts/solace-kubernetes-quickstart) to provide a concrete example of how to deploy redundent Solace VMRs in HA configuration on AWS on a 3 node cluster across 3 zones.

## Description of Solace VMR

Solace Virtual Message Router (VMR) software provides enterprise-grade messaging capabilities so you can easily enable event-driven communications between applications, IoT devices, microservices and mobile devices across hybrid cloud and multi cloud environments. The Solace VMR supports open APIs and standard protocols including AMQP 1.0, JMS, MQTT, REST and WebSocket, along with all message exchange patterns including publish/subscribe, request/reply, fan-in/fan-out, queueing, streaming and more. The Solace VMR can be deployed in all popular public cloud, private cloud and on-prem environments, and offers both feature parity and interoperability with Solace’s proven hardware appliances and Messaging as a Service offering called Solace Cloud.

VMRs can either be deployed as a 3 node HA cluster or a single node. For simple test environments that need to validate application functionality, a single instance will suffice.
Note that in production or any environment where message loss can not be tolerated, an HA cluster is required.

## How to Deploy a VMR into Kubernetes running in AWS

This is a 3 step process:
1. Install Kubernetest in AWS.
2. Load a Solace VMR image into AWS Elastic Container Registery, (ECR).
3. Deploy a Solace VMR release onto Kubernetes.

**Step 1**: Install Kubernetes in AWS.

Create an S3 bucket to contain cluster state:
![alt text](/images/Create_Bucket.png "S3 bucket for Kubernetes cluster state")

Kubernetes documentation outlines several ways to [run Kubernetes on AWS EC2s](https://kubernetes.io/docs/getting-started-guides/aws/). For the remainder of this guide we will use Kubernetes Operations, (kops), folowing this [guideKOPS](https://github.com/kubernetes/kops/blob/master/docs/aws.md).  For simplisity of demonstration, useing .k8s.local and skipped setting up DNS.

When it comes time to create your [cluster](https://github.com/kubernetes/kops/blob/master/docs/aws.md#create-cluster-configuration), recommendation is to create 3 worker nodes across 3 zone in a region.  For example:

```sh
 kops create cluster --node-count 3 \
                    --state s3://solacecluster-k8s \
                    --zones us-east-1a,us-east-1b,us-east-1c \
                    --node-size t2.large \
                    --yes \
                    --kubernetes-version 1.10.0  solacecluster.k8s.local
```

Ensure that you validate the cluster is [up and functional](https://github.com/kubernetes/kops/blob/master/docs/aws.md#use-the-cluster).

Here is an example of a cluster ready to accept deployments:
```sh
kops validate cluster
Using cluster from kubectl context: solacecluster.k8s.local

Validating cluster solacecluster.k8s.local

INSTANCE GROUPS
NAME                    ROLE    MACHINETYPE     MIN     MAX     SUBNETS
master-us-east-1a       Master  c4.large        1       1       us-east-1a
nodes                   Node    m4.large        3       3       us-east-1a,us-east-1b,us-east-1c

NODE STATUS
NAME                            ROLE    READY
ip-172-20-106-109.ec2.internal  node    True
ip-172-20-40-81.ec2.internal    master  True
ip-172-20-48-8.ec2.internal     node    True
ip-172-20-71-48.ec2.internal    node    True

Your cluster solacecluster.k8s.local is ready
```

**Step 2**: Load a Solace VMR image into AWS Elastic Container Registery, (ECR).
(Part I) Download a copy of the Solace VMR Software. Follow Step two from the [Solace Kubernetes QuickStart](https://github.com/SolaceProducts/solace-kubernetes-quickstart#how-to-deploy-a-vmr-onto-kubernetes) to download the VMR software (Community Edition for single-node deployments, Evaluation Edition for VMR HA deployments)

Follow the link in resultant email and it will download a Solace VMR image with a name simular to ```soltr-8.8.0.1027-vmr-evaluation-docker.tar.gz```.

(Part II) Deploy the VMR docker image to your Docker registry of choice. You can utilize the AWS Elastic Container Registry to host the VMR Docker image. For more information, refer to Amazon [Elastic Container Registry](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html).

Assuming you have docker installed here are the basic steps from the above documentation. 
* Download the Docker container image as per Part I above.

| Action | Syntex |
| ------ | ------ |
| Load the image into local registry | ``` docker load -i <soltr-X.X.X.XXX-vmr-evaluation-docker.tar.gz>```    |
| Get image ID of container          | ``` docker images``` note IMAGE ID 
| Create a repository                | ```aws ecr create-repository --repository-name solace-vmr```  note repositoryUri|
| tag docker image                   | ```docker tag <imageID> <repositoryUri>:X.X.X.XXXX-evaluation```  |
| log into ecr                       | ```eval $(aws ecr get-login | sed 's|https://||')```             |
| docker push image                  | ```docker push  <repositoryUri>:X.X.X.XXXX-evaluation```          |

You should now be able to view your images in the Elastic Container Registry,(ECR):

![alt text](/images/ecr-populated.png "Elastic Container Registry")

**Step 3**: Deploy a Solace VMR release onto Kubernetes.
Follow Step five from the [Solace Kubernetes QuickStart](https://github.com/SolaceProducts/solace-kubernetes-quickstart#how-to-deploy-a-vmr-onto-kubernetes)
Ensure you set the environment correctly for:
```sh
  PASSWORD=<YourAdminPassword>
  SOLACE_IMAGE_URL=<repositoryUri>:X.X.X.XXXX-evaluation
  CLOUD_PROVIDER=aws
```
```sh
./start_vmr.sh -c ${CLOUD_PROVIDER} -p ${PASSWORD} -i ${SOLACE_IMAGE_URL} -v values-examples/small-persist-ha-provisionPvc.yaml
```
Follow the solace-kubernetes-quickstart to see how to validate the cluster and test VMR.
