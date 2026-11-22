{% raw %}
# Miri {#miri}

<img src="miri-sticker.png" align="right" width="200px"
 alt="「Miri - 100% safe」と書かれたジュースの瓶から飲んでいる、Unsafe Rusturchin の Corro"
 title="絵: Paige Losare-Lusby、Creative Commons Attribute-ShareAlike">

Miri は、Rust のための[未定義動作（Undefined Behavior）][reference-ub]の検出ツールです。cargo プロジェクトのバイナリやテストスイートを実行し、安全性の要件を守れていない unsafe コードを検出できます。たとえば、次のようなものです。

* 境界外のメモリアクセスと、解放後の使用（use-after-free）
* 未初期化データの不正な使用
* 組み込み関数（intrinsic）の事前条件の違反（[`unreachable_unchecked`] に到達する、重なり合う範囲で [`copy_nonoverlapping`] を呼ぶ、など）
* アラインメントが足りないメモリアクセスと参照
* 型の基本的な不変条件の違反（たとえば、0 でも 1 でもない `bool` や、不正な列挙型の判別値）
* データ競合と、*一部の*弱いメモリ（weak memory）の効果のエミュレーション。つまり、アトミックな読み出しが古い値を返すことがあります
* **実験的**: 参照型のエイリアシング（aliasing）を定める [Stacked Borrows] の規則の違反
* **実験的**: [Stacked Borrows] の代わりに選べる、[Tree Borrows] のエイリアシング規則の違反

これに加えて、Miri はメモリリークも知らせます。実行の終了時にまだ確保されたままのメモリがあり、そのメモリにグローバルな `static` から到達できない場合、Miri はエラーを出します。

