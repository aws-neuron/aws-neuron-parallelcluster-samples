# Create ParallelCluster

1. Once your VPC, ParallelCluster python package, and key pair are set up, you are ready to create a ParallelCluster. Copy the following content into a launch.yaml file in your local desktop where AWS ParallelCluster CLI is installed. Here is an example YAML file content:

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
      Script: s3://neuron-s3/pcluster/post-install-scripts/neuron-installation/v<SDK RELEASE VER>/u20/pt/install_neuron.sh
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
          InstanceType: trn1n.32xlarge
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
          Script: s3://neuron-s3/pcluster/post-install-scripts/neuron-installation/v<SDK RELEASE VER>/u20/pt/install_neuron.sh
      Iam:
        S3Access:
          - BucketName: neuron-s3
            EnableWriteAccess: false
SharedStorage:
- FsxLustreSettings:
    DeploymentType: SCRATCH_2
    StorageCapacity: 1200
  MountDir: /fsx
  Name: pclusterfsx
  StorageType: FsxLustre
```

Please replace the placeholders `<PUBLIC SUBNET ID>`, `<PRIVATE SUBNET ID>`, `<KEY NAME WITHOUT .PEM>` and `<SDK RELEASE VER>` with appropriate values.

For example, to use Neuron SDK release version 2.13.0, replace `<SDK RELEASE VER>` with `2.13.0` in the two `CustomActions: OnNodeConfigured: Script` fields (make sure `2.13.0` is preceded by the letter `v`).

```
  CustomActions:
    OnNodeConfigured:
      Script: s3://neuron-s3/pcluster/post-install-scripts/neuron-installation/v2.13.0/u20/pt/install_neuron.sh
```
The `<PUBLIC SUBNET ID>`and `<PRIVATE SUBNET ID>`values are obtained following the [VPC creation guide](./examples/general/network/vpc-subnet-setup.md). 

The `<KEY NAME WITHOUT .PEM>` is obtained following [key pair setup](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html#having-ec2-create-your-key-pair)

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

2. In the virtual environment where you installed AWS ParallelCluster API, run the following command (assuming you have saved the configurations above in `configuration.yaml`):

```
pcluster create-cluster --cluster-configuration configuration.yaml -n My-PCluster-Trn1 
```
Where

`cluster-configuration` is the YAML file

This will create a ParallelCluster in your AWS account, and you may inspect the progress in AWS CloudFormation console. 

You may also check cluster status using `pcluster` command, for example: 

`pcluster describe-cluster -r us-west-2 -n My-PCluster-Trn1`

3. During the cluster creation process, post-install actions now takes place automatically via `CustomActions` indicated in `configuration.yaml` to configure the head node and any static compute nodes (`MinCount` > 0). `CustomActions` will install Neuron drivers and runtime, EFA drivers, and Neuron tools. 

4. After post-installation actions are complete, the ParallelCluster environment is properly configured to run SLURM jobs. Rerun `pcluster describe-cluster ...` command above to see the head node IP address, such that you may SSH into it for the [next part of the tutorial](../jobs/dp-bert-launch-job.md) where you would launch a training job.

## Custom script update

To make it easier to upgrade to newer versions, you can copy the `install_neuron.sh` to your own s3 bucket s3://``<BUCKET NAME>`` and modify `CustomActions: OnNodeConfigured: Script` and `CustomActions: OnNodeConfigured: Iam: S3Access:` fields (2 sets).

```
  CustomActions:
    OnNodeConfigured:
      Script: s3://<BUCKET NAME>/install_neuron.sh
      Iam:
        S3Access:
          - BucketName: <BUCKET NAME>
            EnableWriteAccess: false
```

Then create the cluster as above.

When you are ready to update your own s3 bucket's copy after a new Neuron SDK release, first stop your ParallelCluster from the system and environment where ParallelCluster API was installed:
```
pcluster update-compute-fleet --cluster-name <YOUR_CLUSTER_NAME> -r <YOUR_REGION> --status STOP_REQUESTED
```
Next copy the new version to your own s3 bucket, then restart the ParallelCluster.
```
pcluster update-compute-fleet --cluster-name <YOUR_CLUSTER_NAME> -r <YOUR_REGION --status START_REQUESTED
```
The above steps only update the runtime and tools packages on the compute fleet. Since the Python environment is on the head node, and the head node is not updated with the above instructions, you will need update your Python environment by manually running the same `install_neuron.sh` script on the head node:

```
# log into the head node, and make sure you have credentials to access s3://<BUCKET NAME>
cd ~/
aws s3 cp s3://<BUCKET NAME>/install_neuron.sh .
source install_neuron.sh
```

## Updating configuration.yaml

You can change the configuration.yaml to update for example compute resources. First stop your ParallelCluster from the system and environment where ParallelCluster API was installed:
```
pcluster update-compute-fleet --cluster-name <YOUR_CLUSTER_NAME> -r <YOUR_REGION> --status STOP_REQUESTED
```
Update the cluster using the new configuration YAML version (assuming it is named `new_configuration.yaml`):
```
pcluster update-cluster -c new_configuration.yaml --cluster-name <YOUR_CLUSTER_NAME> -r YOUR_REGION
```
Then restart the ParallelCluster.
```
pcluster update-compute-fleet --cluster-name <YOUR_CLUSTER_NAME> -r <YOUR_REGION --status START_REQUESTED
```


## Known issues

- The default entries in `/etc/hosts` sometimes does not map to the correct ip address (Trn1 has 8 network interfaces) resulting in potential connection errors when running multi-instance jobs. The default `install_neuron.sh` provided in the above sample YAML file has the workaround along with the neuron package installations. If you prefer to not include the installations and just patch this issue you can include the following as part of your custom OnNodeConfigured script for your Trn1 compute nodes or set it separately after worker launch but before launching any multi-instance jobs. 

```
sudo sed -i "/$HOSTNAME/d" /etc/hosts
```
This removes the hostname ip address mapping. This mapping is not generally needed for normal ParallelCluster operation or Training jobs using Slurm.
