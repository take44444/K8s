https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/
## swapの無効化

```sh
$ sudo swapoff -a
$ sudo systemctl disable dphys-swapfile.service
$ sudo swapon --show
```

## memoryサブシステムの有効化

```sh
$ cat /proc/cgroups | grep memory | awk '{print $1,$4}'
```

memory 0と表示された場合は無効化されているので次のコマンドを実行

```sh
$ sudo sed -i \
"s/$/ cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory/" \
/boot/cmdline.txt
$ cat /boot/cmdline.txt
```

上記2つの設定を反映させるため再起動

```sh
$ sudo reboot
```

```sh
$ sudo systemctl status dphys-swapfile.service
```

activeではないことを確認

```sh
$ cat /proc/cgroups | grep memory | awk '{print $1,$4}'
```

memoryサブシステムが有効化されていることを確認

## iptablesがnftablesバックエンドを使用しないようにする

レガシーバイナリがインストールされていることを確認

```sh
$ sudo apt install -y iptables arptables ebtables
```

レガシーバージョンに切り替え

```sh
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```

## ランタイムのインストール(ここではdocker)他にCRI-Oやcontainerdがある

```sh
$ curl -sSL https://get.docker.com | sh
$ sudo usermod -aG docker ????
$ sudo reboot
$ docker ps
```

## kubeadm、kubelet、kubectlのインストール

```sh
$ sudo apt update && sudo apt install -y apt-transport-https curl
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
$ sudo apt update
$ sudo apt install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```

ここまではすべてのノードで行う

## k8sクラスタの構築(masterノードのみ)

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

マスターノードの初期化(Flannelを使う場合はipは10.244.0.0/16を指定する必要がある)

```sh
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
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

```sh
$ kubectl get node
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