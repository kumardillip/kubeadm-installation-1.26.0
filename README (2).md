


![kubernetes-1 26](https://user-images.githubusercontent.com/120181043/209433967-00b5546a-d9a2-4ecf-a827-9c2dc81c4778.png)
# Kubernetes 1.26 - The electrifying release setup

## Now that we know some of the cool features, let's set up a Kubernetes cluster on Ubuntu 20.04 machines for version 1.26.

- Prerequisites

  2 Ubuntu 20.04 instances with ssh access to them, you can use any cloud provider to launch these instances

  Each instance should have a minimum of 4GB of ram
## Step 1 - Run this on all the machines
- Switch to an interactive session as a root user :
          
      sudo su
- Kubeadm | kubectl | kubelet install
          
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
      sudo apt update -y
      sudo apt -y install vim git curl wget kubelet=1.26.0-00 kubeadm=1.26.0-00 kubectl=1.26.0-00
      sudo apt-mark hold kubelet kubeadm kubectl

- Load the br_netfilter module and let iptables see bridged traffic
          
      sudo modprobe overlay
      sudo modprobe br_netfilter
      sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
      net.bridge.bridge-nf-call-ip6tables = 1
      net.bridge.bridge-nf-call-iptables = 1
      net.ipv4.ip_forward = 1
      EOF
      sysctl --system

- Setup Containerd

      cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
      overlay
      br_netfilter
      EOF


      sudo modprobe overlay
      sudo modprobe br_netfilter

- Setup required sysctl params, these persist across reboots.
         
      cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
      net.bridge.bridge-nf-call-iptables  = 1
      net.ipv4.ip_forward = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      EOF

-  Apply sysctl params without reboot
         
       sudo sysctl --system

- Install and configure containerd 

      curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
      sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
      sudo apt update -y
      sudo apt install -y containerd.io
      sudo mkdir -p /etc/containerd

#  
    containerd config default | sudo tee /etc/containerd/config.toml

- Start containerd

      sudo systemctl restart containerd
      sudo systemctl enable containerd
- Pull the images, pulls the images for Kubernetes 1.26 version .

      sudo kubeadm config images pull --image-repository=registry.k8s.io --cri-socket unix:///run/containerd/containerd.sock --kubernetes-version v1.26.0

## Step2 - Run the kubeadm init command on the control plane node

- The new release images will now be under registry.k8s.io - This will provide faster downloads and also removes a single point of failure

      kubeadm init --image-repository=registry.k8s.io

- Export KUBECONFIG 

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

- deploy a pod network to the cluster on master node

      kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml 

- This is command to generate a token :

      kubeadm token create --print-join-command
## Step 3 - Run the join command on all the worker nodes
- Then you can join any number of worker nodes by running the following on each node as root(copy from master):Then you can join any number of worker nodes by running the following on each node as root(copy from master):
  
      kubeadm join 74.220.27.73:6443 --token 3y24ca.kq73lohh99nzmcl5 \
      --discovery-token-ca-cert-hash sha256:f22dadb9c02bd9ac69b1819cbeaa11330ee70bb5fb6343f8b8a288b9ea83b00f \
      --control-plane --certificate-key 74bfd9237ded9661ca3ee337057caba0be417c19b6493034ec0da3dbcffc8fff
 
 ![Screenshot (118)](https://user-images.githubusercontent.com/120181043/209434695-75258ab9-742a-44fb-adb9-81c5de716b66.png)


- references - 

  https://kubernetes.io/blog/2022/12/09/kubernetes-v1-26-release/

  https://blog.kubesimplify.com/kubernetes-126
  
  https://github.com/containerd/containerd/blob/main/docs/getting-started.md
