---
title: "Rust アプリケーションの性能管理とパフォーマンスチューニング"
date: 2021-09-17T14:45:42+09:00
categories: ["Performance-Tuning"]
tags: ["rust", "profiling", "criterion", "flamegraph"]
---

Rust は基本的にはネイティブなコードを吐き出すし、C/C++と同等な性能の出るプログラミング言語と言われています。

しかし、性能について全く考慮する必要の無いプログラムというのは無いですし、処理の追加や変更での性能変化特に処理性能が下がる事はパフォーマンスデグレードという事でアルゴリズムの変更時等にパフォーマンスの変化は追い続ける必要があります。

パフォーマンスが期待値まで出ていない場合にはパフォーマンスチューニングをする事になります

## 性能計測

性能計測と大きめの題目を上げていますが、これは本当に難しい問題を内在しています。

性能計測環境のハードウェア、ソフトウェアは適切に管理されていなければなりません。特に性能管理（パフォーマンストラッキング）をする場合にはアプリケーションの変更での性能変化を検出する必要があり、ハードウェア構成やソフトウェア構成の異なる環境での結果と比較しても意味がありません。

性能計測環境の維持管理が困難な問題の場合、別のアプローチもありえます。git 等のソース管理ツールがちゃんと利用されていれば、最新のハードウェア、ソフトウェアで過去のバージョンから現在のバージョンまで一連をビルドしてベンチマークしてみる事はできます。

そして、最近のPC(に使われているCPUやGPU)は排熱処理の問題から、短期的な性能変動がかなり大きいです。ベンチマークを繰り返し実行して安定点をさぐろうとしているうちにPCが熱をもってしまって性能だだ下がりとか、夏場は明らかにベンチ結果が低いとかあります。

性能計測をちゃんとやろうとすると、ハードウェアやソフトウェア、気温その他色々な事に気を配る必要があって大変だという事は理解してもらったと思います。その上でこの先の記述では全部無視します。

適切に管理されたパフォーマンストラッキングもアプリケーションによっては必要ですが、「開発者が開発環境内でパフォーマンス変化を確認したりそれを元に遅くなってるから改善しなきゃと気付ける」のも必要で、どちらかというと後者寄りで利用するツールその他を解説していきます。（前者を求めている人は頑張って）

### ベンチマークの作成と実行

Rust のパッケージマネージャでありビルドツールであり、標準タスクランナーであり(ほんとなんでも出来るな)の cargo にはベンチマークを実行する機能があります。

