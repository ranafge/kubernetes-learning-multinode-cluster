# Kubernetes Setup Guide (Ubuntu)

---

# ## ধাপ ০ — ফায়ারওয়াল বন্ধ করা (UFW Disable)

### why need:

Kubernetes নোডগুলোর মধ্যে শত শত পোর্টে যোগাযোগ করে (API Server: 6443, kubelet: 10250, etcd: 2379-2380 ইত্যাদি)। UFW চালু থাকলে এই যোগাযোগ ব্লক হয়ে যায়, ফলে ক্লাস্টার তৈরি হয় না বা অস্থির আচরণ করে। ল্যাব/ডেভ এনভায়রনমেন্টে UFW বন্ধ করাই সহজ ও দ্রুত সমাধান।

| কমান্ড/অপশন                        | মানে কী                                                      |
| ---------------------------------- | ------------------------------------------------------------ |
| `sudo ufw status`                  | বর্তমানে ফায়ারওয়াল চালু না বন্ধ আছে দেখায়                 |
| `sudo ufw disable`                 | ফায়ারওয়াল সম্পূর্ণ বন্ধ করে দেয়                           |
| `sudo systemctl disable --now ufw` | এখনই বন্ধ করে এবং বুট করার পরেও যাতে না চলে সেটি নিশ্চিত করে |

### command

```bash
sudo ufw disable
```

---

# ## ধাপ ১ — সিস্টেম আপডেট ও বেসিক টুলস

### why need:

Kubernetes ইন্সটল করার আগে সিস্টেম আপডেট করা জরুরি, কারণ পুরনো প্যাকেজ ক্লাস্টারে সমস্যা তৈরি করতে পারে। আর `apt-transport-https`, `ca-certificates`, `curl`, `gpg` টুলসগুলো Kubernetes রিপোজিটরি থেকে সুরক্ষিতভাবে ডাউনলোড করার জন্য দরকার।

| কমান্ড/অপশন           | মানে কী                      |
| --------------------- | ---------------------------- |
| `sudo`                | অ্যাডমিন পারমিশনে চালাও      |
| `apt-get update`      | প্যাকেজের তালিকা নামাও       |
| `apt-transport-https` | HTTPS দিয়ে ডাউনলোডের সুবিধা |
| `ca-certificates`     | ভুয়া সাইট চেনার সার্টিফিকেট |
| `curl`                | ইন্টারনেট থেকে ফাইল আনার টুল |
| `gpg`                 | অথেন্টিক কিনা যাচাই করার টুল |
| `-y`                  | সব প্রশ্নে 'হ্যাঁ' বলে দাও   |

### command

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

---

# ## ধাপ ২ — overlay ও br_netfilter কার্নেল মডিউল লোড করা

### why need:

`overlay` মডিউল container runtime (containerd) কে overlay filesystem ব্যবহার করতে দেয়, যা container ইমেজ লেয়ার ম্যানেজের জন্য জরুরি। `br_netfilter` মডিউল bridging ট্রাফিক iptables-এর মাধ্যমে ফিল্টার করতে দেয় — যা Kubernetes নেটওয়ার্ক পলিসি ও Service ইত্যাদির জন্য অপরিহার্য।

| কমান্ড/অপশন                           | মানে কী                                                |
| ------------------------------------- | ------------------------------------------------------ |
| `modprobe overlay`                    | overlay filesystem সাপোর্ট কার্নেলে লোড করে            |
| `modprobe br_netfilter`               | bridge netfilter সাপোর্ট লোড করে                       |
| `/etc/modules-load.d/containerd.conf` | বুট করার সময় যাতে নিজে নিজে লোড হয় সেজন্য কনফিগ ফাইল |

### command

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

---

# ## ধাপ ৩ — Kubernetes-এর GPG চাবি ডাউনলোড

### why need:

GPG চাবি দিয়ে যাচাই করা হয় যে ডাউনলোড করা Kubernetes প্যাকেজগুলো আসল আর ভুয়া নয়। এটা সিকিউরিটি নিশ্চিত করে — যেন কেউ মাঝপথে ভুয়া প্যাকেজ ঢুকাতে না পারে।

