# 論文のテーマ
この論文ではLocation-sensitive problemと呼ばれるタイプの問題をMPCで効率的に解くことをテーマにしている。  
Location-sensitive problemの例
- String Matching
- Longest Palindrome Substring (LPS)
- Longest Common Substring (LCS)
- Longest Common Prefix (LCP)

Location-sensitive problemの特徴
- 入力文字列中の連続した文字列から情報をコンパイルする必要がある
- 前の文字及び文字列の順序に依存する

Location-sensitive problemがMPCで難しい理由
- 入力文字列の部分文字列は各マシンのメモリに乗り切らない場合がある
- 文字列を比較する場合ナイーブに実装すると非常に多くのラウンドが必要になる

この論文では、LPSとLCSをLCQクエリを効率的に解くことができるデータ構造LCPQ oracleを使って解くアルゴリズムを提案する。
また、LCPQ oracleを用いたsuffix arrayとsuffix treeの構築アルゴリズムも考案した。
さらに、Adaptive Massively Parallel Computation(AMPC)モデルにおいて、完全なsuffix treeを構築する手法についても提案した。

# 先行研究
これまでLocally Checkable Labelingと呼ばれるグラフの問題はMPCでよく研究されてきた。  
Locally Checkable Labelingの例
- vertex coloring
- edge coloring
- maximal independent set
- maximal matching

先行研究のリンク乗せる

文字列に関する研究もよくされていて、この論文に関連がある問題としてはString Matchingがある。
[Hajiaghayiらの先行研究]()によってString MatchingとInteger Convlutionを$O(1)$ラウンドで解くアルゴリズムが提案されていて、これによってwildcardを含むString Matchingを効率的に解くアルゴリズムも提案されている。
上記の論文で使われているテクニックは、この論文でも利用している。

PRAMでもこれらの問題についてよく研究されている。
関係が深い論文
- [Apostolicoらの論文]()
  - LPSを$O(n^2)$のトータルメモリで解く
  - この論文で使われているアイディアは本研究でも利用している
    - 複合した回分はperiodicである
- [Iliopoulosらの論文]()
  - suffix arrayを$\hat{O}(n)$のトータルメモリと$O(\log n)$ラウンドで構築する
  - この論文のテクニックは本研究のsuffix tree algorithmで利用している

他にもlongest common subsequenceについての[研究]()や、バイナリ文字列のsufix treeの構築に関する[研究]()がある。

# モデル定義
モデルはMPCとAMPCを利用する。

**MPC**
- サイズ$O(n)$の入力がマシンに分配される
- 各マシンは$O(n^{1 - \epsilon})$のメモリを持つ
- 各ラウンドでマシンは任意の計算を行い、ラウンドの最後に相互に通信する

この研究では$0 < \epsilon \leq 0.5$と仮定している。

**AMPC**  
Adaptive MPCと呼ばれるモデル。
基本的にはMPCと同じモデルだが、各マシンはサイズ$O(n)$のshared memoryが利用できることが特徴。
このshared memoryに書き込まれたデータをマシンは必要なときに読み出せる。

# 問題定義
記号の定義
- $\Sigma$
  - アルファベット
- 長さ$n$の文字列$s \in \Sigma^n$
  - $s = s_0s_1,\dots,s_{n-1}$
- $s[l : r)$
  - インデックス$l$からインデックス$r - 1$までの$s$の部分文字列
  - 特に$s[l: n)$のときsuffixと呼ぶ
  - 同様に$s[0: r)$はprefixと呼ぶ


**Longest common prefix queryとは**
- 古典的な文字列の問題を解くために使われる原始的なツール
- 長さ$n$の2つの文字列$s, s'$に対して$\rm{LCP}_{s,s'}(i, j)$は２つのsuffix$s[i;n), s'[j:n)$の共通のprefixの長さ

