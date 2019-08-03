---
title: "Mathjaxで数式を書く"
description: "Mathjaxを導入して数式を書けるようにしました。"
date: 2019-08-03T14:21:00+09:00
categories:
  - 小ネタ
tags:
  - Mathjax
draft: false
mathjax: true
---

## 動機

記事の中で数式を書けるようにしたかった。

## 方法

[Mathjax](https://www.mathjax.org/)を使います。Mathjaxはブラウザ上で数式を表示できるようにするJavaScriptライブラリです。

このブログはHugoを使って生成しているので、Hugoのthemeテンプレートの中で下記のように呼び出しします。(私はfooter.htmlの中に書きました。)

```
{{ if .Params.mathjax }}
<!-- MathJax -->
<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
<script type="text/x-mathjax-config">
    MathJax.Hub.Config({tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}});
</script>
{{ end }}
```

`{{ if .Params.mathjax }}...{{ end }}` で囲っているのは、MathJaxが必要なページでのみMathjaxをロードするようにするためです。[このように設定する](http://example.com)ことでページ中でMathjaxがロードされるようになります。

また、下記の部分はMathjaxの設定を変更する記述です。

```
<script type="text/x-mathjax-config">
    MathJax.Hub.Config({tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}});
</script>
```

どの文字で囲うとMathjaxでインライン数式が有効になるか、という設定をしています。今回は`$...$`と`\\(...\\)`で囲うと数式になるようになっています。

試しに書いてみると $x = {-b \pm \sqrt{b^2-4ac} \over 2a}.$ のような感じになります。
MathjaxのConfigのデフォルト値は[こちら](http://docs.mathjax.org/en/latest/options/preprocessors/tex2jax.html#configure-tex2jax)に記載されていますので、ご確認ください。

