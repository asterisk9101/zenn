---
title: "trust anchor コマンドと update-ca-trust コマンドの違いについて"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora38]
published: true
---

Fedora38 Server に証明書を追加する際、`trust` コマンドまたは `update-ca-trust` コマンドを使いますが、それぞれの役割や機能について理解できていなかったので調査しました。

## 統合されたシステム全体のトラストストア

[RHEL のドキュメント](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html/system_design_guide/using-shared-system-certificates_system-design-guide#the-system-wide-trust-store_using-shared-system-certificates)を参考にすると、システム全体のトラストストアは以下の２つ存在することが分かります。

- /etc/pki/ca-trust/
- /usr/share/pki/ca-trust-source/

証明書は、上記のディレクトリにインストールされる様です。

まずは優先順位の高い `/etc` の方から内容を確認してみます。`extracted` と `source` というディレクトリがあります。

```bash
ls -l /etc/pki/ca-trust/
#-rw-r--r--. 1 root root 980 Aug  1 09:00 ca-legacy.conf
#drwxr-xr-x. 6 root root  70 Nov  3 14:26 extracted
#-rw-r--r--. 1 root root 166 Aug  1 09:00 README
#drwxr-xr-x. 5 root root  97 Nov  3 14:27 source
```

次に、優先順位の低い `/usr` の方を見てみます。`/etc` の配下と同じ役割のディレクトリであるにも関わらず、内容物は全く異なります。

```bash
ls -l /usr/share/pki/ca-trust-source/
#drwxr-xr-x. 2 root root       6 Aug  1 09:00 anchors
#drwxr-xr-x. 2 root root       6 Aug  1 09:00 blacklist
#drwxr-xr-x. 2 root root       6 Aug  1 09:00 blocklist
#-rw-r--r--. 1 root root 2339585 Aug  1 09:00 ca-bundle.trust.p11-kit
#-rw-r--r--. 1 root root     937 Aug  1 09:00 README
```

実は `/etc/` の配下にも同様の構成がありました。`source` ディレクトリが共通している様です。

```bash
ls -l /etc/pki/ca-trust/source/
#drwxr-xr-x. 2 root root   6 Aug  1 09:00 anchors
#drwxr-xr-x. 2 root root   6 Aug  1 09:00 blacklist
#drwxr-xr-x. 2 root root   6 Aug  1 09:00 blocklist
#lrwxrwxrwx. 1 root root  59 Nov  3 14:27 ca-bundle.legacy.crt -> /usr/share/pki/ca-trust-legacy/ca-bundle.legacy.default.crt
#-rw-r--r--. 1 root root 932 Aug  1 09:00 README
```

## 自己署名証明書の作成

動作を確認するための証明書を作成します。検証なので自己署名で作成します。いくつか質問されますが、適当に回答します。

```bash
openssl genrsa 2048 > private.key
openssl req -new -x509 -days 3650 -key private.key -sha512 -out server.crt
#You are about to be asked to enter information that will be incorporated
#into your certificate request.
#What you are about to enter is what is called a Distinguished Name or a DN.
#There are quite a few fields but you can leave some blank
#For some fields there will be a default value,
#If you enter '.', the field will be left blank.
#-----
#Country Name (2 letter code) [XX]:JA
#State or Province Name (full name) []:osaka
#Locality Name (eg, city) [Default City]:osaka
#Organization Name (eg, company) [Default Company Ltd]:asterisk
#Organizational Unit Name (eg, section) []:home
#Common Name (eg, your name or your server's hostname) []:www
#Email Address []:
```

作成した証明書を確認します。PEM 形式の証明書データです。

```bash
file server.crt
#server.crt: PEM certificate
cat server.crt
#-----BEGIN CERTIFICATE-----
#MIIDmzCCAoOgAwIBAgIUT/QClhntGlbKSKXvplivexmEOzIwDQYJKoZIhvcNAQEN
#BQAwXTELMAkGA1UEBhMCSkExDjAMBgNVBAgMBW9zYWthMQ4wDAYDVQQHDAVvc2Fr
#YTERMA8GA1UECgwIYXN0ZXJpc2sxDTALBgNVBAsMBGhvbWUxDDAKBgNVBAMMA3d3
#dzAeFw0yMzExMDMwNjAzNDZaFw0zMzEwMzEwNjAzNDZaMF0xCzAJBgNVBAYTAkpB
#MQ4wDAYDVQQIDAVvc2FrYTEOMAwGA1UEBwwFb3Nha2ExETAPBgNVBAoMCGFzdGVy
#aXNrMQ0wCwYDVQQLDARob21lMQwwCgYDVQQDDAN3d3cwggEiMA0GCSqGSIb3DQEB
#AQUAA4IBDwAwggEKAoIBAQCgm2ImezVrDle27BmrzaTK6a0Ucgx00nqLj/wlkcsb
#Iykt7020DVmCiQ0eaxslWcDKkCzB4OTC7VrvvX2jzffy6lPGI9Qzo2D787akSTB6
#q9kI2q8aSvHovU8AGBwjDDxz9JRXP7Jf4RhLlxczCrFI9FuG3N5t19s8iig4n/VO
#EzB3M2WfQoAo2zzayArbeY0e61AYLkTM8rYlWQRYy9ctSnvglcBrhtnQUaw1HAZg
#rrt4YpeXgOkm5DjRiHKCP0G09jL3qRu+OVqBm8iKXmnenzTjv5dbeYVivWxL+c/M
#xy7EnMZExQ6uML8Ub6dSMq0HgqdNKhpOYWzzbUeBaLPrAgMBAAGjUzBRMB0GA1Ud
#DgQWBBTsUjWKPrTfXmqkkOFq2cV2H3e0TTAfBgNVHSMEGDAWgBTsUjWKPrTfXmqk
#kOFq2cV2H3e0TTAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBDQUAA4IBAQBl
#fjvu7+vQDt7LJcE4WDqDLU8HR5oCcDWVK0JOWdKTfM6Sfhr+MEJtW9DMy4cj/WK3
#aWuGx4otTvunR3YDlVSGksJfzzKbun/+NzYx20H4q9Q4tdA9QCzeGpf9LV/9ykup
#GvOF0Uxg1Abroi3AfYaQbD1/uT/GJInb3cBKKWbNF7Y3nuqMHu7WVbPvcG8IW2CR
#+YIruJz18kr531kiCDviSbAJZOVUWTIsaPUwZB/3JrS+f1XNkXFhC0E9NCjlDxXD
#B+iD6iXnchYHyhWkSg2UCmNU6LPnxNO+n0iPkHAcstg/2EeUj36xST0vNNeDm6Zh
#EEpQBuYZ8CC/O5XgNXIc
#-----END CERTIFICATE-----
```

証明書の内容は以下のように確認できます。証明書の有効期限（`Not Before` と `Not After`）などは気になることが多いです。何も指定しないと 10 年になるんですね。

```bash
openssl x509 -in server.crt -noout -text
#Certificate:
#    Data:
#        Version: 3 (0x2)
#        Serial Number:
#            4f:f4:02:96:19:ed:1a:56:ca:48:a5:ef:a6:58:af:7b:19:84:3b:32
#        Signature Algorithm: sha512WithRSAEncryption
#        Issuer: C = JA, ST = osaka, L = osaka, O = asterisk, OU = home, CN = www
#        Validity
#            Not Before: Nov  3 06:03:46 2023 GMT
#            Not After : Oct 31 06:03:46 2033 GMT
#        Subject: C = JA, ST = osaka, L = osaka, O = asterisk, OU = home, CN = www
#        Subject Public Key Info:
#            Public Key Algorithm: rsaEncryption
#                Public-Key: (2048 bit)
#                Modulus:
#                    00:a0:9b:62:26:7b:35:6b:0e:57:b6:ec:19:ab:cd:
#                    a4:ca:e9:ad:14:72:0c:74:d2:7a:8b:8f:fc:25:91:
#                    cb:1b:23:29:2d:ef:4d:b4:0d:59:82:89:0d:1e:6b:
#                    1b:25:59:c0:ca:90:2c:c1:e0:e4:c2:ed:5a:ef:bd:
#                    7d:a3:cd:f7:f2:ea:53:c6:23:d4:33:a3:60:fb:f3:
#                    b6:a4:49:30:7a:ab:d9:08:da:af:1a:4a:f1:e8:bd:
#                    4f:00:18:1c:23:0c:3c:73:f4:94:57:3f:b2:5f:e1:
#                    18:4b:97:17:33:0a:b1:48:f4:5b:86:dc:de:6d:d7:
#                    db:3c:8a:28:38:9f:f5:4e:13:30:77:33:65:9f:42:
#                    80:28:db:3c:da:c8:0a:db:79:8d:1e:eb:50:18:2e:
#                    44:cc:f2:b6:25:59:04:58:cb:d7:2d:4a:7b:e0:95:
#                    c0:6b:86:d9:d0:51:ac:35:1c:06:60:ae:bb:78:62:
#                    97:97:80:e9:26:e4:38:d1:88:72:82:3f:41:b4:f6:
#                    32:f7:a9:1b:be:39:5a:81:9b:c8:8a:5e:69:de:9f:
#                    34:e3:bf:97:5b:79:85:62:bd:6c:4b:f9:cf:cc:c7:
#                    2e:c4:9c:c6:44:c5:0e:ae:30:bf:14:6f:a7:52:32:
#                    ad:07:82:a7:4d:2a:1a:4e:61:6c:f3:6d:47:81:68:
#                    b3:eb
#                Exponent: 65537 (0x10001)
#        X509v3 extensions:
#            X509v3 Subject Key Identifier:
#                EC:52:35:8A:3E:B4:DF:5E:6A:A4:90:E1:6A:D9:C5:76:1F:77:B4:4D
#            X509v3 Authority Key Identifier:
#                EC:52:35:8A:3E:B4:DF:5E:6A:A4:90:E1:6A:D9:C5:76:1F:77:B4:4D
#            X509v3 Basic Constraints: critical
#                CA:TRUE
#    Signature Algorithm: sha512WithRSAEncryption
#    Signature Value:
#        65:7e:3b:ee:ef:eb:d0:0e:de:cb:25:c1:38:58:3a:83:2d:4f:
#        07:47:9a:02:70:35:95:2b:42:4e:59:d2:93:7c:ce:92:7e:1a:
#        fe:30:42:6d:5b:d0:cc:cb:87:23:fd:62:b7:69:6b:86:c7:8a:
#        2d:4e:fb:a7:47:76:03:95:54:86:92:c2:5f:cf:32:9b:ba:7f:
#        fe:37:36:31:db:41:f8:ab:d4:38:b5:d0:3d:40:2c:de:1a:97:
#        fd:2d:5f:fd:ca:4b:a9:1a:f3:85:d1:4c:60:d4:06:eb:a2:2d:
#        c0:7d:86:90:6c:3d:7f:b9:3f:c6:24:89:db:dd:c0:4a:29:66:
#        cd:17:b6:37:9e:ea:8c:1e:ee:d6:55:b3:ef:70:6f:08:5b:60:
#        91:f9:82:2b:b8:9c:f5:f2:4a:f9:df:59:22:08:3b:e2:49:b0:
#        09:64:e5:54:59:32:2c:68:f5:30:64:1f:f7:26:b4:be:7f:55:
#        cd:91:71:61:0b:41:3d:34:28:e5:0f:15:c3:07:e8:83:ea:25:
#        e7:72:16:07:ca:15:a4:4a:0d:94:0a:63:54:e8:b3:e7:c4:d3:
#        be:9f:48:8f:90:70:1c:b2:d8:3f:d8:47:94:8f:7e:b1:49:3d:
#        2f:34:d7:83:9b:a6:61:10:4a:50:06:e6:19:f0:20:bf:3b:95:
#        e0:35:72:1c
```

## trust コマンド

`trust list` でシステムにインストールされている証明書一覧が参照できます。

```bash
trust list
#pkcs11:id=%42%3D%2B%24%A6%C1%45%CE;type=cert
#    type: certificate
#    label: A-Trust-Qual-02
#    trust: anchor
#    category: authority
#
#pkcs11:id=%46%06%DF%37%F2%C2%37%10;type=cert
#    type: certificate
#    label: A-Trust-Qual-03
#    trust: anchor
#    category: authority
#
#pkcs11:id=%40%F9%B9%67%BE%03%D2%08;type=cert
#    type: certificate
#    label: A-Trust-Root-05
#    trust: anchor
#    category: authority
#
#...
```

詳細が確認したければ `trust dump` で詳細な一覧が参照できます。インストールされている証明書が PEM 形式で出力されます。付随するメタデータも表示されます。

```bash
trust dump
## pkcs11:
#[p11-kit-object-v1]
#class: nss-builtin-root-list
#private: false
#label: "Trust Anchor Roots"
#modifiable: false
#
#
## pkcs11:
#[p11-kit-object-v1]
#class: nss-builtin-root-list
#private: false
#label: "Trust Anchor Roots"
#modifiable: false
#
#
## pkcs11:id=%57%38%74%BE%5C%36%85%F4%C8%A9%A5%93%87%D5%90%36%2B%A5%93%19
#[p11-kit-object-v1]
#class: x-certificate-extension
#private: false
#label: "A-Trust-Qual-02"
#value: "0%16%06%03U%1D%25%01%01%FF%04%0C0%0A%06%08%2B%06%01%05%05%07%03%03"
#object-id: 2.5.29.37
#id: "W8t%BE%5C6%85%F4%C8%A9%A5%93%87%D5%906%2B%A5%93%19"
#modifiable: false
#-----BEGIN PUBLIC KEY-----
#MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAlpGr146wWbgBvbQerZn9
#ofzqDJZrjy5JSNjp5CSC4di6zOveB1xIxhmCAUdCCeF8uZ+uj/buXmuoqFaIOFaz
#5v7JnU80OFE8nL7zrJwcyD1cmoSu95QsFLJkN2COFLA7Ks2JtSKv9+R/7iwrM/mO
#PK1MolcP+4LFjh8B31OMwRWk3e6akhLSKOliFyVgMe/PMZTzffPkPCGrkPxtz2On
#xhIAcUnXHMig7WOafeOPKN1g+djmFqsm0dAvvecNCd9u1OzlOFP0ZAu6WsyAtThZ
#7IBZZYQxal9VkUwk5SSag4R6voDx7l8gB6p3F2wL4lWqlngIAp6X8CuuWPWAUD/+
#OwIDAQAB
#-----END PUBLIC KEY-----
#
#...
```

証明書をインストールするには `trust anchor` コマンドを使います。

```bash
trust anchor server.crt
```

証明書がインストールされたことを確認します。

```bash
trust list | grep -C 3 www
#pkcs11:id=%EC%52%35%8A%3E%B4%DF%5E%6A%A4%90%E1%6A%D9%C5%76%1F%77%B4%4D;#type=cert
#    type: certificate
#    label: www
#    trust: anchor
#    category: authority

trust dump | grep -A 10 "pkcs11:id=%EC%52%35%8A%3E%B4%DF%5E%6A%A4%90%E1%6A%D9%C5%76%1F%77%B4%4D;type=cert"
## pkcs11:id=%EC%52%35%8A%3E%B4%DF%5E%6A%A4%90%E1%6A%D9%C5%76%1F%77%B4%4D;#type=cert
#[p11-kit-object-v1]
#private: false
#label: "www"
#issuer: #"0%5D1%0B0%09%06%03U%04%06%13%02JA1%0E0%0C%06%03U%04%08%0C%05osaka1%0E0%0C%06#%03U%04%07%0C%05osaka1%110%0F%06%03U%04%0A%0C%08asterisk1%0D0%0B%06%03U%04%0B#%0C%04home1%0C0%0A%06%03U%04%03%0C%03www"
#serial-number: "%02%14O%F4%02%96%19%ED%1AV%CAH%A5%EF%A6X%AF%7B%19%84%3B2"
#trusted: true
#certificate-category: authority
#java-midp-security-domain: 0
#url: ""
#hash-of-subject-public-key: #"%D7%3DF%0EA%24%E7%C9%90%80Q%A4%8A%FC%09%3C%10I%F2%06"
```

`/etc/` 配下の `source` ディレクトリに、`.p11-kit` ファイルが追加されていることが分かります。

```bash
ls -l /etc/pki/ca-trust/source/
#drwxr-xr-x. 2 root root    6 Aug  1 09:00 anchors
#drwxr-xr-x. 2 root root    6 Aug  1 09:00 blacklist
#drwxr-xr-x. 2 root root    6 Aug  1 09:00 blocklist
#lrwxrwxrwx. 1 root root   59 Nov  3 14:27 ca-bundle.legacy.crt -> /usr/share/pki/ca-trust-legacy/ca-bundle.legacy.default.crt
#-rw-r--r--. 1 root root  932 Aug  1 09:00 README
#-r--r--r--. 1 root root 2582 Nov  3 15:20 www.p11-kit
```

内容を確認すると `trust dump` と似たような形式になっています。下の方には PEM ファイルの内容がそのまま残っており、PEM ファイルにヘッダ情報が付加された形式のように見えます。

```bash
cat /etc/pki/ca-trust/source/www.p11-kit
#[p11-kit-object-v1]
#trusted: true
#x-distrusted: false
#private: false
#modifiable: false
#label: "www"
#url: ""
#hash-of-issuer-public-key: ""
#hash-of-subject-public-key: "%D7%3DF%0EA%24%E7%C9%90%80Q%A4%8A%FC%09%3C%10I%F2%06"
#java-midp-security-domain: 0
#check-value: "2N%AC"
#start-date: "20231103"
#end-date: "20331031"
#id: "%ECR5%8A%3E%B4%DF%5Ej%A4%90%E1j%D9%C5v%1Fw%B4M"
#subject: #"0%5D1%0B0%09%06%03U%04%06%13%02JA1%0E0%0C%06%03U%04%08%0C%05osaka1%0E0%0C%06%03U%04%07%0C%05osaka1%110%0F%06%03U%04%0A%0C%08asterisk1%0D0%0B%06%03U%04%0B%0C%04home1%0C0%0A%#06%03U%04%03%0C%03www"
#issuer: #"0%5D1%0B0%09%06%03U%04%06%13%02JA1%0E0%0C%06%03U%04%08%0C%05osaka1%0E0%0C%06%03U%04%07%0C%05osaka1%110%0F%06%03U%04%0A%0C%08asterisk1%0D0%0B%06%03U%04%0B%0C%04home1%0C0%0A%#06%03U%04%03%0C%03www"
#serial-number: "%02%14O%F4%02%96%19%ED%1AV%CAH%A5%EF%A6X%AF%7B%19%84%3B2"
#certificate-category: authority
#-----BEGIN CERTIFICATE-----
#MIIDmzCCAoOgAwIBAgIUT/QClhntGlbKSKXvplivexmEOzIwDQYJKoZIhvcNAQEN
#BQAwXTELMAkGA1UEBhMCSkExDjAMBgNVBAgMBW9zYWthMQ4wDAYDVQQHDAVvc2Fr
#YTERMA8GA1UECgwIYXN0ZXJpc2sxDTALBgNVBAsMBGhvbWUxDDAKBgNVBAMMA3d3
#dzAeFw0yMzExMDMwNjAzNDZaFw0zMzEwMzEwNjAzNDZaMF0xCzAJBgNVBAYTAkpB
#MQ4wDAYDVQQIDAVvc2FrYTEOMAwGA1UEBwwFb3Nha2ExETAPBgNVBAoMCGFzdGVy
#aXNrMQ0wCwYDVQQLDARob21lMQwwCgYDVQQDDAN3d3cwggEiMA0GCSqGSIb3DQEB
#AQUAA4IBDwAwggEKAoIBAQCgm2ImezVrDle27BmrzaTK6a0Ucgx00nqLj/wlkcsb
#Iykt7020DVmCiQ0eaxslWcDKkCzB4OTC7VrvvX2jzffy6lPGI9Qzo2D787akSTB6
#q9kI2q8aSvHovU8AGBwjDDxz9JRXP7Jf4RhLlxczCrFI9FuG3N5t19s8iig4n/VO
#EzB3M2WfQoAo2zzayArbeY0e61AYLkTM8rYlWQRYy9ctSnvglcBrhtnQUaw1HAZg
#rrt4YpeXgOkm5DjRiHKCP0G09jL3qRu+OVqBm8iKXmnenzTjv5dbeYVivWxL+c/M
#xy7EnMZExQ6uML8Ub6dSMq0HgqdNKhpOYWzzbUeBaLPrAgMBAAGjUzBRMB0GA1Ud
#DgQWBBTsUjWKPrTfXmqkkOFq2cV2H3e0TTAfBgNVHSMEGDAWgBTsUjWKPrTfXmqk
#kOFq2cV2H3e0TTAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBDQUAA4IBAQBl
#fjvu7+vQDt7LJcE4WDqDLU8HR5oCcDWVK0JOWdKTfM6Sfhr+MEJtW9DMy4cj/WK3
#aWuGx4otTvunR3YDlVSGksJfzzKbun/+NzYx20H4q9Q4tdA9QCzeGpf9LV/9ykup
#GvOF0Uxg1Abroi3AfYaQbD1/uT/GJInb3cBKKWbNF7Y3nuqMHu7WVbPvcG8IW2CR
#+YIruJz18kr531kiCDviSbAJZOVUWTIsaPUwZB/3JrS+f1XNkXFhC0E9NCjlDxXD
#B+iD6iXnchYHyhWkSg2UCmNU6LPnxNO+n0iPkHAcstg/2EeUj36xST0vNNeDm6Zh
#EEpQBuYZ8CC/O5XgNXIc
#-----END CERTIFICATE-----
```

このとき、`extracted` ディレクトリも更新されていることが分かります(先程追加された `.p11-kit` ファイルと更新日時が同じ)。`trust` コマンドで証明書を追加する際に、`source` ディレクトリ（システム全体のトラストストア）を元に再構成されていると思われます。ディレクトリ名がアプリケーションの名前になっているので、各アプリケーションから参照される証明書ストアと、システム全体のトラストストアと同期するための仕組みだと思われます。

```bash
ls -l /etc/pki/ca-trust/extracted/
#drwxr-xr-x. 2 root root  39 Nov  3 15:20 edk2
#drwxr-xr-x. 2 root root  35 Nov  3 15:20 java
#drwxr-xr-x. 2 root root  47 Nov  3 15:20 openssl
#drwxr-xr-x. 3 root root 123 Nov  3 15:20 pem
#-rw-r--r--. 1 root root 560 Aug  1 09:00 README

ls -l /etc/pki/ca-trust/extracted/edk2/
#-r--r--r--. 1 root root 162230 Nov  3 15:20 cacerts.bin
#-rw-r--r--. 1 root root    566 Aug  1 09:00 README

ls -l /etc/pki/ca-trust/extracted/java/
#-r--r--r--. 1 root root 162772 Nov  3 15:20 cacerts
#-rw-r--r--. 1 root root    726 Aug  1 09:00 README

ls -l /etc/pki/ca-trust/extracted/openssl/
#-r--r--r--. 1 root root 624816 Nov  3 15:20 ca-bundle.trust.crt
#-rw-r--r--. 1 root root    787 Aug  1 09:00 README

ls -l /etc/pki/ca-trust/extracted/pem/
#dr-xr-xr-x. 2 root root  16384 Nov  3 15:20 directory-hash
#-r--r--r--. 1 root root 171543 Nov  3 15:20 email-ca-bundle.pem
#-r--r--r--. 1 root root 496846 Nov  3 15:20 objsign-ca-bundle.pem
#-rw-r--r--. 1 root root    898 Aug  1 09:00 README
#-r--r--r--. 1 root root 223399 Nov  3 15:20 tls-ca-bundle.pem
```

証明書をアンインストールするには以下のコマンドを使います。

```bash
trust anchor --remove server.crt
```

`trust list` および `trust dump` コマンドの結果および `.p11-kit` ファイルも削除されました。

```bash
trust list | grep -C 3 www
trust dump | grep -A 10 "pkcs11:id=%EC%52%35%8A%3E%B4%DF%5E%6A%A4%90%E1%6A%D9%C5%76%1F%77%B4%4D;type=cert"
ls -l /etc/pki/ca-trust/source/
#drwxr-xr-x. 2 root root   6 Aug  1 09:00 anchors
#drwxr-xr-x. 2 root root   6 Aug  1 09:00 blacklist
#drwxr-xr-x. 2 root root   6 Aug  1 09:00 blocklist
#lrwxrwxrwx. 1 root root  59 Nov  3 14:27 ca-bundle.legacy.crt -> /usr/share/pki/ca-trust-legacy/ca-bundle.legacy.default.crt
#-rw-r--r--. 1 root root 932 Aug  1 09:00 README
```

`extracted` ディレクトリも更新されています（結果は省略）

```bash
ls -l /etc/pki/ca-trust/extracted/edk2/
ls -l /etc/pki/ca-trust/extracted/java/
ls -l /etc/pki/ca-trust/extracted/openssl/
ls -l /etc/pki/ca-trust/extracted/pem/
```

## update-ca-trust コマンド

証明書のインストール方法として、手動で証明書を配置する方法があると説明されています。

```bash
cp -p server.crt /etc/pki/ca-trust/source/anchors/
```

この結果、`trust` コマンドは証明書を認識します。

```bash
trust list | grep -C 3 www
#pkcs11:id=%EC%52%35%8A%3E%B4%DF%5E%6A%A4%90%E1%6A%D9%C5%76%1F%77%B4%4D;#type=cert
#    type: certificate
#    label: www
#    trust: anchor
#    category: authority

trust dump | grep -A 10 "pkcs11:id=%EC%52%35%8A%3E%B4%DF%5E%6A%A4%90%E1%6A%D9%C5%76%1F%77%B4%4D;type=cert"
## pkcs11:id=%EC%52%35%8A%3E%B4%DF%5E%6A%A4%90%E1%6A%D9%C5%76%1F%77%B4%4D;#type=cert
#[p11-kit-object-v1]
#private: false
#label: "www"
#issuer: #"0%5D1%0B0%09%06%03U%04%06%13%02JA1%0E0%0C%06%03U%04%08%0C%05osaka1%0E0%0C%06#%03U%04%07%0C%05osaka1%110%0F%06%03U%04%0A%0C%08asterisk1%0D0%0B%06%03U%04%0B#%0C%04home1%0C0%0A%06%03U%04%03%0C%03www"
#serial-number: "%02%14O%F4%02%96%19%ED%1AV%CAH%A5%EF%A6X%AF%7B%19%84%3B2"
#trusted: true
#certificate-category: authority
#java-midp-security-domain: 0
#url: ""
#hash-of-subject-public-key: #"%D7%3DF%0EA%24%E7%C9%90%80Q%A4%8A%FC%09%3C%10I%F2%06"
```

しかし、`extracted` ディレクトリは更新されていません(結果は省略)

```bash
ls -l /etc/pki/ca-trust/extracted/edk2/
ls -l /etc/pki/ca-trust/extracted/java/
ls -l /etc/pki/ca-trust/extracted/openssl/
ls -l /etc/pki/ca-trust/extracted/pem/
```

そこで `update-ca-trust` を実行します。

```bash
update-ca-trust
```

`extracted` ディレクトリが更新されました（結果は省略）

```bash
ls -l /etc/pki/ca-trust/extracted/edk2/
ls -l /etc/pki/ca-trust/extracted/java/
ls -l /etc/pki/ca-trust/extracted/openssl/
ls -l /etc/pki/ca-trust/extracted/pem/
```

`update-ca-trust` コマンドの内容を確認してみます。

```bash
cat $(which update-ca-trust)
```

`extracted` ディレクトリのファイルを単純に上書きしているだけのシェルスクリプトでした。

```bash
#!/usr/bin/sh

#set -vx

# At this time, while this script is trivial, we ignore any parameters given.
# However, for backwards compatibility reasons, future versions of this script must
# support the syntax "update-ca-trust extract" trigger the generation of output
# files in $DEST.

DEST=/etc/pki/ca-trust/extracted

# Prevent p11-kit from reading user configuration files.
export P11_KIT_NO_USER_CONFIG=1

# OpenSSL PEM bundle that includes trust flags
# (BEGIN TRUSTED CERTIFICATE)
/usr/bin/p11-kit extract --format=openssl-bundle --filter=certificates --overwrite --comment $DEST/openssl/ca-bundle.trust.crt
/usr/bin/p11-kit extract --format=pem-bundle --filter=ca-anchors --overwrite --comment --purpose server-auth $DEST/pem/tls-ca-bundle.pem
/usr/bin/p11-kit extract --format=pem-bundle --filter=ca-anchors --overwrite --comment --purpose email $DEST/pem/email-ca-bundle.pem
/usr/bin/p11-kit extract --format=pem-bundle --filter=ca-anchors --overwrite --comment --purpose code-signing $DEST/pem/objsign-ca-bundle.pem
/usr/bin/p11-kit extract --format=java-cacerts --filter=ca-anchors --overwrite --purpose server-auth $DEST/java/cacerts
/usr/bin/p11-kit extract --format=edk2-cacerts --filter=ca-anchors --overwrite --purpose=server-auth $DEST/edk2/cacerts.bin
# Hashed directory of BEGIN TRUSTED-style certs (usable as OpenSSL CApath and
# by GnuTLS)
/usr/bin/p11-kit extract --format=pem-directory-hash --filter=ca-anchors --overwrite --purpose server-auth $DEST/pem/directory-hash
# Debian compatibility: their /etc/ssl/certs has this bundle
/usr/bin/ln -s ../tls-ca-bundle.pem $DEST/pem/directory-hash/ca-certificates.crt
# Backwards compatibility: RHEL/Fedora provided a /etc/ssl/certs/ca-bundle.crt
# since https://bugzilla.redhat.com/show_bug.cgi?id=572725
/usr/bin/ln -s ../tls-ca-bundle.pem $DEST/pem/directory-hash/ca-bundle.crt
```

## p11-kit コマンドについて少しだけ

`update-ca-trust` コマンドには `extracted` ディレクトリは記載されているものの、`source` にあたる部分が見当たりません。

そこで、主要な機能と思われる `p11-kit` コマンドについて調べてみると、[PKCS#11モジュールをロードし、列挙する方法を提供する](https://p11-glue.github.io/p11-glue/p11-kit.html) と説明されていました。

以下のコマンドで、モジュールの一覧が表示できる様です。

```bash
p11-kit list-modules
#module: p11-kit-trust
#    path: /usr/lib64/pkcs11/p11-kit-trust.so
#    uri: pkcs11:library-description=PKCS%2311%20Kit%20Trust%20Module;library-manufacturer=PKCS%2311%20Kit
#    library-description: PKCS#11 Kit Trust Module
#    library-manufacturer: PKCS#11 Kit
#    library-version: 0.25
#    token: System Trust
#        uri: pkcs11:model=p11-kit-trust;manufacturer=PKCS%2311%20Kit;serial=1;token=System%20Trust
#        manufacturer: PKCS#11 Kit
#        model: p11-kit-trust
#        serial-number: 1
#        hardware-version: 0.25
#        flags:
#              token-initialized
#    token: Default Trust
#        uri: pkcs11:model=p11-kit-trust;manufacturer=PKCS%2311%20Kit;serial=1;token=Default%20Trust
#        manufacturer: PKCS#11 Kit
#        model: p11-kit-trust
#        serial-number: 1
#        hardware-version: 0.25
#        flags:
#              write-protected
#              token-initialized
#module: opensc
#    path: /usr/lib64/pkcs11/opensc-pkcs11.so
#    uri: pkcs11:library-description=OpenSC%20smartcard%20framework;library-manufacturer=OpenSC%20Project
#    library-description: OpenSC smartcard framework
#    library-manufacturer: OpenSC Project
#    library-version: 0.23
```

`p11-kit-trust` と `opensc` というモジュールを認識している様です。おそらく前者のモジュールが、`source` ディレクトリ(`/etc` と `/usr` の配下)の `.p11-kit` ファイルや `anchors/*.crt` を列挙する機能を提供しているものと思われます。

## 確認

確認のためにトラストストアを破壊してみます。

まず `extracted` を破壊します。

```bash
rm -f /etc/pki/ca-trust/extracted/edk2/cacerts.bin
rm -f /etc/pki/ca-trust/extracted/java/cacerts
rm -f /etc/pki/ca-trust/extracted/openssl/ca-bundle.trust.crt
rm -f /etc/pki/ca-trust/extracted/pem/*.pem
rm -f /etc/pki/ca-trust/extracted/pem/directory-hash/*
find /etc/pki/ca-trust/extracted/
```

しかし、`source` ディレクトリは存在するため、`update-ca-trust` コマンドで復活できるはずです。

```bash
update-ca-trust
find /etc/pki/ca-trust/extracted/
```

一方で、デフォルトでインストールされている証明書を削除してみます。

```bash
rm -f /usr/share/pki/ca-trust-source/ca-bundle.trust.p11-kit
```

`trust` コマンドの結果が寂しくなりました。

```bash
trust list
#pkcs11:id=%EC%52%35%8A%3E%B4%DF%5E%6A%A4%90%E1%6A%D9%C5%76%1F%77%B4%4D;type=cert
#    type: certificate
#    label: www
#    trust: anchor
#    category: authority

trust dump
## pkcs11:id=%EC%52%35%8A%3E%B4%DF%5E%6A%A4%90%E1%6A%D9%C5%76%1F%77%B4%4D;type=cert
#[p11-kit-object-v1]
#private: false
#label: "www"
#issuer: #"0%5D1%0B0%09%06%03U%04%06%13%02JA1%0E0%0C%06%03U%04%08%0C%05osaka1%0E0%0C%06%03U%04%07%0C%05osaka1%110%0F%06%03U%04%0A%0C%08asterisk1%0D0%0B%06%03U%04%0B%0C%04home1%0C0%0A%#06%03U%04%03%0C%03www"
#serial-number: "%02%14O%F4%02%96%19%ED%1AV%CAH%A5%EF%A6X%AF%7B%19%84%3B2"
#trusted: true
#certificate-category: authority
#java-midp-security-domain: 0
#url: ""
#hash-of-subject-public-key: "%D7%3DF%0EA%24%E7%C9%90%80Q%A4%8A%FC%09%3C%10I%F2%06"
#hash-of-issuer-public-key: ""
#check-value: "2N%AC"
#subject: #"0%5D1%0B0%09%06%03U%04%06%13%02JA1%0E0%0C%06%03U%04%08%0C%05osaka1%0E0%0C%06%03U%04%07%0C%05osaka1%110%0F%06%03U%04%0A%0C%08asterisk1%0D0%0B%06%03U%04%0B%0C%04home1%0C0%0A%#06%03U%04%03%0C%03www"
#id: "%ECR5%8A%3E%B4%DF%5Ej%A4%90%E1j%D9%C5v%1Fw%B4M"
#start-date: "20231103"
#end-date: "20331031"
#modifiable: false
#x-distrusted: false
#-----BEGIN CERTIFICATE-----
#MIIDmzCCAoOgAwIBAgIUT/QClhntGlbKSKXvplivexmEOzIwDQYJKoZIhvcNAQEN
#BQAwXTELMAkGA1UEBhMCSkExDjAMBgNVBAgMBW9zYWthMQ4wDAYDVQQHDAVvc2Fr
#YTERMA8GA1UECgwIYXN0ZXJpc2sxDTALBgNVBAsMBGhvbWUxDDAKBgNVBAMMA3d3
#dzAeFw0yMzExMDMwNjAzNDZaFw0zMzEwMzEwNjAzNDZaMF0xCzAJBgNVBAYTAkpB
#MQ4wDAYDVQQIDAVvc2FrYTEOMAwGA1UEBwwFb3Nha2ExETAPBgNVBAoMCGFzdGVy
#aXNrMQ0wCwYDVQQLDARob21lMQwwCgYDVQQDDAN3d3cwggEiMA0GCSqGSIb3DQEB
#AQUAA4IBDwAwggEKAoIBAQCgm2ImezVrDle27BmrzaTK6a0Ucgx00nqLj/wlkcsb
#Iykt7020DVmCiQ0eaxslWcDKkCzB4OTC7VrvvX2jzffy6lPGI9Qzo2D787akSTB6
#q9kI2q8aSvHovU8AGBwjDDxz9JRXP7Jf4RhLlxczCrFI9FuG3N5t19s8iig4n/VO
#EzB3M2WfQoAo2zzayArbeY0e61AYLkTM8rYlWQRYy9ctSnvglcBrhtnQUaw1HAZg
#rrt4YpeXgOkm5DjRiHKCP0G09jL3qRu+OVqBm8iKXmnenzTjv5dbeYVivWxL+c/M
#xy7EnMZExQ6uML8Ub6dSMq0HgqdNKhpOYWzzbUeBaLPrAgMBAAGjUzBRMB0GA1Ud
#DgQWBBTsUjWKPrTfXmqkkOFq2cV2H3e0TTAfBgNVHSMEGDAWgBTsUjWKPrTfXmqk
#kOFq2cV2H3e0TTAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBDQUAA4IBAQBl
#fjvu7+vQDt7LJcE4WDqDLU8HR5oCcDWVK0JOWdKTfM6Sfhr+MEJtW9DMy4cj/WK3
#aWuGx4otTvunR3YDlVSGksJfzzKbun/+NzYx20H4q9Q4tdA9QCzeGpf9LV/9ykup
#GvOF0Uxg1Abroi3AfYaQbD1/uT/GJInb3cBKKWbNF7Y3nuqMHu7WVbPvcG8IW2CR
#+YIruJz18kr531kiCDviSbAJZOVUWTIsaPUwZB/3JrS+f1XNkXFhC0E9NCjlDxXD
#B+iD6iXnchYHyhWkSg2UCmNU6LPnxNO+n0iPkHAcstg/2EeUj36xST0vNNeDm6Zh
#EEpQBuYZ8CC/O5XgNXIc
#-----END CERTIFICATE-----
#
#...
```

デフォルトの証明書は、以下のコマンドで再インストールできます。

```bash
dnf -y reinstall ca-certificates
```

以上
