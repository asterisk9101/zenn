---
title: "非ドメイン環境の Windows Server で WSUS を参照する"
emoji: "✴"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Windows, Server2022, memo]
published: true
---

お家の検証サーバ用の備忘録です。基本 Administrator です。

## 前置き

非ドメイン環境の Windows で Windows Update を実行するとき、WSUS を参照するように設定する方法です。

複数の Windows Server で設定するのが手間だったので PowerShell で設定できるようにしましたが、`gpedit.msc` で、以下のポリシーを設定した場合と同等になるはずです。

```gpedit.msc
ローカルコンピューターポリシー
    コンピューターの構成
        管理用テンプレート
            Windowsコンポーネント
                Windows Update
                    自動更新を構成する
                    イントラネットの Microsoft 更新サービスの場所を指定する
```

## レジストリの追加

参照先の WSUS サーバの IP アドレスとポート番号は以下とします。

```powershell
$WSUS = "192.168.1.52"
$PORT = "8530"
```

参照元の WSUS クライアント（？）となる Windows Server の PowerShell で以下を実行します。

```powershell
# 現状の確認
# デフォルトでは何も設定されていないはず
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"

# 参照先の設定
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" /v "WUServer" /t REG_SZ /d "http://${WSUS}:${PORT}" /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" /v "WUStatusServer" /t REG_SZ /d "http://${WSUS}:${PORT}" /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" /v "UpdateServiceUrlAlternate" /t REG_SZ /f /d ""
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" /v "SetProxyBehaviorForUpdateDetection" /t REG_DWORD /d 0 /f

# Windows Update の自動更新の確認
# 既定ではダウンロードして通知（インストールせずに）を行う設定(0x3)が入っているはず
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU"

# 自動更新の設定
# 特に設定しなくても変化ないはずだが、UseWUServer を設定しないと、WSUS で認識してくれなかった。
# 自動更新のポリシーを設定するとスケジュールのキーも追加されるが、今回は自動更新はしない設定なので追加しなかった
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v "AUOptions" /t REG_DWORD /d 3 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v "UseWUServer" /t REG_DWORD /d 1 /f
```

以上
