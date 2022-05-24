---
title: "RustプログラムのCoverageをGRCOVで取得する"
date: 2021-09-13T09:10:15+09:00
categories: ["Coverage"]
tags: ["rust", "grcov", "lcov"]
---

rust は単体テストの実行等は cargo に統合されていて非常に扱い安いのですが、カバレッジの取得や確認は統合されていません。

また、カバレッジの取得には nightly コンパイラが必要になる等ちょっとした要件があります。

プロジェクトにカバレッジの取得を追加する度に毎度google検索していたりするので、これを一旦まとめておきます。

## カバレッジの取得に必要な物

- [grcov](https://github.com/mozilla/grcov)

  プロファイラ出力されたカバレッジ情報の集約や変換等を行います。 `cargo install grcov` でインストールする事ができます。

- `llvm-tools-preview`

  `grcov` が依存しています。 `rustup component and llvm-tools-preview` でインストールします。

- Stable rust (rust 1.60 より stable rust でもカバレッジの採取が可能になりました。) ~~ `Nightly rust` ~~

  ~~プロファイラの為のコンパイルオプションは現状 Nightly rust コンパイラでのみサポートされています。~~

  ~~`rustup toolchain install nightly` でインストールします。~~

  ~~Nightlyコンパイラは名前の通り毎晩リリースされ「稀に壊れます」何かうまく行かない等がある場合には `rustup update` でより新しいバージョンを取得してくる、Nightly の最新でうまく行かない場合等は日付指定での Nightly コンパイラへ入れ替える等をしてくる必要があります。 そういう場合には [Working with nightly Rust](https://rust-lang.github.io/rustup/concepts/channels.html#working-with-nightly-rust) を参考にして下さい。~~

## カバレッジ取得の流れ

カバレッジの取得はカバレッジプロファイリングオプションを付けてターゲットをコンパイルする、テストを実行しカバレッジ情報を取得する。
grcov でソースとカバレッジ情報を統合する流れになります。

### カバレッジプロファイル指定でのビルド

プロファイリングオプションを付けてビルドするには基本的には RUSTFLAGS を指定して cargo build するだけです。nightly コンパイラを使うのを考慮すると以下になります。

```shell
CARGO_INCREMENTAL=0, RUSTFLAGS="-Zinstrument-coverage -Ccodegen-units=1 -Cinline-threshold=0 -Clink-dead-code -Coverflow-checks=off" cargo +nightly build
```

インクリメンタルビルドの抑止はオプション違いのバイナリがビルドディレクトリ内で混ざるのを防ぐ為に必要です。

- `instrument-coverage` オプション

  このコンパイルオプションが本命の、カバレッジ情報の出力を行う為のコンパイラ指示になります。

- `codegen-units=1` オプション

  コード生成の並列度を指定します。並列コード生成は `instrument-coverage` ではおすすめできないので 1 を指定しています。（llvm のバグを踏みづらくなるぐらいの効果です）

- `inline-threshold=0` オプション

  inline 化によりソースとの対応関係が発散するのを防止する為に inline 化を抑止します。

- `link-dead-code` オプション

  到達不能なコードの除去を抑止します。到達不能コードでもカバレッジ率の母数としては意識したいため指定します。

- `overflow-checks=off` オプション

  これを指定するかはプロジェクトによると思います。オーバーフロー時の挙動まで含めてしっかりカバレッジする気があるか次第ですが、off にしない場合には通常オーバーフローのありえない計算でのオーバーフローをカバレッジするテストが要求されます。

注意するべきは、このまま実行するとプロジェクトの `/target/debug` 配下にカバレッジプロファイル出力が埋め込まれたバイナリが生成されます。

カバレッジ取得後にそのままのバイナリを使ってテストや諸々を実行すると膨大なカバレッジプロファイル出力が生成される可能性があります。通常のバイナリに戻すには `cargo clean && cargo build` をする必要がありますが、clean 後のビルドは中々重いのでできれば避けたいです。(clean せずに build すると単なるタイムスタンプの違いによって評価されるので最悪)

これを避けるには `CARGO_BUILD_TARGET_DIR` 環境変数でカバレッジ用バイナリを別ディレクトリに生成させる事ができます。

### テストの実行

プロファイルインストルメントが設定されたバイナリが用意できたら、プロファイル情報の出力先を指定してテストを実行します。

[Running the instrumented program](https://clang.llvm.org/docs/SourceBasedCodeCoverage.html#id4)

`LLVM_PROFILE_FILE` 環境変数によりプロファイル情報の出力先が指定されます。 rust プログラムでは target 配下を指定するべきでしょう。

### カバレッジプロファイル情報の集約

`grcov` によってカバレッジプロファイル情報を集約しソースとのマッピングをします。 `grcov` は lcov 等主要なカバレッジ結果フォーマットでの出力を生成できるので、手元のエディタでのハイライト表示に使えるフォーマットを指定するとエディタ上でカバレッジがハイライトされます。

注意点としては grcov は内部でLLVMのプロファイルツールを利用しており、基本的には rust のツールチェインとしてインストールされた物を利用します。

デフォルトを Stable コンパイラでコンパイルしていてカバレッジだけを Nightly でコンパイルしている場合には grcov に Nightly のツールチェインを利用させる必要があります。

環境変数 `RUSTUP_TOOLCHAIN` によって grcov に Nightly ツールチェインを指定する事ができます。

### scripting

ここまでの説明をスクリプト化したのが以下になります。

```bash
#!/usr/bin/env bash

set -eux
export CARGO_BUILD_TARGET_DIR="./target/cover"
export RUSTFLAGS="-Zinstrument-coverage -Ccodegen-units=1 -Cinline-threshold=0 -Clink-dead-code -Coverflow-checks=off"
export RUSTDOCFLAGS="-Cpanic=abort"

cargo +nightly build --verbose

rm -rf ./target/cover/debug/deps/profraw
mkdir ./target/cover/debug/deps/profraw/

export LLVM_PROFILE_FILE="./target/cover/debug/deps/profraw/coverage-%p-%m.profraw"

cargo +nightly test --verbose

RUSTUP_TOOLCHAIN="nightly" grcov ./target/cover/debug/deps/profraw/ -b ./target/cover/debug/deps -s . -t lcov --llvm --branch --ignore-not-existing --ignore "/*" -o ./target/lcov.info
```

最終的に ./target/lcov.info にカバレッジ情報が出力されます。

## `cargo-make` でのカバレッジビルド

[cargo-make](https://github.com/sagiegurari/cargo-make) は rust 向けのメイクツールで、ビルド及びテストの一連の実行が可能です。

この `cargo-make` 向けの grcov でのカバレッジ採取用の Makefile を [cargo-make-grcov-coverage](https://github.com/kazuk/cargo-make-coverage-grcov)で公開しています。

## カバレッジ情報をエディタ表示

vscode の場合には [Code Coverage Highlighter](https://marketplace.visualstudio.com/items?itemName=brainfit.vscode-coverage-highlighter) が良いかなと思いました。 [Code Coverage](https://marketplace.visualstudio.com/items?itemName=markis.code-coverage) としか比較していませんが、uncover を suggestion に上げて来られるのはちょっとうっとおしすぎるので色分けが見れるぐらいがちょうど良いと思います。

拡張機能の設定で生成されたカバレッジ情報を読み込むように指定するだけで、テストでカバーされている行とそうでない行が色分け表示されます。
