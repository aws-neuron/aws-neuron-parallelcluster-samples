# Launch GPT-3 training job
Once the cluster is successfully created and the Neuron packages are installed, please ssh into the head node to begin the training example. As an example here, we expand the [GPT3 pretraining with Megatron-LM](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/megatron_lm_gpt.html) tutorial to run on a cluster using SLURM job scheduler. 

You will be running commands from the head node of the cluster. On ParallelCluster, the home directory is shared between the head node and compute nodes so files in the home directory are visible to worker nodes. 

You may inspect the ParallelCluster with the following command:

```sh
sinfo
```

and expect to see the output which indicates your node's status. An example is:

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST  
compute1*    up   infinite      16  alloc compute1-dy-queue1-i1-[1-16] 
```

For all the commands below, make sure you are in the virtual environment created during setup before you run the commands. SLURM job scheduler will automatically activate the virtual environment when the training script is run on the worker nodes.

```sh
source ~/aws_neuron_venv_pytorch/bin/activate
```

Use this virtual environment in the head node for the steps below.

## Download preprocessed training dataset

In this tutorial, we use the Wikipedia dataset as an example to demonstrate training at scale.
You can download the vocabulary file, the merge table file, and the preprocessed Wikipedia dataset with the following commands:

```sh
export DATA_DIR=~/examples_datasets/gpt2

mkdir -p ${DATA_DIR} && cd ${DATA_DIR}

wget https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-vocab.json
wget https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-merges.txt
aws s3 cp s3://neuron-s3/training_datasets/gpt/wikipedia/my-gpt2_text_document.bin .  --no-sign-request
aws s3 cp s3://neuron-s3/training_datasets/gpt/wikipedia/my-gpt2_text_document.idx .  --no-sign-request
aws s3 cp s3://neuron-s3/training_datasets/gpt/wikipedia/license.txt .  --no-sign-request
```

To prepare your own dataset from scratch, please follow the steps in [Preparing Wikipedia Dataset from Scratch](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/megatron_lm_gpt.html#preparing-wikipedia-dataset-from-scratch)

## Download training scripts

In this section we will download the training scripts and build the necessary dependencies.

First, install Python3 development package needed to build the data helpers tools. If you are on Amazon Linux, do:

```sh
sudo yum install -y python3-devel
```

If you are on Ubuntu, do:

```sh
sudo apt install -y python3-dev
```

Next, clone the AWS Neuron Reference for Megatron-LM package, install dependencies and build helpers tool:

```sh
cd ~/
git clone https://github.com/aws-neuron/aws-neuron-reference-for-megatron-lm.git
pip install pybind11 regex
pushd .
cd aws-neuron-reference-for-megatron-lm/megatron/data/
make
popd
```

There will be an `~/aws-neuron-reference-for-megatron-lm` directory from which you will be running the SLURM commands. The shell scripts needed to run the tutorial are in `~/aws-neuron-reference-for-megatron-lm/examples` directory. 

## GPT-3 6.7B training configuration

In this example, we are going to run a pretraining job for the GPT-3 6.7B model using the following model configuration:

```sh
Hidden size = 4096
Sequence len = 2048
Num heads = 32
Num layers = 32
Microbatch = 1
Gradient accumulation microsteps = 64
```
The distributed configuration is tensor parallel degree 32, pipeline parallel degree 1, and data parallel degree 16 if using 16 nodes. The global batch size is 1024.

## GPT-3 6.7B training script

The training shell script pretrain_gpt3_6.7B_32layers_bf16_bs1024_slurm.sh will be launched by SLURM on each worker node, where it prepares the environment and invokes torchrun to launch the Python script pretrain_gpt.py on 32 workers.  The environment settings are:

- Enable Elastic Fabric Adapter for higher networking performance
- Mark all parameter transfers as static to enable runtime optimizations for wrapped torch.nn modules
- Enables custom lowering for Softmax operation to enable compiler optimizations and improve GPT performance
- Cast training to BF16 and enable stochastic rounding
- Increase Neuron RT execution timeout in case slow compilation causes Neuron RT to wait longer than default timeout
- Ensure enough framework threads to execute collective compute operations to prevent hang
- Separate NeuronCache dir per node, workaround limitation to file locking on NFS
- Run fewer steps and redirect TensorBoard logging when extract graphs only (during neuron_parallel_compile)

The training shell script uses `torchrun` to run multiple pretrain_gpt.py processes, with world size, node rank, and master address extracted from SLURM node information.

```sh
MASTER_ADDR=(`scontrol show hostnames $SLURM_JOB_NODELIST`)
WORLD_SIZE_JOB=$SLURM_NTASKS
RANK_NODE=$SLURM_NODEID
DISTRIBUTED_ARGS="--nproc_per_node 32 --nnodes $WORLD_SIZE_JOB --node_rank $RANK_NODE --master_addr $MASTER_ADDR --master_port 2022"
torchrun $DISTRIBUTED_ARGS pretrain_gpt.py \
     ...(options)...
