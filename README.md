# Train a model on AWS Trn1 ParallelCluster

## Introduction

This document explains how to use AWS ParallelCluster to build HPC compute cluster that uses trn1 compute nodes to run your distributed ML training job. Once the nodes are launched, we will run a training task to confirm that the nodes are working, and use SLURM commands to check the job status. In this tutorial, we will use AWS pcluster command to run a YAML file in order to generate the cluster. As an example, we are going to launch multiple trn1.32xl nodes in our cluster.

We are going to set up our ParallelCluster infrastructure as below:

![image info](./examples/images/vpc-setup.png)

As shown in the figure above, inside a VPC, there are two subnets, a public and a private ones. Head node resides in the public subnet, while the compute fleet (in this case, trn1 instances) are in the private subnet. A Network Address Translation (NAT) gateway is also needed in order for nodes in the private subnet to connect to clients outside the VPC. In the next section, we are going to describe how to set up all the necessary infrastructure for Trn1 ParallelCluster.



## Prerequisite infrastructure

### VPC Creation
A ParallelCluster requires a VPC that has two subnets and a Network Address Translation (NAT) gateway as shown in the diagram above. [Here](./examples/general/network/vpc-subnet-setup.md) are the instructions to create the VPC and enable auto-assign public IPv4 address for the public subnet. 

### Key pair
A key pair is needed for access to the head node of the cluster. You may use an existing one or create a new key pair by following the instruction [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html#having-ec2-create-your-key-pair "Create key pair")

### AWS ParallelCluster Python package

AWS ParallelCluster Python package is needed in a local environment (i.e., your Mac/PC desktop with a CLI terminal or an AWS Cloud9) where you issue the command to launch the creation process for your HPC environment in AWS. See [here](https://docs.aws.amazon.com/parallelcluster/latest/ug/install-v3-virtual-environment.html) for instructions about installing AWS ParallelCluster Python package in your local environment.

### AWS CLI

Make sure you have installed the [AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html):

```
$ pip3 install awscli
```
Next, configure your aws credentials and default region:

```
$ aws configure
AWS Access Key ID [None]: YOUR_KEY
AWS Secret Access Key [None]: YOUR_SECRET
Default region name [us-east-1]:
Default output format [None]:
```

## Create a cluster

See table below for script to create trn1 ParallelCluster:

|Cluster      | Link |
|-------------|------------------|
|16xTrn1 nodes   | [trn1-16-nodes-pcluster.md](./examples/cluster-configs/trn1-16-nodes-pcluster.md)  |

## Launch training job

See table below for script to launch a model training job on the ParallelCluster:

|Job      | Link  |
|-------------|-------------------|
|BERT Large   | [dp-bert-launch-job.md](./examples/jobs/dp-bert-launch-job.md) |
|GPT3 (neuronx-nemo-megatron) | [neuronx-nemo-megatron-gpt-job.md](./examples/jobs/neuronx-nemo-megatron-gpt-job.md) |
|Llama 2 7B (neuronx-nemo-megatron) | [neuronx-nemo-megatron-llamav2-job.md](./examples/jobs/neuronx-nemo-megatron-llamav2-job.md) |

## Launch training job [End of Support]

See table below for scripts that are no longer supported:

|Job      | Link  |
|-------------|-------------------|
|GPT3 (Megatron-LM)        | [gpt3-launch-job.md](./examples/jobs/gpt3-launch-job.md)       |

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the Amazon Software License.


## Release Notes

Please refer to the [Change Log](releasenotes.md).

