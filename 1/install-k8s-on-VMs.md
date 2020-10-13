# Install Kubernetes on regular VMs (with Ubuntu 18.04 should work):
  
  Based on: 
  https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

  This method works for the most naive case, i.e. VMs are not behind a firewall, etc.
  
  Each VM shoud have 2 cores and 4 GB RAM.
  
  # Install tools for Kubernetes Master

  Enter root terminal:

    sudo -i
  
  Letting iptables see bridged traffic:

    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sudo sysctl --system  

  Install & enable docker:
  
    apt update
    apt install -y docker.io
    systemctl enable docker && systemctl start docker
  
  Install kubectl, kubeadm and kubelet:
  
    apt-get update && sudo apt-get install -y apt-transport-https curl
    
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

    cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF

    apt-get update

    apt-get install -y kubelet kubeadm kubectl
    
  Exit root terminal:

    exit

  (Optional) Set autocompletion for kubectl
  
    source <(kubectl completion bash)

# Start Kubernetes Master:
    
  Disable swap before running kubeadm:

    sudo swapoff -a

  Run kubeadm init with pod-cidr that is correct for Flannel network plugin and save the output to file:
 
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16 | tee kubeadm-init.out
  
  From the output run commands:
    
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
  Check weather kubectl show cluster node (should show one node in NotReady state)
  
    kubectl get nodes 

  Set a Network plugin, here we set Flannel:
  
    kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml

  Check pods in on the node (all pods should be running in a minute or so):

    kubectl get pods --all-namespaces

  (Optional) Enable scheduling on master node:
  
    kubectl taint nodes --all node-role.kubernetes.io/master-
    
    
# Grow the cluster - Join with Kubernetes Worker: 

  On each node install kubeadm (very similar steps that are for the master node expect here we need only kubeadm and kubelet)    

  Enter root terminal:

    sudo -i
  
  Letting iptables see bridged traffic:

    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sudo sysctl --system  

  Install & enable docker:
  
    apt update
    apt install -y docker.io
    systemctl enable docker && systemctl start docker
  
  Install kubelet and kubeadm:
  
    apt-get update && sudo apt-get install -y apt-transport-https curl
    
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

    cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF

    apt-get update

    apt-get install -y kubelet kubeadm

  Join with node to the master (the proper key is written in kubeadm init output on the master node):  
  
    # disable swap
    sudo swapoff -a

    cat kubeadm-init.out

    # on worker run something similar:
    kubeadm join 10.128.0.4:6443 --token rdnhok.g8mb6lfgesunanvh --discovery-token-ca-cert-hash sha256:66350d154fc0169b5bb5fd50c04b72468195e356d78d95f137ed55e995402f77
      
  Check nodes on master (should see two in Ready state):
  
    kubectl get nodes 

  Run test deployment:

    kubectl create deployment nginx --image=nginx 

    kubectl get pods -o wide

    curl <POD-IP>

# Destroy cluster node

    sudo kubeadm reset
