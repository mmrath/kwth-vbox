# Kubernetes The Hard Way on VirtualBox

This repository contains the files to implement Kelsey Hightower's "Kubernetes the Hard Way" [tutorial](https://github.com/kelseyhightower/kubernetes-the-hard-way) on VirtualBox. Kelsey's tutorial implements Kubernetes (K8s) on Google Cloud and an implementation on VirtualBox has to be adapted in a number of ways, which I documented here.

I also reworked the sequence of actions taken in the original tutorial to be more step-by-step, for example Kelsey generates all necessary certificates together, I generated them on demand when they become necessary, it is slower but helped me understand when a certificate or config file is actually needed.

I used a Ubuntu 18.04 server at DigitalOcean with 16 GB RAM and 6 CPUs as the host for the 6 Virtual Machines (VMs) that compromise the K8S cluster. Three VMs are for the control plane (cp) that runs the etcd database and the K8s API server, controller manager and scheduler and 3 VMS are for nodes that will run the application pods once  done.

K8s version used is 1.17, current as of March 2020.

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


Standby for next steps...