[cargo-bench : The Cargo Book](https://doc.rust-lang.org/cargo/commands/cargo-bench.html)

しかし、`cargo bench` で実行されるベンチマークの記述には Nightly channel が必要になるなど若干どころではない課題があります。

特にベンチマーク結果を元に性能管理をしようとした場合、 「Nightly コンパイラでのコンパイル結果を Nightly ランタイムで動かしたベンチマーク計測して意味あるの？出荷物はStableでコンパイルするんだよね？」という問題にぶつかります。

前述の `cargo-bench` のページでも紹介されていますが、 [criterion](https://crates.io/crates/criterion) を利用する事で `cargo bench` で実行可能なベンチマークを Stable channel でビルド実行する事が可能です。

また、`criterion` は計測するだけでなくベースラインと性能を比較する機能もある為、性能管理もある程度サポートされます。

[Getting Started: Criterion.rs Documentation](https://bheisler.github.io/criterion.rs/book/getting_started.html)通りで基本的には何の問題もありません。

1. `dev-dependencies` に `criterion` を追加する

   `$ cargo add --dev criterion`

2. `benches` 配下にベンチマークを作成し `cargo.toml` に追加する

3. `cargo bench` で実行する

基本的にはコレでOKです。

性能計測でアレコレ言った通りで計測環境にバイナリを持っていくとかしたい人は以下を参考にどうぞ

```shell-session
kazuk:~/pico-sat$ cargo bench --no-run --bench one_of_expr
    Finished bench [optimized] target(s) in 0.05s
kazuk:~/pico-sat$ ls target/release/deps/one_of_expr*
target/release/deps/one_of_expr-0173b34eca07f896
target/release/deps/one_of_expr-0173b34eca07f896.d
kazuk:~/pico-sat$ ./target/release/deps/one_of_expr-0173b34eca07f896 --bench
WARNING: HTML report generation will become a non-default optional feature in Criterion.rs 0.4.0.
This feature is being moved to cargo-criterion (https://github.com/bheisler/cargo-criterion) and will be optional in a future version of Criterion.rs. To silence this warning, either switch to cargo-criterion or enable the 'html_reports' feature in your Cargo.toml.

Gnuplot not found, using plotters backend
one_of 30               time:   [41.789 ms 42.119 ms 42.482 ms]                      
                        change: [-5.4455% -4.1018% -2.8271%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 8 outliers among 100 measurements (8.00%)
  6 (6.00%) high mild
  2 (2.00%) high severe
```

1. `--no-run` オプション付きで cargo bench を実行するとベンチマークバイナリが target/release/deps 配下に生成される。
2. ベンチマークバイナリを計測環境に持っていく
3. `--bench` オプション付きでベンチマークバイナリを実行するとベンチマークが実行される。

### ベースラインを使用したパフォーマンス比較

[Baselines](https://bheisler.github.io/criterion.rs/book/user_guide/command_line_options.html#baselines)に書いてある通りです。

`--save-baseline` オプションにベースライン名を指定してベンチを実行するとベンチ結果がベースライン名で保存されます。
`--baseline` オプションで比較対象を指定してベンチを実行できます。
`--load-baseline` オプションは `--baseline` オプションで指定された物との比較結果を表示します。（ベンチは実行されません。）

git のブランチを色々切り替えながら `--save-baseline` でベースラインを保存して、 `--load-baseline` `--baseline` で結果を比較して眺めるというのが使い方と言えるでしょう。

通常の開発作業なら main と開発ブランチ、存在していれば直近のリリースタグと性能変化を見ておけば良いと思います。

## パフォーマンスチューニングの為のプロファイリング

ベンチマークを実行した結果解るのは遅い速い、パフォーマンス比較をしていると、特定のバージョンとの相違としての遅い速いです。

具体的にどの処理に時間を使っているかを知るにはプロファイリングが必要になります。

プロファイリングの設定はOSによりますし、管理者権限も必要となりますので、利用OSで扱い安い物を選択しましょう。自分は Linux - Ubuntu上で開発をしていますので、この解説も Linux - Ubuntu での説明になります。

### プロファイラのインストール

Linux ではプロファイラはカーネルに依存する為、カーネルバージョンをまず確認します。

`uname -a` でカーネルバージョンが表示できます。自分の環境では `5.11.0-34-generic` でした。

```shell-session
kazuk:~/kazuk.log$ uname -a
Linux kazuk-ThinkCentre-M75q-1 5.11.0-34-generic #36~20.04.1-Ubuntu SMP Fri Aug 27 08:06:32 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```

対応するプロファイラをインストールします。

```shell-session
sudo apt install -y linux-tools-5.11.0-34-generic
```

プロファイラを有効にする為、 `/etc/sysctl.conf` に `kernel.perf_event_paranoid=1` を設定します。

```shell-session
kazuk:~/kazuk.log$ tail -3 /etc/sysctl.conf 
# Profile enable
kernel.perf_event_paranoid=1
```

`sudo sysctl -p` で設定を反映します。

設定を反映すると `/proc/sys/kernel/perf_event_paranoid` から設定値が見えます。

```shell-session
kazuk:~/kazuk.log$ cat /proc/sys/kernel/perf_event_paranoid 
1
```

プロファイラの出力する結果はとても見やすいとは言えないので、プロファイラの出力結果を見やすく整形するツールを導入します。

`flamegraph` が cargo 連携もあり便利なのでこれをインストールします。

```shell-session
cargo install flamegraph
```

ベンチマークが実装されていれば `cargo flamegraph --bench [bench name]` でフレームグラフが出力されます svg なので好きなビューワーで見て下さい。

フレームグラフは、下が呼び出し元、上が呼び出し先、幅が実行時間を示します。幅が長い関数を見つければそれをチューニングすれば良いという事になりますね。

## パフォーマンスチューニング

いずれ追記します。

メモリ使用量を下げる、より計算量のすくないアルゴリズムに置き換えるのが基本になると思います。
