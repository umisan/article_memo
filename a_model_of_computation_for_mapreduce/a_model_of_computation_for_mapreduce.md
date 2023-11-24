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

自然な発想として、$MRC$や$DMRC$というクラスが$NC$や$P$のような既存の複雑性のクラスとどのような関係にあるのかということが考えられる。
ここでは$DMRC$をメインに取り上げる。

$DMRC \subseteq P$は明らかに成り立つが、$P \subseteq DMRC$は成り立つのか？
同様に$DMRC$と$NC$の関係はどうなっているのか。
7章で、大きなlanguage class $L \in NC$が$L \in DMRC$であることを示す。（条件を満たすPRAMのアルゴリズムはDMRCでシミュレーションできる。）

逆については以下の定理が成り立つ。  
Theorem 4.1  
If $P \neq NC$ then $DMRC \not \subseteq NC$

$P \subseteq DMRC$は成り立たないと考えているが証明はできていない。


## MapReduce: Algorithms and Models
MapReduceはデータセットに1つのワードがどの程度現れるかというような、ナイーブな並列化についてよく研究されているが、近年非自明な問題についてもアルゴリズムが研究されている。
- massive graphsの直径の計算
- グラフにいくつのtrianglesが存在するかの計算
- Em clustering algorithm

これらの研究は実践的なアルゴリズムを与えているが、厳密なモデルの定義はされていない

MapReduceのモデルとしてはFeldmanの研究が存在する。
この研究ではMassively Unordered Distributed(MUD) algorithmsという概念が提案されている。
このモデルとの違いは
- MUDはreducerはデータをstreamで処理する
  - MRCはランダムアクセスができる
- MUDではreducerはpolylogarithmicスペースしか利用できない

したがって、MRCはよりパワフルなモデルになっている

# Finding an MST of a Dense Graph Using MapReduce
モデルの定義が終わったのでここから具体的なアルゴリズムについて紹介していく。
この章ではDense GraphのMSTを計算するMRCアルゴリズムを紹介する。
（発表時間の都合上アルゴリズムの概要のみに留めます。）

入力
- グラフ$G = (V, E)$
- $|V| = N$
- $|E| = m \geq N^{1 + \epsilon}$ for some constant $c > 0$

$n$は引き続き入力の長さを表す。

**アルゴリズム**    
固定した$k$に対して以下を定義する。
- $V = V_1 \cup V_2 \cup \cdots \cup V_k$ 
  - 任意の$i \neq j$について$V_i \cap V_j = \emptyset$
  - 任意の$i$について$|V_i| = N/k$
- 各ペア${i, j}$について頂点集合$V_i \cup V_j$で誘導される辺集合を$E_{i, j} \subseteq E$とする
- 誘導グラフを$G_{i,j} = (V_i \cup V_j, E_{i,j})$とする

ステップ1  
$\binom{k}{2}$個のサブグラフ$G_{i,j}$について、minimum spanning forest $M_{i,j}$を計算する

ステップ2  
$M_{i,j}$から以下のようなグラフ$H$を構成する  
$H = (V, \bigcup_{i,j} M_{i,j})$

ステップ3  
$H$のMSTを計算する

# An Algorithmic Design Technique for $MRC$
ここから$MRC$における多くのアルゴリズムのビルディングブロックになる、$MRC$-parallelizable functionsについて説明する。

Definition 6.1  
Let $S$ be a set. Call a function $f$ on $S$ $MRC$-parallelizable if there are functions $g$ and $h$ so that:
1. For any parition $T = \{T_1, T_2, \dots, T_k\}$ of $S$, where $\bigcup_i T_i = S$ and $T_i \cap T_j = \emptyset$ for $i \neq j$ (of course), $f$ can be expressed as: $f(S) = h(g(T_1), g(T_2), \dots, g(T_k))$.
2. $g$ and $h$ can be expressed in $O(\log n)$ bits.
3. $g$ and $h$ can be computed in time ploynomial in $|S|$ and every output of $g$ can be expressed in $O(\log n)$ bits.

直感的にはこの定義はもし関数$f$を集合$S$について評価したい場合は、以下の手順を取ればいいことを示している。
1. $S$を任意のパーティションで分割する
2. 個別に$g$を適用する
3. 2の結果に$h$を適用する

Lemma 6.1  
Consider a universe $\mathcal{U}$ of size $n$ and a collection $\mathcal{S} = \{S_1, \dots, S_k\}$ of subsets of $\mathcal{U}$, where $S_i \subseteq \mathcal{U}$, $\Sigma_{i=1}^k |S_i| \leq n^{2 - 2\epsilon}$, and $k \leq n^{2-3\epsilon}$. Let $\mathcal{F} = \{f_1, \dots, f_k\}$ be a collection of $\mathcal{MRC}$-parallelizable functions. then the output of $f_1(S_1), \dots, f_k(S_k)$ can be computed using $O(n^{1-\epsilon})$ reducers each with $O(n^{1-\epsilon})$ space.

