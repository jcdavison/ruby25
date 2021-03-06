# Ruby 25 周年によせて　Ruby の今後の性能改善について

Ruby25周年おめでとうございます。

クックパッド株式会社でフルタイム Ruby コミッタをしている笹田です。
こんなふうに Ruby 開発で職を得られるとは、YARV を開発した 2004 年には思ってもいませんでした。

本稿では、私の主務である Ruby の高速化について、今後の展望を述べたいと思います。

# Ruby インタプリタはなぜ遅いのか

Ruby は遅い、と言われています。何が遅いのでしょうか。いくつか要因があります。

## 数値計算

Ruby は数値計算が遅い、と言われています。Ruby では、わざわざ無駄なことをしているから遅い、というわけではなく、色々（多くの場合、無駄に）気を回しすぎて遅い、ということが多いです。

例で示します。C 言語では、`int` 型の整数の足し算 `a + b` は、（大抵の場合）機械語1命令になり、たいていは 1 clock （以下）で実行されます。どちらかというと、機械語 1 命令に落ちる計算が自然に書ける、というのが、C 言語の特徴かもしれません。

Ruby の場合、`a + b` と書いたとき、そもそも変数 `a` と `b` が `int` 型の整数であるか、という情報が無く、例えば `a` は `int` 型の整数で収まる数かもしれないし、収まらない数（いわゆる Bignum。以下 Bignum という用語を利用）かもしれない、もしかしたら文字列や配列かもしれません。Ruby には言語に、そのような情報を伝える手段がないため、Ruby インタプリタは愚直に、すべての計算で、これはどんなオブジェクトだろう、というチェックをしたりしています。また、`int` 型で収まる整数値だとわかっても、足し算の結果、Bignum に拡張しなければならないかもしれません。整数値の足し算とわかっても、整数値の足し算の処理が再定義されているかもしれないので、それもチェックしたほうがいいでしょう。

などなどしていると、`a + b` という処理が`int` 型の整数で収まる数であっても、実行に数百命令かかります。実に、C 言語で記述した場合と比較して数百倍遅いことになります。そりゃ遅いわ。数値計算を記述するプログラマは `a` と `b` が十分小さければ（`int` 型に収まるのであれば）、C 言語と同程度の速度を期待するのは、そんなに変な話じゃありません。ただ、上記の通り、Ruby インタプリタはその情報を（素直に）知る方法がないため、「もしかしたら」を毎回真面目に計算するわけです。

多くの場合、数値計算は2値の計算1回だけではなく、複数の計算の組み合わせで実行されます。例えば、`a + (b * c) * (b * c)` のような。C 言語のような言語には、いろいろな最適化手法が知られていますが（例えば `b * c` は一度だけしか計算しない、とか）、上記「もしかしたら」を真面目に考えていくと、それを素直に適用するのは面倒くさいわけです。

さらに、Ruby のプログラムは仮想マシンの命令に置き換わりますが、仮想マシン1命令を実行するために、一番短い命令（nop）でも、CPU 10命令分程度かかり、それもオーバヘッドになります。

## オブジェクト割り当て（メモリ割り当て）とガーベージコレクション（GC）

Ruby は GC が遅い、と Ruby 2.1 より前は言われていました。今では世代別 GC があるため、そこまで GC が遅いわけでもないのですが（GC での停止時間は、そんなでもなくなっています）、オブジェクトを頻繁に生成するため、その速度が効いてくることがあります。

例で示します。`("abc" + "def").upcase` というプログラムを考えます。このとき、`"abc"`、`"def"`、`"abcdef"` と、`"ABCDEF"` という 4 つの文字列が生成されます。

最近の Ruby では、`frozen_string_literal` プラグマを利用すると、`"abc"` と `"def"` はコンパイル時に1度割り当てられるだけなので、実質的には 2 つ生成されることになります（このように、生成されるオブジェクトの数が減るので、Ruby 3 では `frozen_string_literal` をデフォルトでオンにしておこう、という話があったりします。いや、性能のためだけじゃないですが）。

さて、結果の `"ABCDEF"` は、まぁ生成しないといけない気がしますが、`"abcdef"` についてはすぐに要らなくなる中間データです。でも、今の Ruby では毎回生成しています。

例えば C 言語でしたら、この場合 6 文字であることがわかっているので、スタック上に必要なバイト数だけバッファを確保し、最終的に1度の `malloc` によるメモリ確保、つまりオブジェクト生成を行う、といった処理にするでしょう。スタック上のバッファの割り当ては、関数を呼ぶオーバヘッドに隠蔽されるため、ほぼコストが 0 で実施することができます。

