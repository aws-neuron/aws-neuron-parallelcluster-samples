# AWS ParallelCluster with Trn1

## Introduction

This document explains how to use AWS ParallelCluster to build HPC compute environment that uses Trn1 compute nodes to run your distributed ML training job. Once the nodes are launched, we will run a training task to confirm that the nodes are working, and use slurm commands to check the job status. In this tutorial, we will use AWS pcluster command to run a yaml file in order to generate the cluster. As an example, we are going to launch multiple Trn1.32xl nodes in our cluster.

Some useful slurm commands are `sinfo` and  `squeue`. `sinfo` command displays information about Slurm modes and partitions. `sinfo` command provides information about job queues currently running in the Slurm schedule. Once the job is done, Slurm will generate a log file `slurm-XXXXXX.out`. You may then use `tail -f slurm-XXXXXX.out`, to inspect the job summary.

## Prerequisite infrastructure

This document explains how to use AWS ParallelCluster to build HPC compute environment that uses Trn1 compute nodes to run your distributed ML training job. Once the nodes are launched, we will run a training task to confirm that the nodes are working, and use slurm commands to check the job status. In this tutorial, we will use AWS pcluster command to run a yaml file in order to generate the cluster. As an example, we are going to launch multiple Trn1.32xl nodes in our cluster.

The following are required infrastructure components for Parallel Cluster. You must have these component created and configured before moving on to create the cluster.

### VPC

Trn1 instances of various sizes are being launched and will be available in many AWS regions. Currently, Trn1 instances are available in us-west-2 and us-east-1 regions. You need to find out the availability zones (AZ) that are mapped to your subscription. The way to do this is by copying and pasting the following command in your local or cloud desktop terminal:

```
AZ1=$(aws ec2 describe-availability-zones \
--region us-east-1 \
--query "AvailabilityZones[]" \
--filters "Name=zone-id,Values=use1-az4" \
--query "AvailabilityZones[].ZoneName" \
--output text)

AZ2=$(aws ec2 describe-availability-zones \
--region us-east-1 \
--query "AvailabilityZones[]" \
--filters "Name=zone-id,Values=use1-az5" \
--query "AvailabilityZones[].ZoneName" \
--output text)

AZ3=$(aws ec2 describe-availability-zones \
--region us-west-2 \
--query "AvailabilityZones[]" \
--filters "Name=zone-id,Values=usw2-az4" \
--query "AvailabilityZones[].ZoneName" \
--output text)

echo -e "\nYour Trn1 availability zones are $AZ1, $AZ2, $AZ3\n"

```

An example output of the above snippet may be:

`Your Trn1 availability zones are us-east-1a, us-east-1f, us-west-2d`

Your results may vary. But make a note of the AZ. You may arbitrarily select one from these AZ choices.


