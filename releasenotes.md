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

This fix helps to ensure a dynamic cluster would relaunch successfully.

* Error “Assertion `listp->slotinfo[cnt].gen <= GL(dl_tls_generation)’ failed” followed by ‘RPC failed with status = “UNAVAILABLE: Connection reset by peer”’


```

   INFO: Inconsistency detected by ld.so: ../elf/dl-tls.c: 488: _dl_allocate_tls_init: Assertion `listp->slotinfo[cnt].gen <= GL(dl_tls_generation)' failed!
   INFO: 2022-10-03 02:16:04.488054: W tensorflow/core/distributed_runtime/rpc/grpc_remote_master.cc:157] RPC failed with status = "UNAVAILABLE: Connection reset by peer" and grpc_error_string = "{"created":"@1664763364.487962663","description":"Error received from peer ipv4:10.0.9.150:41677","file":"external/com_github_grpc_grpc/src/core/lib/surface/call.cc","file_line":1056,"grpc_message":"Connection reset by peer","grpc_status":14}", maybe retrying the RPC

```
This error may occur intermittently when using GNU C Library glibc 2.26. To find out what version you have, run ```ldd --version```. glibc 2.27 provides a workaround and therefore the error is fixed in Ubuntu20. For more information on this issue, see [Neuron troubleshooting guide](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/training-troubleshooting.html#error-assertion-listp-slotinfo-cnt-gen-gl-dl-tls-generation-failed-followed-by-rpc-failed-with-status-unavailable-connection-reset-by-peer). 

## Troubleshooting guide

See [Troubleshooting Guide for AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/troubleshooting-v3.html) for more details and fixes to common issues.
