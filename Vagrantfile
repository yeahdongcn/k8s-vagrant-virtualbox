Vagrant.configure("2") do |config|
  config.vm.provision :shell, privileged: true, inline: $install_common_tools

  config.vm.define :master do |master|
    master.vm.provider :virtualbox do |vb|
      vb.name = "master"
      vb.memory = 2048
      vb.cpus = 2
    end
    master.vm.box = "ubuntu/bionic64"
    master.disksize.size = "25GB"
    master.vm.hostname = "master"
    master.vm.network :private_network, ip: "10.0.0.10"
    master.vm.provision :shell, privileged: false, inline: $provision_master_node
  end

  %w{node1 node2 node3}.each_with_index do |name, i|
    config.vm.define name do |node|
      node.vm.provider "virtualbox" do |vb|
        vb.name = "node#{i + 1}"
        vb.memory = 2048
        vb.cpus = 2
      end
      node.vm.box = "ubuntu/bionic64"
      node.disksize.size = "25GB"
      node.vm.hostname = name
      node.vm.network :private_network, ip: "10.0.0.#{i + 11}"
      node.vm.provision :shell, privileged: false, inline: <<-SHELL
sudo /vagrant/join.sh
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.#{i + 11}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
cat /vagrant/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL
    end
  end

  config.vm.provision "shell", inline: $install_multicast
end


$install_common_tools = <<-SCRIPT
# bridged traffic to iptables is enabled for kube-router.
cat >> /etc/ufw/sysctl.conf <<EOF
net/bridge/bridge-nf-call-ip6tables = 1
net/bridge/bridge-nf-call-iptables = 1
net/bridge/bridge-nf-call-arptables = 1
EOF

# Patch OS
apt-get update && apt-get upgrade -y

# Create local host entries
echo "10.0.0.10 master" >> /etc/hosts
echo "10.0.0.11 node1" >> /etc/hosts
echo "10.0.0.12 node2" >> /etc/hosts
echo "10.0.0.13 node3" >> /etc/hosts

# disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Install kubeadm, kubectl and kubelet
export DEBIAN_FRONTEND=noninteractive
apt-get -qq install ebtables ethtool
apt-get -qq update
apt-get -qq install -y docker.io apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get -qq update
apt-get -qq install -y kubelet kubeadm kubectl
SCRIPT

$provision_master_node = <<-SHELL
OUTPUT_FILE=/vagrant/join.sh
KEY_FILE=/vagrant/id_rsa.pub
rm -rf $OUTPUT_FILE
rm -rf $KEY_FILE

# Create key
ssh-keygen -q -t rsa -b 4096 -N '' -f /home/vagrant/.ssh/id_rsa
cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
cat /home/vagrant/.ssh/id_rsa.pub > ${KEY_FILE}

# Start cluster
sudo kubeadm init --apiserver-advertise-address=10.0.0.10 --pod-network-cidr=10.244.0.0/16 | grep -Ei "kubeadm join|discovery-token-ca-cert-hash" > ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.10"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Configure flannel
curl -o kube-flannel.yml https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
sed -i.bak 's|"/opt/bin/flanneld",|"/opt/bin/flanneld", "--iface=enp0s8",|' kube-flannel.yml
kubectl create -f kube-flannel.yml

echo "alias k=/usr/bin/kubectl" >> $HOME/.bash_profile

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL

$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL