# 論文のテーマ
近年大規模データ処理ではMapReduceフレームワークがデファクトスタンダードになっている。
このフレームワークはGoogleによって開発されたもので、OSS版のHadoopは多くの企業や大学で利用されている。(弊社でもめちゃめちゃ使ってます。)

MapReduceフレームワークは既存のよく研究されている並列コンピューティングとは以下の点で異なる。
- 並列計算と逐次計算を織り交ぜている

この新しいフレームワークについて、すでに様々なアルゴリズムが提案されているが、既存の有名な並列計算のモデルであるPRAMのような、しっかりとしたモデルはまだ提案されていなかった。
この研究では、MapReduceの正式なモデルを定義しPRAMモデルと比較する。

主な結果
- $O(n^{2-\epsilon})$のプロセッサーと$O(n^{2-\epsilon})$のトータルメモリを使用するPRAMアルゴリズムのサブラクラスはMapReduceで模倣することができる
- MapReduceで利用できる2つの並列化のためのテクニックを示し、それを応用してdence grapfにおけるMSTとundirected s-t connectivityを解くアルゴリズムを示す

# MapReduce Basics
MapReduceプログラミングにおける基本的な情報の単位はキーバリューペア
- $<key; value>$
  - $key$と$value$はバイナリ文字列

任意のMapReduceアルゴリズムの入力はキーバリューペアのセットである。
このセットは以下の3つのステージで操作される。  
1. map stage
2. shuffle stage
3. reduce stage

(関数型のプログラミングをしたことがあると分かりやすいかも)  

map stageではmapper $\mu$が1つのキーバリューペアーを入力として受け取り、任意の数のキーバリューペアーを出力する。map操作の特徴はstatelessであること。この特性のため、別々のマシンで並列にmap操作を行うことができる。

shuffle stageでは、MapReduceを実装しているシステムが同じキーを持つペアは同じマシンに配置されるように、map stageの出力を分配する。このステージはシームレスに行われ、プログラマーは意識する必要がない。

reduce stageでは、reducer $\rho$がキー $k$を持つ全てのペアを入力に取り$k$をキーに持つペアのマルチセットを出力する。よって、**reducerが処理を実行する前にすべてのmapperの処理が終わっている必要がある。**

reducer単体の処理は逐次的に行われるが、異なるreducerは同時に並列に実行される。

MapReduceではこのようなmapperとreducerが繰り返し実行される。

## MapReduce Example
MapReduceの簡単な例として$k$-th frequency momentの計算を取り上げる。$k$-th frequency momentの定義

$F_k(a) = \Sigma_{i=1}^n a_i^k$

$a$はフリークエンシーの集合

問題定義
- $x$ : 長さ$n$の入力文字列
- $x_i$ : $x$における$i$番目の文字
- $\$$ : 特殊文字
- MapReduceの入力は$<i; x_i>$で表す

プログラムの流れ
1. 最初のmapper $\mu_1$ではキーとバリューを入れ替える
   - $\mu_1(<i;x_i>) = <x_i; i>$
2. 最初のreducer $\rho_1$では、集められたペアーから部分的なfrequency momentを計算する
   - $\rho_1(<x_i; {v_1, \dots, v_m}>) = <x_i; m^k>$
3. 次のmapperでは2の計算結果を足し合わせるために、キーを特殊文字に置き換える
   - $\mu_2(<x_i; v>) = <\$; v>$
4. 全ペアーが同じキーを持つので、すべての入力は一つのマシンに集められる。よって最後はすべてを足し合わせればいい
   - $\rho_2(<\$; {v1, \dots, v_l}>) = <\$, \Sigma_i v_i>$

並列化の容易さの他のMapReduceの利点はフレームワークがフォールトトレランスやデータの分配といった低レベルの処理をプログラマーに対して隠蔽していること。プログラマーはそういったことを意識せずにmapperとreducerのみを開発すれば良い。

一方で欠点は並列性を得るために、プログラマーはプログラムをmapperとreducerの落とし込まないといけないこと。
つまり柔軟性と並列化の容易さがトレードオフになっている。
どのような種類の問題がMapReduceで効率的に解けるのかは自明ではない。この論文では、MapReduceでどのような問題が効率的に解けるのかを明らかにするためにモデルを定義した。

# The Mapreduce Programming Paradigm
ここからMapReduceの定式化を始めていく。
最初にmapperとreducerを定義するところから始める

Definition 2.1  
A mapper is a (possibly randomized) function that takes as input one ordered $<key; value>$ pair of binary strings. As output the mapper produces a finite multiset of new $<key; value>$ pairs

mapperは1度に1つのペアのみ処理することに注意

Definition 2.2  
A reducer is a (possibly randomized) function that takes as input a binary string $k$ which is the key, and a sequence of values $v_1, v_2, \dots$ which are also binary strings. As output, the reducer produces a multiset of pairs of binary strings $<k; v_{k,1}>, <k; v_{k, 2}>, <k; v_{k,3}>, \dots$. The key in the output tuples is identical to the key in the input tuple.

