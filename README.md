# Kubernetes cluster
A vagrant script for setting up a Kubernetes cluster using Kubeadm   
Cloned from : https://github.com/ecomm-integration-ballerina/kubernetes-cluster

This project will allow you to spin up a full 3 node Kubernetes Cluster. This cluster 

## Pre-requisites

 * **[Vagrant 2.2.5+](https://www.vagrantup.com)**
 * **[Virtualbox 6.0.12+](https://www.virtualbox.org)**
 * **A computer with lots of Memory(16GB). Since the 3 virtualBox machines each use 2 Gb from the host**

## How to Run
After cloning the project you can then execute the Vagrant commands to build the box and also configure the nodes to 
joing the Kubernetes cluster. You should have a running/Ready cluster after doing the following.

1. Git clone or download the project
2. Change to the root project directory
3. Run "vagrant up"
4. Wait for all the virtual box machine to be built
5. Run "vagrant ssh k8s-head"
6. kubectl get nodes
7. Deploy your apps to the cluster and expose the services for external use.

Execute the following vagrant command to start a new Kubernetes cluster, this will start one master and two nodes:
```
vagrant up
```

You can also start invidual machines by vagrant up k8s-head, vagrant up k8s-node-1 and vagrant up k8s-node-2

If more than two nodes are required, you can edit the servers array in the Vagrantfile

```
servers = [
    {
        :name => "k8s-node-3",
        :type => "node",
        :box => "ubuntu/xenial64",
        :box_version => "20191005.0.0",
        :eth1 => "192.168.205.13",
        :mem => "2048",
        :cpu => "2"
    }
]
 ```

As you can see above, you can also configure IP address, memory and CPU in the servers array. 

##### Verfiy cluster is up and running with kubectl
```bash
% vagrant ssh k8s-head   ( executed from host )
 On guest master node
vagrant@k8s-head:~$ kubectl get nodes -o wide
NAME         STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-head     Ready    master   73m   v1.16.1   10.0.2.15     <none>        Ubuntu 16.04.6 LTS   4.4.0-165-generic   docker://18.9.9
k8s-node-1   Ready    <none>   69m   v1.16.1   10.0.2.15     <none>        Ubuntu 16.04.6 LTS   4.4.0-165-generic   docker://18.9.9
k8s-node-2   Ready    <none>   64m   v1.16.1   10.0.2.15     <none>        Ubuntu 16.04.6 LTS   4.4.0-165-generic   docker://18.9.9
 
 
```
Verify the above command show  all the nodes to be in the Ready State

##### Do a test deploy of a docker app
```bash
vagrant@k8s-head:~$ kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=8080
vagrant@k8s-head:~$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   0/1     1            0           14s
```
## Clean-up

Execute the following command to remove the virtual machines created for the Kubernetes cluster.
```
vagrant destroy -f
```

You can destroy individual machines by vagrant destroy k8s-node-1 -f

## Licensing

[Apache License, Version 2.0](http://opensource.org/licenses/Apache-2.0).
