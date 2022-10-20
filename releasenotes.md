# Change Log

## October,10th 2022

* Added a parallel cluster example that explains how to use AWS ParallelCluster to build HPC compute cluster using trn1 compute nodes to run the distributed ML training job.

# Known Issues

* **Name or service not known** 

If you encounter an error such as this during execution on ParallelCluster:

```
queue-trn2-dy-compute-resource-trn2-1:10363:10409 [0] include/socket.h:235 NCCL WARN Net : error encountered when getting address info : Name or service not known
```

it is because one of the hostnames in `/etc/hosts` is longer than 63 characters. Please keep the names short to ensure that FQDN does not exceed 63 characters. If they are too long, the workaround is to run the following command: 

```
srun -N <number of nodes> sudo sed -i 's/\([0-9]\) .*pcluster / /' /etc/hosts
```


