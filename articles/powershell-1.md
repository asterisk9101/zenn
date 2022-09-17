---
title: "日本の四半期を計算する関数"
emoji: "💧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Windows, PowerShell]
published: true
---

業務系の人はみんな持ってるはず。再発明をしている。

```powershell
function Get-JpQuarter {
    Param(
        [DateTime]$Date = ( Get-Date ),
        [int]$AddQuarter = 0
    )

    $Date = $Date.AddMonths($AddQuarter * 3)

    $JpYear = $Date.Year
    $Q = [Math]::Floor(($Date.Month - 1) / 3)
    
    # 3ヶ月先～3ヶ月前までの月を列挙して同じQになるDateTimeだけをフィルタしている
    $Months = -2..3 `
        | foreach-object { $Date.AddMonths($_) } `
        | where-object { [Math]::Floor(($_.Month - 1) / 3) -eq $Q } `
        | foreach-object { [datetime]$_.ToString("yyyy/MM/01") } 

    if ($Date.Month -lt 4) {
        $JpYear -= 1
        $Q = 4
    }

    return [pscustomobject]@{
        JpYear = $JpYear
        Q = $Q
        Months = $Months
    }
}
```
