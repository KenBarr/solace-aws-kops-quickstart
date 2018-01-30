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
Kubernetes documentation outlines several ways to [run Kubernetes on AWS EC2s](https://kubernetes.io/docs/getting-started-guides/aws/). For the remainder of this guide we will use Kubernetes Operations, (kops), folowing this [guideKOPS](https://github.com/kubernetes/kops/blob/master/docs/aws.md).  For simplisity of demonstration, useing .k8s.local and skipped setting up DNS.

When it comes time to create your [cluster](https://github.com/kubernetes/kops/blob/master/docs/aws.md#create-cluster-configuration), recommendation is to create 3 worker nodes across 3 zone in a region.  For example:

```sh
 kops create cluster --node-count 3 \
                    --zones us-east-1a,us-east-1b,us-east-1c \
                    --node-size t2.large \
                    --kubernetes-version 1.8.5  solacecluster.k8s.local
```

Ensure that you validate the cluster is [up and functional](https://github.com/kubernetes/kops/blob/master/docs/aws.md#use-the-cluster).


**Step 2**: Load a Solace VMR image into AWS Elastic Container Registery, (ECR).
(Part I) Download a copy of the Solace VMR Software. Follow Step two from the [Solace Kubernetes QuickStart](https://github.com/SolaceProducts/solace-kubernetes-quickstart#how-to-deploy-a-vmr-onto-kubernetes) to download the VMR software (Community Edition for single-node deployments, Evaluation Edition for VMR HA deployments)

Follow the link in resultant email and it will download a Solace VMR image with a name simular to ```soltr-8.7.0.1030-vmr-evaluation-docker.tar.gz```.

(Part II) Deploy the VMR docker image to your Docker registry of choice. You can utilize the AWS Elastic Container Registry to host the VMR Docker image. For more information, refer to Amazon [Elastic Container Registry](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html).

Assuming you have docker installed here are the basic steps from the above documentation. 
* Download the Docker container image as per PartI above.
* Load the image into local registry ```sh docker load -i <soltr-X.X.X.XXX-vmr-yyyy.tar.gz>```
* Create a repository ```aws ecr create-repository --repository-name solace-vmr```
    * note repositoryUri from output
* tag docker image ```docker tag solace-app <repositoryUri>```
* log into ecr ```aws ecr get-login --no-include-email```
* docker push image ```docker push  <repositoryUri>```

**Step 3**: Deploy a Solace VMR release onto Kubernetes.
Follow Step five from the [Solace Kubernetes QuickStart](https://github.com/SolaceProducts/solace-kubernetes-quickstart#how-to-deploy-a-vmr-onto-kubernetes)
Ensure you set the environment correctly for:
```sh
  PASSWORD=<YourAdminPassword>
  SOLACE_IMAGE_URL=<repositoryUri>:latest
  CLOUD_PROVIDER=aws
```

```sh
./start_vmr.sh -c ${CLOUD_PROVIDER} -p ${PASSWORD} -i ${SOLACE_IMAGE_URL} -v values-examples/small-persist-ha-provisionPvc.yaml
```