```

The Python script pretrain_gpt.py calls the Megatron pretraining API with the Megatron GPT model builder, the loss and forward functions, and the dataset provider. It also sets the default compiler flag to model-type transformer for improved transformer support in compiler.

## Precompiling the training graphs

Precompiling the training graphs is an optional step to reduce just-in-time graph compilations during training. This is done using the [Neuron Parallel Compile tool](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/api-reference-guide/training/pytorch-neuron-parallel-compile.html), which extracts the graphs used during training and perform multiple graph compilations in parallel.

To precompile the training graphs, in `~/aws-neuron-reference-for-megatron-lm` directory, issue the following command to submit a SLURM job:

```sh
cd ~/aws-neuron-reference-for-megatron-lm
sbatch --exclusive -N 16 --wrap "srun neuron_parallel_compile ./examples/pretrain_gpt3_6.7B_32layers_bf16_bs1024_slurm.sh"
```

You can also use the SLURM script `examples/pretrain_gpt3_6.7B_compile.slurm` provided for convenience:

```sh
cd ~/aws-neuron-reference-for-megatron-lm
sbatch ./examples/pretrain_gpt3_6.7B_compile.slurm
```

Note that currently each node has it's own separate cache in `~/neuron_cache/gpt/<node name>/neuron-compile-cache` to workaround a known NFS file-locking limitation. This is configured by the following line in the `./examples/pretrain_gpt3_6.7B_32layers_bf16_bs1024_slurm.sh` script.

```sh
export NEURON_CC_FLAGS="--cache_dir=$HOME/neuron_cache/gpt/`hostname`"
```
If the cluster size is larger than 16, use `--nodelist=<node prefix[range]>` to limit the nodes used during precompilation and actual training run to ensure the workers on each node reads from the correct cache. The nodelist must match between precompilation and the actual run.

The job id will be displayed by `squeue` command. You can monitor the results of the compilation job by inspecting the file `slurm-<job id>.out`file generated in `~/aws-neuron-reference-for-megatron-lm` directory. To follow the progress of this SLURM job, you may stream the SLURM output file in real time:

```sh
tail -f slurm-<job id>.out
```
Note that there are many processes across many instances (nodes) running in parallel, and all the outputs are combined into the `slurm-<job id>.out` file. You can examine individual node's log by looking into `run_log_gpt3_6.7B_32layers_bf16.<node id>.16.txt` file. 

The graph extraction sets NEURON_EXTRACT_GRAPHS_ONLY environment variable which cause the graph execution to execute empty graphs with zero outputs. The zero outputs cause execution results to be random, so the TensorBoard log is redirected to `/tmp/parallel_compile_ignored_tb_output`to enable clean TB log of the actual run in the next section.

Currently, the total compilation time for GPT-3 6.7B example is about 30 minutes. When each node is finished compilation, you should see the following at the end of each node's log:

```sh
2023-05-08 20:14:50.000983: INFO ||PARALLEL_COMPILE||: Total graphs: 26
2023-05-08 20:14:50.000983: INFO ||PARALLEL_COMPILE||: Total successful compilations: 26
2023-05-08 20:14:50.000983: INFO ||PARALLEL_COMPILE||: Total failed compilations: 0
```

## Launch training script

Before or after the precompilation job is finished, submit the actual pretraining job by running the following slurm command:

```
cd ~/aws-neuron-reference-for-megatron-lm
sbatch --exclusive -N 16 --wrap "srun ./examples/pretrain_gpt3_6.7B_32layers_bf16_bs1024_slurm.sh"
```

You can also use the SLURM script `examples/pretrain_gpt3_6.7B.slurm` provided for convenience:

```sh
cd ~/aws-neuron-reference-for-megatron-lm
sbatch ./examples/pretrain_gpt3_6.7B.slurm
```

As mentioned above, If the cluster size is larger than 16, use `--nodelist=<node prefix[range]>` to limit the nodes used during precompilation and actual training run to ensure the workers on each node reads from the correct cache. The nodelist must match between precompilation and the actual run.

You can also run the script with more or less number of nodes as long as the cluster size supports the node count. Further more, note that the global batch size (equal to gradient accumulation count times number of nodes) will change when the number of nodes is changed.

If the submission is done before precompilation job is finished, the new submitted job will be queued in the SLURM job queue. You can use `squeue` to see running and queued jobs.

Again, the job id will be displayed by `squeue` command and you can follow the training by inspecting the file `slurm-<job id>.out` file. You can examine individual node's log by looking into `run_log_gpt3_6.7B_32layers_bf16.<node id>.16.txt` file.

After an initial startup, you should see lines like the following that indicate training progress (iteration, loss, elapsed time, throughput, etc).

```sh
 iteration       64/  143051 | consumed samples:        65536 | elapsed time per iteration (ms): 7313.1 | learning rate: 4.956E-05 | global batch size:  1024 | lm loss: 7.281250E+00 | grad norm: 3.562 | throughput: 140.022 |

