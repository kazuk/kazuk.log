---
title: "Rust の中間言語インタープリタ MIRI による検証"
date: 2021-08-20T13:40:47+09:00
draft: false
categories: ["testing"]
tags: ["rust", "miri", "unsafe"]
---

# Rust の中間言語インタープリタ MIRI による検証

Rust の中間言語は [MIR](https://github.com/rust-lang/rfcs/blob/master/text/1211-mir.md) と呼ばれます。

この中間言語コードを実行するインタープリタが [MIRI](https://github.com/rust-lang/miri) です。

インタープリタ実行の為、多少低速ではありますが、コンパイルされたバイナリを実行する時にはできない諸々の検証が行える為、ライブラリの利用方法の誤りや、unsafe コードを含む rust コードに誤りが無いかを確認できます。unsafe コードの動作に問題が無いか確認できるという点で unsafe コードを含むコードを開発している場合には MIRI の利用は強く推奨できます。

「unsafe コード書いて MIRI 回してないってホント大丈夫？」

って言うレベルで unsafe コードを書いているなら必須です。

なので何らか重要なライブラリやフレームワークの導入時にソースを clone してきて回してみるぐらいはした方が良いかなと思います。

## MIRI の導入

MIRI を導入するのは比較的簡単で nightly ツールチェインに対して miri を追加するだけでセットアップできますす。

```shell-session
$ rustup +nightly component add miri
```

## MIRI の実行

既存のテストを MIRI で実行するには `cargo +nightly miri test` で行います。

```shell-session
$ cargo +nightly miri test
```

## MIRI で検出したエラーへの対処

この部分が MIRI の弱さではありますが、エラーログとライブラリのドキュメントその他を熟読して何故エラーとして報告されるのかを確認するしかありません。

unsafe となっているライブラリメソッドには概ね Safety についての解説がありますので、呼び出している unsafe メソッドの Safety についての解説を確認しましょう。

とりあえずエラーを開発者に一報した上でソースとライブラリドキュメントを読み込んで開発者と一緒に問題をクリアしていけると良いですね。

## 自分が見つけた MIRI エラー

何か見つける毎に追記していこうと思います。

- [rg3dengine/rg3d#155: MIRI test failed on scene::mesh::buffer::test::test_add_attribute](https://github.com/rg3dengine/rg3d/issues/155)
