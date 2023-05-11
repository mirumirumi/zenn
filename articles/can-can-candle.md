---
title    : Rust で仮想通貨の OHLCV データ取得用 CLI ツールをつくった
emoji    : 🕯️
type     : tech
topics   : [rust, 仮想通貨, 個人開発]
published: true
---

仮想通貨の bot を書くとき、入力となるチャートのデータがたくさん欲しくなります。これはプログラム内のライブラリから取れる値として欲しいというよりは、手軽にスタンドアロンで動かせるデータフェッチがしたいというのに近いです。

今回はそれを CLI アプリケーションとしてつくってみました。軽量で高速、Rust 製です。

https://github.com/mirumirumi/ro-soku

特徴はこんな感じ：

- いかなるセットアップやコンフィグも必要ありません
- 取引所ごとに異なるエンドポイント仕様や表記方法をいちいち調べる必要がありません
- コマンドをインタラクティブにつくっていけるガイドモード搭載
- 用途に応じたデータ加工方法や出力形式をサポート

![ro-soku-screenshot-1](/images/ro-soku-screenshot-1.png)

![ro-soku-screenshot-2](/images/ro-soku-screenshot-2.png)

シンプルなバイナリなので、プログラムやスクリプトから直接コマンドとして実行するのも簡単です。取引所クライアントライブラリを使えないシーンなどでも役に立つかもしれません。

ぜひ触ってもらって、機能要望やコントリビュートなどしていただければ幸いです！

## どんなツールなの

OHLCV とは、チャートが示す内容を構造化データとして表現したものです。「Kline」と呼ばれることもあります。

- **O**pen price:開始時点の価格
- **H**igh price: 最高価格
- **L**ow price: 最低価格
- **C**lose price: 終了時点の価格
- **V**olume: 取引量

実際にはこれにタイムスタンプ（多くはミリ秒の Unixtime 形式）が先頭に付与されて扱われることが多いです。

![ohlcv-kline.png](/images/ohlcv-kline.png)
*チャート画面上部でこのように表記されている部分が OHLCV で、これが各ローソク足そのものです。*

この緑とか赤の一本一本を「ローソク足」といい、各足の連なりは時間の長さごとで別物のデータとなります。例えば「1 分足」の場合は「1 分ごとのローソクが並んでいるチャート」になる、という具合です。

ちなみに今回つくったツールの名前および実行コマンドは `ro-soku` です :)

```bash
ro-soku \
    --exchange binance \
    --type perpetual \
    --symbol BTC/USDT \
    --interval 1min \
    --term-start 2023-05-10T06:28:28+00:00 \
    --term-end 2023-05-10T06:31:37+00:00 \
    --pick t,o,h,l,c,v \
    --order asc \
    --format csv
```

```raw
1683700140000,27588.7,27596.8,27588.7,27596.7,78.965
1683700200000,27596.8,27596.8,27596.7,27596.8,50.227
1683700260000,27596.7,27596.8,27592,27592,47.811
```

英語でも candleline という表現で同じものを指します。つまり OHLCV ≒ Kline ≒ candleline ですね。 

で、これらが大量に並んでいるものがチャートのデータということになるわけなのですが、所望のデータを得るには意外にもたくさんのインプットが必要とされます：

- 取引所はどこ？
- どの通貨ペアのチャート？
- ローソク足の単位は？
- いつからいつまでのデータがほしいの？

上記のコマンド例にオプションが並んでいることからもわかりますが、これらを毎度調べて必要なデータを得るのはとてもしんどいです。なおかつ、取引所によってパラメータはおろか API の基本仕様すら違うことも忘れてはいけません。

トレード bot を書くときにはこれらを吸収してくれる取引所クライアントラッパーライブラリを使うのが常套手段ですが、OHLCV データを普段の作業の中で得たいユースケースに限ってはその使い方だとどうしても不便です。

なので単品で動かせる CLI ツールは（少ないとしても）一定の需要があるのではと思っています。SaaS にも類似のサービスはあるにはあるのですが、コピペがいらない、プログラム的に扱えるなどはメリットになりそうです。

その他、詳しい使い方はリポジトリの [README](https://github.com/mirumirumi/ro-soku#readme) をどうぞ！

## あとなんかいろいろ

雑感を書く。

### 取引所の突然の仕様変更を検知する

外部のサービスに依存している時点で仕様変更は既に怖い存在ではありますが、仮想通貨界隈には常識的なサービスポリシーなど一切ないわけなので、昨日まで動いていた API が今日は動かないなど驚くに値しません。

:::message
実際、今回実装した取引所のうちのひとつである [Kraken](https://www.kraken.com/) は、API にバグがあり結局リリースを断念しました。
仕様は取引所によって恐ろしいほど違いますが、本当に理解に苦しむ設計になっているものも多く、ラッパーライブラリのありがたみを再認識します…。
:::

というわけで、統合テストを実施するついでにそれを定期実行して失敗を検知する仕組みを導入してみました。

恥ずかしながら GitHub Actions で cron イベントが使えることを少し前まで知らなかったので、どこかで使ってみたいと思っていました。

```yaml
on:
  schedule:
    - cron: "0 8 * * 6"
```

リリース前の CI でのテストと同じものを使えるのでいい感じです :)

### Rust 素晴らしいです

人生で一番プログラムを書くのが楽しかった気がします。人気プログラミング言語ランキング堂々 1 位の実績は伊達じゃない。好かれる理由が書けば書くほどによくわかっていける感じでした。

まだまだ初心者に毛が生えた程度のレベルですが、しばらくはずっと Rust を書いていきたいな。ウェブは合わないうんぬん～とかあるけど、自分は全くそうは思いませんでした。最近はこれまで個人で書いてきたサーバーも Rust に書き換えたりしているくらいなので（これは練習の意味もあるので適材適所ある論に真っ向から反対するつもりはないです）。

もっと流行ってほしくて、サーバーとして Go ではなく Rust を取り入れる風潮が増えてほしくて、そして僕を Rust で迎え入れてくれる求人が増えｔ

 ͏

おわりましょう。

### 絵文字で遊んでたらなんかかわいいブランドマークみたいのが完成した

![ro-soku-emoji](/images/ro-soku-emoji.png)

一体なんの生き物なんでしょうか。