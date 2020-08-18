# AMP story を amp-optimizer にかけたときのバグ調査メモ

- story-sample.html -> amp-img のみの story ページの HTML
- sample.html -> amp-img のみの HTML

## story-sample.html 

amp optimizer をかける

```
$ npx @ampproject/toolbox-cli optimize story-sample.html > story-optimized.html
```

それを amphtml-validator にかけると invalid だと言われる

```
$ npx amphtml-validator story-optimized.html
story-optimized.html:579:12 The parent tag of tag 'img' is 'amp-img', but it can only be 'i-amphtml-sizer-intrinsic'.
```

## sample.html

amp optimizer にかける

```
$ npx @ampproject/toolbox-cli optimize sample.html > sample-optimized.html
```

validatorにかけても valid

```
$ npx amphtml-validator sample-optimized.html
sample-optimized.html: PASS
```

## 調査

今回バリデーションでひっかかっている

`The parent tag of tag 'img' is 'amp-img', but it can only be 'i-amphtml-sizer-intrinsic'.`

は、code: WRONG_PARENT_TAG というエラーで、mandatory_parent で指定されている必須の親要素が存在しない場合に出力される。
spec でいうとこれ。

https://github.com/ampproject/amphtml/blob/master/validator/validator-main.protoascii#L4404-L4435

```
tags: {
  html_format: AMP
  enabled_by: "transformed"
  tag_name: "IMG"
  spec_name: "IMG-I-AMPHTML-INTRINSIC-SIZER"
  mandatory_parent: "I-AMPHTML-SIZER-INTRINSIC"
...
```

img タグの親に i-amphtml-sizer-intrinsic が必要だと言っている。
が、story-optimized.html にはそんな要素は存在しない。
なので、spec だけ見ると、それはそう、という感じなのだが、
しかし、sample-optimized.html にもそんな要素は存在しない。
なのに、こちらは validator が PASS している。

この spec 一体何者なんだろうかと探してみると、↓のPRで入ってることがわかる。

https://github.com/ampproject/amphtml/pull/24119/

Validator support for transformed intrinsic layout なので、intrinsic layout が変換されたときに自動的にできるタグに対して validator をサポートした様子。
よくよく上記specをみてみると、確かに attribute もそれ用にバリデートされている。  

sample.html も story-sample.html も amp-img の layout は responsive なので、intrinsicじゃない。
どこで判断してるかわからないけどおそらく layout が intrinsic の場合に validation する spec なので、
sample.html は valid (そもそもspecを通ってないか、結果が無視されている)だが、
amp-story を使っていると通さなくて良い or 無視すべき spec で検査してしまっているために起きてそう。

ちなみに amp-optimizer の preloadHeroImage をスキップすると img タグの inject がなくなるので、

```
          <amp-img
            src="https://amp.dev/static/samples/img/story_dog2.jpg"
            width="1"
            height="1"
            layout="responsive"
            alt
            class="i-amphtml-layout-responsive i-amphtml-layout-size-defined"
            i-amphtml-layout="responsive"
            ><i-amphtml-sizer
              style="display: block; padding-top: 100%;"
            ></i-amphtml-sizer>
          </amp-img>
```

img タグに対する上記specの対象じゃなくなるので問題なく validator が PASS する。

next.js を普通に使っている人たちは optimizer も validator も埋め込みでoptimizerのオプションやspecを変更なんてしてないはずなので、next build で生成された世の中の web story はほとんど amp invalid になってると思われる。

