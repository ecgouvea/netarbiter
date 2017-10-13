# Ceph Helm
Authors: Hee Won Lee <knowpd@research.att.com> and Yu Xiang <yxiang@research.att.com>    
Created on 10/1/2017  
Adapted for Ubuntu 16.04 and Ceph Luminous  
Based on https://github.com/ceph/ceph-docker/tree/master/examples/helm  

### Qucikstart

Assuming you have a Kubeadm managed Kubernetes 1.7+ cluster and Helm 2.6.1 setup, you can get going straight away! [1]

0. Preflight checklist
```
sudo apt install ceph-common
sudo apt install jq			# used in activate-namespace.sh
```

1. Install helm and tiller
```
curl -O https://storage.googleapis.com/kubernetes-helm/helm-v2.6.1-linux-amd64.tar.gz
tar xzvf helm-v2.6.1-linux-amd64.tar.gz 
sudo cp linux-amd64/helm /usr/local/bin

helm init       # or helm init --upgrade
helm serve &
```

2. Run ceph-mon, ceph-mgr, ceph-mon-check, and rbd-provisioner 
- Usage:
```
./create-secret-kube-config.sh
./helm-install-ceph.sh <release_name> <public_network> <cluster_network>
```

- Example
   ```
   ./helm-install-ceph.sh ceph 172.31.0.0/20 172.31.0.0/20
   ```

3. Run an OSD chart
- Usage:
```
./helm-install-ceph-osd.sh <hostname> <osd_device>
```

- Example:
   - bluestore:
   ```
   ./helm-install-ceph-osd.sh voyager1 /dev/sdc
   ```

   - filestore
   ```
   OSD_FILESTORE=1 ./helm-install-ceph-osd.sh voyager1 /dev/sdc
   ```

   - filestore with journal (recommended for production environment)
   ```
   OSD_FILESTORE=1 OSD_JOURNAL=/dev/sdb1 ./helm-ceph-osd.sh voyager1 /dev/sdc
   ```
      
      - NOTE: Use `diskpart.sh` to prepare for journal disk partitions in each host.
         Example: Create 8 journal partitions in /dev/sdb with the size of 10GiB.
         ```
         ./diskpart.sh /dev/sdb 10 1 8 ceph-journal 
         ```
   
### Namespace Activation

To use Ceph Volumes in a namespace a secret containing the Client Key needs to be present.

Once defined you can then activate Ceph for a namespace by running:
```
./activate-namespace.sh default
```

Where `default` is the name of the namespace you wish to use Ceph volumes in.

### Functional testing

Kubernetes >=v1.6 makes RBAC the default admission controller. We does not currently have RBAC roles and permissions for each
component, so you need to relax the access control rules:
```
# For Kubernetes 1.6 and 1.7
kubectl replace -f relax-rbac-k8s1.7.yaml

# For Kubernetes 1.8+
kubectl replace -f relax-rbac-k8s1.8.yaml
```

Once Ceph deployment has been performed you can functionally test the environment by running the jobs in the tests directory.
```
# Create a pool from a ceph-mon pod (e.g., ceph-mon-0):
ceph osd pool create rbd 100 100

# When mounting a pvc to a pod, you may encounter dmesg errors as follows: 
#    libceph: mon0 172.31.8.199:6789 feature set mismatch
#    libceph: mon0 172.31.8.199:6789 missing required protocol features
# Avoid them by running the following:
ceph osd crush tunables legacy

# Create a pvc and a job:
kubectl create -R -f tests/ceph
```


#### Notes
[1] You actually need to have the nodes setup to access the cluster network, and `/etc/resolv.conf` setup similar to the following:
```
$ cat /etc/resolv.conf
nameserver 10.96.0.10		# K8s DNS IP
nameserver 135.207.240.13	# External DNS IP; You would have a different IP.
search ceph.svc.cluster.local svc.cluster.local cluster.local client.research.att.com research.att.com
```
Otherwise, you can simply replace K8s nodes' `/etc/resolv.conf` with `/etc/resolv.conf` in a ceph-mon pod (e.g., ceph-mon-0) by Ctrl-C & Ctrl-V.

[2] About `docker-image-kubectl-ubuntu-16.04`: 
To generate ceph keys (`ceph/templates/jobs/job.yaml`), we create a docker image with `docker-image-kubectl-ubuntu-16.04/Dockerfile`.