### Subnet
Once you have a VPC, you also need to create two subnets within the VPC for your HPC environment. See [AWS documentation](https://docs.aws.amazon.com/parallelcluster/latest/ug/network-configuration-v3.html#network-configuration-v3-two-subnets "Creating subnets") for creating the VPC and two subnets (**public with NAT gateway for head node, private for compute nodes**). The network configuration for this tutorial uses two subnets as described in [this diagram](https://docs.aws.amazon.com/parallelcluster/latest/ug/network-configuration-v3.html#network-configuration-v3-two-subnets "Network configuration"). As shown [here](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-subnets-commands-example.html#vpc-subnets-commands-example-launch-instance "Launch instance"), you may use AWS CLI to create a subnet within the VPC. During the creation process, observe and note the subnet ID. Make a note of the subnet-ID for public and private subnets. To identify the public subnet, look into your VPC portal in your AWS console, select Subnet tab, and pick a subnet that was just created identifiable by the VPC name. Look into the routing table:

![image info](./document_assets/public-subnet2.png)

In this example, there is a Target `igw-XXXXXXX`. This is the internet gateway. Therefore this particular subnet is the public subnet. A private subnet does not have this gateway.


### Key pair
You also need a key pair. You may use an existing one. But if you wish to create a new key pair, [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html#having-ec2-create-your-key-pair "Create key pair") is the instruction.


### AWS Parallel Cluster 3.2.0 Python package

This is needed in an local environment (i.e., your Mac/PC desktop with a CLI terminal or an AWS Cloud9) where you issue the command to launch the creation process for your HPC environment in AWS. See [this](https://docs.aws.amazon.com/parallelcluster/latest/ug/install-v3-virtual-environment.html) for details on how to install this package. After the installation. A Python virtual environemnt will be created. Activate this particular virtual environment. 



## Create a cluster

Below are the steps to create a cluster and run a distributed training job.

### Step 1: Create AMI 
For now, a prebuilt AMI will be provided to you for this exercise. This AMI is specific for ParallelCluster. 

### Step 2: Define cluster configuration
The cluster configuration is specified in a yaml file. As an example, a `launch.yaml` is shown below:
```
HeadNode:
  InstanceType: c5.4xlarge
  LocalStorage:
    RootVolume:
      Size: 800
  Networking:
    SubnetId: subnet-XXXXXXXX
  Ssh:
    KeyName: trn1
Image:
  CustomAmi: ami-XXXXXXX
  Os: alinux2
Region: us-XXXX-X
Scheduling:
  Scheduler: slurm
  SlurmQueues:
  - CapacityType: ONDEMAND
    ComputeResources:
    - Efa:
        Enabled: true
      InstanceType: trn1.32xlarge
      MaxCount: 2
      MinCount: 0
      Name: queue-0
    ComputeSettings:
      LocalStorage:
        EphemeralVolume: 
          MountDir: /scratch
        RootVolume:
          Size: 800
    Name: compute1
    Networking:
      PlacementGroup:
        Enabled: false
      SubnetIds:
      - subnet-XXXXXXXX
SharedStorage:
- EfsSettings:
    ProvisionedThroughput: 1024
    ThroughputMode: provisioned
  MountDir: /neuron_shared /efs
  Name: ktfefs
  StorageType: Efs
DevSettings:
  InstanceTypesData: '{"trn1.32xlarge": {"InstanceType": "trn1.32xlarge", "SupportedUsageClasses":
    ["on-demand"], "ProcessorInfo": {"SupportedArchitectures": ["x86_64"]}, "VCpuInfo":
    {"DefaultVCpus": 128, "DefaultCores": 64, "DefaultThreadsPerCore": 2}, "MemoryInfo":
    {"SizeInMiB": 520079}, "EbsInfo": {"EncryptionSupport": "supported"}, "NetworkInfo":
    {"MaximumNetworkCards": 8, "EfaSupported": true}}}'
```
Edit the entries above, such that at `HeadNode` section, the subnet ID should be that of the public subnet ID. In `Networking` under '`HeadNode`, put one of the Trn1 AZ mapped to AWS your account as shown in output of the command from **VPC** section above. Edit `CustomAmi` with the AMI-ID provided to you. Also, in `Networking` section, edit SubnetIds with that of the private subnet ID. Once all edits are done, copy the yaml file above to your local desktop, save it as `launch.yaml`. 

Note: keep MinCount at 0 for ComputeResource, as ParallelCluster doesn't yet fully support Trn1 instance yet. MaxCount is set to 2 for this tutorial to avoid hardware capacity issues.


### Step 3: Run pcluster command
Once an AMI-ID is known, and the laml file is in your local desktop, you may use AWS pcluster CLI from your local environment or any cloud desktop that has AWS pcluster CLI installed. With pcluster CLI, all you need to specify is a yaml file that contains information necessary to set up a cluster head node and compute nodes. The command to launch a pcluster creation process is: 

```
pcluster create-cluster --cluster-configuration launch.yaml \
--cluster-name MyTrn1Cluster \
--suppress-validators type:ComputeResourceLaunchTemplateValidator \
\
```

Where 

`--cluster-configuration` is the path to a yaml file (see below)

`--causter-name` is the name of your cluster

`--supress-validators` is used here to generalize this command so it won’t run into error triggered by tagging `policies, if any.

You may check with EC2 console and CloudFormation console for status of the cluster creation process. 

## Run training job
Once the CloudFormation shows the cluster is succesfully created, or when EC2 console shows that Head Node is running, you are ready to launch a training job. You may ssh into the head node from your local environment used to launch the cluster in the previous step. As an example here, we will use [Phase 1 Hugging Face BERT-Large pretraining](https://awsdocs-neuron-staging.readthedocs-hosted.com/en/release_2.3.0rc2/frameworks/torch/tutorials/training/bert.html?next=https%3A%2F%2Fawsdocs-neuron-staging.readthedocs-hosted.com%2Fen%2Frelease_2.3.0rc1%2Fframeworks%2Ftorch%2Ftutorials%2Ftraining%2Fbert.html%3Fnext%3Dhttps%253A%252F%252Fawsdocs-neuron-staging.readthedocs-hosted.com%252Fen%252Frelease_2.3.0rc1%252Fframeworks%252Ftorch%252Ftutorials%252Ftraining%252Fbert.html&ticket=ST-1663365027-jWyjPKGS3TtpDY9Ih0iklXykKnHRSSnL#phase-1-bert-large-pretraining) as the example job to submit to the cluster. This pretraining script will be stored in the head node. Upon launching the job, the head node will distribute it to each compute node. 

In this entire process, there will be three scripts required:

* Python script (.py) that executes the workload
* Shell script that sets up environment definition and runtime argument to be passed into the Python script.
* Slurm script that submit the shell script to the job queue

The Python script `dp_bert_large_hf_pretrain_hdf5.py` in head node is used as the main script that executes the training workloads. This script is in `~/examples/dp_bert_hf_pretrain` of the head node. `~/` or the home directory is shared with compute nodes.

To continue with the training job:

### Step 4: Activate head node's Python virtual environment
In a terminal of the head node, activate the Python virtual environment: `source ~/aws_neuron_venv_pytorch_p37/bin/activate`

### Step 5: Create a shell script
create a shell script in head node (i.e., `pretrain_ph1.sh`) to set up the runtime environment and invoke torchrun command to execute the `pretraining script dp_bert_large_hf_pretrain_hdf5.py`:

`pretrain_ph1.sh`:

```
#!/bin/bash 
WORLD_SIZE_JOB=$SLURM_NTASKS
RANK_NODE=$SLURM_NODEID
MASTER_ADDR_JOB=(`scontrol show hostnames $SLURM_JOB_NODELIST`)
export FI_EFA_USE_DEVICE_RDMA=1
export FI_PROVIDER=efa
export BUCKET_CAP_MB=512

date;hostname;echo $WORLD_SIZE_JOB;echo $RANK_NODE;echo $MASTER_ADDR_JOB

sudo rmmod neuron; sudo modprobe neuron

steps_this_run=28125
if [[ "$NEURON_EXTRACT_GRAPHS_ONLY" == "1" ]]; then
    steps_this_run=10
fi

# run from /tmp to avoid interference among nodes on shared homedir
cd /tmp

sudo sysctl -w net.ipv4.ip_local_reserved_ports=48620    
XLA_USE_BF16=1 torchrun --nproc_per_node=32 ~/examples/dp_bert_hf_pretrain/dp_bert_large_hf_pretrain_hdf5.py --steps_this_run=$steps_this_run --batch_size 16 --grad_accum_usteps 32 |& tee run_pretrain_ph1_log.txt
```

Change `pretrain_ph1.sh` to executable:

`chmod +x ./pretrain_ph1.sh`

This `pretrain_ph1.sh` script will be invoked by a SLURM script running in the head node. 

### Step 6: Compile the model
To do compilation, use the command `sbatch` to run the script on multiple nodes (in this example, 2 nodes):

`sbatch --nodes 2 --exclusive --wrap "neuron_parallel_compile ./pretrain_ph1.sh"`

### Step 7: Launch pretraining job
Wait for compilation to finish on these nodes, then start the actual pretraining: 

`sbatch --nodes 2 --exclusive --wrap "./pretrain_ph1.sh"` 


The job id will be displayed by sbatch. The run output will appear in slurm_\<JOB-ID\>.out file in head node.




## Troubleshooting

If ParallelCluster fails to come up, check the following: 

* If head node fails to create, then check the Events to see the reason for the failure. For example, the AMI maybe created using a different ParallelCluster version.
* Check if subnets are correct. For trn1 cluster, need to use public subnet for c5 head node and  private subnet for compute trn1 nodes.
* For more detailed debugging, need to go to CloudFormation, find the stack that you tried to create, click on Resources tab and find CloudWatchLogGroup link to go to the logs.


## Known Issues
* Parallelcluster 3.2.0 doesn’t support trn1. Therefore, in this version, min count needs to be zero. 

* ParallelCluster hostname in /etc/hosts causing “NCCL WARN Invalid NCCL_COMM_ID“ error

Currently, after creating the ParallelCluster and running a NCCL application, you will encounter the following error:

```
queue-trn2-dy-compute-resource-trn2-1:9131:9167 [0] include/socket.h:233 NCCL WARN Net : error encountered when getting address info : Name or service not known
queue-trn2-dy-compute-resource-trn2-1:9131:9167 [0] bootstrap.cc:30 NCCL WARN Invalid NCCL_COMM_ID [queue-0-dy-compute-resource-trn2-1.tests.pcluster:42323], please use format: <ipv4>:<port> or [<ipv6>]:<port> or <hostname>:<port>
```

The workaround is to run the following command (replace N with the number of nodes in you cluster, and  “queue” in the search string with a prefix that matches your the prefix for your compute host names):

`sbatch --nodes N --wrap "sudo sed -i 's/queue-.*\.pcluster//g' /etc/hosts"`

Internal note: https://t.corp.amazon.com/D53696332/communication


* Internal note: PlacementGroup=True
Currently the yaml setting has PlacementGroup=False. Try PlacementGroup=True in yaml to see if it works for Trn1 and update yaml.




## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the Amazon Software License.

