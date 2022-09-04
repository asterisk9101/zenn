---
title: "FedoraサーバーでKubernetesを使えるようにするときのメモ"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora36, asnible, kubernetes]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 概要と前提

仮想マシン3台(Fedora36 Server)を使って `Kubernetes` を稼働させます。仮想マシンのスペックは、それぞれ 2Core/3GB RAM です。また、仮想マシンの構築に使用する端末として `windows` 上の `wsl` を使います。

`wsl` には、初期構築を行うための `ansible` を用意し、`ssh` で各仮想マシンに接続できることを確認しておきます。`ssh` の認証にはパスワードを使用します。

`Kubernetes` のインストールには `kubeadm` を使用します。ランタイムは `CRI-O`、ネットワークプラグインは `calico` を使用します。

## 仮想マシンの初期設定

`inventory` として以下のファイルを用意します。

```inventory
[fedora36]
192.168.1.22 # controller
192.168.1.23 # worker
192.168.1.24 # worker
```

構築のためのプレイブックを用意します。ファイル名は `part1.yaml` とします。

```yaml
---
- name: part1.yaml
  hosts: fedora36
  become: yes
  gather_facts: no
  tasks:
    - name: configure kernel module load
      lineinfile:
        path: "/etc/modules-load.d/{{ item }}.conf"
        line: "{{ item }}"
        create: true
      with_items:
        - br_netfilter
        - overlay

    - name: load kernel module
      modprobe:
        name: br_netfilter
        state: present

    - name: configure kernel parameter
      sysctl:
        name: "{{ item }}"
        value: "1"
        reload: true
        sysctl_file: /etc/sysctl.d/k8s.conf
      with_items:
        - net.bridge.bridge-nf-call-ip6tables
        - net.bridge.bridge-nf-call-iptables
        - net.ipv4.ip_forward

    - name: disable systemd-resolvd
      systemd:
        name: systemd-resolved
        state: stopped
        enabled: no
      notify:
        - remove resolv.conf
        - restart NetworkManager

    - name: disable SELinux
      selinux:
        state: disabled
      notify:
        - reboot

  handlers:
    - name: remove resolv.conf
      file:
        path: /etc/resolv.conf
        state: absent

    - name: restart NetworkManager
      systemd:
        name: NetworkManager
        state: restarted

    - name: reboot
      reboot:
```

プレイブックを実行します。

```bash
ansible-playbook -i inventory -k -K part1.yaml
```

## パッケージの導入

上モノのパッケージを導入するためのプレイブックを用意します。ファイル名は `part2.yaml` とします。

```yaml
---
- name: part2.yaml
  hosts: fedora36
  become: yes
  serial: 1
  gather_facts: no
  tasks:
    - name: remove swap package
      dnf:
        name: zram-generator
        state: removed

    - name: swapoff
      command: swapoff -a
      changed_when: false

    - name: install iptables-legacy
      dnf:
        name:
          - iptables-legacy
          - iproute-tc
        state: present

    - name: configure iptables backend
      alternatives:
        name: iptables
        path: /usr/sbin/iptables-legacy

    - name: package version 1.24
      command: "dnf -y module enable cri-o:1.24"
      changed_when: false

    - name: install CRI
      dnf:
        name: crio
        state: present

    - name: enable CRI
      systemd:
        name: crio
        state: started
        enabled: yes

    - name: add package repo(k8s)
      copy:
        dest: /etc/yum.repos.d/kubernetes.repo
        content: |
          [kubernetes]
          name=Kubernetes
          baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
          enabled=1
          gpgcheck=1
          repo_gpgcheck=1
          gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

    - name: install k8s package
      dnf:
        name: "{{ item }}"
        state: present
        disable_excludes: kubernetes
      with_items:
        - kubelet
        - kubeadm
        - kubectl

    - name: configure kubelet
      replace:
        path: /etc/sysconfig/kubelet
        regexp: "^KUBELET_EXTRA_ARGS=$"
        replace: "KUBELET_EXTRA_ARGS=--cgroup-driver=systemd"
      notify: restart kubelet

    - name: pull kubernetes images
      command: "kubeadm config images pull"
      changed_when: false

    - name: disable firewalld
      systemd:
        name: firewalld
        state: stopped
        enabled: no

    - name: check iptables-save
      stat:
        path: /root/iptables-save
      register: iptables_save

    - name: backup iptables
      shell: iptables-save > /root/iptables-save
      when: not iptables_save.stat.exists

    - name: configure NetworkManager(calico)
      copy:
        dest: /etc/NetworkManager/conf.d/calico.conf
        content: |
          [keyfile]
          unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:wireguard.cali
      notify: restart NetworkManager

  handlers:
    - name: restart kubelet
      systemd:
        daemon_reload: yes
        name: kubelet
        enabled: yes
        state: restarted

    - name: restart NetworkManager
      systemd:
        name: NetworkManager
        state: restarted
```

