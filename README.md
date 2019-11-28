
Official **kubevirt** documentation you can find [here](https://kubevirt.io/user-guide/docs/latest/welcome/index.html).  
Once **kubevirt** is up and running you can deploy VM.

K8s deployed on openstack  
K8s ExternalIP - openstack **LBaaS**  
K8s StorageClass - openstack **Cinder**


### Tools
- kubevirt v0.23.0
- virtctl 
- kubectl (+kubeconfig)


### Deployment


```sh
## GENERATE ssh keys and add public to 'centos7vm.yml'
$ ssh-keygen -b 1024 -t rsa -N "" -f ./k8s_rsa -C test@kubevirt

## RUN VM

$ kubectl create -f centos7vm.yml

$ kubectl get pods 
NAME                        READY   STATUS    RESTARTS   AGE
importer-centos7-dv-q9chc -f   1/1     Running   0          28s

$ kubectl logs importer-centos7-dv-q9chc -f
(...)
I1127 13:44:02.697165       1 data-processor.go:183] New phase: Resize
W1127 13:44:02.720237       1 data-processor.go:240] Available space less than requested size, resizing image to available space 10447220736.
I1127 13:44:02.720827       1 data-processor.go:246] Expanding image size to: 10447220736
I1127 13:44:04.469317       1 data-processor.go:183] New phase: Complete
I1127 13:44:04.469648       1 importer.go:160] Import complete

# once it's completed start your VM

$ virtctl start testvm

$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
virt-launcher-testvm-dtdxm   1/1     Running   0          27s

# optional check

$ kubectl exec -it virt-launcher-testvm-dtdxm bash
[root@testvm /]$ virsh list
 Id   Name             State
--------------------------------
 1    default_testvm   running

# checks

$ kubectl get vm
NAME     AGE     RUNNING   VOLUME
testvm   5m33s   true      

$ kubectl get vmi
NAME     AGE    PHASE     IP            NODENAME
testvm   107s   Running   10.42.5.191   worker-0

$ kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                              STORAGECLASS   REASON   AGE
persistentvolume/pvc-c2c4d5af-11e0-11ea-876b-fa163e803940   10Gi       RWO            Delete           Bound    default/centos7-dv                                 cinder                  9m15s

NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/centos7-dv   Bound    pvc-c2c4d5af-11e0-11ea-876b-fa163e803940   10Gi       RWO            cinder         9m16s

# once VM is up and running you can access your VM

# using console (password in 'centos7vm.yml')
$ virtctl console testvm
Successfully connected to testvm console. The escape sequence is ^]

CentOS Linux 7 (Core)
Kernel 3.10.0-957.27.2.el7.x86_64 on an x86_64

testvm login:
[centos@testvm ~]$

# using ssh
$ virtctl expose vmi testvm --port=22 --name=testvm-ssh --type=LoadBalancer

$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
testvm-ssh   LoadBalancer   10.43.132.105   <external_ip>   22:58337/TCP   58s

$ ssh -i k8s_rsa centos@<external_ip>
[centos@testvm ~]$


## DELETE VM

# remove LB
$ kubectl delete svc testvm-ssh

$ virtctl stop testvm
$ kubectl delete -f centos7vm.yml
```

### Running VM

```sh
# disks
[root@testvm ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1       9.8G  889M  8.9G   9% /
devtmpfs        473M     0  473M   0% /dev
tmpfs           496M     0  496M   0% /dev/shm
tmpfs           496M   13M  483M   3% /run
tmpfs           496M     0  496M   0% /sys/fs/cgroup
tmpfs           100M     0  100M   0% /run/user/1000

[root@testvm ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    253:0    0  9.7G  0 disk 
└─vda1 253:1    0  9.7G  0 part /
vdb    253:16   0  366K  0 disk 

# docker
[root@testvm ~]$ docker --version
Docker version 19.03.5, build 633a0ea

[root@testvm ~]$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              231d40e811cd        5 days ago          126MB

[root@testvm ~]$ docker run -d -p 80:80 nginx:latest

[root@testvm ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
b691b0475618        nginx:latest        "nginx -g 'daemon of…"   6 seconds ago       Up 3 seconds        0.0.0.0:5050->80/tcp   zealous_euclid

[root@testvm ~]$ curl localhost:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
(...)
```