```
## View TensorBoard trace

You can examine the TensorBoard trace by ssh into the headnode with `-L 6006:localhost:6006' option, then:

```sh
source ~/aws_neuron_venv_pytorch/bin/activate
cd ~/aws-neuron-reference-for-megatron-lm
tensorboard --logdir ./tb_gpt3_6.7B_32layers_bf16
```
On the host from where you ssh into the headnode, you can use a browser to go to `http://localhost:6006/` to view TensorBoard.

## View NeuronCore activities

To inspect NeuronCore activities in the compute node, you can SSH from head node into any compute node, for example:

```sh
ssh compute1-dy-queue1-i1-1
```
and then run `neuron-top` in the compute node to see NeuronCore activities while training in happening

## Tips

Some useful SLURM commands are `sinfo`,  `squeue` and `scontrol`. `sinfo` command displays information about SLURM node names and partitions. `squeue` command provides information about job queues currently running in the Slurm schedule. SLURM will generate a log file `slurm-XXXXXX.out`. You may then use `tail -f slurm-XXXXXX.out`, to inspect the job summary. `scontrol show node <COMPUTE_NODE_NAME>` can show more information such as node state, power consumption, and more.


## Known issues/limitations

- "Failed accept4: Too many open files" error and the solution from [here](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/megatron_lm_gpt.html?highlight=megatron%20lm#failed-accept4-too-many-open-files)
- "cannot import name'helppers' from 'megatron.data' error and the solution from [here](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/megatron_lm_gpt.html?highlight=megatron%20lm#error-cannot-import-name-helpers-from-megatron-data)

## Troubleshooting guide

See [Troubleshooting Guide for AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/troubleshooting-v3.html) for more details and fixes to common issues.