シーケンシャルな設定では二分探索とsuffixのハッシュ値の比較で$O(\log n)$で解くことができる。
もしくは、$s$#$s'$のsufix treeがあれば$s[i:n)$#$s'$と$s'[j:n)$のlowest common ancestorを求めることで解くことができる。

**longest common substring(LCS)とは**
- ２つの長さ$n$の文字列$s,s'$が与えられたとき、共通する最も長い部分文字列を求める問題

**longest palindrome substring(LPS)とは**
- 入力文字列$s$に含まれる最も長い回文を求める問題
- $\bar{s}$を$s$を反転させたものとすると、回文とは$s = \bar{s}$となる文字列

**suffix treeとは**
- trie-likeなデータ構造
- 各辺は文字列でラベル付けされる
- 文字列$s$のsuffix treeは$s$のすべてのsuffixを含む
- 各ノードは$s$の部分文字列を表す
  - あるノードの文字列は根からノードまでたどる辺の文字列を結合したもの

例の画像入れる

suffix treeはよく２つの文字列$s, s'$を特殊文字#で結合した$s$#$s'$に対して作られる。

**suffix arrayとは**  
compressed suffix treeは複雑なので、suffix treeのかわりとして少し柔軟性を失ったかわりにシンプルなデータ構造として作られたのがsuffix array。
suffix arrayはsuffixをある順列に従ってソートしたもの。

# 結果
この研究では、location-sensitiveな文字列問題を解くための汎用的なフレームワークを提供する。
その核となるデータ構造がLCPQ oracleである。LCPQ oracleは多くのlongest common prefixクエリを同時に処理することができるデータ構造となっている。
多くのlocation-sensitiveな文字列問題は多項式数のLCPクエリに変換できるため、LCPQ oracleを利用することで多くの問題を効率的に解くことができる。

## LCPQ Oracle
２つの文字列$s$と$s'$が与えられたとき、LCP query $q = (i, j)$は$s[i;n)$と$s[j:n)$のLCPを返す。

LCPの計算における問題点
- suffixの長さは最大で$O(n)$になる
  - 1つのマシンのローカルメモリは$O(n^{1 - \epsilon})$であるため、単純には比較できない
- ナイーブに長いsuffixのLCPを計算しようとすると$O(n^\epsilon)$ラウンドが必要になる

これを解決するために以下を利用する
- Block-based Data structures
- hash
- Modular Partitioning

イメージとしては、LCPQ oracleは以下のようなデータ構造になっている。
- 入力文字列をブロックと呼ばれるサイズ$n^\epsilon$の部分文字列で管理
- 各$i$について$s[i, i + n^\epsilon], s[i + n\epsilon, i + 2n\epsilon]$のようにハッシュ値を計算し、1つのマシンに集める


クエリの処理は以下の順序で行う
1. LCPをラフに推測するために$s[i:n)$と$s[j:n)$のブロックを1つのマシンに集める
2. ハッシュ値が一致しなくなるブロックを探す
3. そのようなブロックが見つかったら、オリジナルの文字列を持つマシンに対してクエリを送り正確な解を計算する

ブロック数は$O(n^{1 - \epsilon})$個存在するため、3のステップを1つのマシンで行う場合は$O(n^{2\epsilon})$個のマシンを用意し、それぞれにユニークなブロックのペアをもたせる必要がある。

このようなデータ構造であるLCPQ oracleは$k = O(n^{1 + \epsilon})$個の任意のLCPクエリ$Q = \{q_1, q_2, \dots, q_k\}$に$O(1)$ラウンドで答える事ができる。

LCPQ oracleの構築にはハッシュを利用するため、この研究のアルゴリズムはすべて非決定的である。
また、この研究のアルゴリズムでは、$O(\log n)$個の大きな素数を剰余の計算に利用する必要がある。
このコミュニケーションラウンドを避けるために、この研究ではマシンのメモリ制約に$O(\log n)$を追加する。

また、各マシンは$n^\epsilon$個のハッシュ値を保存する必要があるので、$0 < \epsilon \leq 0.5$という制約も追加する必要がある。

