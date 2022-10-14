# AMI 

You need to configure head node and compute nodes via post-installation actions. The following are the actions required:

1. Create a `install_neuron.sh` script that has the following content:

```
# Configure Linux for Neuron repository updates

sudo tee /etc/yum.repos.d/neuron.repo > /dev/null <<EOF
[neuron]
name=Neuron YUM Repository
baseurl=https://yum.repos.neuron.amazonaws.com
enabled=1
metadata_expire=0
EOF
sudo rpm --import https://yum.repos.neuron.amazonaws.com/GPG-PUB-KEY-AMAZON-AWS-NEURON.PUB

# Install OS headers
sudo yum install kernel-devel-$(uname -r) kernel-headers-$(uname -r) -y

# Update OS packages
sudo yum update -y


# Remove preinstalled packages and Install Neuron Driver and Runtime
sudo yum remove aws-neuron-dkms -y
sudo yum remove aws-neuronx-dkms -y
sudo yum remove aws-neuronx-oci-hook -y
sudo yum remove aws-neuronx-runtime-lib -y
sudo yum remove aws-neuronx-collectives -y
sudo yum install aws-neuronx-dkms-2.*  -y
sudo yum install aws-neuronx-oci-hook-2.*  -y
sudo yum install aws-neuronx-runtime-lib-2.*  -y
sudo yum install aws-neuronx-collectives-2.*  -y

# Install EFA Driver(only required for multi-instance training)
curl -O https://efa-installer.amazonaws.com/aws-efa-installer-latest.tar.gz
wget https://efa-installer.amazonaws.com/aws-efa-installer.key && gpg --import aws-efa-installer.key
cat aws-efa-installer.key | gpg --fingerprint
wget https://efa-installer.amazonaws.com/aws-efa-installer-latest.tar.gz.sig && gpg --verify ./aws-efa-installer-latest.tar.gz.sig
tar -xvf aws-efa-installer-latest.tar.gz
cd aws-efa-installer && sudo bash efa_installer.sh --yes
cd
sudo rm -rf aws-efa-installer-latest.tar.gz aws-efa-installer

# Remove pre-installed package and Install Neuron Tools
sudo yum remove aws-neuron-tools  -y
sudo yum remove aws-neuronx-tools  -y
sudo yum install aws-neuronx-tools-2.*  -y
```

2. Change `install_neuron.sh` to executable:
```
chmod +x install_neuron.sh
```

3. Create a slurm script with the following content and name it as `install_neuron.slurm`:

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

After this is completed, all compute nodes will have necessary Neuron packages installed.

4. Set up Python virtual environment in head node. In this step, create a shell script with the following content and name it as `install_python_env.sh`:

```
# Configure Linux for Neuron repository updates

sudo tee /etc/yum.repos.d/neuron.repo > /dev/null <<EOF
[neuron]
name=Neuron YUM Repository
baseurl=https://yum.repos.neuron.amazonaws.com
enabled=1
metadata_expire=0
EOF
sudo rpm --import https://yum.repos.neuron.amazonaws.com/GPG-PUB-KEY-AMAZON-AWS-NEURON.PUB

# Install OS headers
sudo yum install kernel-devel-$(uname -r) kernel-headers-$(uname -r) -y

# Update OS packages
sudo yum update -y

# Remove pre-installed package and Install Neuron Tools

sudo yum remove aws-neuron-tools  -y
sudo yum remove aws-neuronx-tools  -y
sudo yum install aws-neuronx-tools-2.*  -y

# Install Python venv and activate Python virtual environment to install Neuron pip packages.
python3.7 -m venv aws_neuron_venv_pytorch_p37
source aws_neuron_venv_pytorch_p37/bin/activate
python -m pip install -U pip

# Install packages from beta repos

python -m pip config set global.extra-index-url "https://pip.repos.neuron.amazonaws.com"
# Install Python packages - Transformers package is needed for BERT
python -m pip install torch-neuronx=="1.11.0.1.*" "neuronx-cc==2.*" transformers
```

5. Change `install_python_env.sh` to executable:
```
chmod +x install_python_env.sh
```
and run `install_python_env.sh` in head node terminal:
```
sh install_python_env.sh
```
This is run as a shell script instead of Slurm script. This is because we only want to create the virtual environment once, which is in the head node only. 

After this step is complete, the ParallelCluster is properly configured to run SLURM jobs.


## Helpful SLURM commands
Some useful slurm commands are `sinfo`,  `squeue` and `scontrol`. `sinfo` command displays information about slurm node names and partitions. `squeue` command provides information about job queues currently running in the Slurm schedule. Once the job is done, slurm will generate a log file `slurm-XXXXXX.out`. You may then use `tail -f slurm-XXXXXX.out`, to inspect the job summary. `scontrol show node <COMPUTE_NODE_NAME>` can show more information such as node state, power consumption, and more.