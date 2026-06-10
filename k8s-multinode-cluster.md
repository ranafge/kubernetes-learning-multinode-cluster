## ধাপ ১ — সিস্টেম আপডেট ও বেসিক টুলস

**why need:** Kubernetes ইন্সটল করার আগে সিস্টেম আপডেট করা জরুরি, কারণ পুরনো প্যাকেজ ক্লাস্টারে সমস্যা তৈরি করতে পারে। আর apt-transport-https, ca-certificates, curl, gpg টুলসগুলো Kubernetes রিপোজিটরি থেকে সুরক্ষিতভাবে ডাউনলোড করার জন্য দরকার।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `sudo` | অ্যাডমিন পারমিশনে চালাও |
| `apt-get update` | প্যাকেজের তালিকা নামাও |
| `apt-transport-https` | HTTPS দিয়ে ডাউনলোডের সুবিধা |
| `ca-certificates` | ভুয়া সাইট চেনার সার্টিফিকেট |
| `curl` | ইন্টারনেট থেকে ফাইল আনার টুল |
| `gpg` | অথেন্টিক কিনা যাচাই করার টুল |
| `-y` | সব প্রশ্নে 'হ্যাঁ' বলে দাও |

command: `sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gpg`

---

## ধাপ ২ — Kubernetes-এর GPG চাবি ডাউনলোড

**why need:** GPG চাবি দিয়ে যাচাই করা হয় যে ডাউনলোড করা Kubernetes প্যাকেজগুলো আসল আর ভুয়া নয়। এটা সিকিউরিটি নিশ্চিত করে — যেন কেউ মাঝপথে ভুয়া প্যাকেজ ঢুকাতে না পারে।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `mkdir -p` | ফোল্ডার বানাও (প্যারেন্টসহ) |
| `-m 755` | তুমি লিখতে পারো, বাকিরা শুধু পড়তে |
| `/etc/apt/keyrings` | চাবি রাখার নিরাপদ জায়গা |
| `curl -fsSL` | নীরবে ডাউনলোড করো, ভুল হলে জানাও |
| `\|` (পাইপ) | বাঁ দিকের রেজাল্ট ডান দিকে পাঠাও |
| `gpg --dearmor` | GPG চাবিকে বাইনারিতে বদলাও |
| `-o` | আউটপুট ফাইলের নাম দাও |

command: `sudo mkdir -p -m 755 /etc/apt/keyrings && curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`

---

## ধাপ ৩ — Kubernetes রিপোজিটরি যোগ করা

**why need:** রিপোজিটরি যোগ করলে Ubuntu জানে কোথা থেকে Kubernetes প্যাকেজ ডাউনলোড করতে হবে। এটা ঠিকানা দেওয়ার মতো — যাতে apt-get কমান্ড সঠিক জায়গা থেকে kubeadm, kubelet, kubectl খুঁজে পায়।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `echo '...'` | একটা টেক্সট তৈরি করো |
| `deb [...]` | ডেবিয়ান প্যাকেজ সোর্সের ঠিকানা |
| `signed-by=...` | কোন চাবি দিয়ে যাচাই হবে |
| `sudo tee` | ফাইলে লেখো এবং স্ক্রিনেও দেখাও |
| `sources.list.d/` | Ubuntu-র অ্যাপ স্টোরের ঠিকানা রাখার জায়গা |

command: `echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list`

---

## ধাপ ৪ — kubeadm, kubelet, kubectl ইন্সটল

**why need:** kubeadm ক্লাস্টার তৈরি করে, kubelet প্রতিটি নোডে পড চালায়, kubectl ক্লাস্টার কন্ট্রোল করে। তিনটিই Kubernetes-এর মূল উপাদান। apt-mark hold দেওয়া হয় কারণ অটো-আপডেট হলে ক্লাস্টার ভেঙে যেতে পারে।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `kubelet` | প্রতিটি মেশিনে পড চালু রাখার দায়িত্বে |
| `kubeadm` | ক্লাস্টার তৈরি ও কনফিগারের টুল |
| `kubectl` | ক্লাস্টারকে কমান্ড দেওয়ার হাতিয়ার |
| `apt-mark hold` | আপডেট আটকাও — ক্লাস্টার ভাঙবে না |

command: `sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl && sudo apt-mark hold kubelet kubeadm kubectl`

---

## ধাপ ৫ — kubelet সার্ভিস চালু করা

