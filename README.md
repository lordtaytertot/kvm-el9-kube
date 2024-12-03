# kvm-el9-kube

## setup

```bash
ssh-copy-id vm1
ssh vm1
sudo visudo

sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo sed -i '/swap/d' /etc/fstab

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo dnf -y update
sudo dnf -y install kubelet kubeadm kubectl vim yum-utils --disableexcludes=kubernetes
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf -y install containerd

sudo systemctl disable firewalld
sudo systemctl enable containerd
sudo systemctl enable kubelet

containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo poweroff
```

clone to vm2 and vm3, modifying MAC addresses, increasing CPU and memory, and attaching 100 GiB drives

```bash
# on vm1
sudo kubeadm init --apiserver-advertise-address 192.168.1.61 --pod-network-cidr "10.42.0.0/16"

# https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O
sed -i 's/192.168.0.0/10.42.0.0/' custom-resources.yaml
kubectl create -f custom-resources.yaml
```

```bash
# on vm2/vm3
sudo pvcreate /dev/vdb
sudo vgcreate local /dev/vdb

for i in {0..9}; do
  if [[ ${i} == 9 ]]; then
    sudo lvcreate -n lv${i} -l 100%FREE local
    continue
  fi
  sudo lvcreate -n lv${i} -L 10G local
done

for i in {0..9}; do
  sudo mkfs.xfs /dev/mapper/local-lv${i}
  sudo mkdir /mnt/local-lv${i}
  sudo mount /dev/mapper/local-lv${i} /mnt/local-lv${i}
done

sudo kubeadm join 192.168.1.61:6443 --token ... --discovery-token-ca-cert-hash ...
```

```bash
# from kubectl client
./setup-storage.sh
```

## utils
```bash
./power.sh 
usage: ./power.sh ( start || shutdown )
```

## upgrade
```bash
# zeus ~/dev/ansible
ansible vm -b -a 'sudo dnf -y update'

# vm1
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.31.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# zeus ~/dev/ansible
ansible vm2,vm3 -b -a 'kubeadm upgrade node'
ansible vm2,vm3 -b -a 'systemctl daemon-reload'
ansible vm2,vm3 -b -a 'systemctl restart kubelet'
```
