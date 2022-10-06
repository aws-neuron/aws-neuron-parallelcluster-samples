# Subnet setup for ParallelCluster with Trn1

Public subnet is where the head node resides. In this subnet, you need to enable IPv4 address assignment:

1. **Identify subnet by VPC ID** - Now in Subnets tab, find the subnets associated with your ParallelCluster’s VPC. There should be one private and one public subnet. We are going to edit settings in these subnets by clicking on these Subnet ID.

![image info](../../images/subnets.png)

Click on the public subnet's ID.

2. **IPv4 address** - In the public subnet panel, in its `Actions` dropdown box, select `Edit subnet settings`:

![image info](../../images/edit-subnet.png)

This will take you to the next setting page. Here, you need to check `Enable auto-assign public IPv4 address:

![image info](../../images/ipv4.png)

These are all the changes you need to make for the public subnet.