**why need:** kubelet সার্ভিস চালু না থাকলে Kubernetes কোনো পড চালাতে পারবে না। enable দেওয়া হয় যেন কম্পিউটার রিবুট করার পরেও নিজে নিজে চালু হয়।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `systemctl` | সিস্টেম সার্ভিস কন্ট্রোলার |
| `enable` | বুট করার সময় অটো-স্টার্ট করবে |
| `--now` | এখনই স্টার্ট করো (অপেক্ষা না করে) |
| `status` | চালু আছে কিনা দেখো |

command: `sudo systemctl enable --now kubelet && sudo systemctl status kubelet`

---

## ধাপ ৬ — Swap বন্ধ করা (অত্যন্ত জরুরি!)

**why need:** Kubernetes ডিজাইনে Swap ব্যবহার করা নিষিদ্ধ। Swap চালু থাকলে kubelet সঠিকভাবে কাজ করে না, ক্লাস্টার শুরু হয় না। তাই Swap বন্ধ করতেই হবে।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `swapoff -a` | সব swap পার্টিশন বন্ধ করো |
| `sed -i` | ফাইল সরাসরি এডিট করো |
| `/ swap /` | এই শব্দ খোঁজো ফাইলে |
| `s/^(.*)$/#\1/g` | লাইনের আগে `#` বসিয়ে কমেন্ট করো |
| `/etc/fstab` | বুট টাইমে কী মাউন্ট হবে সেই ফাইল |

command: `sudo swapoff -a && sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab`

---

## ধাপ ৭ — Containerd কনফিগার করা

**why need:** Kubernetes পড চালানোর জন্য container runtime দরকার — এখানে containerd। SystemdCgroup চালু না করলে kubelet আর containerd-এর মধ্যে রিসোর্স ম্যানেজমেন্টে গন্ডগোল হয়, যার ফলে ক্লাস্টার অস্থির হয়।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `containerd config default` | ডিফল্ট কনফিগ দেখাও |
| `tee /etc/containerd/config.toml` | ফাইলে সেভ করো |
| `SystemdCgroup = false → true` | systemd পদ্ধতি চালু করো |
| `systemctl restart` | নতুন কনফিগ দিয়ে আবার চালু |

command: `sudo containerd config default | sudo tee /etc/containerd/config.toml && sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml && sudo systemctl restart containerd`

---

## ধাপ ৮ — ক্লাস্টার তৈরি করো (kubeadm init)

**why need:** এই কমান্ডটাই আসলে Kubernetes ক্লাস্টার তৈরি করে। Control Plane নোড সেটআপ করে, API Server, etcd, Controller Manager সব চালু করে। --pod-network-cidr ফ্লানেলের জন্য নির্দিষ্ট রেঞ্জ দেয়।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `kubeadm init` | নতুন ক্লাস্টার শুরু করো |
| `--pod-network-cidr` | পডদের IP ঠিকানার রেঞ্জ |
| `10.244.0.0/16` | Flannel নেটওয়ার্কের জন্য এই রেঞ্জ দরকার |

command: `sudo kubeadm init --pod-network-cidr=10.244.0.0/16`

---

## ধাপ ৯ — kubectl কনফিগার করা

**why need:** kubectl ক্লাস্টারের সাথে কথা বলতে চাইলে পরিচয়পত্র লাগে। admin.conf ফাইলেই সেই পরিচয়পত্র থাকে। ~/.kube/config এ রাখলে kubectl নিজে নিজে খুঁজে পায়, আর বারবার টোকেন দিতে হয় না।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `mkdir -p $HOME/.kube` | `~/.kube` ফোল্ডার তৈরি করো |
| `cp -i admin.conf` | অ্যাডমিন পরিচয়পত্র কপি করো |
| `chown $(id -u):$(id -g)` | ফাইলটার মালিক তুমি হও |

command: `mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config`

---

## ধাপ ১০ — Flannel নেটওয়ার্ক প্লাগইন ইন্সটল

**why need:** Kubernetes পডরা যেন একে অপরের সাথে কথা বলতে পারে তার জন্য নেটওয়ার্ক প্লাগইন দরকার। Flannel হলো একটি ওভারলে নেটওয়ার্ক যা পডদের IP ঠিকানা দিয়ে ক্লাস্টারের ভিতরে যোগাযোগ করতে সাহায্য করে। Flannel ছাড়া পডরা কানেক্ট করতে পারে না।

| কমান্ড/অপশন | মানে কী |
| --- | --- |
| `kubectl apply` | কনফিগারেশন প্রয়োগ করো |
| `-f` | ফাইল বা URL থেকে পড়ো |
| `kube-flannel.yml` | Flannel-এর সম্পূর্ণ কনফিগ ফাইল |

command: `kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml`