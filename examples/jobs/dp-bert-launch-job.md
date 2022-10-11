# Launch training job
Once the cluster is successfully created, you may ssh into the head node from your local environment used to launch the cluster in the previous step. As an example here, we will use [Phase 1 BERT-Large pretraining](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/bert.html#phase-1-bert-large-pretrainingg) as the example job to submit to the cluster. This pretraining script will be stored in the head node. Upon launching the job, the head node will distribute it to each compute node.

In this entire process, there will be three scripts required:

1. Python script (.py) that executes the workload, which is `dp_bert_large_hf_pretrain_hdf5.py` and may be downloaded per instruction [here](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/bert.html#phase-1-bert-large-pretraining).
2. Shell script that sets up environment definition and runtime argument to be passed into the Python script.
3. Slurm script that submit the shell script to the job queue

The Python script `dp_bert_large_hf_pretrain_hdf5.py` in head node is used as the main script that executes the training workloads. This script is already in `~/examples/dp_bert_hf_pretrain` of the head node. Home directory (`~/`) are shared by the head node and comute nodes.

Following are steps for launching a training job:

1. In a terminal of the head node, activate the Python virtual environment: `source ~/aws_neuron_venv_pytorch_p37/bin/activate`.

2. Download dataset. In a terminal of the head node, follow [instructions in Neuron documentation](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/bert.html#downloading-tokenized-and-sharded-dataset-files)

3. Download a shell run script to head node to set up the runtime environment: 

```
cd ~/examples/dp_bert_hf_pretrain
wget https://raw.githubusercontent.com/aws-neuron/aws-neuron-samples/torch-neuronx/training/dp_bert_hf_pretrain/run_dp_bert_large_hf_pretrain_bf16_s128.sh
```

4. Change `run_dp_bert_large_hf_pretrain_bf16_s128.sh` to executable:
    
    `chmod +x ./run_dp_bert_large_hf_pretrain_bf16_s128.sh`

This `run_dp_bert_large_hf_pretrain_bf16_s128.sh` script will be invoked by SLURM scripts running in the head node. To do compilation, use the following SLURM compilation script (you may save it as `compile_ph1.slurm`) to run the script on multiple nodes (in this example, 16 nodes):

```
#!/bin/bash
#SBATCH --nodes=16
#SBATCH --exclusive
srun neuron_parallel_compile ./run_dp_bert_large_hf_pretrain_bf16_s128.sh
```

Also, save the following SLURM pretraining script as `pretrain_ph1.slurm` :

```
#!/bin/bash
#SBATCH --nodes=16
#SBATCH --exclusive
srun ./run_dp_bert_large_hf_pretrain_bf16_s128.sh
```

5. Then run the slurm compilation script using:

```
sbatch compile_ph1.slurm
```

6. Wait for compilation to finish on these nodes, then start the actual pretraining:

```
sbatch pretrain_ph1.slurm
```

The job id will be displayed by sbatch. The run output will appear in slurm_<job id>.out file in head node.


## Tips

Some useful slurm commands are `sinfo` and `squeue`. sinfo command displays information about Slurm modes and partitions. sinfo command provides information about job queues currently running in the Slurm schedule. Once the job is done, slurm will generate a log file slurm-XXXXXX.out. You may then use `tail -f slurm-XXXXXX.out` to inspect the job summary.

## Troubleshooting guide

See [Troubleshooting Guice in Neuron documentation](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/tutorials/training/bert.html#troubleshooting) for more details and fixes to common issues.