| কমান্ড/অপশন         | মানে কী                            |
| ------------------- | ---------------------------------- |
| `mkdir -p`          | ফোল্ডার বানাও (প্যারেন্টসহ)        |
| `-m 755`            | তুমি লিখতে পারো, বাকিরা শুধু পড়তে |
| `/etc/apt/keyrings` | চাবি রাখার নিরাপদ জায়গা           |
| `curl -fsSL`        | নীরবে ডাউনলোড করো, ভুল হলে জানাও   |
| `\|` (পাইপ)         | বাঁ দিকের রেজাল্ট ডান দিকে পাঠাও   |
| `gpg --dearmor`     | GPG চাবিকে বাইনারিতে বদলাও         |
| `-o`                | আউটপুট ফাইলের নাম দাও              |

### command

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

---

# ## ধাপ ৪ — Kubernetes রিপোজিটরি যোগ করা

### why need:

রিপোজিটরি যোগ করলে Ubuntu জানে কোথা থেকে Kubernetes প্যাকেজ ডাউনলোড করতে হবে। এটা ঠিকানা দেওয়ার মতো — যাতে `apt-get` কমান্ড সঠিক জায়গা থেকে `kubeadm`, `kubelet`, `kubectl` খুঁজে পায়।

| কমান্ড/অপশন       | মানে কী                                    |
| ----------------- | ------------------------------------------ |
| `echo '...'`      | একটা টেক্সট তৈরি করো                       |
| `deb [...]`       | ডেবিয়ান প্যাকেজ সোর্সের ঠিকানা            |
| `signed-by=...`   | কোন চাবি দিয়ে যাচাই হবে                   |
| `sudo tee`        | ফাইলে লেখো এবং স্ক্রিনেও দেখাও             |
| `sources.list.d/` | Ubuntu-র অ্যাপ স্টোরের ঠিকানা রাখার জায়গা |

### command

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

---

# ## ধাপ ৫ — kubeadm, kubelet, kubectl ইন্সটল

### why need:

`kubeadm` ক্লাস্টার তৈরি করে, `kubelet` প্রতিটি নোডে পড চালায়, `kubectl` ক্লাস্টার কন্ট্রোল করে। তিনটিই Kubernetes-এর মূল উপাদান। `apt-mark hold` দেওয়া হয় কারণ অটো-আপডেট হলে ক্লাস্টার ভেঙে যেতে পারে।

| কমান্ড/অপশন     | মানে কী                                |
| --------------- | -------------------------------------- |
| `kubelet`       | প্রতিটি মেশিনে পড চালু রাখার দায়িত্বে |
| `kubeadm`       | ক্লাস্টার তৈরি ও কনফিগারের টুল         |
| `kubectl`       | ক্লাস্টারকে কমান্ড দেওয়ার হাতিয়ার    |
| `apt-mark hold` | আপডেট আটকাও — ক্লাস্টার ভাঙবে না       |

### command

```bash
sudo apt-get update && \
sudo apt-get install -y kubelet kubeadm kubectl && \
sudo apt-mark hold kubelet kubeadm kubectl
```

---

# ## ধাপ ৬ — kubelet সার্ভিস চালু করা

### why need:

`kubelet` সার্ভিস চালু না থাকলে Kubernetes কোনো পড চালাতে পারবে না। `enable` দেওয়া হয় যেন কম্পিউটার রিবুট করার পরেও নিজে নিজে চালু হয়।

| কমান্ড/অপশন | মানে কী                        |
| ----------- | ------------------------------ |
| `systemctl` | সিস্টেম সার্ভিস কন্ট্রোলার     |
| `enable`    | বুট করার সময় অটো-স্টার্ট করবে |
| `--now`     | এখনই স্টার্ট করো               |
| `status`    | চালু আছে কিনা দেখো             |

### command

```bash
sudo systemctl enable --now kubelet
sudo systemctl status kubelet
```

---

# ## ধাপ ৭ — Swap বন্ধ করা (অত্যন্ত জরুরি!)

### why need:

Kubernetes ডিজাইনে Swap ব্যবহার করা নিষিদ্ধ। Swap চালু থাকলে `kubelet` সঠিকভাবে কাজ করে না, ক্লাস্টার শুরু হয় না। তাই Swap বন্ধ করতেই হবে।

| কমান্ড/অপশন      | মানে কী                          |
| ---------------- | -------------------------------- |
| `swapoff -a`     | সব swap পার্টিশন বন্ধ করো        |
| `sed -i`         | ফাইল সরাসরি এডিট করো             |
| `/ swap /`       | এই শব্দ খোঁজো ফাইলে              |
| `s/^(.*)$/#\1/g` | লাইনের আগে `#` বসিয়ে কমেন্ট করো |
| `/etc/fstab`     | বুট টাইমে কী মাউন্ট হবে সেই ফাইল |

