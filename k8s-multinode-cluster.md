# 🎓 Kubernetes kubeadm ইন্সটলেশন গাইড — বাংলায়

> **সিস্টেম:** Ubuntu 24.04 LTS | **K8s ভার্সন:** v1.36

---

## ধাপ ১ — সিস্টেম আপডেট ও বেসিক টুলস

> 🛒 **উদাহরণ:** মোবাইলের Play Store রিফ্রেশ করলে যেমন নতুন অ্যাপের তালিকা আসে — এই কমান্ডও Ubuntu-র জন্য ঠিক সেটাই করে।

```bash
# প্যাকেজ তালিকা রিফ্রেশ করো
sudo apt-get update

# দরকারি টুলস ইন্সটল করো
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `sudo` | অ্যাডমিন পারমিশনে চালাও |
| `apt-get update` | প্যাকেজের তালিকা নামাও |
| `apt-transport-https` | HTTPS দিয়ে ডাউনলোডের সুবিধা |
| `ca-certificates` | ভুয়া সাইট চেনার সার্টিফিকেট |
| `curl` | ইন্টারনেট থেকে ফাইল আনার টুল |
| `gpg` | অথেন্টিক কিনা যাচাই করার টুল |
| `-y` | সব প্রশ্নে 'হ্যাঁ' বলে দাও |

---

## ধাপ ২ — Kubernetes-এর GPG চাবি ডাউনলোড

> 🔑 **উদাহরণ:** Kubernetes কোম্পানির আসল পরিচয়পত্র (ID card) ডাউনলোড করে সুরক্ষিত জায়গায় রাখছি — যাতে ভুয়া সফটওয়্যার না ঢোকে।

```bash
# চাবির আলমারি বানাও
sudo mkdir -p -m 755 /etc/apt/keyrings

# অফিসিয়াল চাবি নামিয়ে রাখো
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `mkdir -p` | ফোল্ডার বানাও (প্যারেন্টসহ) |
| `-m 755` | তুমি লিখতে পারো, বাকিরা শুধু পড়তে |
| `/etc/apt/keyrings` | চাবি রাখার নিরাপদ জায়গা |
| `curl -fsSL` | নীরবে ডাউনলোড করো, ভুল হলে জানাও |
| `\|` (পাইপ) | বাঁ দিকের রেজাল্ট ডান দিকে পাঠাও |
| `gpg --dearmor` | GPG চাবিকে বাইনারিতে বদলাও |
| `-o` | আউটপুট ফাইলের নাম দাও |

---

## ধাপ ৩ — Kubernetes রিপোজিটরি যোগ করা

> 🏪 **উদাহরণ:** ফোনের সেটিংসে নতুন অ্যাপ স্টোরের ঠিকানা লেখার মতো — এখন থেকে Ubuntu Kubernetes এর অ্যাপ ঐ ঠিকানা থেকে আনতে পারবে।

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `echo '...'` | একটা টেক্সট তৈরি করো |
| `deb [...]` | ডেবিয়ান প্যাকেজ সোর্সের ঠিকানা |
| `signed-by=...` | কোন চাবি দিয়ে যাচাই হবে |
| `sudo tee` | ফাইলে লেখো এবং স্ক্রিনেও দেখাও |
| `sources.list.d/` | Ubuntu-র অ্যাপ স্টোরের ঠিকানা রাখার জায়গা |

---

## ধাপ ৪ — kubeadm, kubelet, kubectl ইন্সটল

> 🔧 **উদাহরণ:** তিনজন কর্মী নিয়োগ — **kubeadm** = মিস্ত্রি (ক্লাস্টার বানায়), **kubelet** = কর্মী (পড চালায়), **kubectl** = বস (কমান্ড দেয়)।

```bash
# আবার রিফ্রেশ (নতুন স্টোর ধরতে)
sudo apt-get update

# তিনটি মূল টুলস ইন্সটল
sudo apt-get install -y kubelet kubeadm kubectl

# অটো-আপডেট বন্ধ (ফ্রিজ) করো — ক্লাস্টার ভাঙবে না
sudo apt-mark hold kubelet kubeadm kubectl
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `kubelet` | প্রতিটি মেশিনে পড চালু রাখার দায়িত্বে |
| `kubeadm` | ক্লাস্টার তৈরি ও কনফিগারের টুল |
| `kubectl` | ক্লাস্টারকে কমান্ড দেওয়ার হাতিয়ার |
| `apt-mark hold` | আপডেট আটকাও — ক্লাস্টার ভাঙবে না |

> ⚠️ **কেন hold করছি?** নতুন ভার্সন নিজে নিজে ইন্সটল হলে ক্লাস্টার ভেঙে যেতে পারে। ম্যানুয়ালি আপগ্রেড করতে হবে।

---

## ধাপ ৫ — kubelet সার্ভিস চালু করা

> ⚡ **উদাহরণ:** কর্মী (kubelet) কে বললাম — 'কম্পিউটার চালু হলেই কাজ শুরু করো এবং এখনই শুরু করো!'

```bash
# kubelet সক্রিয় ও এখনই চালু করো
sudo systemctl enable --now kubelet

# স্ট্যাটাস চেক করো
sudo systemctl status kubelet
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `systemctl` | সিস্টেম সার্ভিস কন্ট্রোলার |
| `enable` | বুট করার সময় অটো-স্টার্ট করবে |
| `--now` | এখনই স্টার্ট করো (অপেক্ষা না করে) |
| `status` | চালু আছে কিনা দেখো |

> ⚠️ **স্বাভাবিক:** `kubeadm init` চালানোর আগে kubelet বারবার restart নিতে পারে — এটা চিন্তার কিছু নেই!

