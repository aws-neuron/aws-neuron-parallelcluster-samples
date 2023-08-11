# Launch a GPT-3 pretraining job using neuronx-nemo-megatron

This tutorial explains how to run GPT-3 pretraining jobs with AWS EC2 trn1.32xl instances using [neuronx-nemo-megatron](https://github.com/aws-neuron/neuronx-nemo-megatron) and [AWS ParallelCluster](https://aws.amazon.com/hpc/parallelcluster/).

neuronx-nemo-megatron (also known as "AWS Neuron Reference for NeMo Megatron") includes modified versions of the open-source packages [NeMo](https://github.com/NVIDIA/NeMo) and [Apex](https://github.com/NVIDIA/apex) that have been adapted for use with AWS Neuron and AWS EC2 Trn1 instances. neuronx-nemo-megatron allows for pretraining models with hundreds of billions of parameters across thousands of Trainium accelerators, and enables advanced training capabilities such as 3D parallelism, sequence parallelism, and activation checkpointing.

## Prerequisites
Before proceeding with this tutorial, please follow [these instructions](https://github.com/aws-neuron/aws-neuron-parallelcluster-samples#train-a-model-on-aws-trn1-parallelcluster) to create a ParallelCluster consisting of 1 or more trn1.32xl or trn1n.32xl nodes. ParallelCluster automates the creation of trn1 clusters, and provides the SLURM job management system for scheduling and managing distributed training jobs. Please note that the home directory on your ParallelCluster head node will be shared with all of the worker nodes via NFS.

## Install neuronx-nemo-megatron

With your trn1 ParallelCluster in place, begin by logging into the head node of your cluster using SSH. To provide access to TensorBoard (required in a later step), please make sure that you enable port forwarding for TCP port 6006 when you login, ex:
```
ssh -i YOUR_KEY.pem ubuntu@HEAD_NODE_IP_ADDRESS -L 6006:127.0.0.1:6006
```

Once logged into the head node, activate the provided PyTorch Neuron virtual environment that was created when you set up your ParallelCluster. **Note**: if your PyTorch Neuron environment is lower than Neuron 2.11, please refer to the [Neuron documentation](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/setup/pytorch-update.html#pytorch-neuronx-update) for instructions on updating to Neuron 2.11 or later.
```
cd ~
source ./aws_neuron_venv_pytorch/bin/activate
```

Next, clone the neuronx-nemo-megatron repo to the head node:
```
cd ~
git clone https://github.com/aws-neuron/neuronx-nemo-megatron.git
cd neuronx-nemo-megatron
```

Install the `wheel` Python package and run the build script to create the neuronx-nemo-megatron wheels:
```
pip3 install wheel
./build.sh
```

Install the neuronx-nemo-megatron packages and dependencies in your virtual environment:
```
pip3 install ./build/*.whl
pip3 install -r requirements.txt torch==1.13.1 protobuf==3.20.3
```

Build the Megatron helper module
```
cd ~
python3 -c "from nemo.collections.nlp.data.language_modeling.megatron.dataset_utils import compile_helper; \
compile_helper()"
```

The above utility will help make this file : ```nemo.collections.nlp.data.language_modeling.megatron.dataset_utils``` and below is the expected output (You can ignore the error) 
```
2023-Aug-17 22:53:01.0674 47940:47940 ERROR  TDRV:tdrv_get_dev_info                       No neuron device available
[NeMo W 2023-08-17 22:53:03 optimizers:67] Could not import distributed_fused_adam optimizer from Apex
[NeMo W 2023-08-17 22:53:04 experimental:27] Module <class 'nemo.collections.nlp.data.language_modeling.megatron.megatron_batch_samplers.MegatronPretrainingRandomBatchSampler'> is experimental, not ready for production and is not fully supported. Use at your own risk.
```

## Download GPT dataset
This tutorial makes use of a preprocessed Wikipedia dataset that is stored in S3. The dataset can be downloaded to your cluster by running the following commands on the head node:
```sh
export DATA_DIR=~/examples_datasets/gpt2
mkdir -p ${DATA_DIR} && cd ${DATA_DIR}
wget https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-vocab.json
wget https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-merges.txt
aws s3 cp s3://neuron-s3/training_datasets/gpt/wikipedia/my-gpt2_text_document.bin .  --no-sign-request
aws s3 cp s3://neuron-s3/training_datasets/gpt/wikipedia/my-gpt2_text_document.idx .  --no-sign-request
aws s3 cp s3://neuron-s3/training_datasets/gpt/wikipedia/license.txt .  --no-sign-request
```

## GPT-3 training configurations
We tested with the following model sizes: 23B, 46B, 175B
### GPT-3 23B
- Model configuration
    - Attention heads: 64
    - Layers: 28
    - Sequence length: 2048
    - Hidden size: 8192
    - Hidden FFN size: 32768
    - Microbatch size: 1
    - Global batch size: 32 * number_of_nodes

- Distributed training configuration
    - Number of nodes: 4
    - Tensor parallel degree: 8
    - Pipeline parallel degree: 4
    - Data parallel degree: 4

### GPT-3 46B
- Model configuration
    - Attention heads: 64
    - Layers: 56
    - Sequence length: 2048
    - Hidden size: 8192
    - Hidden FFN size: 32768
    - Microbatch size: 1
    - Global batch size: 32 * number_of_nodes

- Distributed training configuration
    - Number of nodes: 8
    - Tensor parallel degree: 8
    - Pipeline parallel degree: 8
    - Data parallel degree: 4

### GPT-3 175B
- Model configuration
    - Attention heads: 96
    - Layers: 96
    - Sequence length: 2048
    - Hidden size: 12288
    - Hidden FFN size: 49152
    - Microbatch size: 1
    - Global batch size: 32 * number_of_nodes

- Distributed training configuration
    - Number of nodes: 8
    - Tensor parallel degree: 32
    - Pipeline parallel degree: 8
    - Data parallel degree: 1

## Pre-compile the model
By default, PyTorch Neuron uses a just in time (JIT) compilation flow that sequentially compiles all of the neural network compute graphs as they are encountered during a training job. The compiled graphs are cached in a local compiler cache so that subsequent training jobs can leverage the compiled graphs and avoid compilation (so long as the graph signatures and Neuron version have not changed).

An alternative to the JIT flow is to use the included [neuron_parallel_compile](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/api-reference-guide/training/pytorch-neuron-parallel-compile.html?highlight=neuron_parallel_compile) command to perform ahead of time (AOT) compilation. In the AOT compilation flow, the compute graphs are first identified and extracted during a short simulated training run, and the extracted graphs are then compiled and cached using parallel compilation, which is considerably faster than the JIT flow.

Run the following commands to launch an AOT pre-compilation job on your ParallelCluster:
```
cd ~/neuronx-nemo-megatron/nemo/examples/nlp/language_modeling
sbatch --nodes 4 compile.slurm ./gpt_23b.sh
```
For the 46B and 175B the `--nodes 8` would be used instead of 4.

Once you have launched the precompilation job, run the `squeue` command to view the SLURM job queue on your cluster. If you have not recently run a job on your cluster, it may take 4-5 minutes for the requested trn1.32xlarge nodes to be launched and initialized. Once the job is running, `squeue` should show output similar to the following:
```
    JOBID  PARTITION  NAME           USER    ST  TIME  NODES NODELIST(REASON)
    10     compute1   compile.slurm  ubuntu  R   5:11  4     compute1-dy-queue1-i1-[1-4]
```

You can view the output of the precompilation job by examining the file named `slurm-compile.slurm-ZZ.out` where ZZ represents the JOBID of your job in the `squeue` output, above. Ex:
```
tail -f slurm-compile.slurm-10.out
```

Once the precompilation job is complete, you should see a message similar to the following in the logs:
```
2023-06-11 23:04:08.000738: INFO ||PARALLEL_COMPILE||: Total graphs: 22
2023-06-11 23:04:08.000738: INFO ||PARALLEL_COMPILE||: Total successful compilations: 22
2023-06-11 23:04:08.000738: INFO ||PARALLEL_COMPILE||: Total failed compilations: 0
```

At this point, you can press `CTRL-C` to exit the tail command.

## Launch a pretraining job
The GPT-3 pretraining job can be launched in the same manner as the precompilation job described above. In this case, we change the SLURM script from `compile.slurm` to `run.slurm`, but the other parameters remain the same:
```
cd ~/neuronx-nemo-megatron/nemo/examples/nlp/language_modeling
sbatch --nodes 4 run.slurm ./gpt_23b.sh
```
For the 46B and 175B the `--nodes 8` would be used instead of 4, like the compile above.


As outlined above, you can again use the `squeue` command to view the job queue. Once you see that your pretraining job is running, you can view the output of the training job by examining the file named `slurm-run.slurm-ZZ.out` where ZZ represents the JOBID of your job:
```
tail -f slurm-run.slurm-11.out
```

Once the model is loaded onto the Trainium accelerators and training has commenced, you will begin to see output indicating the job progress:
```
Epoch 0:   0%|          | 189/301501 [59:12<1573:03:24, 18.79s/it, loss=7.75, v_num=3-16, reduced_train_loss=7.560, global_step=188.0, consumed_samples=24064.0]
Epoch 0:   0%|          | 190/301501 [59:30<1572:41:13, 18.79s/it, loss=7.74, v_num=3-16, reduced_train_loss=7.560, global_step=189.0, consumed_samples=24192.0]
Epoch 0:   0%|          | 191/301501 [59:48<1572:21:28, 18.79s/it, loss=7.73, v_num=3-16, reduced_train_loss=7.910, global_step=190.0, consumed_samples=24320.0]
```

Note : Checkpointing is enabled by default and all your checkpoints are getting stored by default in the nemo_experiments directory. This can cause your host memory to fill up and can cause disk space issues, to change the directory to a file system path, you can introduce another parameter in test.sh like below : 
```
vi ~/neuronx-nemo-megatron/nemo/examples/nlp/language_modeling/test.sh 

#Add the below param : 

exp_manager.exp_dir: PATH_TO_FILE_SYSTEM_DIRECTORY
```

## Monitor training
### TensorBoard
In addition to the text-based job monitoring described in the previous section, you can also use standard tools such as TensorBoard to monitor training job progress. To view an ongoing training job in TensorBoard, you first need to identify the experiment directory associated with your ongoing job. This will typically be the most recently created directory under `~/neuronx-nemo-megatron/nemo/examples/nlp/language_modeling/nemo_experiments/megatron_gpt`. Once you have identifed the directory, `cd` into it, and then launch TensorBoard:
```
cd ~/neuronx-nemo-megatron/nemo/examples/nlp/language_modeling/nemo_experiments/megatron_gpt
ls -alt|head
# Identify the correct experiment directory in the
# output of the ls command, ex: 2023-06-10_00-22-42
cd YOUR_EXPERIMENT_DIR  # <- replace this with your experiment directory
tensorboard --logdir ./
```

With TensorBoard running, you can then view the TensorBoard dashboard by browsing to http://localhost:6006 on your local machine. If you cannot access TensorBoard at this address, please make sure that you have port-forwarded TCP port 6006 when SSH'ing into the head node, ex: `ssh -i YOUR_KEY.pem ubuntu@HEAD_NODE_IP_ADDRESS -L 6006:127.0.0.1:6006`

### neuron-top / neuron-monitor / neuron-ls
The [neuron-top](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/tools/neuron-sys-tools/neuron-top-user-guide.html?highlight=neuron-top) tool can be used to view useful information about NeuronCore utilization, vCPU and RAM utilization, and loaded graphs on a per-node basis. To use neuron-top during on ongoing training job, first SSH into one of your compute nodes from the head node, and then run `neuron-top`:
```
ssh compute1-dy-queue1-i1-1  # to determine which compute nodes are in use, run the squeue command
neuron-top
```

Similarly, once you are logged into one of the active compute nodes, you can also use other Neuron tools such as [neuron-monitor](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/tools/neuron-sys-tools/neuron-monitor-user-guide.html) and [neuron-ls](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/tools/neuron-sys-tools/neuron-monitor-user-guide.html) to capture performance/utilization statistics and to understand NeuronCore allocation.

## Known issues/limitations
* The initial release of neuronx-nemo-megatron supports GPT pretraining only. Model evaluation will be available in a future release.
* The Neuron compiler's modular flow (ex: `--enable-experimental-O1`) is not supported by this initial release of neuronx-nemo-megatron.
* neuronx-nemo-megatron currently requires pytorch-lightning v1.8.6
* Checkpointing has stability issues and could cause timeout issues, to disable checkpointing, follow the below steps : 
```
vi ~/neuronx-nemo-megatron/nemo/examples/nlp/language_modeling/test.sh

    # Change model.save_xser from True to False
    +model.save_xser=False \
    
    # Change exp_manager.create_checkpoint_callback from True to False
    exp_manager.create_checkpoint_callback=$CHECKPOINT_CALLBACK \
```

## Troubleshooting guide
See [Troubleshooting Guide for AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/troubleshooting-v3.html) for more details and fixes to common issues.
