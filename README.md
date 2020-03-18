# Kubernetes The Hard Way on VirtualBox

This repository contains the files to implement Kelsey Hightower's "Kubernetes the Hard Way" [tutorial](https://github.com/kelseyhightower/kubernetes-the-hard-way) on VirtualBox. Kelsey's tutorial implements Kubernetes (K8s) on Google Cloud and an implementation on VirtualBox has to be adapted in a number of ways, which I documented here.

I also reworked the sequence of actions taken in the original tutorial to be more step-by-step, for example Kelsey generates all necessary certificates together, I generated them on demand when they become necessary, it is slower but helped me understand when a certificate or config file is actually needed.

I used a Ubuntu 18.04 server at DigitalOcean with 16 GB RAM and 6 CPUs as the host for the 6 Virtual Machines (VMs) that compromise the K8S cluster. Three VMs are for the control plane (cp) that runs the etcd database and the K8s API server, controller manager and scheduler and 3 VMS are for nodes that will run the application pods.

The k8s version used is v1.17, current as of March 2020 and etcd is at v3.4.

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
- apt-get update && apt-get upgrade
- apt-get install virtualbox vagrant
- reboot
- ssh-keygen to generate ssh keys
  - copy Vagrantfile and insert the private ssh key where indicated (line 36)
  - vagrant up to start the VMs (will take 5-10 minutes)

- install tools: cfssljson and kubectl
  - cfssljson is a tool authored by Cloudflare that helps to generate certificates
  - kubectl is part of k8s and will be used to generate config files 

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

- update /etc/hosts file with 

    192.168.100.100 cp0
    192.168.100.101 cp1
    192.168.100.102 cp2
    192.168.100.200 node0
    192.168.100.201 node1
    192.168.100.202 node2

cp0,1,2 and node0,1,2 can now be accessed via SSH from the host.

## Step 2: Install etcd on cp0, cp1 and cp2

To install the etcd database we will need to generate certificates that are used for authentication. The certificates will be signed by our own Certificate Authority (CA) that we will create as well. The CA and the certificates will be created on the host and copied to the VMs necessary.

- On the host use the following to create the CA:
```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Mountain View",
      "O": "Kubernetes CA",
      "OU": "Lab",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
These commands will create the JSON files necessary for cfssljson and then generate ca.pem and ca-key.pem the CA certificates that we are interested in.

Now the kubernetes certificate which will be used for etcd. 

```
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Mountain View",
      "O": "Kubernetes",
      "OU": "Lab",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,192.168.100.100,192.168.100.101,192.168.100.102,127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```
Similarly generates kubernetes.pem and kubernetes-key.pem

- Copy the certificates to cp0, cp1 and cp2
```
scp ca.pem kubernetes.pem kubernetes-key.pem cp0:
```

- on cp0, cp1 and cp2 download etcd and install it
```
mkdir -p /etc/etcd /var/lib/etc
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.4/etcd-v3.4.4-linux-amd64.tar.gz"
tar -xf etcd-v3.4.4-linux-amd64.tar.gz
mv etcd-v3.4.4-linux-amd64/etcd* /usr/local/bin/
```
- on cp1, cp1 and cp2 generate a startup file for systemctl. The values for the variables ETCD_NAME and INTERNAL_IP have to be set differently for each host, i.e. cp0 and 192.168.100.100, cp1 and 192.168.100.101 and cp2 and 192.168.100.102
```
ETCD_NAME=cp0
INTERNAL_IP=192.168.100.100
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://${INTERNAL_IP}:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster cp0=https://192.168.100.100:2380,cp1=https://192.168.100.101:2380,cp2=https://192.168.100.102:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- edit the host file on cp0, cp1 and cp2 with the IP addresses:
```
192.168.100.100 cp0
192.168.100.101 cp1
192.168.100.102 cp2
```