**Lemma 3.1**  
For $\epsilon \in (0, 0.5]$, there is an $O(1)$-round MPC algorithm, with $\tilde{O}(n^{1 + \epsilon})$ total memory and $\tilde{O}(n^{1 - \epsilon})$ memory per machine, which initializes the Longest Common Prefix Query(LCPQ) Oracle in $O(1)$ rounds w.h.p., and then computes a collection of $k = O(n^{1 + \epsilon})$ queries, $Q = \{q_1, q_2, \dots, q_k\}$, in $O(1)$ rounds.

これまで説明しててきた基本的なLCPQ oracleから発展させてcompressed LCPQ oracleというものも提案している。

**Theorem 3.2**
For $\epsilon \in (0, 0.5]$, there is an $O(1)$-round MPC algorithm with $\tilde{O}(n^{1 - \epsilon})$ memory per machine which initializes an LCPQ oracle in $O(1)$ rounds w.h.p., and then processes a collection of $k$ queries, $Q = \{q_1, q_2, \dots, q_k\}$, $O(1)$ rounds. The total memory used by this algorithm is $\tilde{O}(n + k + \min(n, k) \cdot n^\epsilon)$.

table 3を乗せる

## LPSとLCS
LPSとLCSはLCQ queryを使って解くことができる。

**Theorem 3.3**  
For $\epsilon \in (0, 0.5]$, there is an $O(1)$-round MPC algorithm that solves Longest Palindrome SubString(LPS) with $\tilde{O}(n)$ total memory and $\tilde{O}(n^{1 - \epsilon})$ memory per processor, w.h.p.

**Theorem 3.4**
For $\epsilon \in (0, 0.5]$, there is an $O(1)$-round MPC algorithm that solves Longest Common Substring(LCS) with $\tilde{O}(n^{1 + \epsilon})$ total memory and $\tilde{O}(n^{1 - \epsilon})$ memory per processor, w.h.p..

既存のテクニックを使ったアルゴリズムでは、LPSとLCSともに$O(\frac{1}{\epsilon})$ラウンドが必要だったが、これが$O(1)$ラウンドに改善できている。
LPSについてはトータルメモリも$O(n^{1 + \epsilon})$から$\tilde{O}(n)$に改善できている

## Suffix Tree
LCPQ oracleはsuffix arrayの構築にも利用できる。suffix arrayをsuffix treeに変換することができるため、LCPQ oracleを用いてsuffix treeを構築することができる。このsuffix treeの構築は既存の手法とは異なりAMPCを利用するとラウンド数とトータルメモリを削減することができる。

**Theorem 3.5**  
For $\epsilon \in (0, 0.5]$, there is an $O(1/\epsilon)$-round MPC algorithm for computing the suffix array of a given string $s$ with $\tilde{O}(n^{1 + \epsilon})$ total memory and $\tilde{O}(n^{1 - \epsilon})$ memory per processor w.h.p..

**Theorem 3.6**  
(Suffix Tree in AMPC). For $\epsilon \in (0, 1]$, there is an $O(1)$-round AMPC algorithm for computing the suffix tree of a given string $s$ with $\tilde{O}(n)$ total memory and $\tilde{O}(n^{1 - \epsilon})$ memory per processor with high probability.

# ビルディングブロック
ここからこの研究のアルゴリズムで利用しているツールの説明に移る。
- Block-based Data Structures
- Modular Partitioning
- Weighted Load Balancing

また、ここでアルゴリズムで利用しているハッシュ関数についても定義しておく。
$$
hash(s[l:r)) = (\Sigma_{i = l}^{r - 1} s_i \cdot |\Sigma|^{r - 1 - i}) \mod p
$$
$p$は十分に大きな素数

## Block-based Data Structures
Block-based data structuresでは、入力文字列$s$はサイズ$O(n^{1 - \epsilon})$の$n^\epsilon$個の連続するブロックに分割される。(おそらくアルゴリズムによってというよりは、初期の入力として分割して与えられる。)


