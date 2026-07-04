You can now join any number of control-plane nodes running the following command on each as root:

  kubeadm join 192.168.0.99:6443 --token z5tbvt.h9qj1gxo7731l3wm \
        --discovery-token-ca-cert-hash sha256:fae47d16ffc1169d3f90b5e3ba29af82ab9a34fb74d4180c4d16f36111528f4d \        --control-plane --certificate-key b769e8e81013c7d835009f77a88f051644609351880dda79169dcb71e3459fe2

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.99:6443 --token z5tbvt.h9qj1gxo7731l3wm --discovery-token-ca-cert-hash sha256:fae47d16ffc1169d3f90b5e3ba29af82ab9a34fb74d4180c4d16f36111528f4d 