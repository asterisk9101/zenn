---
title: "同じ構成の表の値を合計する"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Windows, PowerShell]
published: true
---

R とか SQL だと上手いやり方がある気がする。再発明をしている。

こんな感じの表が２つある時に

|label  |item1  |item2  |
|-------|-------|-------|
|A  |1  |2  |
|B  |2  |4  |
|C  |3  |6  |
|D  |4  |8  |

|label  |item1  |item2  |
|-------|-------|-------|
|A  |2  |1  |
|B  |4  |2  |
|C  |6  |3  |
|D  |8  |4  |
|E  |10 |5  |

これらの B 列と C 列を足した新しい表を作りたい。尚、label はユニークで、行は増減する可能性がある。列の増減は無い。

まず、前提となる表を 2 つ作る。

```powershell
$table1 = 1..4 | % { [pscustomobject]@{ label=[char](64+$_); item1=$_; item2=($_*2) } }
$table2 = 5..1 | % { [pscustomobject]@{ label=[char](64+$_); item1=($_*2); item2=$_ } }
```

この状態では使いにくいので、label で要素にアクセスできるよう辞書に詰め込む。

```powershell
$dic1 = @{}; $table1 | % { $dic1.add($_.label, $_) }
$dic2 = @{}; $table2 | % { $dic2.add($_.label, $_) }
```

label を列挙する。途中の `% { $_ }` は、いわゆる flatten をしており、`sort -u` で重複する label を排除している。

```powershell
$table1.label, $table2.label | % { $_ } | sort -u | % {
    [pscustomobject]@{
        label = $_
        item1 = $dic1[$_].item1 + $dic2[$_].item2
        item2 = $dic1[$_].item2 + $dic2[$_].item2
    }
}
```

以上