Miri を使うと、ほかのターゲット向けのプログラムをエミュレートできます。たとえば、バイト単位のデータ操作が、リトルエンディアンのシステムでもビッグエンディアンのシステムでも正しく動くことを確かめられます。後述の[クロスインタープリテーション](#cross-interpretation-running-for-different-targets)を参照してください。

Miri はすでに、多くの[現実のバグ](#bugs-found-by-miri)を見つけています。Miri でバグを見つけたら、ぜひ知らせてください。一覧に追加します！

既定では、Miri はプログラムをホストシステムから隔離することで、完全に決定的な実行を保証します。通常ならホストにアクセスする一部の API、たとえば乱数生成器のためのエントロピーの収集、環境変数、時計などは、決定的な「偽の」実装に置き換えられます。代わりに本物のシステム API にアクセスするには、`MIRIFLAGS="-Zmiri-disable-isolation"` を設定します。（この隔離は本格的なサンドボックスでは*ない*ことに注意してください。隔離の抜け穴は、セキュリティ上のバグではなく通常のバグとして扱われます。また、「偽の」システム乱数生成器の API があるため、Miri は**暗号用途には向きません**！ Miri を使って鍵を生成しないでください。）

とはいえ、Miri は、プログラム中の **Rust の仕様への違反をすべて捕まえるわけではない**ことに注意してください。そもそも、そのような仕様が存在しないことが大きな理由です。Miri は、Rust で何が未定義動作で何がそうでないかについて、独自の近似を使っています。私たちの知るかぎり、プログラムの正しさに影響しうる未定義動作は、（[バグ][I-misses-ub]を除けば）すべて Miri が検出*します*。ただし、未定義動作の公式な定義については[リファレンス][reference-ub]を確かめてください。Miri は Rust コンパイラとともに更新され、その時点のコンパイラが理解している未定義動作（UB）から守りますが、将来のバージョンの rustc については何も約束しません。

Miri の利用者が知っておくべき注意点は、ほかにもあります。

* データがどう配置されるかという、規定されていない詳細にプログラムが依存していても、Miri ではそのまま問題なく動きます。しかし、別のバージョンのコンパイラや別のプラットフォームでは壊れるかもしれません（UB を引き起こすことも含みます）。（こうした場合の一部は、`-Zrandomize-layout` を使うと検出できます。）
* プログラムの実行は、たとえばアロケーションがメモリ上の正確にどこに置かれるかや、並行するスレッドの正確なインターリーブ（実行の入り混じり方）に依存する場合、非決定的になります。Miri は、プログラムのありうる多くの実行のうちの 1 つをテストしますが、別の実行でしか起きないバグは見逃します。`-Zmiri-seed` に異なる値を与えて Miri を実行すれば、ある程度は緩和できます。それでも、ありうる実行をすべて調べるにはほど遠いままです。
* Miri は、プラットフォームに依存しないインタプリタとしてプログラムを実行します。そのため、プログラムは、プラットフォーム固有の API や FFI のほとんどにアクセスできません。いくつかの API（標準出力への表示、環境変数へのアクセス、基本的なファイルシステムへのアクセスなど）は実装されていますが、ほとんどは実装されていません。たとえば、Miri は現在、ネットワークに対応していません。システム API への対応はターゲットによって異なります。Windows で実行する場合は、より良い対応を得るために `--target x86_64-unknown-linux-gnu` を使うのがよいでしょう。
* 弱いメモリのエミュレーションは完全ではありません。正当な挙動のなかにも、Miri が決して生み出さないものがあります。それでも、Miri は実際のハードウェアでは観察しにくい挙動を数多く生み出すので、弱いメモリに関する並行性のバグを見つけるのにかなり役立ちます。複雑なアトミックのコードについて本当に確信を持ちたい場合は、[loom](https://github.com/tokio-rs/loom) のような専用のツールを使ってください。

さらに、Miri は原理的に、あなたのコードが*健全*であることを保証できません。[健全性（Soundness）][Soundness]とは、任意の安全なコードから呼び出されたときに、ほかの健全なコードと組み合わせた場合でも、決して未定義動作を引き起こさないという性質です。これに対して Miri が教えられるのは、*あなたのコードとの特定のやり取りの仕方*（たとえばテストスイート）が、*特定の実行*で未定義動作を引き起こすかどうかだけです（実行は、たとえば並行性やそのほかの非決定性が関わる場合、数多くありえます）。Miri が UB を見つけたなら、あなたのコードは確実に健全ではありません。しかし、Miri が UB を見つけなかったときは、もっと多くの入力や、もっと多くのありうる非決定的な選択をテストする必要があるだけかもしれません。

[rust]: https://www.rust-lang.org/
[mir]: https://github.com/rust-lang/rfcs/blob/master/text/1211-mir.md
[`unreachable_unchecked`]: https://doc.rust-lang.org/stable/std/hint/fn.unreachable_unchecked.html
[`copy_nonoverlapping`]: https://doc.rust-lang.org/stable/std/ptr/fn.copy_nonoverlapping.html
[Stacked Borrows]: https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md
[Tree Borrows]: https://perso.crans.org/vanille/treebor/
[Soundness]: https://rust-lang.github.io/unsafe-code-guidelines/glossary.html#soundness-of-code--of-a-library
[reference-ub]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html
[I-misses-ub]: https://github.com/rust-lang/miri/labels/I-misses-UB


## Miri を使う {#using-miri}

`rustup` を使って、Rust の nightly に Miri をインストールします。

```sh
rustup +nightly component add miri
```

以下のコマンドはすべて、`rustup override set nightly` で nightly ツールチェーンに固定してあることを前提にしています。あるいは、以下の各コマンドで `cargo +nightly` を使ってください。

これで、プロジェクトを Miri で実行できます。

- プロジェクトのすべてのテストを Miri で実行するには、`cargo miri test` を使います。
- バイナリのプロジェクトであれば、`cargo miri run` を使って Miri で実行できます。

Miri を初めて実行するときは、追加のセットアップが行われ、いくつかの依存物がインストールされます。何かをインストールする前には、確認を求められます。

`cargo miri run/test` は、`cargo run/test` とまったく同じフラグに対応しています。たとえば `cargo miri test filter` は、名前に `filter` を含むテストだけを実行します。

`MIRIFLAGS` を通じて、Miri に[フラグ][miri-flags]を渡せます。たとえば `MIRIFLAGS="-Zmiri-disable-stacked-borrows" cargo miri run` は、参照のエイリアシングを検査せずにプログラムを実行します。

`cargo miri` でコードをコンパイルするとき、Miri の下で解釈実行されるコードには、設定フラグ `cfg(miri)` が設定されます。これを使うと、Miri が対応していないことを行うために Miri の下では失敗するテストケースを、無視できます。

```rust
#[test]
#[cfg_attr(miri, ignore)]
fn does_not_work_on_miri() {
    tokio::run(futures::future::ok::<_, ()>(()));
}
```

Miri にできないことは無限にあるので、すべてを列挙する方法はありません。しかし、インタプリタは、対応していないものを見つけたときに、はっきりと知らせます。

```
error: unsupported operation: can't call foreign function: bind
    ...
    = help: this is likely not a bug in the program; it indicates that the program \
            performed an operation that Miri does not support
```

### クロスインタープリテーション: 別のターゲット向けに実行する {#cross-interpretation-running-for-different-targets}

Miri は、ホストのターゲット向けにバイナリやテストスイートを実行できるだけではありません。任意の別のターゲット向けに、クロスインタープリテーション（cross-interpretation）を行うこともできます。`cargo miri run --target x86_64-unknown-linux-gnu` は、ホストの OS が何であっても、プログラムを Linux のプログラムであるかのように実行します。Linux のターゲットは Windows のターゲットよりずっとよく対応されているので、Windows を使っている場合にはとくに便利です。

これを使って、ホストのプラットフォームとは性質の異なるプラットフォームをテストすることもできます。たとえば `cargo miri test --target s390x-unknown-linux-gnu` は、テストスイートをビッグエンディアンのターゲットで実行します。エンディアンに左右されるコードをテストするのに便利です。

### ターゲット機能を制御する {#controlling-target-features}

ターゲット機能（target feature）の制御は、通常の rustc の呼び出しと同じように働きます。`RUSTFLAGS="-Ctarget-feature=+avx512f" cargo miri test` は、AVX512 を有効にしてテストを実行します。（Miri が対応している AVX512 の組み込み関数は、今のところごくわずかです。）`-Ctarget-cpu` も使えます。ターゲット機能が doctest にも関係する場合は、`RUSTDOCFLAGS` も設定しなければなりません。

通常の rustc と違って、このフラグには *2 つ*の効果があります。1 つは、そのターゲット機能が使える状態でコードをビルドすることです（これは `cfg(target_feature)` に影響します）。もう 1 つは、解釈実行されるプログラムが動く「仮想 CPU」がその機能を備えているものとして扱うよう、Miri に伝えることです（つまり、コードは対応する組み込み関数を呼び出してよいことになります）。

### Nextest との連携 {#nextest-integration}

Miri は [`cargo-nextest`](https://nexte.st) と組み合わせられます。

```
cargo install --locked cargo-nextest
cargo miri nextest run
```

Nextest は、テストごとに別々の Miri のインスタンスを起動します。これにはいくつかの利点があります。

- テストを並列に実行できます。Miri 自体は 1 つのインタプリタにつき 1 つのスレッドしか使わないので、大幅な高速化になりえます（ただし、後述の注意点を見てください）。
- 問題が 1 つ見つかっても、テストが止まりません。Miri は、未定義動作や対応していない操作に出会うと実行を中止します（続行する方法が実際には無いことが多いためです）。そのため、テストが 1 つ失敗すると、残りのテストは実行できません。Nextest はテストごとにプロセスを分ける方式なので、どのテストが Miri で動き、どのテストに問題があったかの完全な一覧が得られます。

しかし、大きな注意点もあります。Miri は、[呼び出されるたびにテストクレートを再コンパイルします](https://github.com/rust-lang/miri/issues/5013)。つまり、N 個のテストを持つクレートは N+1 回コンパイルされます。テストクレートのビルドに長い時間がかかる場合、これは並列化の利点を上回ることがあります。

nextest についての詳しい情報は、[`cargo-nextest` の Miri のドキュメント](https://nexte.st/book/miri.html)を参照してください。

注: Nextest は 1 つのプロセスで 1 つのテストを実行する方式です。そのため、2 つのテストが共有リソースをめぐって競合するデータ競合を、`cargo miri test` は検出できますが、`cargo miri nextest run` は検出しません。

注: `cargo-nextest` は [doctest に対応していません](https://github.com/nextest-rs/nextest/issues/16)。

### 複数の異なる実行をテストする {#testing-multiple-different-executions}

実行の一部は、Miri がランダムに選びます。たとえば、アロケーションが格納される正確なベースアドレスや、並行して実行されるスレッドのインターリーブです。ときには、複数の異なる実行を調べると役に立ちます。たとえば、新しいアロケーションがたまたま「余分に強くアラインされている」ことにコードが依存していないと確かめるためや、異なるスレッドのインターリーブをテストするためです。これは `-Zmiri-many-seeds` フラグで行えます。

```
MIRIFLAGS="-Zmiri-many-seeds" cargo miri test # tries the seeds in 0..64
MIRIFLAGS="-Zmiri-many-seeds=0..16" cargo miri test
```

Miri は、与えられた範囲のシードを、並列のインタプリタのインスタンスでテストします。既定の 64 個の異なるシードはかなり遅くなることがあるので、多くの場合はもっと狭い範囲を指定するとよいでしょう。

### CI で Miri を実行する {#running-miri-on-ci}

CI で Miri を実行するときは、次の断片を使って、Miri コンポーネントつきの nightly ツールチェーンをインストールします。

```sh
rustup toolchain install nightly --component miri
rustup override set nightly

cargo miri test
```

GitHub Actions のジョブの例を示します。

```yaml
  miri:
    name: "Miri"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Miri
        run: |
          rustup toolchain install nightly --component miri
          rustup override set nightly
          cargo miri setup
      - name: Test with Miri
        run: cargo miri test
```

`cargo miri setup` を明示的に実行しておくと、実際のテストのステップの出力をきれいに保てます。

### 対応しているターゲット {#supported-targets}

Miri は、Rust が対応しているすべてのターゲットに対応しているわけではありません。しかし良い知らせとして、ホストの OS やプラットフォームが何であっても、`--target` を使えば、*どの*ターゲット向けのコードも簡単に実行できます！

次のターゲットは CI でテストされているので、（以下に記した程度まで）常に動くはずです。

- Rust のすべての [Tier 1 ターゲット](https://doc.rust-lang.org/rustc/platform-support.html)に、Miri は対応しています。これらはすべて Miri の CI で検査されています。一部（OS ごとに少なくとも 1 つ）は、Rust のすべての PR でも検査されています。そのため、配布される Miri は、これらのターゲットで常に動くはずです。
- `s390x-unknown-linux-gnu` は、私たちが「ビッグエンディアンのターゲットとして選んだもの」として対応しています。
- OS が `linux`、`macos`、`windows` であるそのほかのすべてのターゲットでも、Miri はおおむね動くはずです。ただし、私たちは何も約束しませんし、そうしたターゲットのテストも実行していません。
- さらにいくつかのオペレーティングシステムには、非公式の対応（Miri チーム自身が保守しているものではありません）があります。
  - `solaris` / `illumos`: @devnexen が保守しています。テストスイート全体に対応しています。
  - `freebsd`: @YohDeadfall と @LorrensP-2158466 が保守しています。テストスイート全体に対応しています。
  - `android`: **メンテナー募集中**。テストスイート全体に対応しています。
- そのほかのオペレーティングシステムのターゲットでは、Miri は `main` 関数に到達する前に失敗するかもしれません。

ただし、私たちが対応しているターゲットであっても、プラットフォームの API（ファイルシステムなど）へのアクセスに対応している度合いは、ターゲットによって異なります。一般に、Linux のターゲットがもっともよく対応されていて、macOS のターゲットもたいてい同じ水準です。Windows への対応は、それより劣ります。

### `miri` ドライバを直接呼び出す {#directly-invoking-the-miri-driver}

Miri を呼び出す推奨の方法は、`cargo miri` を使うことです。その下にある `miri` ドライバを直接呼び出すことには対応していません。このバイナリが PATH にインストールすらされないのは、そのためです。しかし、多数の小さなテストで Miri を実行する必要があり、`rustc` を呼び出すのと同じように直接呼び出したい場合は、少し手間をかければ可能です。

```sh
# one-time setup
cargo +nightly miri setup
SYSROOT=$(cargo +nightly miri setup --print-sysroot)
# per file
~/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/bin/miri --sysroot "$SYSROOT" file.rs
```

### よくある問題 {#common-problems}

上の手順を使っていると、分かりにくいコンパイラエラーにいくつか出会うことがあります。

#### "note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace" {#note-run-with-rust_backtrace1-environment-variable-to-display-a-backtrace}

Miri にバックトレースを表示させようとすると、これが表示されることがあります。既定では、Miri はプログラムに環境変数を一切見せないので、`RUST_BACKTRACE=1 cargo miri test` を実行しても、期待どおりにはなりません。

バックトレースを得るには、[`-Zmiri-disable-isolation` を使って][miri-flags]隔離を無効にする必要があります。

```sh
RUST_BACKTRACE=1 MIRIFLAGS="-Zmiri-disable-isolation" cargo miri test
```

#### "found crate `std` compiled by an incompatible version of rustc" {#found-crate-std-compiled-by-an-incompatible-version-of-rustc}

Miri が使う専用の libstd をビルドしたときとは異なるバージョンのコンパイラで `cargo miri` を実行していて、Miri がそれを検出できなかったのかもしれません。`cargo miri clean` を実行してみてください。


## Miri の `-Z` フラグと環境変数 {#miri--z-flags-and-environment-variables}

[miri-flags]: #miri--z-flags-and-environment-variables

Miri は独自の `-Z` フラグ一式を追加します。これらは通常、環境変数 `MIRIFLAGS` を通じて設定します。まず、もっとも関係が深く、もっともよく使われるフラグを説明します。

* `-Zmiri-backtrace=<0|1|full>` は、Miri がバックトレースをどう表示するかを設定します。`1` が既定で、バックトレースは刈り込まれた形で表示されます。`full` は刈り込まずにバックトレースを表示し、`0` はバックトレースを完全に無効にします。
* `-Zmiri-deterministic-concurrency` は、Miri の並行性に関する挙動を完全に決定的にします。厳密に言えば、隔離が有効なとき（既定のモード）、Miri は常に完全に決定的です。しかし、この決定性は、固定のシードを持つ乱数生成器（RNG）を使うことで実現されています。そのため、プログラムへの一見無害な変更や、別のターゲットアーキテクチャ向けに実行するだけのことで、その先のプログラムの挙動がまったく違うものになりえます。このフラグは、並行性に関する判断に RNG を使うことをやめさせます。そのため、Miri は特定の状況でしか起きないバグを見つけられなくなりますが、Miri の挙動はバージョンやターゲットをまたいでより安定します。これは `-Zmiri-fixed-schedule -Zmiri-compare-exchange-weak-failure-rate=0.0 -Zmiri-address-reuse-cross-thread-rate=0.0 -Zmiri-disable-weak-memory-emulation` と同等です。
* `-Zmiri-disable-isolation` は、ホストからの隔離を無効にします。その結果、プログラムは、環境変数、ファイルシステム、乱数性といったホストのリソースにアクセスできます。これは、それより前に指定した `-Zmiri-isolation-error` を上書きします。
* `-Zmiri-disable-leak-backtraces` は、メモリリークについてのバックトレースの報告を無効にします。既定では、リークした場合に備えて、すべてのアロケーションについて、作成時にバックトレースが記録されます。このため、ほとんど使われることのないデータを保存するための、いくらかのメモリのオーバーヘッドが生じます。このフラグは、`-Zmiri-ignore-leaks` を指定すると暗黙に有効になります。
* `-Zmiri-env-forward=<var>` は、環境変数 `var` を、解釈実行されるプログラムに引き渡します。複数回使って、いくつかの変数を引き渡せます。引き渡した変数の値が変わらなければ、実行は決定的なままです。`-Zmiri-disable-isolation` が設定されている場合は、効果がありません。
* `-Zmiri-env-set=<var>=<value>` は、解釈実行されるプログラムの中で、環境変数 `var` を `value` に設定します。ホストの環境を変えずに環境変数を渡すのに使えます。複数回使って、いくつかの変数を設定できます。`-Zmiri-disable-isolation` または `-Zmiri-env-forward` が設定されている場合、このオプションで設定した値が、ホストの環境から来る値より優先されます。
* `-Zmiri-ignore-leaks` は、メモリリークの検査を無効にします。また、メインスレッドが終了するときに、残りのスレッドがいくつか存在していることも許します。
* `-Zmiri-isolation-error=<action>` は、隔離が有効なあいだに、ホストへのアクセスを必要とする操作に対して Miri がどう応じるかを設定します。対応しているアクションは、`abort`、`hide`、`warn`、`warn-nobacktrace` です。既定は `abort` で、マシンを停止します。一部の（すべてではありません）操作は、「permission denied」エラーをプログラムに返して実行を続けることにも対応しています。`warn` は、それが起きるたびに完全なバックトレースを表示します。`warn-nobacktrace` はもっと簡潔で、1 つの操作につき多くても 1 回だけ表示されます。`hide` は警告を完全に隠します。これは、それより前に指定した `-Zmiri-disable-isolation` を上書きします。
* `-Zmiri-many-seeds=[<from>]..<to>` は、Miri の RNG に異なるシードを与えて、プログラムを複数回実行します。シードが異なると、Miri は、並行するスレッドがスケジュールされる順序や、アロケーションに割り当てられる正確なアドレスといった非決定性を解消するために、異なる選択をします。これは、並行するスレッドの特定のインターリーブの下でしか起きないバグや、そのほか非決定性に依存するバグを見つけるのに便利です。`<from>` の部分を省略すると、既定の `0` になります。値なしでも使えます。その場合、範囲は既定の `0..64` になります。
* `-Zmiri-many-seeds-keep-going` は、失敗するシードがすでに見つかっていても、与えられた範囲のすべてのシードを本当に試すよう Miri に指示します。これは、シードのうちどのくらいの割合が失敗するかを調べるのに便利です。
* `-Zmiri-max-extra-rounding-error` は、精度が保証されていない浮動小数点演算に、常に最大の誤差を適用するよう Miri に指示します。誤差の符号は、非決定的なままです。
* `-Zmiri-no-extra-rounding-error` は、精度が保証されていない浮動小数点演算に、Miri が追加の丸め誤差を加えるのをやめさせます。
* `-Zmiri-no-short-fd-operations` は、`read` / `write` 操作がバッファの一部だけを処理するように Miri が人為的に強制するのを、やめさせます。なお、Miri が `read` / `write` の実装にホストの操作を使う場合（たとえば、ファイルに結びついたファイルディスクリプタの場合）は、ホストシステムの側で、やはり短い読み書きが起きえます。
* `-Zmiri-num-cpus` は、miri が報告する、利用可能な CPU の数を指定します。既定では、利用可能な CPU の数は `1` です。なお、このフラグは、miri がスレッドをどう扱うかにはまったく影響しません。
* `-Zmiri-permissive-provenance` は、整数からポインタへのキャストと [`ptr::with_exposed_provenance`](https://doc.rust-lang.org/nightly/std/ptr/fn.with_exposed_provenance.html) に対する警告を無効にします。こうした操作は、サニタイザでは効率よく正確に実装できないので、必然的に一部のバグを見逃すことになります。ただし、見逃すのは、こうした操作の対象になるメモリやポインタに関わるバグだけです。
* `-Zmiri-report-progress` は、現在のスタックトレースを Miri にときどき表示させます。プログラムがただ動き続けているときに、何をしているのかが分かります。報告を表示する頻度は、`-Zmiri-report-progress=<blocks>` で調整できます。これは、基本ブロック N 個ごとに報告を表示します。
* `-Zmiri-seed=<num>` は、Miri が非決定性を解消するのに使う RNG のシードを設定します。この RNG は、アロケーションのベースアドレスを選ぶこと、プリエンプションと `compare_exchange_weak` の失敗を決めること、弱いメモリのエミュレーションのためのストアバッファリングを制御することに使われます。隔離が有効なとき（既定）は、システムのエントロピーをエミュレートするのにも使われます。既定のシードは 0 です。異なるシードで Miri を複数回実行すると、テストの網羅範囲を広げられます。
* `-Zmiri-strict-provenance` は、Miri で[厳密な来歴（strict provenance）](https://doc.rust-lang.org/nightly/std/ptr/index.html#strict-provenance)の検査を有効にします。つまり、整数をポインタにキャストすると、そのポインタの来歴（プロベナンス）を決められないため、実行が止まります。
* `-Zmiri-symbolic-alignment-check` は、アラインメントの検査をより厳しくします。既定では、アラインメントは、ポインタを整数にキャストし、それがアラインメントの倍数であることを確かめることで検査されます。このやり方では、「たまたま」十分にアラインされていたために、プログラムがまったくの偶然でアラインメントの検査を通ってしまう場合がありえます。その実行には UB がありませんが、ほかの実行には UB があるはずです。このような場合を避けるため、シンボリックなアラインメント検査は、該当するアロケーションに要求されたアラインメントと、そのアロケーション内でのオフセットだけを考慮します。これにより、そうしたバグの見逃しを避けられます。しかし、コードがアラインメントを確保するために整数演算を手作業で行っている場合には、いくらかの偽陽性も生じます。（標準ライブラリの `align_to` メソッドは、どちらのモードでも問題なく動きます。シンボリックなアラインメントの下では、アロケーションが十分なアラインメントを保証する場合にだけ、中央のスライスを埋めます。）
* `-Zmiri-user-relevant-crates=<crate>,<crate>,...` は、Miri が「利用者に関係がある」とみなすクレートの一覧を広げます。これは、バックトレースの表示に影響します（利用者に関係があるクレートについては、Miri は関数名だけでなく実際のコードも表示します）。また、データ競合とエイリアシング違反について集められるスパンにも影響します（Miri は、利用者に関係があるクレートの中で、`#[track_caller]` ではない最上位のフレームのスパンを表示します）。`cargo miri` を使う場合、ローカルのワークスペースのクレートは、常に利用者に関係があるものとみなされます。

残りのフラグは上級者向けに限られ、変更されたり削除されたりする可能性がより高いものです。このうちいくつかは**健全ではありません**。つまり、プログラム中の未定義動作を Miri が検出できなくなることがあります。

* `-Zmiri-address-reuse-rate=<rate>` は、解放された*スタック以外の*アロケーションがアドレス再利用のためのプールに追加される確率と、新しい*スタック以外の*アロケーションがプールから取り出される確率を変えます。スタックのアロケーションは、プールに追加されることも、プールから取り出されることもありません。既定は `0.5` です。
* `-Zmiri-address-reuse-cross-thread-rate=<rate>` は、以前に解放されたメモリブロックを再利用しようとするアロケーションが、*ほかのスレッド*の解放したブロックも候補にする確率を変えます。既定は `0.1` です。つまり既定では、アドレスの再利用を試みる場合の 90% で、同じスレッドのアドレスだけが候補になります。別のスレッドのアドレスを再利用すると、それらのスレッドのあいだに同期が生じます。これは、データ競合や弱いメモリのバグを覆い隠すことがあります。
* `-Zmiri-compare-exchange-weak-failure-rate=<rate>` は、`compare_exchange_weak` 操作の失敗率を変えます。既定は `0.8` です（つまり、weak の操作の 5 回に 4 回が失敗します）。`0.0` から `1.0` までの任意の値に変えられます。`1.0` は常に失敗することを、`0.0` は決して失敗しないことを意味します。なお、`1.0` に設定すると、おそらくハングします。`compare_exchange_weak` を使うプログラムが先に進めなくなるためです。
* `-Zmiri-deterministic-floats` は、Miri の浮動小数点の挙動を完全に決定的にします。つまり、演算は常に優先される NaN を返し、精度の低い演算にランダムな誤差が加えられることはなく、`min` / `max` と「融合されるかもしれない」積和演算はすべて決定的に振る舞います。なお、Miri は一部の演算に依然としてホストの浮動小数点を使うので、ホストのターゲットや構成によって挙動が異なることはありえます。ランダムな誤差だけを無効にするフラグについては、`-Zmiri-no-extra-rounding-error` を参照してください。
* `-Zmiri-disable-alignment-check` は、ポインタのアラインメントの検査を無効にします。ほかの失敗に集中できますが、Miri がプログラム中のバグを見逃しうることになります。このフラグを使うことは**健全ではありません**。
* `-Zmiri-disable-data-race-detector` は、データ競合の検査を無効にします。このフラグを使うことは**健全ではありません**。これを指定すると、`-Zmiri-disable-weak-memory-emulation` も暗黙に有効になります。
* `-Zmiri-disable-stacked-borrows` は、借用を追跡するための実験的なエイリアシング規則（[Stacked Borrows] と [Tree Borrows]）の検査を無効にします。これで Miri の実行が速くなることがありますが、エイリアシング違反が一切検出されなくなることでもあります。このフラグを使うことは**健全ではありません**（ただし、影響を受ける健全性の規則は実験的なものです）。後から指定したフラグが優先されます。借用の追跡は、`-Zmiri-tree-borrows` でふたたび有効にできます。
* `-Zmiri-disable-validation` は、既定で強制されている、妥当性の不変条件（validity invariant）の強制を無効にします。これが無効にするのは、型つきのコピーについての検査だけです。そのほかの操作で不正な値を使えば、やはりエラーになります。これは、エイリアシングモデル（Stacked Borrows / Tree Borrows）も無効にします。おもに、ほかの失敗（境界外アクセスなど）に先に集中するのに便利です。このフラグを設定すると、Miri はプログラム中のバグを見逃しうることになります。その一方で、Miri の実行を速くする助けにもなります。このフラグを使うことは**健全ではありません**。
* `-Zmiri-disable-weak-memory-emulation` は、C++11 の一部の弱いメモリの効果のエミュレーションを無効にします。
* `-Zmiri-fixed-schedule` は、プリエンプションを無効にします（`-Zmiri-preemption-rate=0.0` と同様です）。さらに、次に選ばれるスレッドのランダム化も無効にし、代わりにラウンドロビンのスケジュールに固定します。ただし、Miri の並行性に関する挙動のほかの面は、依然としてランダム化されていることに注意してください。それらをすべて無効にするには、`-Zmiri-deterministic-concurrency` を使います。
* `-Zmiri-native-lib=<path to a shared object file or folder>` は、インタプリタの内側から FFI を通じてネイティブの関数を呼び出すことに対応するための、実験的なフラグです。このフラグは、Unix システムでのみ対応しています。そのファイルが提供しない関数は、これまでどおり通常の Miri のシム（shim）を通じて実行されます。ディレクトリへのパスを指定すると、そのディレクトリのすべてのファイルが、再帰せずに取り込まれます。このフラグは複数回渡せて、複数のファイルやディレクトリを指定できます。
  **警告**: 不正な、または誤った `.so` ファイルを指定すると、Miri 自体に未定義動作を引き起こすことがあります！ そして当然ながら、Miri は多くの場合、ネイティブコードが行う動作を何も検査できません。
  なお、Miri はファイルディスクリプタを独自に扱っています。そのため、ファイルディスクリプタを扱う関数の*一部*を置き換えたい場合は、その*すべて*を置き換えなければなりません。そうしないと、2 種類のファイルディスクリプタが混ざってしまいます。
  これは**開発途中**です。現在のところ、引数と戻り値として対応しているのは整数とポインタだけで、ネイティブコードが確保したメモリには Rust からアクセスできません（その逆だけが可能です）。ネイティブコードは、呼び出しが Rust に戻った後もバックグラウンドで動き続け、Rust が確保したメモリにアクセスするようなスレッドを、起動してはなりません。
  最後に、このフラグは次の意味で**健全ではありません**。Miri は、ネイティブコードと共有するメモリについて、初期化や来歴といった詳細の追跡をやめます。そのため、UB を含んでいながら Miri に見逃されるコードが、簡単に書けてしまいます。
* `-Zmiri-native-lib-enable-tracing` は、ネイティブコードを呼び出すための、開発途中の詳細なトレースモードを有効にします。なお、このフラグが意味を持つのは Linux システムだけです。ほかの Unix は（現在のところ）トレースモードに対応していません。
* `-Zmiri-measureme=<name>` は、解釈実行されるプログラムについて、`measureme` によるプロファイリングを有効にします。
   これは、プログラムのどの部分が Miri の下で遅く実行されているかを見つけるのに使えます。
   プロファイルは `<name>` という名前のディレクトリの中のファイルに書き出され、リポジトリ https://github.com/rust-lang/measureme にあるツールで処理できます。
* `-Zmiri-mute-stdout-stderr` は、標準出力と標準エラー出力へのすべての書き込みを黙って無視しますが、プログラムには実際に書き込んだと報告します。これは、プログラムの実際の出力には関心がなく、Miri のエラーと警告だけを見たいときに便利です。
* `-Zmiri-recursive-validation` は、妥当性の検査を参照の *1 段*下まで再帰させる、*きわめて実験的な*フラグです。メモリ上の値は、`MaybeDangling` の内側にあるかのように扱われます。つまり、入れ子になった参照は、参照外しが可能である必要すらありません。
* `-Zmiri-preemption-rate` は、基本ブロックの終わりで、実行中のスレッドがプリエンプトされる確率を設定します。既定は `0.01`（つまり 1%）です。これを `0` に設定すると、プリエンプションが無効になります。なお、プリエンプションが無くても、スケジュールは依然として非決定的です。スレッドがブロックしたり実行を譲ったりすると、次のスレッドはランダムに選ばれます。
* `-Zmiri-provenance-gc=<blocks>` は、ポインタの来歴のガベージコレクタが動く頻度を設定します。既定では、基本ブロック `10000` 個ごとに 1 回、到達できない来歴を探して取り除きます。これを `0` に設定すると、ガベージコレクタが無効になります。その場合、プログラムによっては、メモリ使用量が爆発的に増えたり、実行時間が線形を超えて増えたりします。
* `-Zmiri-track-alloc-accesses` は、追跡しているアロケーションについて、確保と解放のイベントだけでなく、読み出しと書き込みも表示します。
* `-Zmiri-track-alloc-id=<id1>,<id2>,...` は、指定したアロケーションが確保または解放されるときに、バックトレースを表示します。これは、メモリリークや解放後の使用のバグをデバッグするのに役立ちます。この引数を複数回指定しても、以前の値は上書きされません。代わりに、その値が一覧に追加されます。同じ ID を複数回挙げても、効果はありません。`miri_track_alloc` を使って、実行時に ID を追加することもできます。
* `-Zmiri-track-pointer-tag=<tag1>,<tag2>,...` は、指定したポインタのタグが作られるときと、（もしあれば）借用スタックから取り除かれるときに、バックトレースを表示します（取り除かれた時点でタグは無効になり、それ以降にそのタグを使うとエラーになります）。これは、なぜ UB が起きているのか、コードのどこを調べるとよさそうかを突き止めるのに役立ちます。この引数を複数回指定しても、以前の値は上書きされません。代わりに、その値が一覧に追加されます。同じタグを複数回挙げても、効果はありません。
* `-Zmiri-track-weak-memory-loads` は、弱いメモリのエミュレーションが読み出しに対して古い値を返すときに、バックトレースを表示します。これは、`-Zmiri-disable-weak-memory-emulation` の下では消える問題を診断するのに役立ちます。
* <a name="-Zmiri-tree-borrows"><!-- The playground links here --></a>
  `-Zmiri-tree-borrows` は、[Stacked Borrows] を [Tree Borrows] の規則に置き換えます。Tree Borrows は、Stacked Borrows よりもさらに実験的です。Tree Borrows は、現在のバージョンのコンパイラが利用しうるエイリアシング違反をすべて捕まえるという意味では、依然として健全です。しかし、Rust の最終的なエイリアシングモデルは、Tree Borrows より厳しいものになる可能性が高いです。言い換えると、Tree Borrows を使う場合、あなたのコードが今日は受け入れられても、将来 UB と宣言されるかもしれません。Stacked Borrows では、その可能性はずっと低くなります。
* `-Zmiri-tree-borrows-implicit-writes` は、`&mut` であるすべての関数引数について、暗黙の書き込みを有効にします。これにより、Tree Borrows はより厳しくなります。
* `-Zmiri-tree-borrows-no-precise-interior-mut` は、Tree Borrows に、内部可変なデータを、既定のバイト単位ではなく参照の単位で追跡させます。そのため、このフラグを付けると、Tree Borrows はより寛容になります。
* `-Zmiri-force-page-size=<num>` は、アーキテクチャの既定のページサイズを、1k の倍数で上書きします。ほとんどのターゲットで、既定は `4` です。この値は、常に 2 のべき乗で、0 以外であるべきです。

[function ABI]: https://doc.rust-lang.org/reference/items/functions.html#extern-function-qualifier

さらに、Miri はいくつかの環境変数を認識します。

* `MIRIFLAGS` は、Miri に渡す追加のフラグを定義します。
* `MIRI_LIB_SRC` は、Miri がビルドして解釈実行に使う標準ライブラリのソースがあるはずのディレクトリを定義します。このディレクトリは、`rust-lang/rust` リポジトリのチェックアウトの `library` サブディレクトリを指していなければなりません。
* `MIRI_SYSROOT` は、使用する sysroot を示します。`cargo miri test` / `cargo miri run` を使う場合、これを設定すると自動のセットアップが省かれます。自動で作られる sysroot を使いたくない場合にだけ設定してください。`cargo miri setup` を呼び出す場合、これは sysroot が置かれる場所を示します。
* `MIRI_NO_STD` は、ターゲットの sysroot が libstd なしでビルドされるようにします。これにより、no_std のプログラムをテストしたり実行したりできます。これは*通常は使うべきではありません*。Miri には、ターゲット名にもとづいて no-std のターゲットを検出するヒューリスティックがあります。libstd に対応しているターゲットでこれを設定すると、分かりにくい結果になることがあります。

[testing-miri]: https://github.com/rust-lang/miri/blob/master/CONTRIBUTING.md#testing-the-miri-driver

## Miri の `extern` 関数 {#miri-extern-functions}

Miri は、プログラムが Miri 固有の機能を使うためにインポートできる、いくつかの `extern` 関数を提供しています。これらは [/tests/utils/miri\_extern.rs](https://github.com/rust-lang/miri/blob/master/tests/utils/miri_extern.rs) で宣言されています。

## no-std のバイナリのエントリポイント {#entry-point-for-no-std-binaries}

標準ライブラリを使わないバイナリは、Miri がどこから実行を始めればよいかを知れるように、次のような関数を宣言することになっています。

```rust
#[cfg(miri)]
#[unsafe(no_mangle)]
fn miri_start(argc: isize, argv: *const *const u8) -> isize {
    // Call the actual start function that your project implements, based on your target's conventions.
}
```

## コントリビュートと助けの求め方 {#contributing-and-getting-help}

Miri にコントリビュートしたいなら、大歓迎です！ 私たちの[コントリビューションガイド](https://github.com/rust-lang/miri/blob/master/CONTRIBUTING.md)を見てください。

Miri の実行について助けが必要なときは、この GitHub で issue を開くか、[Rust の Zulip にある Miri のストリーム][zulip]を使ってください。

[zulip]: https://rust-lang.zulipchat.com/#narrow/stream/269128-miri

## 歴史 {#history}

このプロジェクトは、2015 年に、[サスカチュワン大学（University of Saskatchewan）][usask]の学部生向け研究科目の一環として、@solson によって始まりました。そのプロジェクトの[スライド][slides]と[報告書][report]が公開されています。2016 年には @oli-obk が加わり、Miri が最終的に Rust コンパイラ自体の定数評価器として使われるよう準備を進めました（基本的には、`const` や `static` といったもののためです）。これは、AST を直接扱っていた古い評価器を置き換えるものでした。2017 年には、@RalfJung が Mozilla でインターンシップを行い、Miri を未定義動作の検出ツールへと発展させ始めました。あわせて、Rust の未定義動作についてのさまざまな定義の候補がもたらす帰結を探る手段としても、Miri を使い始めました。@oli-obk による、Miri のエンジンのコンパイラへの移行は、2018 年の初めにようやく完了しました。一方、同じ年の後半に、@RalfJung は 2 回目のインターンシップを行い、Miri をさらに発展させました。型の基本的な不変条件の検査と、参照がエイリアシングの制約に従って使われていることの検証に対応させたのです。

[usask]: https://www.usask.ca/
[slides]: https://solson.me/miri-slides.pdf
[report]: https://solson.me/miri-report.pdf

## Miri が見つけたバグ {#bugs-found-by-miri}

Miri はすでに、Rust の標準ライブラリやそれ以外で、数多くのバグを見つけています。その一部をここに集めています。Miri のおかげで、あなたのコードの見つけにくい UB のバグが見つかったなら、一覧に追加する PR を送ってもらえるとうれしいです！

見つかった、確実なバグ:

* [`Debug for vec_deque::Iter` が未初期化メモリにアクセスしていた](https://github.com/rust-lang/rust/issues/53566)
* [`Vec::into_iter` がアラインされていない ZST の読み出しを行っていた](https://github.com/rust-lang/rust/pull/53804)
* [`From<&[T]> for Rc` がアラインメントの足りない参照を作っていた](https://github.com/rust-lang/rust/issues/54908)
* [`BTreeMap` が小さすぎるアロケーションを指す共有参照を作っていた](https://github.com/rust-lang/rust/issues/54957)
* [`Vec::append` がダングリング参照を作っていた](https://github.com/rust-lang/rust/pull/61082)
* [Future が共有参照を可変参照に変えていた](https://github.com/rust-lang/rust/pull/56319)
* [`str` が共有参照を可変参照に変えていた](https://github.com/rust-lang/rust/pull/58200)
* [`rand` がアラインされていない読み出しを行っていた](https://github.com/rust-random/rand/issues/779)
* [Unix のアロケータが `posix_memalign` を不正な方法で呼んでいた](https://github.com/rust-lang/rust/issues/62251)
* [`getrandom` が `getrandom` システムコールを不正な方法で呼んでいた](https://github.com/rust-random/getrandom/pull/73)
* [`Vec`](https://github.com/rust-lang/rust/issues/69770) と [`BTreeMap`](https://github.com/rust-lang/rust/issues/69769) が、一部の（パニックが絡む）条件でメモリをリークしていた
* [`beef` がメモリをリークしていた](https://github.com/maciejhirsz/beef/issues/12)
* [`EbrCell` が未初期化メモリを誤って使っていた](https://github.com/Firstyear/concread/commit/b15be53b6ec076acb295a5c0483cdb4bf9be838f#diff-6282b2fc8e98bd089a1f0c86f648157cR229)
* [TiKV がアラインされていないポインタアクセスを行っていた](https://github.com/tikv/tikv/issues/7613)
* [`servo_arc` がダングリングな共有参照を作っていた](https://github.com/servo/servo/issues/26357)
* [TiKV が境界外のポインタ（と、重なり合う可変参照）を作っていた](https://github.com/tikv/tikv/pull/7751)
* [`encoding_rs` が境界外のポインタ演算を行っていた](https://github.com/hsivonen/encoding_rs/pull/53)
* [TiKV が `Vec::from_raw_parts` を誤って使っていた](https://github.com/tikv/agatedb/pull/24)
* [`AtomicPtr`](https://github.com/rust-lang/rust/pull/84052) と [`Box::from_raw_in`](https://github.com/rust-lang/rust/pull/84053) の誤った doctest
* [`ThinVec` のアラインメント不足](https://github.com/Gankra/thin-vec/pull/27)
* [`crossbeam-epoch` が、部分的にしか初期化されていない `MaybeUninit` に対して `assume_init` を呼んでいた](https://github.com/crossbeam-rs/crossbeam/pull/779)
* [`integer-encoding` が、アラインメントのずれたポインタを参照外ししていた](https://github.com/dermesser/integer-encoding-rs/pull/23)
* [`rkyv` が、過剰にアラインされたアロケーションから `Box<[u8]>` を作っていた](https://github.com/rkyv/rkyv/commit/a9417193a34757e12e24263178be8b2eebb72456)
* [`arc-swap` のデータ競合](https://github.com/vorner/arc-swap/issues/76)
* [`thread::scope` のデータ競合](https://github.com/rust-lang/rust/issues/98498)
* [`regex` が、アラインされていない `Vec<u8>` バッファを誤って扱っていた](https://www.reddit.com/r/rust/comments/vq3mmu/comment/ienc7t0?context=3)
* [`once_cell` での `compare_exchange_weak` の誤った使用](https://github.com/matklad/once_cell/issues/186)
* [`vec::IntoIter` での、アラインされていないポインタによるドロップ](https://github.com/rust-lang/rust/pull/106084)
* [インプレースの `Iterator::collect` の新しい特殊化での、誤ったレイアウトによる解放](https://github.com/rust-lang/rust/pull/118460)
* [`portable-atomic-util` での、アラインメントの大きい型に対する誤ったオフセット計算](https://github.com/taiki-e/portable-atomic/pull/138)
* [`std::mpsc` チャネルでときどき起きるメモリリーク](https://github.com/rust-lang/rust/issues/121582)（元のコードは [crossbeam](https://github.com/crossbeam-rs/crossbeam/pull/1084) にあります）
* [Windows のスレッドローカルストレージでの、弱いメモリが原因のメモリリーク](https://github.com/rust-lang/rust/pull/124281)
* [`RwLock::downgrade` の新しい実装のバグ](https://rust-lang.zulipchat.com/#narrow/channel/269128-miri/topic/Miri.20error.20library.20test)（Rust のリポジトリに入る前に Miri が捕まえました）
* [Mockall が、`std::io::Read::read` をモックするとき、すべての期待が満たされていても未初期化メモリを読んでいた](https://github.com/asomers/mockall/issues/647)（Tokio のテストスイートを実行していた Miri が捕まえました）
* [`ReentrantLock` が、異なるスレッドの TLS ストレージでのアドレスの再利用を正しく扱っていなかった](https://github.com/rust-lang/rust/pull/141248)
* [スレッドのパーキング（とその解除）のコード例での、まれなデッドロック](https://github.com/rust-lang/rust/issues/145816)
* [`winit` が、Windows でグローバルコンストラクタを誤った ABI で登録していた](https://github.com/rust-windowing/winit/issues/4435)
* [`VecDeque::splice` が物理インデックスと論理インデックスを取り違えていた](https://github.com/rust-lang/rust/issues/151758)
* [`oneshot` チャネルのデータ競合](https://github.com/faern/oneshot/issues/69)
* [serde-yaml-bw のメモリリーク](https://github.com/bourumir-wyngs/serde-yaml-bw/issues/197)
* [chacha20 の SSE2 バックエンドでの、SSE4.1 の組み込み関数の誤った使用](https://github.com/RustCrypto/stream-ciphers/issues/579)

見つかった [Stacked Borrows] の違反のうち、おそらくバグであるもの（ただし、Stacked Borrows は現在のところ実験にすぎません）:

* [`VecDeque::drain` が重なり合う可変参照を作っていた](https://github.com/rust-lang/rust/pull/56161)
* `BTreeMap` のさまざまな問題
    * [`BTreeMap` のイテレータが、共有参照と重なり合う可変参照を作っていた](https://github.com/rust-lang/rust/pull/58431)
    * [`BTreeMap::iter_mut` が重なり合う可変参照を作っていた](https://github.com/rust-lang/rust/issues/73915)
    * [`BTreeMap` のノードの挿入が、生ポインタを有効なメモリ領域の外で使っていた](https://github.com/rust-lang/rust/issues/78477)
* [`LinkedList` のカーソルによる挿入が、重なり合う可変参照を作っていた](https://github.com/rust-lang/rust/pull/60072)
* [`Vec::push` が、そのベクタの中を指す既存の参照を無効にしていた](https://github.com/rust-lang/rust/issues/60847)
* [`align_to_mut` が可変参照の一意性に違反していた](https://github.com/rust-lang/rust/issues/68549)
* [`sized-chunks` がエイリアスする可変参照を作っていた](https://github.com/bodil/sized-chunks/issues/8)
* [`String::push_str` が、その文字列の中を指す既存の参照を無効にしていた](https://github.com/rust-lang/rust/issues/70301)
* [`ryu` が、生ポインタを有効なメモリ領域の外で使っていた](https://github.com/dtolnay/ryu/issues/24)
* [ink! が重なり合う可変参照を作っていた](https://github.com/rust-lang/miri/issues/1364)
* [TiKV が、重なり合う可変参照と生ポインタを作っていた](https://github.com/tikv/tikv/pull/7709)
* [Windows の `Env` イテレータが、生ポインタを有効なメモリ領域の外で使っていた](https://github.com/rust-lang/rust/pull/70479)
* [`VecDeque::iter_mut` が重なり合う可変参照を作っていた](https://github.com/rust-lang/rust/issues/74029)
* [生ポインタが関わる、標準ライブラリのさまざまなエイリアシングの問題](https://github.com/rust-lang/rust/pull/78602)
* [`<[T]>::copy_within` が、借用を無効にした後でそれを使っていた](https://github.com/rust-lang/rust/pull/85610)

## Miri を利用している学術論文 {#scientific-papers-employing-miri}

* [Stacked Borrows: An Aliasing Model for Rust](https://plv.mpi-sws.org/rustbelt/stacked-borrows/)
* [Using Lightweight Formal Methods to Validate a Key-Value Storage Node in Amazon S3](https://www.amazon.science/publications/using-lightweight-formal-methods-to-validate-a-key-value-storage-node-in-amazon-s3)
* [SyRust: Automatic Testing of Rust Libraries with Semantic-Aware Program Synthesis](https://dl.acm.org/doi/10.1145/3453483.3454084)
* [Crabtree: Rust API Test Synthesis Guided by Coverage and Type](https://dl.acm.org/doi/10.1145/3689733)
* [Rustlantis: Randomized Differential Testing of the Rust Compiler](https://dl.acm.org/doi/10.1145/3689780)
* [A Study of Undefined Behavior Across Foreign Function Boundaries in Rust Libraries](https://arxiv.org/abs/2404.11671)
* [Tree Borrows](https://plf.inf.ethz.ch/research/pldi25-tree-borrows.html)
* [Miri: Practical Undefined Behavior Detection for Rust](https://plf.inf.ethz.ch/research/popl26-miri.html) **（この論文は Miri 自体について述べています）**

## ライセンス {#license}

次のいずれかのライセンスの下で提供されます。

  * Apache License, Version 2.0（[LICENSE-APACHE](https://github.com/rust-lang/miri/blob/master/LICENSE-APACHE) または
    <http://www.apache.org/licenses/LICENSE-2.0>）
  * MIT license（[LICENSE-MIT](https://github.com/rust-lang/miri/blob/master/LICENSE-MIT) または
    <http://opensource.org/licenses/MIT>）

どちらにするかは、あなたが選べます。

### コントリビューション {#contribution}

あなたが明示的に別の定めをしないかぎり、この著作物に含めるためにあなたが意図的に提出したコントリビューションは、いかなる追加の条項や条件もなしに、上記のとおりデュアルライセンスされるものとします。
{% endraw %}
