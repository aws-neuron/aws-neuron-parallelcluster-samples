# Launch training job
Once the cluster is successfully created, please ssh into the head node to begin the training example. As an example here, we will use [Phase 1 BERT-Large pretraining](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/bert.html#phase-1-bert-large-pretrainingg) as the example job to submit to the cluster using SLURM job scheduler. You will be running commands from the head node.

First, on the headnode, setup the virtual Python environment in the home directory. The following is for setting up in Amazon Linux 2 OS: 

```
# Install Python venv and activate Python virtual environment to install
# Neuron pip packages.
python3 -m venv aws_neuron_venv_pytorch
source aws_neuron_venv_pytorch/bin/activate
python -m pip install -U pip

# Install packages from beta repos
python -m pip config set global.extra-index-url "https://pip.repos.neuron.amazonaws.com"

# Install Python packages - Transformers package is needed for BERT
python -m pip install torch-neuronx=="1.11.0.1.*" "neuronx-cc==2.*" transformers
```

On ParallelCluster, the home directory is shared between the head node and compute nodes, so this virtual environment is also visible from all compute nodes.

For all the commands below, make sure you are in the virtual environment that you have created above before you run the commands. SLURM job scheduler will automatically activate the virtual environment when the training script is run on the worker nodes.

```
source ~/aws_neuron_venv_pytorch/bin/activate
```

Next, download the Python-based training script `dp_bert_large_hf_pretrain_hdf5.py`, the SLURM shell script `dp_bert_large_hf_pretrain_hdf5.sh` and the requirements file into `~/examples/dp_bert_hf_pretrain` and install the requirements:
```
mkdir -p ~/examples/dp_bert_hf_pretrain
cd ~/examples/dp_bert_hf_pretrain
wget https://raw.githubusercontent.com/aws-neuron/aws-neuron-samples/master/torch-neuronx/training/dp_bert_hf_pretrain/run_dp_bert_large_hf_pretrain_bf16_s128.sh
chmod +x ./run_dp_bert_large_hf_pretrain_bf16_s128.sh
wget https://raw.githubusercontent.com/aws-neuron/aws-neuron-samples/master/torch-neuronx/training/dp_bert_hf_pretrain/dp_bert_large_hf_pretrain_hdf5.py
wget https://raw.githubusercontent.com/aws-neuron/aws-neuron-samples/master/torch-neuronx/training/dp_bert_hf_pretrain/requirements.txt
python3 -m pip install -r requirements.txt
```

The pretraining scripts will be stored in the home directory of the head node. Upon launching the job using SLURM, the job will run the script on each specified compute node.

Download the tokenized and sharded dataset files needed for this tutorial:

```
mkdir -p ~/examples_datasets/
pushd ~/examples_datasets/
aws s3 cp s3://neuron-s3/training_datasets/bert_pretrain_wikicorpus_tokenized_hdf5/bert_pretrain_wikicorpus_tokenized_hdf5_seqlen128.tar .  --no-sign-request
tar -xf bert_pretrain_wikicorpus_tokenized_hdf5_seqlen128.tar
rm bert_pretrain_wikicorpus_tokenized_hdf5_seqlen128.tar
popd
```

The `run_dp_bert_large_hf_pretrain_bf16_s128.sh` script will be invoked by SLURM commands running on the head node. To do compilation, run the following command from `~/examples/dp_bert_hf_pretrain` directory on the head node:

```
cd ~/examples/dp_bert_hf_pretrain
sbatch --exclusive --nodes=16 --wrap "srun neuron_parallel_compile ./run_dp_bert_large_hf_pretrain_bf16_s128.sh"
```

The job id will be displayed by sbatch. You can monitor the results of the compilation job by inspecting the file `slurm_<job id>.out` file generated in `~/examples/dp_bert_hf_pretrain`.


After the compilation job is finished, start the actual pretraining:

```
cd ~/examples/dp_bert_hf_pretrain
sbatch  --exclusive --nodes=16 --wrap "srun ./run_dp_bert_large_hf_pretrain_bf16_s128.sh"
```

Again, the job id will be displayed by sbatch and you can follow the training by inspecting the file `slurm_<job id>.out` file generated in `~/examples/dp_bert_hf_pretrain`.

The SLURM shell script automatically adjust the gradient accumulation microsteps to keep the global batch size for phase 1 at 16384 (strong scaling) with the following line in the script:

```
GRAD_ACCUM_USTEPS=$(($GRAD_ACCUM_USTEPS/$WORLD_SIZE_JOB))
```
To see performance for larger global batch size (weak scaling), please comment out the line above.

## Tips

Some useful slurm commands are `sinfo` and `squeue`. sinfo command displays information about SLURM modes and partitions. sinfo command provides information about job queues currently running in the Slurm schedule. While a job is running, SLURM generates a log file `slurm_<job id>.out`. You may then use `tail -f slurm_<job id>.out` to inspect the job summary.

## Known issues/limitations

- The current setup supports up to 16 nodes BERT pretraining.

## Troubleshooting guide

See [Troubleshooting Guide for AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/troubleshooting-v3.html) for more details and fixes to common issues.
