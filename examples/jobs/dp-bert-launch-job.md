# Launch training job
Once the cluster is successfully created, you may ssh into the head node from your local environment used to launch the cluster in the previous step. As an example here, we will use [Phase 1 BERT-Large pretraining](https://awsdocs-neuron-staging.readthedocs-hosted.com/en/release_2.3.0rc2/frameworks/torch/tutorials/training/bert.html?next=https%3A%2F%2Fawsdocs-neuron-staging.readthedocs-hosted.com%2Fen%2Frelease_2.3.0rc1%2Fframeworks%2Ftorch%2Ftutorials%2Ftraining%2Fbert.html%3Fnext%3Dhttps%253A%252F%252Fawsdocs-neuron-staging.readthedocs-hosted.com%252Fen%252Frelease_2.3.0rc1%252Fframeworks%252Ftorch%252Ftutorials%252Ftraining%252Fbert.html&ticket=ST-1663365027-jWyjPKGS3TtpDY9Ih0iklXykKnHRSSnL#phase-1-bert-large-pretraining) as the example job to submit to the cluster. This pretraining script will be stored in the head node. Upon launching the job, the head node will distribute it to each compute node.

In this entire process, there will be three scripts required:

1. Python script (.py) that executes the workload
2. Shell script that sets up environment definition and runtime argument to be passed into the Python script.
3. Slurm script that submit the shell script to the job queue

The Python script `dp_bert_large_hf_pretrain_hdf5.py` in head node is used as the main script that executes the training workloads. This script is already in `~/examples/dp_bert_hf_pretrain` of the head node. Home directory (`~/`) are shared by the head node and comute nodes.

Following are steps for launching a training job:

1. In a terminal of the head node, activate the Python virtual environment: `source ~/aws_neuron_venv_pytorch_p37/bin/activate`.
2. Create a shell script with the content shown below in head node (i.e., `pretrain_ph1.sh`) to set up the runtime environment and invoke torchrun command to execute the pretraining script `dp_bert_large_hf_pretrain_hdf5.py`:

```
#!/usr/bin/env bash
set -o pipefail

WORLD_SIZE_JOB=$SLURM_NTASKS
RANK_NODE=$SLURM_NODEID
MASTER_ADDR=(`scontrol show hostnames $SLURM_JOB_NODELIST`)
export FI_EFA_USE_DEVICE_RDMA=1
export FI_PROVIDER=efa
export BUCKET_CAP_MB=512

MASTER_PORT=2022
NUM_NEURONCORES=32
DISTRIBUTED_ARGS="--nproc_per_node $NUM_NEURONCORES --nnodes $WORLD_SIZE_JOB --node_rank $RANK_NODE --master_addr $MASTER_ADDR --master_port $MASTER_PORT"
echo $DISTRIBUTED_ARGS

sudo rmmod neuron; sudo modprobe neuron
sudo sysctl -w net.ipv4.ip_local_reserved_ports=48620

steps_this_run=28125
if [[ "$NEURON_EXTRACT_GRAPHS_ONLY" == "1" ]]; then
    steps_this_run=10
fi

GRAD_ACCUM_USTEPS=32
# adjustment to keep global batch size at 16k for AdamW (phase 1)
GRAD_ACCUM_USTEPS=$(($GRAD_ACCUM_USTEPS/2))

# run from /tmp to avoid interference among nodes on shared homedir
cd /tmp
XLA_USE_BF16=1 torchrun $DISTRIBUTED_ARGS ~/examples/dp_bert_hf_pretrain/dp_bert_large_hf_pretrain_hdf5.py --steps_this_run=$steps_this_run --batch_size 16 --grad_accum_usteps $GRAD_ACCUM_USTEPS |& tee run_pretrain_ph1_log.txt
```

3. Change `pretrain_ph1.sh` to executable:
    
    `chmod +x ./pretrain_ph1.sh`

This `pretrain_ph1.sh` script will be invoked by SLURM scripts running in the head node. To do compilation, use the following SLURM compilation script (save as compile_ph1.slurm) to run the script on multiple nodes (in this example, 2 nodes):

```
#!/bin/bash
#SBATCH --nodes=2
#SBATCH --exclusive
srun neuron_parallel_compile ./pretrain_ph1.sh
```

Save the following SLURM pretraining script as `pretrain_ph1.slurm` :

```
#!/bin/bash
#SBATCH --nodes=2
#SBATCH --exclusive
srun ./pretrain_ph1.sh
```

4. Then run the slurm compilation script using:

```
sbatch compile_ph1.slurm
```

5. Wait for compilation to finish on these nodes, then start the actual pretraining:

```
sbatch pretrain_ph1.slurm
```

The job id will be displayed by sbatch. The run output will appear in slurm_<job id>.out file in head node.


## Tips

Some useful slurm commands are `sinfo` and `squeue`. sinfo command displays information about Slurm modes and partitions. sinfo command provides information about job queues currently running in the Slurm schedule. Once the job is done, slurm will generate a log file slurm-XXXXXX.out. You may then use `tail -f slurm-XXXXXX.out` to inspect the job summary.

