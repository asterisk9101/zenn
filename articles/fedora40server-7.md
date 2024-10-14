---
title: "Fedora40 で fail2ban を動かす"
emoji: "🚁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Linux, Fedora, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 root です。

## 前提

続きです。

<https://zenn.dev/asterisk9101/articles/fedora40server-5>

## インストールする

```bash
dnf -y install fail2ban
systemctl enable --now fail2ban
systemctl status fail2ban
```

以上（気休め？）
