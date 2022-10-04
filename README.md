# Train a model on AWS Trn1 ParallelCluster

## Introduction

This document explains how to use AWS ParallelCluster to build HPC compute cluster that uses trn1 compute nodes to run your distributed ML training job. Once the nodes are launched, we will run a training task to confirm that the nodes are working, and use slurm commands to check the job status. In this tutorial, we will use AWS pcluster command to run a yaml file in order to generate the cluster. As an example, we are going to launch multiple trn1.32xl nodes in our cluster.

We are going to set up our ParallelCluster infrastructure as below:

![image info](./examples/images/vpc-setup.png)

As shown in the figure above, inside a VPC, there are two subnets, a public and a private ones. Head Node resides in the public subnet, while the compute fleet (in this case, trn1 instances) are in the private subnet. A Network Address Translation (NAT) gateway is also needed in order for nodes in the private subnet to connect to clients outside the VPC. In the next section, we are going to describe how to set up all the necessary infrastructure for Trn1 ParallelCluster.

Some useful slurm commands are `sinfo` and  `squeue`. `sinfo` command displays information about slurm modes and partitions. `sinfo` command provides information about job queues currently running in the Slurm schedule. Once the job is done, slurm will generate a log file `slurm-XXXXXX.out`. You may then use `tail -f slurm-XXXXXX.out`, to inspect the job summary.

## Prerequisite infrastructure
The following are required infrastructure components for ParallelCluster. You must have these component created and configured before moving on to create the cluster.

### VPC

Trn1 instances of various sizes are being launched and will be available in many AWS regions. Follow the instructions [here](./examples/general/network/vpc-setup.md) to set up your VPC.

### Subnets
Once you have a VPC, you also need to create two subnets within the VPC for your HPC environment. See [AWS documentation](https://docs.aws.amazon.com/parallelcluster/latest/ug/network-configuration-v3.html#network-configuration-v3-two-subnets "Creating subnets") for creating the VPC and two subnets (**public with NAT gateway for head node, private for compute nodes**). The network configuration for this tutorial uses two subnets as described in [this diagram](https://docs.aws.amazon.com/parallelcluster/latest/ug/network-configuration-v3.html#network-configuration-v3-two-subnets "Network configuration"). As shown [here](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-subnets-commands-example.html#vpc-subnets-commands-example-launch-instance "Launch instance"), you may use AWS CLI to create a subnet within the VPC. Follow the instructions [here](./examples/general/network/subnet-setup.md) to set up your public subnet.

### Peering Connection
A peering connection is needed between your default VPC and the VPC for ParallelCluster. Follow [this](./examples/general/network/peering-connection-setup.md) instruction for how to create a peering connection, and add it to the public subnet's route table. 

### NAT gateway
A Network Address Translation (NAT) gateway is required for compute nodes in the private subnet to connect to service outside (i.e., web access for essential software updates or packages) the VPC. This is a one-way connection such that outside services cannot connect to the nodes inside the private subnet. For convenience, choose NAT gateway option during the time you create the VPC, so that a NAT gateway is created automatically. In this example, it is created while we set up VPC as shown in [the VPC setup instructions](./examples/general/network/vpc-setup.md).

### Key pair
You also need a key pair. You may use an existing one. But if you wish to create a new key pair, [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html#having-ec2-create-your-key-pair "Create key pair") is the instruction.

### AMI
You also need a ParallelCluster AMI to run on your cluster. For this example, you may find a list of available ParallelCluster AMI in [here](https://github.com/aws/aws-parallelcluster/blob/v2.11.7/amis.txt). For the example here, we will use Amazon Linux 2 AMI. You may find the AMI ID based on the region accessible by your account. Once the AMI is chose, you will use this AMI to create a ParallelCluster. After the cluster is created, you will update your cluster with Neuron stacks by running the fresh installation instructions, as shown in [the Fresh install section](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/neuron-intro/pytorch-setup/pytorch-install.html#develop-on-aws-ml-accelerator-instance) of the AWS Neuron SDK documentation.

### AWS Parallel Cluster Python package

This is needed in a local environment (i.e., your Mac/PC desktop with a CLI terminal or an AWS Cloud9) where you issue the command to launch the creation process for your HPC environment in AWS. See [this](https://docs.aws.amazon.com/parallelcluster/latest/ug/install-v3-virtual-environment.html) for details on how to install this package. After the installation. A Python virtual environment will be created. Activate this particular virtual environment. 


## Create a cluster

See table below for script to create trn1 ParallelCluster:

|example      | cluster creation |
|-------------|------------------|
|BERT Large   | [trn1-8-nodes-pcluster.md](./examples/cluster-configs/trn1-8-nodes-pcluster.md)  |

## Launch training job

See table below for script to launch a model training job on the ParallelCluster:

|example      | slurm job launch  |
|-------------|-------------------|
|BERT Large   | [dp-bert-launch-job.md](./examples/jobs/dp-bert-launch-job.md) |

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the Amazon Software License.

