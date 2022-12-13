# Create ParallelCluster

1. Once your VPC, ParallelCluster python package, and key pair are set up, you are ready to create a ParallelCluster. Copy the following content into a launch.yaml file in your local desktop where AWS ParallelCluster CLI is installed. Here is the YAML if your operating system (OS) choice is Amazon Linux 2:

```
Region: <YOUR REGION> # i.e., us-west-2
Image:
  Os: alinux2
HeadNode:
  InstanceType: c5.4xlarge
  Networking:
    SubnetId: subnet-<PUBLIC SUBNET ID>
  Ssh:
    KeyName: <KEY NAME WITHOUT .PEM>
  LocalStorage:
    RootVolume:
      Size: 1024
  CustomActions:
    OnNodeConfigured:
      Script: s3://neuron-s3/pcluster/post-install-scripts/neuron-installation/v2.4.0/al2/pt/install_neuron.sh
  Iam:
    S3Access:
       - BucketName: neuron-s3
         EnableWriteAccess: false
Scheduling:
  Scheduler: slurm
  SlurmQueues:
    - Name: compute1
      CapacityType: ONDEMAND
      ComputeSettings:
        LocalStorage:
          RootVolume:
            Size: 1024
          EphemeralVolume:
            MountDir: /local_storage
      ComputeResources:
        - Efa:
            Enabled: true
          InstanceType: trn1.32xlarge
          MaxCount: 16
          MinCount: 0
          Name: queue1-i1
      Networking:
        SubnetIds:
          - subnet-<PRIVATE SUBNET ID>
        PlacementGroup:
          Enabled: true
      CustomActions:
        OnNodeConfigured:
          Script: s3://neuron-s3/pcluster/post-install-scripts/neuron-installation/v2.4.0/al2/pt/install_neuron.sh
      Iam:
        S3Access:
          - BucketName: neuron-s3
            EnableWriteAccess: false
SharedStorage:
- EfsSettings:
    ProvisionedThroughput: 1024
    ThroughputMode: provisioned
  MountDir: /efs
  Name: neuron
  StorageType: Efs
```

If your OS choice is Ubuntu 20.04, here is an example YAML:


```
Region: <YOUR REGION> # i.e., us-west-2
Image:
  Os: ubuntu2004
HeadNode:
  InstanceType: c5.4xlarge
  Networking:
    SubnetId: subnet-<PUBLIC SUBNET ID>
  Ssh:
    KeyName: <KEY NAME WITHOUT .PEM>
  LocalStorage:
    RootVolume:
      Size: 1024
  CustomActions:
    OnNodeConfigured:
      Script: s3://neuron-s3/pcluster/post-install-scripts/neuron-installation/v2.4.0/u20/pt/install_neuron.sh
  Iam:
    S3Access:
       - BucketName: neuron-s3
         EnableWriteAccess: false
Scheduling:
  Scheduler: slurm
  SlurmQueues:
    - Name: compute1
      CapacityType: ONDEMAND
      ComputeSettings:
        LocalStorage:
          RootVolume:
            Size: 1024
          EphemeralVolume:
            MountDir: /local_storage
      ComputeResources:
        - Efa:
            Enabled: true
          InstanceType: trn1.32xlarge
          MaxCount: 16
          MinCount: 0
          Name: queue1-i1
      Networking:
        SubnetIds:
          - subnet-<PRIVATE SUBNET ID>
        PlacementGroup:
          Enabled: true
      CustomActions:
        OnNodeConfigured:
          Script: s3://neuron-s3/pcluster/post-install-scripts/neuron-installation/v2.4.0/u20/pt/install_neuron.sh
      Iam:
        S3Access:
          - BucketName: neuron-s3
            EnableWriteAccess: false
SharedStorage:
- EfsSettings:
    ProvisionedThroughput: 1024
    ThroughputMode: provisioned
  MountDir: /efs
  Name: neuron
  StorageType: Efs
  ```


The YAML file above will create a ParallelCluster with a c5.4xlarge head node, and 16 trn1.32xl compute nodes. All `MaxCount` trn1 nodes are in the same queue. In case you need to isolate compute nodes with different queues, simply append another instanceType designation to the current instanceType, and designate `MaxCount` for each queue, for example, `InstanceType` section would be become:

```
InstanceType: trn1.32xlarge
MaxCount: 8
MinCount: 0
Name: queue-0
InstanceType: trn1.32xlarge
MaxCount: 8
MinCount: 0
Name: queue-1
```

So now you have two queues, each queue is designated to a number of trn1 compute nodes. An unique feature for trn1.32xlarge instance is the EFA interfaces built for high performance/low latency network data transfer. This is indicated by:

```
- Efa:
    Enabled: true
```

If you are using trn1.2xl instance, this feature is not enabled, and in which case, you don’t need such designation.

You also need to designate an EC2 private key . This is indicated by the following line in launch.yaml:

```
Ssh:
  KeyName: <KEY NAME WITHOUT .PEM>
```

2. In the virtual environment where you installed AWS ParallelCluster API, run the following command:

```
pcluster create-cluster --cluster-configuration launch.yaml \
--cluster-name My-ParallelCluster-Trn1 \
--suppress-validators type:ComputeResourceLaunchTemplateValidator \
\
```
Where

`cluster-configuration` is the path to YAML file

`cluster-name` is the name of your cluster

`suppress-validators` is used here to generalize this command so it will not run into error triggered by tagging policies, if any.

This will create a ParallelCluster in your AWS account, and you may inspect the progress in AWS CloudFormation console. 

You may also check cluster status using `pcluster` command, for example: 

`pcluster describe-cluster -r us-west-2 -n My-ParallelCluster-Trn1`

3. During the cluster creation process, post-install actions now takes place automatically via `CustomActions` indicated in `launch.yaml` to configure the head node and any static compute nodes (`MinCount` > 0). `CustomActions` will install Neuron drivers and runtime, EFA drivers, and Neuron tools. 

4. After post-installation actions are complete, the ParallelCluster environment is properly configured to run SLURM jobs. Rerun `pcluster describe-cluster ...` command above to see the head node IP address, such that you may SSH into it for the [next part of the tutorial](../jobs/dp-bert-launch-job.md) where you would launch a training job.