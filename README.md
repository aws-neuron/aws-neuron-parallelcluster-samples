# Train a model on AWS Trn1 ParallelCluster

## Introduction

This document explains how to use AWS ParallelCluster to build HPC compute cluster that uses trn1 compute nodes to run your distributed ML training job. Once the nodes are launched, we will run a training task to confirm that the nodes are working, and use slurm commands to check the job status. In this tutorial, we will use AWS pcluster command to run a yaml file in order to generate the cluster. As an example, we are going to launch multiple trn1.32xl nodes in our cluster.

We are going to set up our ParallelCluster infrastructure as below:

![image info](./examples/images/vpc-setup.png)

As shown in the figure above, inside a VPC, there are two subnets, a public and a private ones. Head Node resides in the public subnet, while the compute fleet (in this case, trn1 instances) are in the private subnet. A Network Address Translation (NAT) gateway is also needed in order for nodes in the private subnet to connect to clients outside the VPC. In the next section, we are going to describe how to set up all the necessary infrastructure for Trn1 ParallelCluster.

Some useful slurm commands are `sinfo` and  `squeue`. `sinfo` command displays information about slurm modes and partitions. `sinfo` command provides information about job queues currently running in the Slurm schedule. Once the job is done, slurm will generate a log file `slurm-XXXXXX.out`. You may then use `tail -f slurm-XXXXXX.out`, to inspect the job summary.

## Prerequisite infrastructure

### VPC
A ParallelCluster requires a VPC. First, you must have these component created and configured before creating the cluster. [Here](./examples/general/network/vpc-setup.m) is the instruction to create a VPC. This VPC has two subnets as described in [this diagram](https://docs.aws.amazon.com/parallelcluster/latest/ug/network-configuration-v3.html#network-configuration-v3-two-subnets "Network configuration"). Second, after VPC is created, follow [this instruction](./examples/general/network/subnet-setup.md) to configure the public subnet.

### Key pair
You also need a key pair. You may use an existing one. But if you wish to create a new key pair, [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html#having-ec2-create-your-key-pair "Create key pair") is the instruction. A key is pair is needed so you may SSH into the head node of the cluster.

Details and steps are provided in [here](./examples/general/ami/ami_setup.md)

### AWS Parallel Cluster Python package

This is needed in a local environment (i.e., your Mac/PC desktop with a CLI terminal or an AWS Cloud9) where you issue the command to launch the creation process for your HPC environment in AWS. See [this](https://docs.aws.amazon.com/parallelcluster/latest/ug/install-v3-virtual-environment.html) for details on how to install this package. After the installation. A Python virtual environment will be created. Activate this particular virtual environment. 

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

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the Amazon Software License.

