# ১. আপডেট ও টুলস
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# ২. GPG চাবি
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# ৩. রিপোজিটরি যোগ
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# ৪. ইন্সটল ও ফ্রিজ
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# ৫. kubelet চালু
sudo systemctl enable --now kubelet

# ৬. Swap বন্ধ
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# ৭. Containerd কনফিগ
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd

# ৮. ক্লাস্টার তৈরি
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# ৯. kubectl কনফিগ
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# ১০. Flannel নেটওয়ার্ক
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# ১১. ভেরিফিকেশন
kubectl get nodes
kubectl get pods --all-namespaces
