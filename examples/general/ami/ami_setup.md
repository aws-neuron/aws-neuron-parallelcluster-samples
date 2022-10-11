# AMI 

You need a ParallelCluster AMI for you to run the cluster. For now, the approach is to use a base AMI and post-installation script to build your cluster environment. The following are steps required:

1. Find a list of available ParallelCluster AMI in [here](https://github.com/aws/aws-parallelcluster/blob/v2.11.7/amis.txt). For the example here, we will use Amazon Linux 2 AMI. You may find the AMI ID based on the region accessible by your account. 

2. Once the AMI is chosen, make a note of the AMI ID; you will need this AMI to create a ParallelCluster. 

3. After the cluster is created, create a post installation script to install Neuron packages by creating `install_neuron.sh` script that has same content as in [here](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/setup/pytorch-install.html#develop-on-trn1-instance) for Amazon Linux 2.


4. Change `install_neuron.sh` to executable:

```
chmod +x install_neuron.sh
```

5. Create a slurm script with the following content and name it as `install_neuron.slurm`:

```
#!/bin/bash
#SBATCH --nodes=16
#SBATCH --exclusive
srun ./install_neuron.sh
```

and run it:

```
sbacth install_neuron.slurm
```

