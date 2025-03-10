# External ceph cluster for kubernetes

https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd

## Setup

### Proxmox

Set-up a ceph cluster in proxmox following [these instructions](https://pve.proxmox.com/pve-docs/chapter-pveceph.html).

The following commands are run in proxmox to create the pool and retreive the necessary keys:

```shell
ceph --cluster ceph auth get-key client.admin
ceph --cluster ceph osd pool create kube 64 64
ceph auth add client.kube mon 'allow r' osd 'allow rwx pool=kube'
ceph auth get-key client.kube
ceph osd pool application enable kube rbd
ceph osd crush tunables hammer
```

(the last command, `ceph osd crush tunables hammer` is necessary to resolve an error during usage from kubernetes)

### kubernetes nodes

The `ceph` packages need to be installed on each of the k8s nodes (`apt-get install ceph`).  The reason for this is that the kubelet container running on the hosts needs to be able to lead the libceph kernel drivers.  It should be noted that the default version of ceph installed for buuntu 16.04 is v10.2.10 while the ceph cluster on the proxmox servers is v12.2.8 (luminous).  It doesn't seem a big issue though.

The rke kubelet configuration needs to expose the `/lib/modules` directory to the kubelet via

```yaml
extra_binds:
    - "/lib/modules:/lib/modules"
```

## Installation

Follow the instructions from [the guide](https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd#test-instruction).  For the parts about creating the Ceph admin & pool secrets, reference the keys obtained from the earlier proxmox section.

```shell
kubectl apply -f ./rbac
kubectl apply -f storageclass.yaml
```

## Working with rbd volumes

Example of finding and mounting a pvc in the proxmox host

Find the mappings between kubernetes pvc and ceph rbds:

```shell
➜ for i in $(kubectl get pv | grep rbd | awk '{print $1}')
do
echo "$i => $(kubectl describe pv "$i" | grep RBDImage | awk '{print $2}')"
done
pvc-880f2c76-1d23-11e9-86f8-36db0084bcac => kubernetes-dynamic-pvc-8dc13bbc-1d23-11e9-b4f5-524dbb781fe7
pvc-881d660e-1d23-11e9-86f8-36db0084bcac => kubernetes-dynamic-pvc-882c0839-1d23-11e9-b4f5-524dbb781fe7
pvc-c0ed5031-1d23-11e9-86f8-36db0084bcac => kubernetes-dynamic-pvc-c36d7e3a-1d23-11e9-b4f5-524dbb781fe7
pvc-de0ff848-1d23-11e9-86f8-36db0084bcac => kubernetes-dynamic-pvc-de1ca1be-1d23-11e9-b4f5-524dbb781fe7
pvc-ec6878d8-1d23-11e9-86f8-36db0084bcac => kubernetes-dynamic-pvc-f01af949-1d23-11e9-b4f5-524dbb781fe7
```

Then, acess the rbd image on the ceph server:

```shell
root@proxmox:/# rbd list kube
kubernetes-dynamic-pvc-2672ff6d-f6b5-11e8-a795-0a580a2a001c
root@proxmox:/# rbd map kube/kubernetes-dynamic-pvc-2672ff6d-f6b5-11e8-a795-0a580a2a001c
/dev/rbd0
root@proxmox:/# mkdir /tmp/rbd
root@proxmox:/# mount /dev/rbd0 /tmp/rbd
root@proxmox:/# ls -al /tmp/rbd
total 24
drwxr-xr-x  3 root root  4096 Dec  2 23:37 .
drwxrwxrwt 11 root root  4096 Dec  2 23:39 ..
drwx------  2 root root 16384 Dec  2 23:37 lost+found
-rw-r--r--  1 root root     0 Dec  2 23:37 SUCCESS
root@proxmox:/# umount /tmp/rbd
root@proxmox:/# rbd unmap /dev/rbd0
```
