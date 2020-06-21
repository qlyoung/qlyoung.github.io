picluster-k3s
=============

Ansible playbook for setting up k3s on a group of stock Raspberry Pis.

Usage
-----
This playbook has been tested on a cluster of 4 Raspberry Pis, each with the
following starting configuration:

- 128gb sdcard
- Raspbian Lite fresh install, plus `/boot/ssh` to enable ssh access
- One Ethernet connection with a DHCP-assigned IPv4 address

The Pis were booted exactly once prior to running the playbook.

To use this playbook on your cluster:

1. Edit `inventory.yaml` to reflect the IP addresses of your Pis, choosing one
   as the master; the inventory file should be clear enough by itself
2. `ansible-playbook -i inventory.ini ./provision.yaml`
3. ???
4. Profit

After it's done, `ssh` into your master node, note the hostname for future
reference. Check that it worked with

```
sudo kubectl get nodes
```

E.g., in my cluster, the end state after running the playbook:
 
```
pi@clusterpi-69:~ $ sudo kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
clusterpi-69   Ready    master   11m   v1.18.2+k3s1
clusterpi-67   Ready    <none>   10m   v1.18.2+k3s1
clusterpi-42   Ready    <none>   10m   v1.18.2+k3s1
clusterpi-86   Ready    <none>   10m   v1.18.2+k3s1
```

Known Issues
------------

- Hostnames are changed to `picluster-<X>`, where `X` is a random number in
  [0,100], with replacement. This is subject to the birthday problem; if your
  cluster is larger than ~5 nodes you should submit a patch to make sure
  hostnames aren't reused. Or just increase the range :)
