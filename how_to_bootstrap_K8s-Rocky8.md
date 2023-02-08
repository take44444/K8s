https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/

## Rocky Linuxのアップデート

```sh
dnf makecache --refresh
dnf update -y
reboot
```

## SELinuxをpermissiveモードに設定する

```sh
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## swapの無効化

```sh
swapoff -a
sed -e '/swap/s/^/#/g' -i /etc/fstab
free -m
```

## iptablesがブリッジを通過するトラフィックを処理できるようにする

```sh
lsmod | grep overlay
modprobe overlay
lsmod | grep br_netfilter
modprobe br_netfilter

cat > /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF

cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

## 必須ポートの確認 
### master

```sh
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10251/tcp
firewall-cmd --permanent --add-port=10252/tcp
firewall-cmd --reload
```

### worker

```sh
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
```

## ランタイムのインストール(ここではcontainerd)他にCRI-Oやdockerがある

```sh
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf makecahe
dnf install docker-ce -y
```

```sh
mv /etc/containerd/config.toml /etc/containerd/config.toml.orig
containerd config default > /etc/containerd/config.toml
vi /etc/containerd/config.toml
```

/etc/containerd/config.tomlの以下の場所を書き換えます

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd

```toml
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```

```sh
systemctl enable --now containerd.service
systemctl status containerd.service
systemctl enable --now docker.service
systemctl status docker.service
```

プロキシを利用している場合は以下を実施

```sh
mkdir /etc/systemd/system/docker.service.d
mkdir /etc/systemd/system/containerd.service.d
```

/etc/systemd/system/docker.service.d/http-proxy.confに以下を書き込む

```conf
[Service]
Environment="HTTP_PROXY=http://proxy-host:proxy-port/"
Environment="HTTPS_PROXY=http://proxy-host:proxy-port/"
Environment="NO_PROXY=localhost,127.0.0.0/8,localip_of_machine"
```

```sh
systemctl daemon-reload 
systemctl restart docker
```

.bashrcに以下を書き込む

```sh
export http_proxy=http://proxy-host:proxy-port/
export HTTP_PROXY=$http_proxy
export https_proxy=$http_proxy
export HTTPS_PROXY=$http_proxy
printf -v lan '%s,' localip_of_machine
printf -v pool '%s,' 10.244.{0..255}.{1..253}
printf -v service '%s,' 10.96.0.{1..253}
export no_proxy="${lan%,},${service%,},${pool%,},127.0.0.1";
export NO_PROXY=$no_proxy
```

```sh
source .bashrc
```

## kubeadm、kubelet、kubectlのインストール

see https://github.com/kubernetes/website/pull/34546
```sh
cat > /etc/yum.repos.d/k8s.repo << EOF 
[kubernetes] 
name=Kubernetes 
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1 
gpgcheck=1 
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg 
exclude=kubelet kubeadm kubectl 
EOF

dnf makecache
dnf install -y {kubelet,kubeadm,kubectl} --disableexcludes=kubernetes
```

```sh
systemctl enable --now kubelet
systemctl status kubelet
```

マスターノードの初期化(Flannelを使う場合はipは10.244.0.0/16を指定する必要がある)

```sh
kubeadm init --apiserver-advertise-address= localip_of_machine --service-cidr=10.96.0.0/24 --pod-network-cidr=10.244.0.0/16
```

以下のようなものが表示されること

また，これをコピーしておくこと
kubeadm join 192.168.2.201:6443 --token k9oipi.grpjhyspab2jjk9b \
        --discovery-token-ca-cert-hash sha256:841868d22f78bb3691cab610287f6df194b813e1fd564c9fbcd2808733739a9a

```sh
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

コマンドの入力補完を設定

```sh
$ echo "source <(kubectl completion bash)" >> $HOME/.bashrc
$ source ~/.bashrc
```

kubectlが使えることを確認

## PodネットワークAddon(Flannel)のインストール(masterノードのみ)

```sh
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
$ kubectl get pods -n kube-system
```

すべてRunningになるまで待つ

```sh
$ kubectl get node
```

Readyと表示されることを確認(Pod間通信が有効であることを確認)

## workerノードをクラスタにjoin

masterノードの~/.kube/configをworkerに転送

```sh
$ scp /home/master/.kube/config worker1@192.168.2.202:/home/worker1/
```

workerノードで，転送されたファイルを~/.kube/configに移動

```sh
$ mkdir .kube
$ mv config .kube/
```

/etc/containerd/config.tomlの以下の場所を書き換えます

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd

```toml
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```

```sh
$ containerd config default > config.toml
$ vi config.toml
$ cat config.toml | sudo tee /etc/containerd/config.toml
```

```sh
$ sudo systemctl restart containerd
```

workerノードで，コピペしておいた以下のコマンドを実行(sudo必要)

```sh
$ sudo kubeadm join 192.168.2.201:6443 --token k9oipi.grpjhyspab2jjk9b \
  --discovery-token-ca-cert-hash sha256:841868d22f78bb3691cab610287f6df194b813e1fd564c9fbcd2808733739a9a
```

workerがReadyになるまで待つ

```sh
$ kubectl get node
```

### コピペを忘れていた場合はmasterノードで確認

```sh
$ kubeadm token list
```

トークンの有効期限が切れてしまった場合は再発行

```sh
$ sudo kubeadm token create
```

CA証明書のhash値を出力

```sh
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

## workerラベルを付与(masterノードのみ)

```sh
$ kubectl label nodes k8sworker1 node-role.kubernetes.io/worker=""
```

```sh
$ kubectl get nodes
```

以下が表示されること

```sh
NAME         STATUS   ROLES           AGE     VERSION
k8smaster    Ready    control-plane   165m    v1.26.0
k8sworker1   Ready    worker          4m48s   v1.26.0
```