記号の定義
- ブロック
  - $b_0, b_1, \dots, b_{n^\epsilon - 1}$
- マシン
  - ブロック$b_\alpha$を持つマシンを$\mu_{\alpha}$とする
  
問題に2つ目の入力文字列$s'$がある場合は、ブロックとマシンをそれぞれ$b'_\beta, \mu'_\beta$と表記する。
入力文字列のサイズが異なる場合は、$s$と$s'$に含まれない特殊な文字列で調整する。

あるマシン$\mu_\alpha$は以下のブロックを持つとする
- $b_{\alpha - 1}$
- $b_\alpha$
- $b_{\alpha + 1}$

すなわち、自分のブロックの前後のブロックにもアクセスすることができる。

**Definition 4.1**  
(Block $b_\alpha$ and Block Machine $\mu_\alpha$). There is a collection of $n^\epsilon$ machines $\{\mu_0, \mu_1, \dots, \mu_{n^\epsilon - 1}\}$ refered to as block machine for string $s$ (and $\{\mu'_0, \mu'_1, \dots, \mu'_{n^\epsilon - 1}\}$ for string $s'$), where $\mu_\alpha$ (and $\mu'_\beta$) contains the substring of the $\alpha$-th block of $s$ (and the $\beta$-th block $s'$) denoted by $b_\alpha = s[\alpha n^{1 - \epsilon}: (\alpha + 1)n^{1 - \epsilon})$.

このデータ構造は以下のように利用される。
- ある関数$f$をすべてのブロックに適用する
  - $f$はブロックから何らかの情報を抽出する関数
- $f$で抽出した情報を1つのマシンに集める
  - $\epsilon \leq 0.5$なのでこれが可能

$f$には以下のようなマージ関数$g$が存在する必要がある。
- $f(s) = g(f(s[0:i]), f(s[i:n]), i, n)$

また、各マシンは自身のブロックについてセグメント木を構成し、$f(b_\alpha[l:r])$を高速に計算できるようにしておく。
これによって、$O(n)$個の$f(s[l:r])$クエリを2ラウンドで計算することができる。

## Modular Partitioning
長さ$n$の配列のModular Partitioningは、配列の要素を$n^\epsilon$の剰余に分割すること。

Block-based data structuresで分配されたブロック$b_\alpha$の部分文字列のハッシュ値を計算する場合を例に考える。  

- 各$\alpha n^\epsilon \leq i \leq \min\{(\alpha + 1)n^\epsilon, n\}$について$hash(s[i: i + n^\epsilon])$を計算する
- 各$0 \leq x < n^\epsilon$についてマシン$\lambda_x$を用意し、$i \mod n^\epsilon = x$となる部分文字列のハッシュを割り当てる

これによって、1つのマシンで任意の$i$から始まる部分文字列についての計算を行えるようになる。

Figure3入れる

**Definition 4.2**  
We denote a collection of $n^\epsilon$ mod machines $\{\lambda_0, \lambda_1, \lambda_{n^\epsilon - 1}\}$ for string $s$ (and $\{\lambda'_0, \lambda'_1, \dots, \lambda'_{n^\epsilon - 1}\}$ for string $s'$), where $\lambda_x$ (and $\lambda'_x$) contains the hash of substrings of length $O(n^\epsilon)$ starting at indices $i$ so that $i \mod n^\epsilon = x$.


## Weighted Load Balancing
この研究のアルゴリズムでは多くのクエリを並列に処理する必要がある。
このクエリを適切にマシンに分配する必要がある。
Weighted load balancingはメモリ制約に違反しないように、データを割り当てられたクエリに比例して配布する2-roundアルゴリズムである。

アルゴリズム1を貼る

# 論文情報
[Location-Sensitive String Problems in MPC](https://dl.acm.org/doi/pdf/10.1145/3558481.3591090)