プレイブックを実行します。

```bash
ansible-playbook -i inventory -k -K part2.yaml
```

## kubeadm の実行

コントロールプレーン用の `192.168.1.22` にログインして作業します。

`kubeadm` を実行します。`pod-network-cider` はノードの IP レンジと被っていなければOKです。検証用なのでロードバランサは指定しません。

```bash
# kubeadm の実行
kubeadm init --pod-network-cidr="10.0.0.0/8"
```

`kubeadm join` で始まる以下のような行を控えておきます。後でノードを追加するときに使用します。

```bash
kubeadm join 192.168.1.22:6443 --token 28r6mx.4id5loxvio6g8lxh \
        --discovery-token-ca-cert-hash sha256:6e65778be3b8002dda6a298ed249ba292d5210797bdf277b303acf998845787c
```

`kubectl` が使えるように準備をします。

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc
```

`kubectl` で Pod が稼働していることを確認をします。このとき、`coredns` が ready になっていませんが、問題ありません。

```bash
kubectl get pod -A
```

## ワーカーノードの追加

ワーカー用のノード `192.168.1.23` と `192.168.1.24` で、`kubeadm` を実行したときのコマンドを実行します。

```bash
kubeadm join 192.168.1.22:6443 --token 28r6mx.4id5loxvio6g8lxh \
        --discovery-token-ca-cert-hash sha256:6e65778be3b8002dda6a298ed249ba292d5210797bdf277b303acf998845787c
```

コントロールプレーン用のノード `192.168.1.22` で以下のコマンドを実行し、ワーカーノードが表示されることを確認します。

```bash
kubectl get node
```

## ネットワークプラグインの導入

引き続きコントロールプレーンで作業します。

とりあえず `calicoctl` をインストールします。ただ、今回は使用しません。

```bash
curl -LO https://github.com/projectcalico/calico/releases/download/v3.23.2/calicoctl-linux-amd64

install -m 0744 calicoctl-linux-amd64 /usr/local/bin/calicoctl
```

`tigera-operator` を kubernetes にインストールします。

```bash
curl -LO https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
```

`tigera-operator` を稼働させるためにカスタムリソースをインストールします。インストール前に yaml の IP レンジを、`kubeadm` をインストールしたときの IP レンジ(10.0.0.0/8)に合わせて書き換えます。

```bash
curl -LO https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml
sed -i 's;192.168.0.0/16;10.0.0.0/8;' custom-resources.yaml
kubectl create -f custom-resources.yaml
```

`calico` 関連のリソースが展開されるのを待ちます。しばらく時間がかかります。

```bash
watch kubectl get pod -A
```

## metrics-server の導入

脇道です。

`kubernetes` を使っていると Pod による負荷状況を確認したくなることが多いので `metrics-server` をインストールします。

```bash
curl -LO https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
sed -i '/- args:/a\        - --kubelet-insecure-tls' components.yaml
kubectl apply -f components.yaml
```

インストール直後はエラーが出ますが、しばらく待つと負荷状況が表示されるようになります。

```bash
kubectl top pod -A
```

## 稼働確認

稼働確認として、NGINX を `kubernetes` クラスターに展開してみます。

```bash
cat <<EOF > nginx.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-nodeport-dep
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-svc
spec:
  selector:
    app: nginx
  ports:

- port: 80
    nodePort: 30903
  type: NodePort
EOF
kubectl apply -f nginx.yaml
```

以下のコマンドでポート 30903 で稼働していることを確認します。

```bash
kubectl get svc
```

構築に使用した Windows のブラウザで `192.168.1.22:30903` にアクセスし、NGINX の Welcome 画面が表示されることを確認します。

## 結果

いくつか不足がありますが、コンテナを稼働させるところまで出来たので良しとします。

`kubeadm` を実行したとき、以下の警告が出て気持ち悪いので解消方法ご存知の方教えてください。

```bash
[WARNING SystemVerification]: missing optional cgroups: blkio
```

以上
