---
title: "Kubernetes で AWX 使えるようにするときのメモ"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Kubernetes, AWX, Ansible]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 参考

<https://github.com/ansible/awx-operator>

## 外部サービス

今回、AWX が使用する外部サービスとして `PostgreSQL` と `FreeIPA` を使うことにします。

コンテナを使って、仮想マシン1台(`192.168.1.21`)でまとめて展開します。従って、各サービスへのアクセスは以下のようになります。

|Service      |IP Address:Port  |
|-------------|-----------------|
|PostgreSQL   |192.168.1.21:5432|
|FreeIPA(LDAP)|192.168.1.21:646 |

構築方法については、それぞれ以下の記事を参照してください。

- <https://zenn.dev/asterisk9101/articles/fedora36server-2>
- <https://zenn.dev/asterisk9101/articles/fedora36server-4>

## Kubernetes

AWX は (awx-operatorを使って) Kubernetes 上に展開することが推奨されているため、Kubernetes クラスタを用意します。IPアドレスと役割は以下の通りです。

|Role      |IP Address  |
|----------|------------|
|Controller|192.168.1.22|
|Worker1   |192.168.1.23|
|Worker2   |192.168.1.24|

構築方法については、以下の記事を参照してください。

- <https://zenn.dev/asterisk9101/articles/fedora36server-3>

## FreeIPAの設定

AWX から LDAP 認証するための準備をします。

`192.168.1.21` にログインし、以下のコマンドで `FreeIPA` コンテナにログインします。

```bash
podman exec -it ipa bash
```

`FreeIPA` コンテナで Kerberos チケットを取得します。

```bash
# パスワードを問われるので、コンテナ起動時に指定したパスワードを入力する
kinit admin

# チケットが取得できたことを確認する。
klist
```

続けて、AWX からバインドするユーザーを作成し、グループに追加します。

```bash
# AWX 用ユーザーの作成
ipa user-add ansible --first=ansible --last=admin --password

# グループの作成と、ユーザーの追加
ipa group-add awx
ipa group-add-member awx --users=ansible

# AWX にログインに使用するユーザーの作成
ipa user-add asterisk9101 --first=asterisk --last=9101 --password
ipa group-add-member awx --users=asterisk9101
```

更に、外部から LDAPS で接続するために証明書を採取します（※正しい方法なのか怪しい）

```bash
cat /etc/ipa/ca.crt
```

この証明書は Controller ノード(`192.168.1.22`) にコピーしておきます。

以上で `FreeIPA` コンテナでの作業は完了です。

## yamlファイルの作成

Controller ノード (`192.168.1.22`) で作業します。

適当なディレクトリを作って、その中で作業します。前工程で採取した `ca.crt` ファイルは、このディレクトリに入れておきます。

```bash
mkdir awx
cd awx
```

まず、データベースへの接続定義 `awx-postgres-configuration.yaml` を用意します。

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: awx-postgres-configuration
  namespace: awx
stringData:
  host: "192.168.1.21"
  port: "5432"
  database: postgres
  username: postgres
  password: mysecretpassword
  sslmode: prefer
  type: unmanaged
type: Opaque
```

次に、LDAP サーバーへの接続パスワード `ldap-secret.yaml` を用意します。

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: awx-demo-ldap-password
  namespace: awx
stringData:
  ldap-password: P@ssw0rd
type: Opaque
```

マニュアルに従って `awx-demo.yaml` ファイルを作ります。`GroupOfNamesType()` や `lda.OPT_X_TLS_REQUIRE_CERT` を使用するために `from ... import ...` を注入しています。他に綺麗なやり方が見つからなかったのでやむを得ず。

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
spec:
  service_type: nodeport
  nodeport_port: 30080
  postgres_configuration_secret: awx-postgres-configuration
  bundle_cacert_secret: awx-demo-custom-certs
  ldap_cacert_secret:   awx-demo-custom-certs
  ldap_password_secret: awx-demo-ldap-password
  extra_settings:
  - setting: AUTH_LDAP_SERVER_URI
    value: '"ldaps://192.168.1.21:636"; from django_auth_ldap.config import GroupOfNamesType; import ldap'
  - setting: AUTH_LDAP_BIND_DN
    value: '"uid=ansible,cn=users,cn=accounts,dc=localdomain,dc=intra"'
  - setting: AUTH_LDAP_USER_SEARCH
    value: 'LDAPSearch("cn=users,cn=accounts,DC=localdomain,DC=intra",ldap.SCOPE_SUBTREE,"(uid=%(user)s)",)'
  - setting: AUTH_LDAP_GROUP_SEARCH
    value: 'LDAPSearch("cn=groups,cn=accounts,dc=localdomain,dc=intra",ldap.SCOPE_SUBTREE,"(objectClass=groupofnames)",)'
  - setting: AUTH_LDAP_GROUP_TYPE
    value: "GroupOfNamesType()"
  - setting: AUTH_LDAP_CONNECTION_OPTIONS
    value: '{ ldap.OPT_X_TLS_REQUIRE_CERT: ldap.OPT_X_TLS_ALLOW }'
  - setting: AUTH_LDAP_REQUIRE_GROUP
    value: '"cn=awx,cn=groups,cn=accounts,dc=localdomain,dc=intra"'
```

事前に準備するファイルは以上です。

## awx-operatorのインストール

まず Controller ノード(`192.168.1.22`)にて、`Kustomize` をインストールします。

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash

# カレントディレクトリにインストールされますが、邪魔なので移動します
mv kustomize /usr/local/bin/
```

`kustomization.yaml` ファイルを作ります。`awx-operator` のバージョンは、`0.28.0` にしました。

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/ansible/awx-operator/config/default?ref=0.28.0
  - awx-postgres-configuration.yaml
  - ldap-secret.yaml
  - awx-demo.yaml

images:
  - name: quay.io/ansible/awx-operator
    newTag: 0.28.0

namespace: awx
```

`namespace` を先に作って、`awx-demo-custom-certs` を作成した上で、`kustomization.yaml` を展開します。

```bash
kubectl create namespace awx
kubectl create -n awx secret generic awx-demo-custom-certs --from-file=ldap-ca.crt=ca.crt --from-file=bundle-ca.crt=/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
kustomize build . | kubectl apply -f -
```

`kubectl` の名前空間を `awx` に変更して、`awx-operator` が稼働していることを確認します。しばらく待つと `awx-demo-...` が作成されます。

```bash
kubectl config set-context --current --namespace=awx
kubectl get pod
```

## 稼働確認

Controller ノード(`192.168.1.22`)にて `admin` ユーザーのパスワードを取得します。

```bash
kubectl get secret awx-demo-admin-password -ojson | jq '.data.password' -r | base64 -d | xargs
```

ブラウザで Kubernetes ノードのいずれかの 30080 ポートにアクセスし、`admin` でログインできることを確認します。また、`asterisk9101` ユーザーでログインし、LDAP 連携できていることも確認します。

## トラブルシューティング

上手く動かず中々手こずったので、トラブルシューティングの手法もメモしておきます。

まず最初に見るべきは、Pod の稼動状況です。Pod が起動しない原因（リソース不足など）の調査ができます。

```bash
kubectl describe pod/awx-demo....
```

次に、Pod の標準出力を確認します。`-c` でコンテナを指定する必要があります。ブラウザでアクセスできない原因（DB接続に失敗しているなど）の調査ができます。

```bash
kubectl logs -f awx-demo... -c awx-demo-web
```

それでも駄目なら、Pod にログインします。設定値が想定通りに設定されているか（設定値のダブルクォートが抜けてエラーになっていたり）などの調査ができます。尚、awx-operator が設定を挿入するファイルは `/etc/tower/settings.py` にあります。

```bash
kubectl exec -it awx-demo... -c awx-demo-web -- bash
```

以上
