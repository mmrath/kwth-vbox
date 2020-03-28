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

## Step 1: Prepare the Host and VMs

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
- On the host: generate the config file
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
- On cp0: Generate RBAC role for access
```
cat <<EOF > rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF

cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
kubectl apply -f rbac.yaml
```

## Step 4: Setting up processing nodes
In k8s user applications run on processing nodes are where the applications run. K8s clusters can have 100s of processing nodes. Our example will have 3 nodes: node0, node1 and node2.

The k8s applications kubelet and kube-proxy are installed on all processing nodes and are responsible for their basic management and networking. They authenticate to the API server through client certificates.

POD_CIDR is the address range that is to be used for pods on that node:
10.200.{dpnode#}.0/24 for node0, node1, node2.

- On node0, node1 and node2: install necessary OS packages
```
apt-get -y install socat conntrack ipset
```
- On node0, node1 and node2: Download K8s applications
```
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.17.0/crictl-v1.17.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc10/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.8.5/cni-plugins-linux-amd64-v0.8.5.tgz \
  https://github.com/containerd/containerd/releases/download/v1.2.13/containerd-1.2.13.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.17.2/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.17.2/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.17.2/bin/linux/amd64/kubelet
```
- On node0, node1 and node2: create directories for binaries and config files
```
  mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```
- On node0, node1 and node2: install binaries
```
  mkdir containerd
  tar -xvf crictl-v1.17.0-linux-amd64.tar.gz
  tar -xvf containerd-1.2.13.linux-amd64.tar.gz -C containerd
  sudo tar -xvf cni-plugins-linux-amd64-v0.8.5.tgz -C /opt/cni/bin/
  sudo mv runc.amd64 runc
  chmod +x crictl kubectl kube-proxy kubelet runc 
  sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
  sudo mv containerd/bin/* /bin/
```
- On node0, node1 and node2: configure networking - adjust POD_CIDR depending on node
```
POD_CIDR=10.200.0.0/24 # or 10.200.1.0 or 10.100.2.0 if on node1 and node2
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF

cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF
```
- On node0, node1 and node2: configure containerd and systemctl startup file
```
mkdir -p /etc/containerd/

cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF

cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```
#### On node0, node1 and node2: configure the kubelet
- On the host: generate the certs, they are different for each node
```
for instance in node0 node1 node2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Mountain View",
      "O": "system:nodes",
      "OU": "Lab",
      "ST": "California"
    }
  ]
}
EOF
done

INTERNAL_IP=192.168.100.200 # .201,.202 for node0, node1 and node2
instance=node0              # node1, node2 

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
```
- On the host: KUBERNETES_PUBLIC_ADDRESS is the API server to contact (cp0 here at least initially)
```
KUBERNETES_PUBLIC_ADDRESS=192.168.100.100
for instance in node0 node1 node2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```
- On the host: copy to node0, node1 and node2
```
scp node0*.pem node0.kubeconfig ca.pem node0: # same for node1 and node2
```
- On node0, node1 and node2: move config files
```
mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
mv ca.pem /var/lib/kubernetes/
```
-  On node0, node1 and node2: generate YAML config file
```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```
- On node0, node1 and node2: generate systemctl startup file - node-ip changes!
```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --node-ip=192.168.100.200 \\
  --resolv-conf=/run/systemd/resolve/resolv.conf \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```



#### On node0, node1 and node2: configure the kube-proxy

- On the Host: generate certs
```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Mountain View",
      "O": "system:node-proxier",
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
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```
- On the Host: generate kubeconfig file
```
KUBERNETES_PUBLIC_ADDRESS=192.168.100.100
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
- On the Host: copy file to node0, node1 and node2
```
scp kube-proxy.kubeconfig node0:
```
- On node0, node1and node2: move configfile into required directory 
```
mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig 
```
- On node0, node1 and node2: the YAML config file
```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```
- On node0, node1 and node2: generate the systemctl startup file
```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
- On node0, node1 and node2: starts all apllications
```
systemctl daemon-reload
systemctl enable containerd kubelet kube-proxy
systemctl start containerd kubelet kube-proxy
```
- On cp0: see if nodes are registered
```
kubectl get nodes
```
- should give:
```
NAME    STATUS   ROLES    AGE    VERSION
node0   Ready    <none>   6d2h   v1.17.2
node1   Ready    <none>   5d7h   v1.17.2
node2   Ready    <none>   5d7h   v1.17.2
```
- On cp0, cp1 and cp2: Th econtrol plane needs to be able to resolve the node names to an IP address. Add IPs to /etc/hosts on cp0, cp1 and cp3
```
192.168.100.200 node0
192.168.100.201 node1
192.168.100.202 node2
```


## Step 5: Setup kubectl on the host
kubectl on the host needs to be able to reach the admin server and present a valid certificate.
- On the host: generate a kubeconfig file
```
KUBERNETES_PUBLIC_ADDRESS=192.168.100.100

kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
```
- On the Host: Test
```
kubectl get nodes
```
- On the Host: Run a small application, pulled from dockerhub
```
cat <<EOF > baseapachedeployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: baseapache
  labels:
    app: baseapache
spec:
  replicas: 1
  strategy: 
    type: RollingUpdate
  selector:
    matchLabels:
      app: baseapache
  template:
    metadata:
      labels:
        app: baseapache
    spec:
      containers:
       - name: baseapache
         image: wkandek/baseapache:latest
         imagePullPolicy: Always
      terminationGracePeriodSeconds: 0 
EOF


kubectl -f baseapachedeployment.yaml
kubectl get pods
kubectl exec -ti <podid> /bin/bash
```
The cluster is now able to run pods. wkandek/baseapache is a docker image that serves a simple HTML page. At this point the HTML page can  be accessed only on the node that it is running on, as we are still missing the network setup that maps pods to outside addresses.

Test: Access the Pods IP address with 'kubectl get pods -o wide', find the node that it running on, ssh to the node and try curl to the address of the pod 10.200.x.y. A simple HTML page should be returned.



## Step 6: Networking Installation

K8s by itself does not come with the necessary networking capabilities to communicate to other nodes or map services to outside addresses. Instead it leaves that functionality to a Container Network Interface (CNI) implementation.

This tutorial uses flannel is a simple implementation of a CNI.

- install flannel - on the host: download the YAML file, edit 2 parameters
  - replace vxlan -> host_gw
  - add --iface-regex=192\.168\.100\. in the command definition
```
wget -q --show-progress --https-only --timestamping \
	"https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"
vxlan -> host-gw
command:
  - /opt/bin/flanneld
  args:
  - --ip-masq
  - --kube-subnet-mgr
  - --iface-regex=192\.168\.100\.
```
- On the host: kubectl apply -f kube-flannel.yml
- On the host: kubectl -n kube-system get pods - to see if the flannel pods are running

flannel alone will still not get a mapping to an external IP address, in our case the 192.168.100.x space. For that we need to install additional network software. MetalLB is a load balancer that works with k8s to provide the necessary mapping from internal service addresses to external IP addresses.

-  On the host: download YAML for MetalLB
```
wget https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
```
- On the host:  create a configmap to define the range of IP addresses that MetalLB can map
```
cat <<EOF > metalLBconfigmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.100.240-192.168.100.250
EOF
kubectl apply -f metalLBconfigmap.yaml
kubectl apply -f metallb.yaml
```
Now a pod can be mapped to an external IP through a service definition of type: LoadBalancer
```
cat <<EOF > baseapacheservice.yaml
kind: Service
apiVersion: v1
metadata:
  name: baseapache
spec:
  selector:
    app: baseapache
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

kubectl apply -f baseapacheservice.yaml
kubectl get service
curl http://192.168.100.24X
```

To have k8s DNS integration we need to install a DNS service, for example coredns.
```
wget https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
```
- edit the coredns YAML file and add forward 8.8.8.8 in the .53 stanza just under the prometheus line to provide external DNS resolution
```
kubectl apply -f coredns.yaml
```
Now an nslookup on a newly started pod will work. 

## Step 7: API Loadbalancer

The API server runs on cp0, cp1 and cp2 but it is only accessed on cp0  through IP address ```192.168.100.100``` for example by the kubelet on the nodes and kubectl on the host. There is no k8s native way to loadbalance API requests, but an external loadbalancer works fine.

We will install an addtional VM with ```haproxy``` to implement the load balancer.
```
                           +-----------------+
                           |      API LB     |
                           | 192.168.100.50  |
                           +-----------------+  

    +-----------------+    +-----------------+    +-----------------+
    |       cp0       |    |       cp1       |    |       cp2       |
    | 192.168.100.100 |    | 192.168.100.101 |    | 192.168.100.102 |
    +-----------------+    +-----------------+    +-----------------+
```

Add the following lines to the ```Vagrantfile``` to define the additional VM:

```
  config.vm.define "loadbalancer" do |loadbalancer|
    loadbalancer.vm.box = 'ubuntu/bionic64'
    loadbalancer.vm.hostname = "loadbalancer"
    loadbalancer.vm.network :private_network, ip: "192.168.100.50"
    loadbalancer.vm.provision "shell", inline: $vmscript
  end
```

and run 

```
vagrant up loadbalancer
```

to provision the VM.

Then as root install haproxy on the VM:

```
apt-get update
apt-get upgrade
apt-get haproxy
```

and edit the haproxy.cfg file in /etc/haproxy:

```
defaults
        log     global
        mode    tcp
        option  tcplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

frontend k8sapi
        bind 192.168.100.50:6443
        default_backend k8sapiservers

backend k8sapiservers
        balance roundrobin
        mode tcp
        server cp1 192.168.100.100:6443 check
        server cp2 192.168.100.101:6443 check
        server cp3 192.168.100.102:6443 check
```

and restart the haproxy service: ```systemctl restart haproxy```

When we access the API server through https://192.168.100.50:6433 we get a certificate warning as the certificate does not include that IP as a valid target. We need to regenerate the certificate on the host and copy to cp0, cp1 and cp2 and restart the API servers.

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,192.168.100.50,192.168.100.100,192.168.100.101,192.168.100.102,127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

scp kubernetes.pem kubernetes-key.pem cp0:/var/lib/kubernetes/
scp kubernetes.pem kubernetes-key.pem cp1:/var/lib/kubernetes/
scp kubernetes.pem kubernetes-key.pem cp2:/var/lib/kubernetes/

ssh cp0 'systemctl restart kube-apiserver'
ssh cp1 'systemctl restart kube-apiserver'
ssh cp2 'systemctl restart kube-apiserver'
```

Test via firefox on the host to see if everything works. To get rid of the firefox warnings on the self signed CA, import the CA into firefox via preferences, secuirty, view certificates, import.

Obs: This type of test will require a way to run firefox on the host but have its window displayed on the workstation. XWindows works, but other solutions such as VNC work as well. 

Finally the kubelets need to be configured to access the new IP address. Generate a new kubeconfig file with the external address set to 192.168.100.50, copy to node0, node1, and node2 and restart the service. 


## Standby for more...
