# 続・Nominatimが広島平和記念資料館を見つけられない理由とその解決方法

author
:   阿部 智晃

institution
:   ClearCode Inc.

theme
:   clear-code

# 自己紹介

* 阿部 智晃
  * GitHub: abetomo
* 株式会社クリアコード
  * FOSSの開発・サポートを仕事にしている
  * 全文検索エンジンの開発が主

# FOSS4G 2026 Hiroshima

**Nominatimで「広島平和記念資料館」が見つからない**

* 原因: 日本語だといい感じに検索できない！
  * **検索最適化が裏目に出る**
  * 漢字は中国語の読み
* 解決: PGroongaで全文検索！
  * PostgreSQLの拡張機能

# 「北海道大学付属図書館」も見つからない！

# 「北海道大学付属図書館」も見つからない！

![](images/openstreetmap-example.png)

# 今回のお話

* PostgreSQLの拡張機能の追加なしでなんとか見つけたい！
* 「検索最適化が裏目に出る」を改善
* Pythonのコードを50行くらいで実現
  * データベースのデータはそのままでOK

# なぜ見つけられないか

「ペナルティ」がいい感じに設定されないから。

# お知らせ

このあと「ペナルティ」についての説明をしていきますが、わかりやすさ優先のため単純化しています。

ご了承ください。

# ペナルティ

ペナルティは単語（文字）と単語（文字）のくっつきやすさ。

* 「北海」と「道」はくっつきやすい => ペナルティ0
* 「北海」と「阿部」はくっつきにくい => ペナルティ0.5
* 「北海道」は住所でよく使われる => （住所検索においては）ペナルティ0
* 「阿部智晃」は住所で使われにくい => （住所検索においては）ペナルティ0.5

{::note}ペナルティの値が小さいほど、より良い検索と見なす{:/note}

# ペナルティがいい感じに設定されない

* 漢字（日本語・中国語）の場合、基本的に一文字区切り
  * 形態素解析のようなことは行わず、単純に1文字区切る
  * `北 | 海 | 道 | 大 | 学 | 付 | 属 | 図 | 書 | 館`
  * 意味を無視して単純に分割しているのでくっつきやすいうんぬんが関係なくなる…
* 検索クエリーが長ければ長いほどペナルティが高くなる
  * 次のスライドで説明

# 検索クエリーが長いとペナルティ高

* 長: `北 | 海 | 道 | 大 | 学 | 付 | 属 | 図 | 書 | 館`
  * `|` ごとにペナルティ0.3
    * 区切りに意味もないので共通で0.3
  * 0.3 * 9 = 2.7
* 短: `北 | 海 | 道`
  * `|` ごとにペナルティ0.3
    * 区切りに意味もないので共通で0.3
  * 0.3 * 2 = 0.6

{::note}いい感じに分割されないものの、クエリーが短いとペナルティ高の影響を受けにくい{:/note}

# ペナルティ高の補足

いい感じに分割されるとペナルティもいい感じに低く設定されるはず。

例: `北海道 | 大学 | 付属 | 図書館`

* 仮に単語で分割しているとすると、くっつきやすさの概念が出てきて「北海道 | 大学」はあり得るからペナルティ0などになる
* この例だと全部くっつきそうなので0 + 0 + 0 = 0でペナルティ0とかの可能性もある

# なぜ高いと見つからないか

1. 「どこまでが名前で、どこからが住所か」の候補をたくさん作る
   * {::note}詳細はFOSS4G 2026 Hiroshimaの資料の巻末をご覧ください{:/note}
2. ペナルティの小さい順に試す
3. 一番小さい候補 + 1.5 を超えたら打ち切り
   * 「北海道大学付属図書館」の例ではクエリーが長くペナルティが高くなり打ち切られている

**正解の候補は作られているのに、ペナルティが高すぎて試されない**

# 今回のアイディア

ペナルティを適切に設定するために日本語の分割（形態素解析）を改善してうんぬんを言い始めると大変。

日本語（特に漢字）が連続するものはペナルティが低めで良いのでは？

# 今回のアイディア: 具体例

`北 | 海 | 道 | 大 | 学 | 付 | 属 | 図 | 書 | 館`

* Before
  * `|` ごとにペナルティ0.3
  * 0.3 * 9 = 2.7
* After
  * `|` の前後が日本語（かな・カナ・漢字）だったらペナルティ0.1
    * くっつきやすいはずなので低めにする
  * 0.1 * 9 = 0.9

# 実装

{::note}追加したコードの一部{:/note}

```python
def get_cj_partial_penalty(self, trange: TokenRange) -> float:
    penalty = self.nodes[trange.start].partial.penalty
    for pos in range(trange.start + 1, trange.end):
        if self.is_cj_continuation(pos): # 日本語が続いていたら
            penalty += PENALTY_CJ_CHAR # ペナルティは0.1
        else:
            node = self.nodes[pos]
            # そうでなかったら既存のペナルティの追加
            penalty += node.partial.penalty + node.word_break_penalty
    return penalty
```

# デモ: ペナルティ改善なし

```bash
docker compose exec -e NOMINATIM_CJK_PENALTY=off nominatim \
  nominatim search --query "広島平和記念資料館"
```

```bash
docker compose exec -e NOMINATIM_CJK_PENALTY=off nominatim \
  nominatim search --query "北海道大学付属図書館"
```

# デモ: ペナルティ改善あり

```bash
docker compose exec -e NOMINATIM_CJK_PENALTY=on nominatim \
  nominatim search --query "広島平和記念資料館"
```

```bash
docker compose exec -e NOMINATIM_CJK_PENALTY=on nominatim \
  nominatim search --query "北海道大学付属図書館"
```

# 副作用

見つからなくても良さそうなデータも見つかる…。

{::note}私は真に驚くべき解説をここでしたいが、このスライドはそれを書くには狭すぎるので、ブログ記事で解説します。{:/note}

# 効果がある範囲

* 日本語（J）
  * 今回発表した通り
* 中国語（C）
  * 未検証。同様に分割されるので効果はあると思われる
* 韓国語（K）にはない
  * 一文字ずつ分割されないので前提が違う
  * 先ほどの実装例が`cj`だったのはこのため

# 他のアプローチ

**日本語の単語ごとに正しく区切って、正しくペナルティを設定する。**
が素直なやり方。

ICUのBreakIteratorを使うといい感じに分割できそうではある。
が、分割結果が変わるとデータの再インポートが必要だったり大変な側面も…。

**もっと良いアイディアがあれば教えてください！**

# まとめ

* ペナルティがいい感じに設定されていなかった
  * **日本語が連続するものはペナルティを低めにして改善**
* そもそも一文字ずつの分割がよくない
  * ここから改善できるとより良い

# ありがとうございました

* Email: abe@clear-code.com
* 本PoC（ペナルティ調整）
  * https://github.com/abetomo/Nominatim （`penalty-tuning-for-cjk`ブランチ）
* デモ環境（Docker Compose）
  * https://github.com/abetomo/Nominatim-docker-compose
* FOSS4G 2026 Hiroshima
  * https://www.clear-code.com/blog/2026/8/19/nominatim-with-pgroonga-ja.html
