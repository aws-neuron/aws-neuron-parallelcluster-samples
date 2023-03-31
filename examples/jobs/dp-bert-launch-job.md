# Launch training job
Once the cluster is successfully created and the Neuron packages are installed, please ssh into the head node to begin the training example. As an example here, we will use [Phase 1 BERT-Large pretraining](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/bert.html#phase-1-bert-large-pretrainingg) as the example job to submit to the cluster using SLURM job scheduler. You will be running commands from the head node. On ParallelCluster, the home directory is shared between the head node and compute nodes so files in the home directory are visible to worker nodes.

For all the commands below, make sure you are in the virtual environment created during setup before you run the commands. SLURM job scheduler will automatically activate the virtual environment when the training script is run on the worker nodes.

```
source ~/aws_neuron_venv_pytorch/bin/activate
```

## Download training scripts
Clone the AWS Neuron Samples to obtain the Python-based training script `dp_bert_large_hf_pretrain_hdf5.py`, the SLURM shell scripts, and the public Python-based implementation of the Layerwise Adaptive Moments (LAMB) optimizer `lamb.py`. Install the requirements. 
```
cd ~/
git clone https://github.com/aws-neuron/aws-neuron-samples.git
python3 -m pip install -r ~/aws-neuron-samples/torch-neuronx/training/dp_bert_hf_pretrain/requirements.txt
```

The pretraining scripts are stored in the home directory of the head node which is shared with the compute nodes via NFS. As you will see later, when launching a training job using SLURM, the job will run a script on each specified compute node.

## Download data set

Download the tokenized and sharded dataset files needed for this tutorial in the home directory that is shared with the compute nodes via NFS:

```
mkdir -p ~/examples_datasets/
pushd ~/examples_datasets/
aws s3 cp s3://neuron-s3/training_datasets/bert_pretrain_wikicorpus_tokenized_hdf5/bert_pretrain_wikicorpus_tokenized_hdf5_seqlen128.tar .  --no-sign-request
tar -xf bert_pretrain_wikicorpus_tokenized_hdf5_seqlen128.tar
rm bert_pretrain_wikicorpus_tokenized_hdf5_seqlen128.tar
popd
```

## Compile model
The `run_dp_bert_large_hf_pretrain_bf16_s128.sh` script will be invoked by SLURM commands running on the head node. To do compilation, run the following command from `~/aws-neuron-samples/torch-neuronx/training/dp_bert_hf_pretrain` directory on the head node:

```
cd ~/aws-neuron-samples/torch-neuronx/training/dp_bert_hf_pretrain
sbatch --exclusive --nodes=16 --wrap "srun neuron_parallel_compile ./run_dp_bert_large_hf_pretrain_bf16_s128_lamb.sh"
```

The job id will be displayed by sbatch. You can monitor the results of the compilation job by inspecting the file `slurm_<job id>.out` file generated in `~/aws-neuron-samples/torch-neuronx/training/dp_bert_hf_pretrain`.

## Launch training
After the compilation job is finished, start the actual pretraining:

```
cd ~/aws-neuron-samples/torch-neuronx/training/dp_bert_hf_pretrain
sbatch  --exclusive --nodes=16 --wrap "srun ./run_dp_bert_large_hf_pretrain_bf16_s128_lamb.sh"
```

Again, the job id will be displayed by sbatch and you can follow the training by inspecting the file `slurm_<job id>.out` file generated in `~/examples/dp_bert_hf_pretrain`.

### Cluster scalability

In a Trn1 cluster, multiple interconnected Trn1 instances run a large model training workload in parallel and reduce total computation time, or time to convergence. There are two measures of scalability of a cluster: strong scaling and weak scaling. Typically, for model training, the need is to speed up training run, because usage cost is determined by sample throughput for rounds of gradient updates. This means strong scaling is an important measure of scalability for model training. Strong scaling refers to the scenario where the total problem size stays the same as the number of processors increases. In evaluating strong scaling, or the impact of parallelization, we want to keep global batch size same and see how much time it takes to convergence. In such scenario, we need to adjust gradient accumulation micro-step according to number of compute nodes. This is achieved with the following in the downloaded training shell script `run_dp_bert_large_hf_pretrain_bf16_s128_lamb.sh`:

```
GRAD_ACCUM_USTEPS=$(($GRAD_ACCUM_USTEPS/$WORLD_SIZE_JOB))
```

The SLURM shell script automatically adjust the gradient accumulation microsteps to keep the global batch size for phase 1 at 65536 with LAMB optimizer (strong scaling).

On the other hand, if the interest is to evaluate how much more workloads can be executed at a fixed time by adding more nodes, then use weak scaling to measure scalability. In weak scaling, the problem size increases at the same rate as number of processors, thereby keeping amount of work per processor the same. To see performance for larger global batch size (weak scaling), please comment out the line above. Doing so would keep number of steps for gradient accumulation constant with a default value (i.e., 128) provided in the training script `run_dp_bert_large_hf_pretrain_bf16_s128_lamb.sh`.

## Tips

Some useful SLURM commands are `sinfo`,  `squeue` and `scontrol`. `sinfo` command displays information about SLURM node names and partitions. `squeue` command provides information about job queues currently running in the Slurm schedule. SLURM will generate a log file `slurm-XXXXXX.out`. You may then use `tail -f slurm-XXXXXX.out`, to inspect the job summary. `scontrol show node <COMPUTE_NODE_NAME>` can show more information such as node state, power consumption, and more.


## Known issues/limitations

- The current setup supports up to 128 nodes BERT pretraining with LAMB optmizer when using strong scaling.

## Troubleshooting guide

See [Troubleshooting Guide for AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/troubleshooting-v3.html) for more details and fixes to common issues.
