# K8s local installation quick start
This repo is a quick start tutorial for running K8s locally on VMWare Desktop. There is no much theory and mostly listing down essential steps to get things running.
You can imagine this is WiP :)
Heabily inspired by this guide - https://dev.to/iamzahid/setting-up-a-kubernetes-with-crio-3f3h

## Tools needed
- Helm CLI
- Kubectl
  
## Install VMWare desktop
You may need to register on Broadcom web site to get setup dist. Pretty sure you can handle that :)

## Nodes setup
Nodes setup will look the following:
- 192.168.60.100 master-node
- 192.168.60.101 worker1-node
- 192.168.60.102 worker2-node

You'll have to create additional Host network 192.168.60.0/24 (no DHCP) and add second adapter pointing to that network. NAT network should be present as well to enable Internet access.
Recommended specs:
- 4 CPUs
- 4Gb RAM
- 20+Gb HDD

## Nodes OS install
Ubutu installation is done using .iso image and you have to enable OpenSSH. During installation second adapter is configured manually with node IP address.

## OS configuration (all nodes)
Comment out swap mount in /etc/fstab and run:
`sudo swapoff -a`

Run modules setup and forwarding:
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```
sudo modprobe br_netfilter
sudo modprobe overlay
```
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
```
sudo sysctl --system
sysctl net.ipv4.ip_forward
sysctl -w net.ipv4.ip_forward=1
```

## Install Kubernetes (all nodes)
Current install version is 1.33, however you may want to replace it with more up to date.
```
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

## Install container runtime (all nodes)
We using CRIo as container runtime.
```
CRIO_VERSION=v1.33
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list
```
```
sudo apt-get update
sudo apt-get install -y cri-o
systemctl start crio.service
sudo systemctl status crio
sudo systemctl daemon-reload
sudo systemctl enable crio --now
```

## Init K8s (master node)
```
sudo kubeadm init --apiserver-advertise-address 192.168.60.100 --control-plane-endpoint 192.168.60.100 --pod-network-cidr=10.244.0.0/16
```

## Join worker nodes
Run join command from init output on worker nodes.
This ends separate nodes installs and all other steps are executed either locally or on master node (you'll need kubectl configured)

## Network layer (CNI) install (master node)
Cilium is used as CNI
```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
```
cilium install --set nodePort.enabled=true --set ingressController.enabled=true --set ingressController.default=true --set ingressController.loadbalancerMode=dedicated
```
------
## Hubble install for observability - optional
```
cilium hubble enable 
```
```
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

## MetalLB load balancer install 
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

## Rancher install - WiP
```
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set crds.enabled=true
```
```
helm upgrade --install rancher .\rancher\rancher\ --namespace cattle-system --set hostname=rancher --set replicas=1 --set bootstrapPassword=admin --create-namespace
```