mapperはkeyを自由に操作することができるが、reducerはkeyを操作することはできない。

次にシステムがどのようにMapReduceを実行するのかを示す。
MapReduceプログラムはmapperとreducerの系列で構成される。  

$<\mu_1, \rho_1, \mu_2, \rho_2, \dots, \mu_R, \rho_R>$

入力は$<key; value>$のマルチセットで$U_0$とする。
入力$U_0$に対してプログラムは以下のように実行される。

For $r = 1, 2, \dots, R$, do:  
1. $U_{r - 1}$の各ペア$<k; v>$を$\mu_r$に投入し実行する。mapperはtupleのシーケンス$<k_1; v_1>, <k_2; v_2>, \dots$を生成する。$U'_r$を$\mu_r$によって生成されるマルチセットとする。すなわち$U'_r = \bigcup_{<k;v> \in U_{r - 1}} \mu_r (<k; v>)$
2. 各$k$に対して$V_{k, r}$を$U'_r$でキーに$k$を持つバリューのマルチセットとする。MapReduceを実装するシステムは$V_{k,r}$を$U'_r$から生成する。
3. 各$k$に対して、1つの分離された$\rho_r$に$k$と$V_{k, r}$の任意の順列を入力し実行する。reducerは$<k;v'_1>,<k;v'_2>,\dots$を生成する。$U_r$をreducerが生成するマルチセットとする。すなわち$U_r = \bigcup_k \rho_r(<k; V_{k,r}>)$

最後のreducer $\rho_r$が停止すると計算が終了する。

mapperもreducerも同時に複数のインスタンスを動作させることができるため、MapReduceは容易に並列化できる。

# The MapReduce Class ($MRC$)
この章ではMapReduce Class ($MRC$)を定義する。
定義に組み込みたい3つの原理が存在する。

**Memory**  
MapReduceでは問題を分解して並列に解くことができるので、一つのマシンが問題のすべてをメモリに載せることが出来たらあまり意味がない。
したがって、mapperとreducerに対する入力のサイズは問題のサイズのsublinearである必要がある。
そうでなければ、$P$に属するすべての問題は最初のmapperで1つのreducerにマップしreducerでそのまま問題を解くことができてしまう。

**Machines**  
マシンの台数に制限がない場合、一気に現実との関連性がなくなってしまうため、マシンの台数に対しても制限を加える。
具体的にはマシンの台数も問題のサイズのsublinearとする。

**Time**  
[先行研究]()とは異なり、MRCではmapperとreducerの計算能力を基本的には制限しない。
mapperとreducerはオリジナルの入力サイズの多項式時間で動作するとする。
またshuffleステージは時間がかかる処理のため、全体で必要になるラウンド数も少ないことが求められる。

以上より$MRC$を以下のように定義する。  
入力は有限の長さのペア$<k_j; v_j>$のシーケンスで、$k_j$と$v_j$はバイナリ文字列。  
入力の長さ$n$は$n = \Sigma_j (|k_j| + |v_j|)$

