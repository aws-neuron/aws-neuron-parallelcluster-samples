# Neuron environment and packages installation

This part is already automated via [launch.yaml](../../cluster-configs/trn1-16-nodes-pcluster.md) via `CustomActions` options. It is executed automatically as a part of cluster launch. For your information, this script will install Neuron drivers and Runtime, EFA drivers (if multi-instance training), and Neuron tools. 

After this step is complete, the ParallelCluster environment is properly configured to run SLURM jobs.

## Helpful SLURM commands
Some useful slurm commands are `sinfo`,  `squeue` and `scontrol`. `sinfo` command displays information about slurm node names and partitions. `squeue` command provides information about job queues currently running in the Slurm schedule. Once the job is done, slurm will generate a log file `slurm-XXXXXX.out`. You may then use `tail -f slurm-XXXXXX.out`, to inspect the job summary. `scontrol show node <COMPUTE_NODE_NAME>` can show more information such as node state, power consumption, and more.