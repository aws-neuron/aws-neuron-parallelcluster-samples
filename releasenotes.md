# Change Log

## October,10th 2022

* Added a parallel cluster example that explains how to use AWS ParallelCluster to build HPC compute cluster using trn1 compute nodes to run the distributed ML training job.

# Known Issues

* **Name or service not known** 

If you encounter an error such as this during execution on ParallelCluster:

```
queue-trn2-dy-compute-resource-trn2-1:10363:10409 [0] include/socket.h:235 NCCL WARN Net : error encountered when getting address info : Name or service not known


queue-trn2-dy-compute-resource-trn2-1:10363:10409 [0] bootstrap.cc:33 NCCL WARN Invalid NCCL_COMM_ID [queue-trn2-dy-compute-resource-trn2-1.integ-tests-2kjfpvjzyi0qc3pz-trn.pcluster:37917], please use format: <ipv4>:<port> or [<ipv6>]:<port> or <hostname>:<port>
```

it is because one of the hostnames in `/etc/hosts` is longer than 63 characters. Please keep the names short to ensure that FQDN does not exceed 63 characters. If they are too long, the workaround is to run the following command: 

```
srun -N <number of nodes> sudo sed -i 's/\([0-9]\) .*pcluster / /' /etc/hosts
```

* Relaunch a dynamic cluster created with `MinCount = 0` may fail due to compute nodes IP address mismatch.

For dynamic cluster with `MinCount = 0`, /etc/hosts IP addresses of compute nodes may not match with what's in `nslookup` upon cluster relaunch. Therefore, for your information, a temporary workaround is included in `install_neuron.sh` post-install script:

```
IP="$(host $HOSTNAME| awk '{print $4}')"
DOMAIN=$(jq .cluster.dns_domain /etc/chef/dna.json | tr -d \")
sudo sed -i "/$HOSTNAME/d" /etc/hosts
sudo bash -c "echo '$IP $HOSTNAME.${DOMAIN::-1} $HOSTNAME' >> /etc/hosts"
```
This fix helps to ensure a dynamic cluster will relaunch successfully.