- and startup etcd on cp0, wait for a minute then cp1, wait 1 minute then cp2. Watch by tailing -f /var/log/syslog on the VM
```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

- check on the status of the etcd database
```
export ETCDCTL_API=3
etcdctl --endpoints=https://192.168.100.100:2379,https://192.168.100.101:2379,https://192.168.100.102:2379   --cacert=/etc/etcd/ca.pem   --cert=/etc/etcd/kubernetes.pem   --key=/etc/etcd/kubernetes-key.pem -w table endpoint status
```
- should return:
```
https://192.168.100.100:2379, 1e895fb4f3d83cc6, 3.4.4, 2.5 MB, false, false, 1097, 385412, 385412,
https://192.168.100.101:2379, c3ff911dc3e31606, 3.4.4, 2.4 MB, true, false, 1097, 385412, 385412,
https://192.168.100.102:2379, 941e6b6454ac9941, 3.4.4, 2.5 MB, false, false, 1097, 385413, 385413,
```

- the -w table option gives more context:
```
etcdctl --endpoints=https://192.168.100.100:2379,https://192.168.100.101:2379,https://192.168.100.102:2379   --cacert=/etc/etcd/ca.pem   --cert=/etc/etcd/kubernetes.pem   --key=/etc/etcd/kubernetes-key.pem -w table endpoint status
```



## Step 3: Install K8s on cp0, cp1 and cp2

Three k8s applications need to run on the control plane servers:
- The API Server, the only application that accesses the etcd server
- The Controller Manager, accesses the API server
- The Scheduler, accesses the API Server

They each need certificates and k8s config files, the API server and scheduler also need an additional YAML file.

- On the host: create an encryption key:
```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```
- On the host: create certificates for the service account:
```
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Mountain View",
      "O": "Kubernetes",
      "OU": "Lab",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```
- On the host: copy encryption key and necessary certificates to cp0, cp1 and cp2:
```
scp ca.pem ca-key.pem kubernetes.pem kubernetes-key.pem service-account.pem service-account-key.pem encryption-config.yaml cp0:
```
- On cp0, cp1, cp2: move the certificates to the required directories:
```
mkdir -p /var/lib/kubernetes
mv ca.pem ca-key.pem kubernetes.pem kubernetes-key.pem service-account.pem service-account-key.pem encryption-config.yaml /var/lib/kubernetes/
```
- On cp0, cp1 and cp2: download the required k8s applications:
```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.17.2/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.17.2/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.17.2/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.17.2/bin/linux/amd64/kubectl"
```
- On cp0, cp1 and cp2: install the applications
```
chmod +x kube-* kubectl
mv kube-* kubectl /usr/local/bin/
```
- On cp0, cp1 and cp2: systemctl startup file for kube-apiserver. The API Server is the only application that accesses the etcd database, so its needs to be told where etcd lives: --etcd-servers= line.
```
INTERNAL_IP=$(ip a show enp0s8 | grep 'inet ' | awk '{ print $2 }' | cut -f1 -d/)
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.100.100:2379,https://192.168.100.101:2379,https://192.168.100.102:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all=true \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- On cp0, cp1 and cp2: configure the controller manager. The Controller Manager accesses only the API Server. It needs client certificates for the authentication.  They are generated on the host and then embedded in the config file for the Controller Manager. That config file is then copied to cp0, cp1 and cp2. 
- on the host: generate certificates
```
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Mountain View",
      "O": "system:kube-controller-manager",
      "OU": "Lab",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```
  - On the host: generate kubeconfig file controller manager:
```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```
- On the host: copy the kubeconfig file to cp0, cp1 and cp2. The file has the certificates embedded, so they do not need to be copied.
```
scp kube-controller-manager.kubeconfig cp0:
```
- On cp0, cp1 and cp2: move file to the required directory
```
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```
- On cp0, cp1 and cp2: generate a systemctl startup file for controller manager
```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --bind-address=0.0.0.0 \
  --cluster-cidr=10.200.0.0/16 \
  --allocate-node-cidrs=true \
  --node-cidr-mask-size=24 \
  --cluster-name=kubernetes \
   --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem  \
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --leader-elect=true \
  --cert-dir=/var/lib/kubernetes \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --use-service-account-credentials=true \
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

-  On cp0, cp1 and cp2: configure the scheduler. The Scheduler accesses only the API Server. It needs client certificates for the authentication.  They are generated on the host and then embedded in the config file for the Scheduler. That config file is then copied to cp0, cp1 and cp2.
-  On the host: generate the client certificates:
```
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Mountain View",
      "O": "system:kube-scheduler",
      "OU": "Lab",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```
-  On the host: generate the config file with embedded certificates
```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```
- On the host: copy the kubeconfig file to cp0, cp1 and cp2. The file has the certificates embedded, so they do not need to be copied.
```
scp kube-scheduler.kubeconfig cp0:
```
- On cp0, cp1 and cp2: move file to the required directory
```
mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```
- On cp0, cp1 and cp2:generate a YAML config file for the scheduler
```
mkdir -p /etc/kubernetes/config
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```
- On cp0, cp1 and cp2: generate a systemctl startup file for scheduler
```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- On cp0, cp1 and cp2: set the /etc/hosts file, the hostname has to be set to the IP address that provides DNS resolution, normally that is 10.0.2.15 - the first interface on the host. Typically the hostname points to 127.0.0.1
```
10.0.2.15 cp0 # or cp1 or cp2
```

- On cp0, cp1 and cp2: activate the k8s control plane applications
```
systemctl daemon-reload
systemctl enable kube-apiserver kube-controller-manager kube-scheduler
systemctl start kube-apiserver kube-controller-manager kube-scheduler
```
- On cp0, cp1 and cp2: monitor syslog for the startup process
- On cp0, cp1 and cp2: configure kubectl for local access. kubectl is the command line interface (CLI) used to interact with k8s through the API Server. It requires a client certificate.
-  On the host: generate certificate
```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Mountain View",
      "O": "system:masters",
      "OU": "Lab",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```
- On the host::  generate the config file
```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
```
- On the host: copy to cp0, cp1 and cp2
```
scp admin.kubeconfig cp0:
```
- On cp0, cp1 and cp2: test the access:
```
mkdir .kube
mv admin.kubeconfig .kube/config
kubectl get componentstatuses
```
should return:
```
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```

## Standby for next steps...
