# UTF-8じゃないの？Pythonの文字列処理で火傷を防ぐ

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Soliton-Analytics-Team/BytePos/blob/main/BytePos.ipynb)

皆さんはPythonでUTF8の文字列のバイト位置を知りたいと思ったことありませんか。私はあります。DBから取得したデータがUTF8でそれを変換せずに直接処理したいことがあったからです。

UTF-8を直接処理？Pythonの文字列型はUTF-8じゃないの？と思った方は、[こちらの記事](https://www.haya-programming.com/entry/2020/04/24/035151?amp=1)を見てください。

例えば、Pythonの正規表現モジュールは、以下の様に検索語と被検索対象を共にバイト列を指定すればUTF8のまま処理が可能です。


```python
import re
key = '第\d+回'.encode('UTF8')
target = '本日は第3回全国大会にお集まりいただき、ありがとうございます。'.encode('UTF8')
m = re.search(key, target)
m.group().decode('UTF8')
```




    '第3回'



しかし、文字列をUTF8で表した場合の各文字のバイト位置が必要な時、どうすればいいでしょうか。


```python
s = "今日の日付は10月10日です。"
```

ナイーブな実装をすれば以下の様になるでしょうか。


```python
result = [0]
for c in s[:-1]:
    result.append(result[-1] + len(c.encode('UTF8')))
for c, pos in zip(s, result):
    print(c, pos, sep='\t')
```

    今	0
    日	3
    の	6
    日	9
    付	12
    は	15
    1	18
    0	19
    月	20
    1	23
    0	24
    日	25
    で	28
    す	31
    。	34


しかし、for文で回すのは格好が悪い。処理時間も短くしたい。

まずは文字列を増やします。


```python
s1k = "今日の日付は10月10日です。" * 1000
```

それでは処理時間を測ってみましょう。


```python
%%timeit
result = [0]
for c in s1k[:-1]:
    result.append(result[-1] + len(c.encode('UTF8')))
```

    100 loops, best of 5: 5.59 ms per loop


いや待ってください。嫌な予感がします。encode('UTF8')で文字列の引数を渡しているところです。encode()はデフォルトでUTF8なので、省いてみます。


```python
%%timeit
result = [0]
for c in s1k[:-1]:
    result.append(result[-1] + len(c.encode()))
```

    100 loops, best of 5: 4.68 ms per loop


なんと2割も処理時間が削減されました。文字列引数、恐るべし。指定してもしなくても同じだと思ったら大間違い。

次に内包表記で一行で書いたものを計測。もちろんもうencode()の引数は使いません。


```python
%%timeit
result = [len(s1k[:i].encode()) for i in range(len(s))]
```

    1 loop, best of 5: 283 ms per loop


シンプルに一行になりましたが、処理時間が全然だめです。そもそもアルゴリズムがだめです。これではオーダーがO(n^2)になってしまいます。なぜだか分かりますか？どんどん長くなる文字列のi番目までの文字列s[:i]を毎度encode()しているからです。

単に1文字ずつのUTF8のバイト長を得るだけならそんな時間はかかりません。


```python
%%timeit
result = [len(c.encode()) for c in s1k]
```

    100 loops, best of 5: 2.78 ms per loop


このリストの内容を累積すれば目的は達成できそうです。そして、リストの各要素を累積するaccumulate()という関数がitertoolsモジュールの中にありました。


```python
from itertools import accumulate
```

これを使って計測してみましょう。


```python
%%timeit
result = [0] + list(accumulate([len(c.encode()) for c in s1k[:-1]]))
```

    100 loops, best of 5: 3.47 ms per loop


さらに2割ほど処理時間が削減された上、一行で綺麗に書けました。めでたしめでたし。

本当にそうでしょうか。実際に実行された方でキャッシュ使われてるんじゃ？という警告文を目にした方、いらっしゃいませんか。%%timeitは100回実行してベスト5の平均をとりますが、どうもaccumulate()の引数に渡しているリストがキャッシュされて、初回以外の計測値が意図しないものになっている可能性がありそうです。

他の方法でも計測してみましょう。


```python
from itertools import accumulate
from timeit import timeit
timeit("result = [0] + list(accumulate([len(c.encode()) for c in s1k[:-1]]))", globals = globals(), number=100) / 100
```




    0.0037341063500002745



これもキャッシュ疑惑が拭えません。

繰り返さずに一回だけの処理速度を測ってみましょう。処理時間が短いと誤差が大きくなるので、文字列の長さをさらに10000倍にして測定します。

まずはナイーブな実装。


```python
s10m = "今日の日付は10月10日です。" * 1000 * 10000
result = [0]
%time for c in s10m[:-1]: result.append(result[-1] + len(c.encode()))
```

    CPU times: user 1min, sys: 2.55 s, total: 1min 3s
    Wall time: 1min 3s


次はaccumulate()を使った実装。


```python
from itertools import accumulate
s10m = "今日の日付は10月10日です。" * 1000 * 10000
%time result = [0] + list(accumulate([len(c.encode()) for c in s10m[:-1]]))
```

    CPU times: user 38 s, sys: 4.9 s, total: 42.9 s
    Wall time: 42.9 s


3割ほど処理時間が短い！

でもリストを２回も作成しているのが気になる。accumulate()はジェネレーターを返すということなので、expand()に渡してみましょう。


```python
from itertools import accumulate
s10m = "今日の日付は10月10日です。" * 1000 * 10000
result = [0]
%time result.extend(accumulate([len(c.encode()) for c in s10m[:-1]]))
```

    CPU times: user 36.1 s, sys: 2.46 s, total: 38.5 s
    Wall time: 38.6 s


おお！本日最速！

ただ、一行で書けてないのが残念。extend()が結果の配列を返してくれる仕様だったら良かったのに。

ちなみに、accumulate()の引数はこうも書けます。


```python
from itertools import accumulate
s10m = "今日の日付は10月10日です。" * 1000 * 10000
result = [0]
%time result.extend(accumulate(len(c.encode()) for c in s10m[:-1]))
```

    CPU times: user 37.7 s, sys: 2.15 s, total: 39.8 s
    Wall time: 39.9 s


あれ？全く同じだと思ったらちょっと遅い。この書き方だとリストじゃなくてジェネレータが渡るのかな？

確認のため、明示的にジェネレータを渡してみましょう。


```python
from itertools import accumulate
s10m = "今日の日付は10月10日です。" * 1000 * 10000
result = [0]
%time result.extend(accumulate((len(c.encode()) for c in s10m[:-1])))
```

    CPU times: user 37.8 s, sys: 2.14 s, total: 39.9 s
    Wall time: 39.9 s


同じだ。やはりこの書き方ではジェネレータが渡されるようです。そしてこの場合はリスト渡しのほうが速いと。試してみないと分からないものですね。

現場からは以上です。
