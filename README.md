# Kubernetes The Hard Way on VirtualBox

This repository contains the files to implement Kelsey Hightower's "Kubernetes the Hard Way" [tutorial](https://github.com/kelseyhightower/kubernetes-the-hard-way) on VirtualBox. Kelsey's tutorial implements Kubernetes (K8s) on Google Cloud and an implementation on VirtualBox has to be adapted in a number of ways, which I documented here.

I also reworked the sequence of actions taken in the original tutorial to be more step-by-step, for example Kelsey generates all necessary certificates together, I generated them on demand when they become necessary, it is slower but helped me understand when a certificate or config file is actually needed.

I used a Ubuntu 18.04 server at DigitalOcean with 16 GB RAM and 6 CPUs as the host for the 6 Virtual Machines (VMs) that compromise the K8S cluster. Three VMs are for the control plane (cp) that runs the etcd database and the K8s API server, controller manager and scheduler and 3 VMS are for nodes that will run the application pods once  done.

The VMs have the default network interface provided by VirtualBox, plus an secondary interface that is used for communication between the VMs. All secondary interfaces are in the 192.168.100.0/24 network:

- 192.168.100.100/102/103 for cp0, cp1 and cp2
- 192.168.100.201/201/202 for node0, node1 and node2

```
    +-----------------+    +-----------------+    +-----------------+
    |       cp0       |    |       cp1       |    |       cp2       |
    | 192.168.100.100 |    | 192.168.100.101 |    | 192.168.100.102 |
    +-----------------+    +-----------------+    +-----------------+
```
and
```
    +-----------------+    +-----------------+    +-----------------+       
    |      node0      |    |       node1     |    |      node2      |
    | 192.168.100.200 |    | 192.168.100.201 |    | 192.168.100.202 |
    +-----------------+    +-----------------+    +-----------------+
```


We will use vagrant to automate the setup of the VirtualBox VMs.

## Step1: Prepare the Host and VMs

All actions are taken as root.
1. apt-get update && apt-get upgrade

2. apt-get install virtualbox vagrant

3. reboot

4. ssh-keygen to generate ssh keys
   copy Vagrantfile and insert the private ssh key where indicated (line 36)
   vagrant up to start the VMs (will take 5-10 minutes)

5. install tools: cfssljson and kubectl
    cfssljson is a tool authored by Cloudflare that helps to generate certificates and kubectl is part of k8s and will be used to generate config files 
  
```
    wget -q --show-progress --https-only --timestamping \
    https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
    https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

    chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
    sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
    sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

    curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.17.0/bin/linux/amd64/kubectl
    chmod +x kubectl
    mv kubectl /usr/local/bin
```  
    
6. update /etc/hosts file with 

    192.168.100.100 cp0
    192.168.100.101 cp1
    192.168.100.102 cp2
    192.168.100.200 node0
    192.168.100.201 node1
    192.168.100.202 node2

cp0,1,2 and node0,1,2 can now be accessed via SSH from the host.



Standby for next steps...