Definition 3.1  
Fix an $\epsilon > 0$. An algorithm in $MRC^i$ consists of a sequence $<\mu_1, \rho_1, \mu_2, \rho_2, \cdots, \mu_R, \rho_R>$ of operations which outputs the correct answer with probability af least $3/4$ where:
- Each $\mu_r$ is a randomized mapper implemented by a $RAM$ with $O(\log n)$-length words, that uses $O(n^{1 - \epsilon})$ space and time ploynomial in n.
- Each $\rho_r$ is a randomized reducer implemented by a $RAM$ with $O(\log n)$-length words, that uses $O(n^{1-\epsilon})$ space and time plynomial in n.
- The total space $\Sigma_{<k;v> \in U'_r}(|k| + |v|)$ used by $<key; value>$ pairs output by $\mu_r$ is $O(n^{2 - 2\epsilon})$.
- The number of rounds $R = O(\log^i n)$.

RAMがペアーのシーケンスを出力するとしても、マルチセットとして解釈する。

決定的な$MRC$は$DMRC$と呼ぶ。 

reducerについて注意することは、reducerは入力として任意の順序でシーケンスを受け取る。よって、reducerはどんな順序で入力が渡されても正しい答えを出力する必要がある。 

mapperとreducerは自身が受け取った入力ではなく、元の問題の入力サイズである$n$の多項式時間で動作する。

先ほど紹介したfrequency momentのアルゴリズムはこの定義では利用できない。
入力文字列が同じ文字の$n$個のコピーで構成されている場合、メモリ量の制約に違反してしまうため。
後ほど正しい$MRC$アルゴリズムを示す。

## Disucssion
ここから定義したモデルの妥当性について議論する。

### Machines
前述したように$n$がとても大きい場合、マシンの数が$n$に線形であるというのは現実的ではない。
一方この定義では小さい$\epsilon$も許容しており、この場合も$n$がとても大きい場合には、線形のときと同様に大量のマシンが必要になってしまうが、マシンの台数を$O(n^{1/2})$とするのも不自然だと考えられる。

### Memory Restrictions
mapperとreducerは$O(n^{1 - \epsilon})$のメモリを持つマシンの上で動作するので、すべてのペア$<key; value>$のサイズは$O(n^{1-\epsilon})$である必要がある。

また、トータルのメモリのサイズは$O(n^{2 - 2\epsilon})$である。
reducerはすべてのmapperの処理が終わってからしか動作できないので、全mapperの出力である$U'_r$のサイズは$O(n^{2 - 2\epsilon})$である必要がある。

一方mapperは入力を一つずつ処理できるため、reducerには上記のような制限はない。

### Shuffle Step
このステップでは、reducerを実行するために、同じキーを持つペアを同じマシンに集める。
このときにマシンでメモリの制約違反が起こってはいけない。
次のLemmaでDefinition 3.1においてメモリの制約違反を起こさずに適切にキーのshuffleが行えることを証明する。

Lemma 3.1  
Consider round $r$ of the execution of an algorithm in $MRC$. Let $K_r$ be the set of keys in $U'_r$, let $V_r$ be the multiset of values in $U'_r$ and let $V_{k,r}$ denote the multiset of values in $U'_r$ that have key $k$.  
Then $K_r$ and $V_r$ can be partitioned across $\Theta(n^{1 - \epsilon})$ machines such that all machines get $O(n^{1-\epsilon})$ bits, and the pair $<k, V_{k,r}>$ gets sent to the same machine.

証明には以下の事実とminimum makespan scheduling problemに対するGrahan's greedy algorithmを利用する。  
[参考文献1]()  
[参考文献2]()

事実
- $s(V_r) + S(K_r) \leq s(U'_r) = O(n^{2 - 2\epsilon})$
  - $s(B) = \Sigma_{b \in B} |b|$
- $|k| + s(V_{k, r})$は$O(n^{1 - \epsilon})$
  - reducerのスペースが$O(n^{1 - \epsilon})$に制限されているため


Graham's greedy algorithmより、一つのマシンに割り当てられる最大のビット数は平均ロードと$<k, V_{k,r}>$の最大ビット数の和よりも大きくならない。

$$
\leq \frac{s(V_r) + s(K_r)}{\rm{number\ of\ machines}} + \max_{k \in K_r}(|k| + s(V_{k,r})) \\
\leq \frac{O(n^{2 - 2\epsilon})}{\Theta(n^{1-\epsilon})} + O(n^{1 - \epsilon}) \\
\leq O(n^{1-\epsilon})
$$

これより、Definition 3.1はshuffle stepの実行のためにも必要であることがわかる。

### Time Restrictions
mapperとreducerが元の問題の入力サイズの多項式時間の計算能力を持つことが非現実的だという指摘も考えられる。
しかし、$O(n \log n)$のような任意の関数で制限することもまた自然ではない。

実際のMapReduceの1ラウンドの計算時間は非常に長くなることがある。
よって、ラウンド数を抑えることが重要となる。
理想的にはアルゴリズムは$MRC^0$で有るべきだが、多くの重要なアルゴリズムが$MRC^1$になっている。

# Related Work
最初にPRAMとMRCを比較し、その後MapReduceを利用した他の論文について議論する。

## Comparing MapReduce and PRAMs
現在最も広く有名な並列計算のモデルPRAM
これに次ぐのが以下の2つ
- [LogP]()
- [BSP]()

これらの3つのモデルはアーキテクチャに依存しないが、アーキテクチャに依存したモデルを研究している研究者もいる。

ここでは最も有名なPRAMとMRCを比較する。
PRAMでは以下のように計算が実行される。
- 任意の数のプロセッサーが
- 制限のない巨大なメモリを共有し
- 共有された入力に対して同時に処理を実行し、出力を生成する

プロセッサー数は通常サイズ$n$の問題に対して多項式に制限される。

PRAMの研究には2つの種類が存在する。
- PRAMではどのような問題が多項式のプロセッサー数かつpolylog時間で解くことができるのか
  - このような問題を$NC$と呼ぶ
- PRAMではどのような問題が効率よく並列化できるのか
  - 逐次的なアルゴリズムより早く、プロセッサータイムのproductは逐次的なアルゴリズムの実行時間に近い

PRAMは理論的には美しいが、現実では問題点も大きい
- 大量のプロセッサー数を持つshared memory machineは存在せずシミュレーションは遅い
- shared memoryで巨大な計算機を作ることは難しい
- プロセッサー数が$n$の多項式というのは非現実的