---

## ধাপ ৬ — Swap বন্ধ করা (অত্যন্ত জরুরি!)

> 🚫 **উদাহরণ:** Kubernetes বলে — "আমি Swap ব্যবহার করব না!" তাই Swap বন্ধ না করলে ক্লাস্টার শুরুই হবে না।

```bash
# এখনই Swap বন্ধ করো
sudo swapoff -a

# রিবুটের পরও বন্ধ থাকবে (fstab এডিট)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `swapoff -a` | সব swap পার্টিশন বন্ধ করো |
| `sed -i` | ফাইল সরাসরি এডিট করো |
| `/ swap /` | এই শব্দ খোঁজো ফাইলে |
| `s/^(.*)$/#\1/g` | লাইনের আগে `#` বসিয়ে কমেন্ট করো |
| `/etc/fstab` | বুট টাইমে কী মাউন্ট হবে সেই ফাইল |

---

## ধাপ ৭ — Containerd কনফিগার করা

> 🍳 **উদাহরণ:** kubelet আর containerd দুজনকেই "systemd পদ্ধতিতে রিসোর্স ভাগ করো" বলতে হবে। মিল না থাকলে ঝগড়া হয়, ক্লাস্টার চলে না।

```bash
# ডিফল্ট কনফিগ তৈরি করো
sudo containerd config default | sudo tee /etc/containerd/config.toml

# SystemdCgroup চালু করো
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# containerd রিস্টার্ট দাও
sudo systemctl restart containerd
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `containerd config default` | ডিফল্ট কনফিগ দেখাও |
| `tee /etc/containerd/config.toml` | ফাইলে সেভ করো |
| `SystemdCgroup = false → true` | systemd পদ্ধতি চালু করো |
| `systemctl restart` | নতুন কনফিগ দিয়ে আবার চালু |

---

## ধাপ ৮ — ক্লাস্টার তৈরি করো (kubeadm init)

> 🏗️ **উদাহরণ:** মিস্ত্রি (kubeadm) কে বললাম — "একটা Kubernetes ক্লাস্টার বানাও, পডদের ঠিকানা হবে 10.244.x.x রেঞ্জে!"

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `kubeadm init` | নতুন ক্লাস্টার শুরু করো |
| `--pod-network-cidr` | পডদের IP ঠিকানার রেঞ্জ |
| `10.244.0.0/16` | Flannel নেটওয়ার্কের জন্য এই রেঞ্জ দরকার |

> ✅ **গুরুত্বপূর্ণ:** সফল হলে শেষে নিচের মতো একটা কমান্ড দেখাবে — সেটা **কপি করে রেখো!** Worker Node যোগ করতে লাগবে।
>
> ```
> kubeadm join 192.168.1.100:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
> ```

---

## ধাপ ৯ — kubectl কনফিগার করা

> 🪪 **উদাহরণ:** বস (kubectl) কে ক্লাস্টারের সাথে কথা বলতে হলে পরিচয়পত্র লাগে। এই কমান্ড সেই পরিচয়পত্র তোমার ফোল্ডারে রাখে।

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `mkdir -p $HOME/.kube` | `~/.kube` ফোল্ডার তৈরি করো |
| `cp -i admin.conf` | অ্যাডমিন পরিচয়পত্র কপি করো |
| `chown $(id -u):$(id -g)` | ফাইলটার মালিক তুমি হও |

---

## ধাপ ১০ — Flannel নেটওয়ার্ক প্লাগইন ইন্সটল

> 🌐 **উদাহরণ:** পডরা যেন একে অপরের সাথে কথা বলতে পারে তার জন্য নেটওয়ার্ক প্লাগইন লাগে — Flannel হলো সেই 'ফোন লাইন'।

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

| কমান্ড/অপশন | মানে কী |
|---|---|
| `kubectl apply` | কনফিগারেশন প্রয়োগ করো |
| `-f` | ফাইল বা URL থেকে পড়ো |
| `kube-flannel.yml` | Flannel-এর সম্পূর্ণ কনফিগ ফাইল |

---

## ধাপ ১১ — সব কিছু ঠিক আছে কিনা চেক

> ✅ **উদাহরণ:** হাজিরা খাতা — ক্লাসে (ক্লাস্টারে) কে কে আছে এবং তারা সুস্থ কিনা দেখো।

```bash
# নোডের অবস্থা দেখো
kubectl get nodes

# সব পডের অবস্থা দেখো
kubectl get pods --all-namespaces
```

**আশানুরূপ আউটপুট:**

```
NAME         STATUS   ROLES           AGE   VERSION
workspace    Ready    control-plane   2m    v1.36.0
```

`STATUS: Ready` দেখলে বুঝবে সব ঠিকঠাক! 🎉

---

## বোনাস — Worker Node যোগ করা

> 👨‍🎓 **উদাহরণ:** নতুন ছাত্র (Worker) কে ক্লাসে অ্যাডমিট করার ফরম পূরণ করা। টোকেন হলো তার আইডি কার্ড।

```bash
# Worker Node-এ বসে এই কমান্ড চালাও
sudo kubeadm join 192.168.1.100:6443 \
  --token <টোকেন> \
  --discovery-token-ca-cert-hash sha256:<হ্যাশ>
```

**টোকেন ভুলে গেলে — Control Plane থেকে দেখো:**

```bash
# টোকেন লিস্ট দেখো
kubeadm token list

# নতুন টোকেন বানাও
kubeadm token create
```

---

## 📋 দ্রুত রেফারেন্স — সব কমান্ড একসাথে

```bash
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
```

---

*Ubuntu 24.04 LTS — Kubernetes v1.36 — বাংলায় লেখা*