### command

```bash
sudo swapoff -a

sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

# ধাপ ১: containerd ইনস্টল করুন
```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y containerd.io
```

| কমান্ড | সংক্ষিপ্ত অর্থ |
|--------|---------------|
| `sudo mkdir -p /etc/containerd` | containerd-এর কনফিগ ফাইল রাখার জন্য ডিরেক্টরি তৈরি করো |
| `sudo containerd config default \| sudo tee /etc/containerd/config.toml` | ডিফল্ট কনফিগারেশন জেনারেট করে ফাইলে সেভ করো |
| `sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml` | SystemdCgroup false-কে true-তে পরিবর্তন করো (kubelet-এর সাথে সামঞ্জস্যের জন্য) |
| `sudo systemctl restart containerd` | নতুন কনফিগ লোড করতে containerd রিস্টার্ট করো |
| `sudo systemctl enable containerd` | সিস্টেম বুট হলে containerd অটোমেটিক চালু হবে |

# ধাপ ৩: ভেরিফাই করুন
```bash
sudo systemctl status containerd --no-pager
grep SystemdCgroup /etc/containerd/config.toml
```

# ## ধাপ ৯ — ক্লাস্টার তৈরি করো (kubeadm init)

### why need:

এই কমান্ডটাই আসলে Kubernetes ক্লাস্টার তৈরি করে। Control Plane নোড সেটআপ করে, API Server, etcd, Controller Manager সব চালু করে। `--pod-network-cidr` Flannel-এর জন্য নির্দিষ্ট রেঞ্জ দেয়।

| কমান্ড/অপশন          | মানে কী                                  |
| -------------------- | ---------------------------------------- |
| `kubeadm init`       | নতুন ক্লাস্টার শুরু করো                  |
| `--pod-network-cidr` | পডদের IP ঠিকানার রেঞ্জ                   |
| `10.244.0.0/16`      | Flannel নেটওয়ার্কের জন্য এই রেঞ্জ দরকার |

### command

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

---

# ## ধাপ ১০ — kubectl কনফিগার করা

### why need:

`kubectl` ক্লাস্টারের সাথে কথা বলতে চাইলে পরিচয়পত্র লাগে। `admin.conf` ফাইলেই সেই পরিচয়পত্র থাকে। `~/.kube/config` এ রাখলে `kubectl` নিজে নিজে খুঁজে পায়।

| কমান্ড/অপশন               | মানে কী                     |
| ------------------------- | --------------------------- |
| `mkdir -p $HOME/.kube`    | `~/.kube` ফোল্ডার তৈরি করো  |
| `cp -i admin.conf`        | অ্যাডমিন পরিচয়পত্র কপি করো |
| `chown $(id -u):$(id -g)` | ফাইলটার মালিক তুমি হও       |

### command

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

# ## ধাপ ১১ — Flannel নেটওয়ার্ক প্লাগইন ইন্সটল

### why need:

Kubernetes পডরা যেন একে অপরের সাথে কথা বলতে পারে তার জন্য নেটওয়ার্ক প্লাগইন দরকার। Flannel হলো একটি ওভারলে নেটওয়ার্ক যা পডদের IP ঠিকানা দিয়ে ক্লাস্টারের ভিতরে যোগাযোগ করতে সাহায্য করে।

| কমান্ড/অপশন        | মানে কী                        |
| ------------------ | ------------------------------ |
| `kubectl apply`    | কনফিগারেশন প্রয়োগ করো         |
| `-f`               | ফাইল বা URL থেকে পড়ো          |
| `kube-flannel.yml` | Flannel-এর সম্পূর্ণ কনফিগ ফাইল |

### command

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

---

# ✅ Verification Commands

```bash
kubectl get nodes
kubectl get pods -A
```

---

# ✅ Expected Output

```bash
NAME           STATUS   ROLES           AGE   VERSION
master-node    Ready    control-plane   10m   v1.36.0
```

---

# ✅ Notes

* Ubuntu 22.04 / 24.04 recommended
* Minimum 2 CPU & 2GB RAM
* Disable Swap before cluster init
* Use same Kubernetes version across nodes
* For production use proper firewall rules instead of disabling UFW

---