Ruby の場合、オブジェクト生成のルーチンがかかり（これが10命令くらい）、さらに必要なら `malloc` を呼び、そして時々 GC を起こす（時々起こる GC のコストは、数万クロック以上かかるんじゃないかと思います）、ということになり、やはりこれも C 言語と比べるとずいぶん遅いです。

# 性能向上に関する展望

なぜ遅いか、その理由（の一部）を示したので、それらを解決する手法について紹介します。

## JIT コンパイルによる解決

現在、Ruby 2.6 に向けて、MJIT という JIT コンパイラ（正確には、JIT コンパイラを搭載するための基盤）の導入が進められています。JIT コンパイラを利用すると、いくつかのボトルネックが解消する可能性があります。

まず、仮想マシンの命令を実行するために必要になるオーバヘッドが不要になります（ナイーブに JIT コンパイラを使うと、まずここが効きます）。プログラムカウンタをどうこうして、オペランドをフェッチして、といった処理が、コンパイル時に（だいたい）計算できるため、実行時に行う必要がなくなります。

次に、実行時プロファイリングが可能になります（厳密には、JIT コンパイラでなくても平気ですが、作りやすいです）。例えば、ループの各繰り返しでは、同じようなことを行うことが多いです。ということは、変数の型や、呼び出されるメソッドは同じものを使うことができる可能性が高いです。そこで、以前の繰り返しと同じだと仮定して、いろいろ特殊化してしまうことで、大抵は成功する、最適化されたプログラムを得ることができます（多分。多くの場合）。上記の例では、変数 `a`、`b` が十分小さい整数だ、とか、そういう情報です。

また、JIT コンパイラによって、インライン化もやりやすくなります（なる可能性があります）。

色々夢が膨らみますね。

## プログラムの解析による解決

JIT コンパイラの方でインライン化と書いたのですが、その辺と関連する話で。

オブジェクト割り当てで紹介したような、余分な割り当てはなぜ起こるのか、というと、中間オブジェクトがどのように使われるか、ちゃんと調べないとわからないからです。逆に言うと、ちゃんと調べれば、中間オブジェクトはその場でしか使われないから、例えばスタックに実体を置いてしまう、というようなことができます。スタックアロケーションみたいに言います。ちゃんと調べるということをエスケープ解析とかいったりします。現状、エスケープ解析を行うために必要な情報が決定的に足りていないので、その辺の整備（情報をどう加えていくか、といったノーテーションの話から、解析の実装まで、多岐にわたります）をしなければならないのですが、ちゃんとやれば出来る話なので、やっていきたいと思っています。

## 並列化による高速化

現在は並列化がしづらいプログラミング言語になっていますが、しづらい点を整理して、並列化をするための仕組みをいくつか入れて、マルチコアといった並列計算機を十分活用できるようにすれば、性能改善が期待できます。

他の解決方法とは独立に行うことができるので、他の解決方法で m 倍、その上で n 並列実行すれば、n * m 倍の性能向上が得られます（すごーく理想的には。アムダールの法則を参照）。

# おわりに

他にもいろいろあるんですが（例えば、precompilation、bulk definition による起動速度の高速化、ブロックセットアップ情報のキャッシュ、新しいメソッドキャッシュによる特化命令の統一、セカンドレベルヒープの導入とその世代別管理、等々）、面倒になってきたのでそろそろ終わりにします。

他のインタプリタの高性能さを見てください。Ruby（MRI） は、やはりまだまだ遅いです。
つまり、Ruby にはまだまだ高速化の余地があります。

25年で、Ruby は昔と比べて、とても高性能になりました。次の 25 年も、まだまだ性能改善していくことができるんじゃないかと思います。
微力ながら、私も力を尽くし、高速化していきたいと思っています（それでお給料もらってるし）。

やっぱり Ruby プログラミングは楽しいですね。やることイッパイあって。

2018年2月 笹田耕一拝

# 著者について

笹田耕一（クックパッド株式会社）

大学在学時からRuby向け仮想マシンYARVを開発し、2007年に「Ruby 1.9」に採用される。以降、Rubyコミッターとして、言語処理系の高速化に従事し、仮想マシンやガーベージコレクションの性能改善などを行なう。東京大学大学院情報理工学系研究科助手、助教、講師（2006～2012）。株式会社セールスフォース・ドットコム、Heroku, Inc.（2012～2017）。現在はクックパッド株式会社にてRubyインタプリタ開発に従事（2017～）。Rubyアソシエーション理事 （2012～現任）。博士（情報理工学）。未踏ユーススーパークリエータ（2004）、情報処理学会山下記念研究賞（2015）。他、受賞多数（詳細）。 
 
