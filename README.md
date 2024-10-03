# Kubernetes-
It consist of commands to setup the Kubernetes Master and Worker Node.



### Command to Run on both Master and Worker Node
#!/bin/bash
#
# Setup for Control Plane (Master) servers

set -euxo pipefail

# If you need public access to API server using the servers Public IP adress, change PUBLIC_IP_ACCESS to true.

PUBLIC_IP_ACCESS="true"
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"

# Pull required images

sudo kubeadm config images pull

# Initialize kubeadm based on PUBLIC_IP_ACCESS

if [[ "$PUBLIC_IP_ACCESS" == "false" ]]; then
    
    MASTER_PRIVATE_IP=$(ip addr show eth0 | awk '/inet / {print $2}' | cut -d/ -f1)
    sudo kubeadm init --apiserver-advertise-address="$MASTER_PRIVATE_IP" --apiserver-cert-extra-sans="$MASTER_PRIVATE_IP" --pod-network-cidr="$POD_CIDR" --node-name "$NODENAME" --ignore-preflight-errors Swap

elif [[ "$PUBLIC_IP_ACCESS" == "true" ]]; then

    MASTER_PUBLIC_IP=$(curl ifconfig.me && echo "")
    sudo kubeadm init --control-plane-endpoint="$MASTER_PUBLIC_IP" --apiserver-cert-extra-sans="$MASTER_PUBLIC_IP" --pod-network-cidr="$POD_CIDR" --node-name "$NODENAME" --ignore-preflight-errors Swap

else
    echo "Error: MASTER_PUBLIC_IP has an invalid value: $PUBLIC_IP_ACCESS"
    exit 1
fi

# Configure kubeconfig

mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config

# Install Claico Network Plugin Network 

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml


# =======================================================================================================================================================================
### Commands to run on only Master NOde
#!/bin/bash

# Exit on any error
set -e

# Update and install required packages
echo "Updating packages..."
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y apt-transport-https ca-certificates curl

# Add Kubernetes APT repository
echo "Adding Kubernetes APT repository..."
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update

# Install kubeadm, kubelet, and kubectl
echo "Installing Kubernetes components..."
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Disable swap
echo "Disabling swap..."
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab  # Disable swap permanently

# Initialize the Kubernetes master node
POD_NETWORK_CIDR="10.244.0.0/16"  # Change if needed
echo "Initializing Kubernetes master node..."
sudo kubeadm init --pod-network-cidr=$POD_NETWORK_CIDR

# Set up kubectl access
echo "Setting up kubectl access..."
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel network plugin
echo "Installing Flannel network plugin..."
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifest.yaml

# Allow scheduling on the master node (optional)
echo "Allowing scheduling on master node..."
kubectl taint nodes --all node-role.kubernetes.io/master-

echo "Kubernetes master node installation completed!"

# To join worker nodes to a Kubernetes master node, you need to generate a token on the master node and use it with the kubeadm join command.
kubeadm token create --print-join-command
# ===> After running this command it gives output which looks like this:
# kubeadm join 54.166.41.207:6443 --token rdwwik.7c1lzfql2yrdp0jt --discovery-token-ca-cert-hash sha256:809d0f762303a2f2bd4713d908b68ae0688213812f97c8d24e7c979c38c8d4e3

# Now go to Worker Nodes and run this command on worker nodes 
# As you execute the command you will be able to see this:

# =========================================================================================
# This node has joined the cluster:
# * Certificate signing request was sent to apiserver and a response was received.
# * The Kubelet was informed of the new secure connection details.

# Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
# ==========================================================================================
