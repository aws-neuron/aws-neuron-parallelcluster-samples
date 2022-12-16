# Launch training job
Once the cluster is successfully created and the Neuron packages are installed, please ssh into the head node to begin the training example. As an example here, we will use [Phase 1 BERT-Large pretraining](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/bert.html#phase-1-bert-large-pretrainingg) as the example job to submit to the cluster using SLURM job scheduler. You will be running commands from the head node. On ParallelCluster, the home directory is shared between the head node and compute nodes via NFS so files in the home directory are visible to worker nodes.

For all the commands below, make sure you are in the virtual environment created during setup before you run the commands. SLURM job scheduler will automatically activate the virtual environment when the training script is run on the worker nodes.

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

Some useful SLURM commands are `sinfo`,  `squeue` and `scontrol`. `sinfo` command displays information about SLURM node names and partitions. `squeue` command provides information about job queues currently running in the Slurm schedule. SLURM will generate a log file `slurm-XXXXXX.out`. You may then use `tail -f slurm-XXXXXX.out`, to inspect the job summary. `scontrol show node <COMPUTE_NODE_NAME>` can show more information such as node state, power consumption, and more.


## Known issues/limitations

- The current setup supports up to 16 nodes BERT pretraining.

- Error “Assertion `listp->slotinfo[cnt].gen <= GL(dl_tls_generation)’ failed” followed by ‘RPC failed with status = “UNAVAILABLE: Connection reset by peer”’


```

   INFO: Inconsistency detected by ld.so: ../elf/dl-tls.c: 488: _dl_allocate_tls_init: Assertion `listp->slotinfo[cnt].gen <= GL(dl_tls_generation)' failed!
   INFO: 2022-10-03 02:16:04.488054: W tensorflow/core/distributed_runtime/rpc/grpc_remote_master.cc:157] RPC failed with status = "UNAVAILABLE: Connection reset by peer" and grpc_error_string = "{"created":"@1664763364.487962663","description":"Error received from peer ipv4:10.0.9.150:41677","file":"external/com_github_grpc_grpc/src/core/lib/surface/call.cc","file_line":1056,"grpc_message":"Connection reset by peer","grpc_status":14}", maybe retrying the RPC

```
This error may occur intermittently when using GNU C Library glibc 2.26. To find out what version you have, run ```ldd --version```. glibc 2.27 provides a workaround and therefore the error is fixed in Ubuntu20. For more information on this issue, see [Neuron troubleshooting guide](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/training-troubleshooting.html#error-assertion-listp-slotinfo-cnt-gen-gl-dl-tls-generation-failed-followed-by-rpc-failed-with-status-unavailable-connection-reset-by-peer). 

## Troubleshooting guide

See [Troubleshooting Guide for AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/troubleshooting-v3.html) for more details and fixes to common issues.
