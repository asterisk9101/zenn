---
title: "Fedora44 workstation を作るときのメモ"
emoji: "👒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, memo]
published: true
---

お家の検証用の備忘録です。

## 前提

Proxmox VE の上に作ります。

CPU 4 core, メモリ 8GB ぐらいあると安心です。

## インストール

live 環境が表示されるので `Install Fedora Linux...` をクリック。

![1](/images/fedora44workstation-1/001.jpg)

日本語を選択し、キーボードのレイアウトの変更をクリックします。

![2](/images/fedora44workstation-1/002.jpg)

設定画面が表示されるので、「Add Input Source」をクリックします。

![3](/images/fedora44workstation-1/003.jpg)

キーボードレイアウトの選択画面が表示されるので、「Japanese」をクリックします。

![4](/images/fedora44workstation-1/004.jpg)

元の画面に戻るので、日本語を優先に並べ替え、右上の「x」で画面を閉じます。

![5](/images/fedora44workstation-1/005.jpg)

「次へ」をクリックします。

![6](/images/fedora44workstation-1/006.jpg)

インストール先を確認されますが、変更せずに「次へ」をクリックします。

![7](/images/fedora44workstation-1/007.jpg)

ストレージを暗号化するか聞かれるので、変更せずに「次へ」をクリックします。

![8](/images/fedora44workstation-1/008.jpg)

確認画面が表示されるので、「データを消去してインストールします」をクリックします。

![9](/images/fedora44workstation-1/009.jpg)

インストールの完了画面が表示されたら、「ライブデスクトップに戻る」をクリックします。

![10](/images/fedora44workstation-1/010.jpg)

右上のシステムメニューをクリックし、「Restart...」をクリックします。

![11](/images/fedora44workstation-1/011.jpg)

確認画面が表示されるので、「Boot Options」をクリックします（多分表記ミスだと思います。本来は Restart now みたいな感じかと）

![12](/images/fedora44workstation-1/012.jpg)

## 再起動

再起動するとセットアップ画面が表示されるので、「セットアップを開始」をクリックします。

![13](/images/fedora44workstation-1/013.jpg)

プライバシーの設定画面が表示されるので、「次へ」をクリックします。

![14](/images/fedora44workstation-1/014.jpg)

タイムゾーンの選択画面が表示されるので、日本付近をクリックして「次へ」をクリックします。

![15](/images/fedora44workstation-1/015.jpg)

サードパーティのリポジトリを有効にするか聞かれるので、変更せずに「次へ」をクリックします。

![16](/images/fedora44workstation-1/016.jpg)

ユーザー情報を聞かれるので、ユーザー名を入力して「次へ」をクリックします。

![17](/images/fedora44workstation-1/017.jpg)

パスワードを聞かれるので、パスワードを入力して「次へ」をクリックします。

![18](/images/fedora44workstation-1/018.jpg)

完了画面が表示されるので、「Fedora Linux を使い始める」をクリックします。

![19](/images/fedora44workstation-1/019.jpg)

## セットアップ完了後

ツアーが表示されるので、「スキップ」をクリックします。

![20](/images/fedora44workstation-1/020.jpg)

アプリケーション画面を開き、「ターミナル」を右クリックして「ダッシュボードにピン留め」をクリックします。

![22](/images/fedora44workstation-1/022.jpg)

ターミナルをクリックします。

![23](/images/fedora44workstation-1/023.jpg)

`sudo su -` できるので `wheel` グループであることが分かります。

![24](/images/fedora44workstation-1/024.jpg)

ディスクの使用量を確認。

![25](/images/fedora44workstation-1/025.jpg)

メモリの使用量を確認。

![26](/images/fedora44workstation-1/026.jpg)

CPU コア数を確認。

![27](/images/fedora44workstation-1/027.jpg)

`dnf -y update` して再起動を行います。

![28](/images/fedora44workstation-1/028.jpg)

`dnf -y update` は何度か実施する必要がありました。

この時点で一度バックアップを作成しておきます。

長かったので一旦以上です。

## 次

<https://zenn.dev/asterisk9101/articles/fedora44workstation-2>
