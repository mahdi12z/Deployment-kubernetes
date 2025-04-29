# k8s-master-setup

## Kubernetes Cluster Setup - Master Node
This guide walks you through setting up a Kubernetes Master Node on Ubuntu with detailed explanations for every step. It uses `kubeadm`, `containerd` as the container runtime, and Flannel as the CNI plugin.

----
```bash
sudo nft delete table ip nat
sudo nft flush ruleset
sudo nft list ruleset
sudo systemctl stop nftables
sudo systemctl disable nftables
sudo rm -f /etc/nftables.conf
sudo apt purge nftables -y
sudo apt autoremove --purge -y
which nft
nft --version
```



## 1 Set the Hostname

```bash
sudo hostnamectl set-hostname --static km1
```

This sets a persistent hostname for the machine. Use a unique name like `km1`, `km2`, or `km-master

---

## 2 Edit /etc/hosts File
```bash
sudo vi /etc/hosts
```

Add the IP and hostname mappings for all nodes in your cluster:

```text
192.168.100.248   km1
192.168.100.249   km2
192.168.100.250   km-master
```
This ensures that nodes can resolve each other's hostnames without relying on DNS

---

## 3 Configure Static IP Address (Netplan)
Edit the Netplan config:
```bash
sudo vi /etc/netplan/00-installer-config.yaml
```
Set it like this:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens34:
      dhcp4: false
      addresses:
        - */24
      routes:
        - to: default
          via: *
      nameservers:
        addresses:
          - 185.51.200.1
          - 8.8.8.8
```
Then apply the settings:
```bash
sudo netplan apply
```
----
## 4-Lock DNS Configuration
```bash
sudo rm -f /etc/resolv.conf
sudo vi /etc/resolv.conf
```
Add:
```
nameserver 185.51.200.1
nameserver 172.22.122.101
```
```bash
sudo chattr +i /etc/resolv.conf
```

```bash
sudo apt update
sudo apt install openvswitch-switch
sudo systemctl enable ovsdb-server
sudo systemctl start ovsdb-server
sudo systemctl enable openvswitch-switch
sudo systemctl start openvswitch-switch

```

```bash
vi /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
network: {config: disabled}

```

```bash
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -X
sudo iptables -t nat -X
sudo systemctl restart kubelet

```
----

## 5 Setup and Configure NTP

```bash
sudo apt update && sudo apt install ntp -y
```
Edit:
```bash
sudo vi /etc/ntp.conf
```
Example NTP servers (Iranian or global):
```
server ntp1.ptb.ir iburst
server ntp2.ptb.ir iburst
server time.iranserver.com iburst
server ir.pool.ntp.org iburst
```

Then restart and enable the service:
```bash
sudo systemctl restart ntp
sudo systemctl enable ntp
ntpq -p
```
Time synchronization is critical for cluster certificates and log consistency
-----
## 6 Disable Swap (Required for K8s)
```bash
sudo swapoff -a
sudo sed -i '\|swap|s|/|#/|g' /etc/fstab
```
 *Kubernetes requires swap to be turned off for proper scheduling and memory management.*
 
 ----
 
## 7 Configure Kernel Parameters
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

```bash
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf

```

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
modprobe br_netfilter overlay
modprobe br_netfilter overlay && lsmod | grep br && lsmod | grep ov
```


 *These parameters and modules are needed for Kubernetes networking to function properly.*

 -----

 ## 8 Install containerd (Container Runtime)

```bash
sudo apt update && sudo apt install ca-certificates curl apt-transport-https conntrack gpg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker repo:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install and configure:

```bash
sudo apt update && sudo apt install containerd.io -y
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo modprobe br_netfilter overlay
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart containerd
```

 *containerd is a lightweight, Kubernetes-compatible container runtime.*

 ----

 ## 9 Install Kubernetes (kubelet, kubeadm, kubectl)

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install kubelet kubeadm kubectl -y
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
sudo kubeadm config images pull
```
##  Enable Shell Completion & Set Kubeconfig

```bash
sudo kubeadm completion bash > /etc/bash_completion.d/kubeadm
sudo kubectl completion bash > /etc/bash_completion.d/kubectl
exec bash -l
```

## Initialize Kubernetes Control Plane

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.*.* \
  --pod-network-cidr=10.244.0.0/16 \
  --upload-certs \
  --control-plane-endpoint="192.168.*.*:6443"
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```

 *This sets up the first control plane node with Flannel-compatible networking and shared certificates for HA.*

 -----
 
##  Apply Flannel CNI

```bash
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml

```

Your Kubernetes Master Node is now fully configured and ready to join worker nodes!

##  Make iptables Rules Persistent
```bash
sudo apt update
sudo apt install iptables iptables-persistent -y
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6
```

 *Ensures your firewall rules are saved and reloaded after reboot.*



---
----

##  Troubleshooting

### PKI Errors on `kubeadm init`

If you see:
```
error loading /etc/kubernetes/pki/...
```

 You probably forgot `--upload-certs` when using multiple control planes.

 ### CoreDNS or CNI Not Working?

```bash
sudo mkdir -p /opt/cni/bin
curl -Lo cni-plugins.tgz https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz
sudo tar -xzf cni-plugins.tgz -C /opt/cni/bin
sudo rm cni-plugins.tgz
sudo systemctl restart kubelet
```
### kubelet Not Listening on Port 6443?

```bash
sudo systemctl restart kubelet
sudo systemctl status kubelet
```