この補題は、同じユニバースの部分集合の上に定義された$\mathcal{MRC}$-parallelizable functionsはMapReduceのサブルーチンとして計算可能であること示している。
オーバーフローを起こさないように入力をreducerに分配する部分をこの補題が管理するので、アルゴリズムの設計者はその部分を気にする必要がなくなることが重要。

基本的には1つのreducerに$S_i$と$f_i$を割り当てて計算を行いたい。
しかし、$S_i$は非常に大きくなる可能性があるため、これは実現できない。
したがって、$|S_i| > n^{1-\epsilon}$の場合は、$f_i(S_i)$の計算は複数のreducerに分散して行う必要がある。

この問題に対処するために、$f_i$が$\mathcal{MRC}$-parallelizableであること利用する。
具体的には
1. reducerを$t$個のブロックに分割する
2. $S_i$を割り当てられたブロックに属するreducerに分配し、$g_i(S_i)$の中間値を計算する
3. 2の結果を1つのreducerに集め、最終的な$h$を計算する

より形式的に定義する。

**Input**  
サブルーチンへの入力は$i \in [k]$について、以下の3種類で構成される。
- $<i; u>$のリスト
  - $u \in S_i$ 
- $g_i$
- $h_i$

**Initialize**  
- $M = n^{1 - \epsilon}$をサブルーチンが利用するreducerの数とする。  
- それをサイズ$B = \Theta(n^\epsilon)$のブロックに分割する。  
- $t = \lceil M / B \rceil$をブロック数とする。  
- univarsal hash functions $hash_1$と$hash_2$を定義する。
  - $[k] \to [t]$

**Map 1**  
各$<i; u>$を$<r; (u; i)>$にmapする。
- $r$はブロック$B_{hash_1(i)}$に属するreducerから一様ランダムに選ぶ

各$g_i$と$h_i$を$<b;(g_i, i)>$と$<b; (h_i, i)>$にmapする。
- $b \in B_{hash_1(i)}$
  - すべてのブロック内のreducerに分配するため
  
**Reduce 1**  
reducerへの入力は以下のような形式になっている  
$<r; ((u_1, i), \dots, (u_k, i), (g_i, i), (h_i, i))>$
- $\{u_1, u_2, \dots, u_k\} = T_j \subseteq S_i$
  - パーティションされた$S_i$の1つのパート

reducerは$g_i(T_j)$を計算し$<r; (g_i(T_j), i, h_i)>$を出力する。

**Map 2**  
$<r; (g_i(T_j), i, h_i)>$を$<hash_2(i); (g_i(T_j), h_i)>$に変換する。

**Reduce 2**  
最後のreducerに対する入力は以下のような形式になっている。  
$<hash_2(i); ((g_i(T_1), h_i), (g_i(T_2), h_i), \dots, (g_i(T_B), h_i))>$  
reducerは$h_i$を計算し、$<hash_2(i); f_i(S_i)>$を出力する。

次の補題によって、Reduce 1でオーバーフローが起きないことが保証される。

Lemma 6.2  
Each reducer in step Reduce 1 will have $\tilde{O}(n^{1-\epsilon})$ elements mapped to it with high probability.

証明の方針
1. $n^\epsilon$個のreducerで構成されるブロック全体に高い確率で$\tilde{O}(n)$個の要素しかマップされないことを示す。
2. ブロック内のどのreducerに個別の要素がマップされるかは一様ランダムに選ばれるので、チェルノフバウンドを利用して補題を示す。

細かい証明は省略します。

Lemma 6.3  
With high probability, each reducer in step Reduce 2 will have at most $n^{1-\epsilon}$ values of $g_i$ mapped to it.

証明には$hash_2$がユニバーサルハッシュ関数であることを利用する。
- 1つのブロックにマップされる要素の期待値は$\frac{k}{t}$

$k \geq t$のとき$\frac{k}{t} \leq n^{1-\epsilon}$であることを利用して、チェルノフバウンドを適用することで、あるブロック$b$に割り当てられる要素の数が多くなる確率を抑える。  
最後にunion boundを取ってオーバーロードになるreducerが存在する確率が低いことを示す。

Lemma 6.3と同じような議論で、reducerは$g_i$と$h_i$を持つのに十分なメモリを持っていることを示せる。
Lemma 6.2とLemma 6.1と$g_i$と$h_i$がpolynomial-time computableである事実からLemma 6.1が証明できる。
