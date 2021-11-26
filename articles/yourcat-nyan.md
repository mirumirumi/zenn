---
title: "🐱 だけ限定のフォト共有サイトをつくったよ（Vue + SAM）"
emoji: "🐾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vue, aws, githubactions, 個人開発]
published: true
---

猫の写真だけをアップロードできて、かつその表示は毎回ランダムで一期一会、というフォト共有サービスをつくってみたという話。

技術要素は

- Vue3 (Vue Router、Vuex)
- AWS SAM (一部Terraform)
- GitHub Actions

あたりです。

よければ一枚、あなたも置き土産していってくださいませ。

## 概要

こんなサイトです。

![](/images/yourcat-screenshot.png)

👉 [YourCat](https://mirumi.me/apps/yourcat)

特徴は以下。

- 投稿できる写真は猫が写っているものだけ（機械学習でフィルタリングします）
- アカウント登録も写真へのコメントもなーんもいらないです
- 毎回サイト上に表示される猫は全登録写真からランダム→数が増えれば増えるほど二度目以降の再会は難しい…！

本当に一瞬で使えるので、何か 🐱 写真をお持ちの方はとりあえず一枚だけでもお試しでアップロードしていってくださると嬉しいです～～～

## 技術要素と所感

使用したもの、および雑感です。  
以下はリポジトリのリンク。

https://github.com/mirumirumi/your-cat-vue

https://github.com/mirumirumi/your-cat-sam

### Vue

Web で見かける小さな UI 部品の中で自分が好きだなーと思うものをちゃんとリリースできる成果物に含めてみたいなーとずっと思っていて、それを今回は Vue でたくさん書けたのが楽しかったです。

いくつか紹介させてください。

#### blur なモーダルバック

しょっぱなから Vue 全然関係ないんですが、モーダルのバックを暗くするんじゃなくてぼかすやつをやりたかったんですよ～。

![](/images/yourcat-screenshot-19.png)

Apple の UI っぽくなってモダンになるのと、一応ギャラリーサイトの側面があるので暗くしちゃうのはなんか悲しいかなーと。ぼやかすとカラフルな写真タイル感が上品に残る気がします。

#### スケルトンローディング

最も Vue の恩恵を受けられた部分だったと思います。

![](/images/49a254b302d2.gif)

表示のたびにスケルトンブロックのサイズや個数をランダム変動させる仕組みにしてみました。

```vue
<template>
  <div class="loading-block-wrap" :style="{'width': size.width * 133 / size.height + 'px', 'flex-grow': size.width * 133 / size.height}">
    <i :style="{'padding-bottom': size.height / size.width * 100 + '%'}"></i>
    <div class="loading-block" :class="{'stop-loading': isStopAnimation}" :height="size.height"></div>
  </div>
</template>
```

表示まで含めても大した実装ではないこと、写真表示と同様のインターフェースにしたかったことなどからライブラリの採用は見送りました。

#### 進捗アニメーション

ユーザーからの操作１つにつきアプリケーションからの応答１つ、というのは最もよくあるインタラクションですが、それらが連なって一連の動きを取るものを作ってみたいと考えていました。

![](/images/1bb019e62584.gif)

これも細かいところでかなり時間を要しました。フロントエンドの実装って本当に工数を必要としますよね。でも「見た目やデザインだから」って言われたりしやすいわけで、実務でやっている方たちの苦労が少し想像できた感じでした…。

#### フォトタイル CSS

これは Vue とは直接的には関係ないのですが、フロントエンド側の話題ということでここに書きます。

![](/images/yourcat-screenshot-9.png)

このようなデザインは比較的よく見かけるので「簡単な CSS で書けそうやな？」って思うでしょう？

ところがどっこい、全然そんなことはありませんでした。

最初は「横幅を統一して縦に並んでいくデザイン」にしたのですが（下図参照）、これだと画像の配列要素が横向きに進んでいかないためプログラム的にかなり扱いづらいということでやめました。

![](/images/image1.jpg)
*出典：https://www.mdn.co.jp/di/contents/3265/39761/*

そこでこの「Google フォト」もしくは「Adobe ストック」風のデザインを目指したわけなのですが、自分で書こうと思ってもすぐに筆が止まりました。止まるのはタイピングか。

というわけでこのやたら長い [GitHub issue スレッド](https://github.com/xieranmaya/blog/issues/6) に全面的に助けられました。

https://github.com/xieranmaya/blog/issues/6

最終的には JS 含めこんな感じになります。

```vue
<div class="photo-wrap">
    <div class="photo" v-for="photo, index in photoArray" :key="photo" :style="{'width': photo.size.width * 200 / photo.size.height + 'px', 'flex-grow': photo.size.width * 200 / photo.size.height}">
    <i :style="{'padding-bottom': photo.size.height / photo.size.width * 100 + '%'}"></i>
    <img :src="photo.src" :alt="photo.title" @click="onClickPhoto(index)" @load="loaded(index)" crossorigin="Anonymous"/>
</div>
```

おかげで美しいフォトギャラリーになったと思ってます！

ちなみに画像ビューアには spotlight.js を使いました。

https://github.com/nextapps-de/spotlight

マークアップで反応させられるようにする形式と、API 的に使える方法とで使い分けられるのが気に入りました。シンプルなデザインもよいところ。

![](/images/yourcat-screenshot-10.png)

ただちょっとだけ bug があって、CSS などで暫定対応せざるを得ないシーンがいくつかありました。issue を継続監視しているのでいつか直るかもです。

```js
onClickPhoto(index) {
    Spotlight.show(this.photos, {
        index: index,
        animation: "slide, fade",  // "play"
        control: "zoom, close",
        title: false,
        // play: 3,  bug (http://github.com/nextapps-de/spotlight/issues/49)
        onshow: (index) => {
            Spotlight.addControl("dl-btn", (event) => {
                execDownload(this.photos[index].src);
            });
        },
        onclose: () => {
            Spotlight.removeControl("dl-btn");
        },
    });
},
```

### AWS SAM

SAM は何度となく使ってきているので特に目新しいことはないのですが、初めて個人アカウントで環境分け、およびそれにともなう CI/CD パイプラインの構築を行いました。

実務では

- 開発 (dev)
- 検証 (stg)
- 本番 (prd)

と３つに分けることがよくあるパターンですが、個人では真ん中は特に必要としないので

- 開発 (dev)
- 本番 (prd)

としました。

AWS が[規約](https://d1.awsstatic.com/legal/AWSFreeTierTerms/AWS%20Free%20Tier%20terms%20-%20JAPANESE%20%20%282018-08-14%29%20UPDATED.pdf)で定める「不当に無料利用枠を得ようとする行為」に該当しないか不安になるところですが、

> お客様またはお客様の事業体が複数のアカウントを開設し、本オファー基づく追加特典を受けている場合、お客様は本オファーを受けることはできません。  

[Discussion Forums](https://forums.aws.amazon.com/thread.jspa?messageID=381770) にて AWS 側が下記のような回答を正式に行っているのを発見したので安心です。

> Creating multiple accounts is not against the terms of service. It is OK to have different accounts for test and production systems, for example, or for separate business units which have completely distinct email sending that doesn't overlap. However, attempting to circumvent service limits or policies is prohibited by the terms of service, and creating multiple accounts for the purpose of circumventing limits would be a violation of the TOS.

というかごくごく普通のソリューションなので、それを個人でやることが咎められるのは考えにくいですね。

#### AWS Rekognition

あと強いて言うなら [AWS Rekognition](https://aws.amazon.com/jp/rekognition/) は初めて使いました。

今回のような物体ラベルづけはもとより、顔認証や顔画像の一致度比較程度なら API １つでラクに使えるのは驚きでした。ほんでもってめっちゃ安いです。

例えばラベリングである `detect_labels()` は下記のように使います。

```python
from boto3 import Session

session = Session(region_name="ap-northeast-1")
client = session.client("rekognition")

res = client.detect_labels(Image={
    "Bytes": file
})
```

レスポンスのデータを使って画像内にバウンディングボックスを表示することができます。

![](/images/yourcat-screenshot-4.png)

API リファレンスは下記。

https://docs.aws.amazon.com/rekognition/latest/dg/API_Reference.html

ただし実際はランタイムごとの SDK のリファレンスを見た方がわかりやすかったです。少なくとも Python(boto3) はそうでした。

https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/rekognition.html

今回はサービスの軸として使いましたが、「要らんものを弾く」という簡易なフィルタリングとしても十分実用に耐えうるサービスだと思います。レスポンスはだいたい 200~300ms くらいでした。

### CI/CD

今回は Code Build などは使わず GitHub Actions のみの運用としました。

理由は

- フロントエンドとバックエンドでリポジトリを分ける体制を取っていたため、両者で使うツールは統一したかった
- フロントエンド側は AWS ではなくホスティングサーバーへのデプロイのため、AWS 側のサービスを使う必要性がなかった
- 単にお手軽

という感じ。

SAM は `sam deploy` するだけなのでいいとしても、ホスティングサーバーへのデプロイというのは初めてなのでちょっとだけ苦戦しました。

`scp` したいわけなのですが、定義済みの GitHub Actions である [scp-action](https://github.com/appleboy/scp-action) を使えば一発かなーと思いきや、ディレクトリ構成がうまく制御できず結局セルフベタ書きすることになりました。

その結果クソゴリラコードになってしまったのですが、まあよい勉強になりました。

SAM 側はいつもどおり `samconfig.toml` で環境区分しています。本番環境へのフックはプルリクのマージのみ許容です（これは Vue 側も同様）。

```toml
- run: sam build --use-container
- if: ${{ github.ref == 'refs/heads/main' }}
  run: sam deploy --no-confirm-changeset --no-fail-on-empty-changeset --config-env dev
- if: ${{ github.ref == 'refs/heads/release/prd' }}
  run: sam deploy --no-confirm-changeset --no-fail-on-empty-changeset --config-env prd
```

### Twitter API

新しい 🐱 が投稿されたらその写真をツイートしてくれる Twitter アカウントもつくりました。

https://twitter.com/YourCat_photos/status/1462679063211147268

サーバーは Python で書いたのですが、Twitter の API には [TwitterAPI](https://github.com/geduldig/TwitterAPI) というダイレクトな名前のパッケージが公式で紹介されていたのを発見したのでそれを使いました。

https://developer.twitter.com/en

https://github.com/geduldig/TwitterAPI

```python
from TwitterAPI import TwitterAPI

api = TwitterAPI(
        consumer_key,
        consumer_secret,
        access_token_key,
        access_token_secret
      )
api.request('statuses/update', {'status':'This is a tweet!'})
```

見てわかる通り、OAuth 認証に使う例の４つのキー類もそのまま扱えて、次の行ではもうツイートのリクエストができちゃいます。

インストールは `pip` で。

```
pip install TwitterAPI
```

今度から毎回これ使おう。

## おわりに

個人開発あるあるだと思うのですが、「利用者さえ揃えば素敵なものになる」ってやつあるじゃないですか。

ユーザーコンテンツがないと成立しないのはビジネス観点で言えば当然なんですけど、でもそこにマーケティングしてお金かけて…というモチベーションはやっぱり湧かないと。お金使ったからといって素人が簡単に人を集められる保証もないですしね。

そんなわけで、みなさんが にゃんこ🐱写真 を提供してくれないと悲しいことになってしまいます。

ぜひ！！！
なにとぞ！！！！

…お願いします。

<br>

P.S.  
わんこ🐶 の写真をお持ちの方も一度お試しあれ。











