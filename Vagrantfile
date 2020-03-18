#-*- mode: ruby -*-
# vi: set ft=ruby :

######################################################################################
# User Variables Section CP = Control plane NODE = nodes
######################################################################################

VM_SUBNET = "192.168.100."
CP_OCTET = 100
DP_OCTET = 200
BOX_IMAGE = "ubuntu/bionic64"
CP_COUNT = 3
NODE_COUNT = 3
CPU = 1
CPMEMORY = 1024
NODEMEMORY = 2048

######################################################################################
# VM Configuration Script
######################################################################################

$vmscript = <<-SCRIPT

echo "Turning off swap..."
swapoff -a
echo "Configuring swap off at boot time..."
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "Setting up ssh..."
sed -i -e 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
sed -i -e 's/#PermitRoot/PermitRoot/' /etc/ssh/sshd_config
sed -i -e 's/#PubkeyA/PubkeyA/' /etc/ssh/sshd_config
systemctl reload sshd

mkdir /root/.ssh
echo "<Input .ssh/id_rsa.pub here>" > /root/.ssh/authorized_keys

SCRIPT

######################################################################################
# Create VMs Section
######################################################################################

Vagrant.configure("2") do |config|
  (0..CP_COUNT-1).each do |i|
    config.vm.define "cp#{i}" do |cp|
      cp.vm.box = BOX_IMAGE
      cp.vm.hostname = "cp#{i}"
      cp.vm.network :private_network, ip: "#{VM_SUBNET}#{CP_OCTET+i}"
      cp.vm.synced_folder "./" , "/vagrant"
      cp.vm.provider :virtualbox do |v|
        v.name = "cp#{i}"
        v.memory = CPMEMORY
        v.cpus = CPU
      end
      cp.vm.provision "shell", inline: $vmscript
    end
  end

  (0..NODE_COUNT-1).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.box = BOX_IMAGE
      node.vm.hostname = "node#{i}"
      node.vm.network :private_network, ip: "#{VM_SUBNET}#{DP_OCTET+i}"
      node.vm.synced_folder "./" , "/vagrant"
      node.vm.provider :virtualbox do |v|
        v.name = "dp#{i}"
        v.memory = NODEMEMORY
        v.cpus = CPU
      end
      node.vm.provision "shell", inline: $vmscript
    end
  end
end
