# 論文のテーマ
並列でのソーティングには以下の2種類の大きな種類がある
- partition-based sorting algorithms
- merge-based sorting algorithms

partition-based sorting algorithmsでは、元データを均等に分割するパーティションを決定し、そのパーティションに基づいてデータを再配布することによって、ソートを行う。
partition-based sorting algorithmsには、merge-basedに比べてコミュニケーションコストが低いため、現代のアーキテクチャに向いているという利点がある。
既存の最先端の並列ソーティングのアルゴリズムでは、samplingとhistogramingというテクニックが利用されている。
この論文では、samplingとiterative histogramingを組み合わせてpartitionを探索するHistogram sort with sampling(HSS)を提案している。
このアルゴリズムは、既存の最も良いアルゴリズムに対して、$\Theta(\log(p)/\log\log(p))$分の1のコミュニケーションコストでpartitionを発見することができる。

# 問題定義
- $p$
    - プロセッサー数
- $N$
    - 入力シーケンスの長さ
- $A(i) 0 \leq i \leq N - 1$
    - 入力シーケンスのi番目のキー
- $I(i) 0 \leq i \leq N - 1$
    - 全体の順序におけるi番目のキー
    - $A(j) = I(r)$の場合、ランク$r$を持つとする

各プロセッサー$p$は$N / p$個のキーを持つ。  
注意点
- 入力シーケンスに重複はないと仮定する。重複があるケースの変換は少ないオーバーヘッドで可能
- 初期状態でキーが均等に分割されていないケースにも証明とアルゴリズムを変換することができる

$i$番目のプロセッサーはソート済みのシーケンス$I(0), \dots, I(N - 1)$の$i$番目のサブシーケンスを持つようにキーを再分配することがアルゴリズムのゴール


並列ソーティングでは、ソート後のキーの分布についてバランスが取れていることを求められるのが一般的である。
バランスについての制約には以下の2種類が存在する
- locally balanced
- grobally balanced

locally balancedでは、各プロセッサーは$(1 + \epsilon)N/p$より多くのキーを持たない。  
globally balancedは、locally balancedよりも強い制約で以下の条件を各プロセッサーが満たす必要がある。  
$$
S(i) = I(\chi(i))\ with \\
\chi \in T_i \\
T_i = [N_i / p - N\epsilon / 2p, N_i / p + N\epsilon / 2p]
$$
このときプロセッサー$i$は、$S(i)$以上かつ$S(i + 1)$よりも小さいすべてのキーを持っていなければいけない。
この論文のアルゴリズムはglobally balanced distributionを達成する。  繰り返しアルゴリズムにおけるglobally balanced distributionの利点は、初期の分布がほぼソートされていてかつglobal balancedの場合、データの交換ステップのコミュニケーションが少ないこと

先行研究[12](https://math.mit.edu/~edelman/publications/scalable_parallel.pdf)で、どちらの分布のもとでもexact splittingが得られることは証明されている。しかし、locally balancedの場合は、自身が持っているデータすべてを1つか2つのプロセッサーと交換しなければならないプロセッサがー存在するかもしれないのに対して、globally balancedの場合は、各プロセッサーにおいて交換が必要なキーの数はたかだか$N\epsilon/p$となる。よって、根本的な違いとしては、アルゴリズムの全実行時間においてload balanceが維持されているかになる。

この研究では、partition-based sorting algorithmにおけるdata-partitionステップに注力する。
partition-based sorting algorithmには以下が存在する
- Sample sort by regular sampling
- histogram sort
- sample sort by random sampling
- parallel sorting by over paritioning
- AMS sort
- HyKsort (今一番速いらしい)


# 先行研究
## Sample sort
スタンダードなよく研究されていてる並列ソートアルゴリズム  
各プロセッサーで$s$個のkeyをサンプリングし、1つのプロセッサーにすべてのサンプルを集める。サンプルを集めたプロセッサーは、サンプルをから$p - 1$個のキーをsplitterとして選択する。
- $s$
    - 各プロセッサーがサンプルするkeyの数
- $M$
    - 全サンプル数
    - $M = ps$
- $\Lambda = \{\lambda_0, \lambda_1, \dots, \lambda_{ps - 1}\}$
    - 全サンプルを結合しソートしたもの

一般的にSample sortは下記の3つのフェーズで構成される
1. Sampling Phase
2. Splitter Determination
    - サンプルを受け取ったプロセッサーが$\Lambda$を均等に分割する$p - 1$個のsplitterを決定する
    - splitterの集合$S = \{S(1), S(2), \dots, S(p - 1)\}$
    - splitterが決定されると、すべてのプロセッサーにブロードキャストされる
3. Data Exchange
    - splitterを受け取ったら、各プロセッサーはkeyを最終的に配置されるべきプロセッサーに送信する
    - プロセッサー$i$は$\lbrack S(i), S(i + 1))$に含まれるkeyを持つ
    - 交換が完了したら直列のアルゴリズムでローカルにソートする

## Sample sort: Sampling methods
Sample sortで利用される2種類のsampling methodsについて取り上げる。
- random sampling
- regular sampling

### Random sampling
random samplingでは、各プロセッサーは自身が持つソート済みの入力を$s$個のブロックに分割し、各ブロックから1つランダムにキーを選択する。$s$はoversampling ratioと呼ばれる。
splitterは集めたサンプルを均等に分割するように選択される。
[先行研究](https://link.springer.com/article/10.1007/s002240000083)で以下の定理が示されている

Theorem 3.1  
With $O(p\log N/\epsilon ^2)$ samples overall, sample sort with random sampling achives $(1 + \epsilon)$ load balance w.h.p..

### Regular sampling
regular samplingでは、各プロセッサーは自身が持つソート済みの入力を均等に分割する$s$個のキーを計算し、サンプルとする。そのサンプルを1つのプロセッサーに集め再度splitterを計算する。これについても先行研究[1](https://www.sciencedirect.com/science/article/abs/pii/016781919390019H),[2](https://www.sciencedirect.com/science/article/abs/pii/074373159290075X)が存在し、以下の定理が示されている

Theorem 3.2
If $s = p / \epsilon$ is the oversampling ratio, then sample sort with regular sampling achieves $(1 + \epsilon)$ load balance.

samplingサイズの大きさから、regular samplingではsampling phaseはスケールしない。random samplingはより効率的だが、それでもload-balanced splittingを達成するためには大きなサンプルサイズが必要になるため、こちらもスケーラビリティは低い。

## Histogram Sort
histogram sortは、splittersを決定するために大きなサイズのサンプルを利用するのではなく、splitterが存在するキーの区間を管理する。アルゴリズムの実行を繰り返すことで、この区間の範囲を狭めていき、全splitterの区間が所定の閾値よりも小さくなったら、splitterを確定する。データの交換フェーズはsample sortと同じ。

histogram sortの流れ
1. セントラルプロセッサーはM個のソートされたキーで構成されたprobeを全プロセッサーに配布する
    - 通常初期のprobeはkeyの区間を均等に分割するように作成される
2. 各プロセッサーはprobeのキーによって作られる区間に自身の入力が何個含まれるかをカウントする
    - これはローカルのヒストグラムとなる
3. セントラルプロセッサーでローカルヒストグラムを足し合わせて、グローバルヒストグラムを計算する
4. もし、$p - 1$個のsplitterに対する十分小さい区間が見つかったら、セントラルプロセッサーはsplitterを確定し、ブロードキャストする。それ以外の場合は、ヒストグラムを利用してprobeを更新し配布し、splitterを決定するまで繰り返す

probeの改良は、ヒストグラムで隣接する区間を分割することで行われる。
Histogram sortは、任意のレベルのload balanceを実現することができ、各ラウンド使用するヒストグラムのサイズは$O(p)$程度となっており比較的小さいため、スケールさせることができる。しかし、ラウンド数は大きくなる可能性があり、特にデータの分布に偏りがある場合に顕著になる。$Z$を入力の範囲とすると、ラウンド数はたかだか$\log(Z)$となる。Histogram sortは実際に超並列の科学アプリケーションで利用されている。例 [ChaNGa](https://ieeexplore.ieee.org/document/5470406)

先行研究
(1)[https://ieeexplore.ieee.org/document/4134268]  
(2)[https://ieeexplore.ieee.org/document/5470406]

## Large scale parallel sorting algorithms
[HykSort](https://users.flatironinstitute.org/~dmalhotra/files/pubs/ics13sort.pdf)は直近で最も優れた実践的なアルゴリズム。
HykSortはsample sortとhypercube quick sortのハイブリッドになっている。しかし、HykSortがsplitterの決定に使っているsamplingとhistogrammingの手法はHSSとは異なっている。HykSortはHSSと比較してワーストケースで少なくとも$\Omega(\log (p) / \log^2\log (p))$倍の実行時間になる。実際にこの論文では、HSSとHykSortのsamplingをHSSのアルゴリズムに置き換えたものを実装して、収束が速いことを確認している。

[AMS-sort](https://dl.acm.org/doi/10.1145/2755573.2755595)はoverpartitioningをsamplingに使っているアルゴリズム。
AMS-sortがsplitterの決定に使っているscanning algorithmはhistogrammingを1ラウンドしかしなかった場合のHSSより優れている。
しかしhistogrammingを複数回繰り返すと、HSSのほうがより効率的になる。AMS-sortのscanning algorithmはマルチラウンドに拡張しづらいものになっている。また、AMS-sortはlocally-balancedしか達成しないがHSSはglobally-balancedを達成できる。

## Single stage AMS sort
AMS-sortの流れ

1. サンプルを収集
2. 1ラウンドのhistogrammingを実行
3. ヒストグラムに基づいてlocally balancedなsplittingを行う

アルゴリズムは高確率でlocally balancedなsplitterを決定することができ、oversampling parameterはrandom samplingのsample sortより小さい。

5章で示す通り、histogrammingのコストは同じ数のkeyをsamplingするコストと漸近的に等しいため、AMS-sortは理論的にsample sortよりも優れている。

### Scanning algorithm
AMS-sortではヒストグラムを計算したあとにscanning techniqueを用いてsplitterを決定する。このアルゴリズムはヒストグラムをscanしていって、可能な限り最大のキーを順番にプロセッサーに割り当てていく。（実際にはヒストグラムのバケットを割り当てていく）これによって、各プロセッサーのロードは$N(1 + \epsilon)/p$を超えないようにする。サンプルサイズが十分に大きければ、最初の$p - 1$個のプロセッサーの平均のロードは高い確率で$N/p$より大きくなる。

locally balanced partitioningを達成するためにはサンプルサイズは$\Theta(p(\log p + 1/\epsilon))$必要になる。

![table2](https://github.com/umisan/myimages/blob/main/table2.PNG)


# HISTOGRAM SORT WITH SAMPLING
HSSはsampling phaseとhistogramming phaseを交互に行う。これによって、sample sortに比べてサンプリングサイズを抑えることができる

## HSS with one histogram round
最初に1ラウンドのhistogrammingを行うHSSについて考える。
この方式はAMS-sortよりも若干効率が悪くなっている。（$\Theta(\min(\log p, 1 / \epsilon))$倍になる）
しかし、HSSは1ラウンドのhistogrammingをマルチラウンドに拡張することによって、globally balanced partitionを達成することができる。AMS-sortのscanning algorithmをマルチラウンドに拡張することは簡単ではない。
マルチラウンドのHSSはAMS-sortより効率的で、$O(1)$回のhistogrammingで漸近的な改善には十分となる。

HSSでは、各ターゲットレンジ$T_i = [Ni/p - N\epsilon/2p, Ni/p + N\epsilon/2p]$に含まれるsplitterの候補を見つけることが目的となる。
もし各$T_i$について、$T_i$にランクが含まれるキーが少なくとも1つサンプリングされれば、すべてのsplitterを決定することができる。直感的には、十分に大きなサンプルサイズであれば、このようなことが高確率で起こると考えられる。

Lemma 4.1  
If every key is independently picked in the sample with probability, $ps/N = (2p\ln p)/\epsilon N$, where s, the oversampling ration is choesn to be $(2\ln p)/\epsilon$, then at least one key is chosen with rank in $T_i$ for each i, w.h.p..

証明の方針  
$T_i$について、1つもキーがサンプリングされない確率を上から抑える  
$(1 - ps/N)^{|T_i|} \leq 1/p^2$  
splitterは$p - 1$個あるので、キーがサンプリングされない$T_i$が存在する確率は  
$(p - 1) \times p^{-2} < 1/p$

Lemma 4.1からTheorem 4.2を導出できる。
このTheoremはマルチラウンドのHSSの解析にも利用できる。

Theorem 4.2  
With one round of histogramming and sample size $O({p \log(p)} / \epsilon)$, HSS achives $(1 + \epsilon )$ load balance w.h.p..

## HSS with multiple rounds
ここからはHSSはラウンドを複数にすることでより効率的になることを示す。

### Sampling method
HSSのsampling phaseでは、サンプルを入力のサブセット$\gamma$から選択する。初期状態では$\gamma$は入力全体となる。ラウンドが進むごとに$\gamma$は小さくなっていく。

HSSでのサンプリングでは、確率$ps/N$で$\gamma$の各キーが独立にサンプリングされる。（これによって解析が簡単になる。）
$s$はsampling ratioと呼ぶ。これまで出てきたoversampling ratioとは異なる概念のため注意。サンプルサイズの期待値は$(ps |\gamma|/N)$となる。（oversampling ratioの場合は、バケットごとにサンプリングされる）

### HSS with k histogramming rounds: Algorithm
アルゴリズムの流れ
1. 1ラウンド目のsampling phaseでは、入力の全キーが確率$ps_1/N$でサンプリングされる。$s_i$は初期のsampling ratio。サンプルはセントラルプロセッサーに集められ、その後probeがhistogramming phaseのために配布される
2. 各プロセッサーはローカルヒストグラムを計算し、それを足し合わせグローバルのヒストグラムを計算する。
3. 各splitter $i$について、セントラルプロセッサーは$L_j(i)$と$U_j(i)$を管理し、splitter intervalを各プロセッサーに配布する
    - $L_j(i)$はjラウンド後に$Ni/p$に最も下から近かったキーのランク
    - $U_j(i)$はjラウンド後に$Ni/p$に最も上から近かったキーのランク
    - $I_j(i) = [I(L_j(i)), I(U_j(i))]$をsplitter intervalと呼ぶ
4. 各プロセッサーは新しいsplitter intervalを受け取ると、splitter intervalの中に存在するキーを確率$ps_{j + 1}/N$でサンプリングする。$s_{j + 1}$はj + 1ラウンドのsampling ratio
    - $j = k$になるまでこれを繰り返す
5. histogramming phaseが終了したら、サンプルの中から最も$Ni/p$に近いキーを$i$番目のsplitterにする

figure 1入れる

HykSortのサンプリングとHSSのサンプリングの違いは、HykSortはすべてのsplitter intervalから均等にサンプルするのに対して、HSSはintervalの長さに比例してサンプリングすること。これによって、HSSはintervalの縮小が速い。

アルゴリズムの進展によってsplitter intervalは縮小していくので、サンプリングの対象は毎回小さくなっていく。
$\gamma_j$を$j$ラウンド後にいずれかのsplitter intervalに含まれるキーの集合とする。すると以下のようにサイズを上から抑えることができる

$|\gamma_j| \leq \Sigma_i U_j(i) - L_j(i)$

不等式なのは重複があるから。
しかし、splitter intervalの選び方から、部分的な重複はなく、重複がないか一致しているかのどちらかしかない。

HSSの性能の証明の流れ
1. 最後のラウンドでsampling ratioが十分に大きければ良いsplittingを達成することの証明
2. 各ラウンドのサンプルサイズを上から抑える
3. サンプルサイズが定数分の1に減少していくようにsampling ratioを設定する

Lemma 4.3  
if $s_k = \frac{2\ln p}{\epsilon}$ be the sampling ratio for the $k^{th}$ round, then at least one key is chosen from each $T_i$ after k rounds w.h.p..

これが証明の流れの1にあたる補題  
これはLemma 4.1を$s_j = \frac{2p \ln p}{\epsilon}$として適用すれば即座に求まる。  

Lemma 4.4  
Let $s_j$ be the sampling ratio for the $j^{th}$ round, $I_j(i)$ be the splitter interval for the $i^{th}$ splitter after $j$ rounds and $\gamma_j$ denote the set of input keys that lie in one of the $I_j$'s, then $E(|\gamma_j|) \le \frac{2N}{s_j}$.

証明の流れの2のための準備1  
この補題は、$L_j(i)$と$U_j(i)$が毎ラウンド改善されていくことを利用して、$E[U_j(i) - \frac{Ni}{p}]$と$E[L_j(i) - \frac{Ni}{p}]$を上から抑えることで証明される。$U_j(i) - \frac{Ni}{p}$は、$j$ラウンド目のsplitter intervalの上半分。上半分と下半分を上から抑えることで全体のサイズを上から抑える。  

Lemma4.5  
If $s_j < \sqrt{\frac{2p}{\ln p}}$, then $|\gamma_j| \leq \frac{4N}{s_j}$ w.h.p..

証明の流れの2のための準備2  
Lemma 4.4では期待値だったが、この補題で高確率で$\gamma_j$のサイズが小さいことを示す。  
この補題の証明では、新しく$U_j'(i)$と$L_j'(i)$を定義する。  

$U_j'(i) = \min (\frac{N(i + 1)}{p}, U_j(i)), L_j'(i) = \max (\frac{N(i - 1)}{p}, L_j(i))$

これによって、2つのsplitter intervalで重複していた部分を分割し、個別に扱えるようにする。($|\gamma_j$|は変わらないことに注意)  
これを使って

$P[\Sigma_i U_j'(i) - \frac{Ni}{p} > \frac{2N}{s_j}] \leq \frac{1}{p^2}$

を示す。($L_j'$側も同じ手順で示す)  
よって

$|\gamma_j| \leq \Sigma_i (U_j'(i) - \frac{Ni}{p}) + (\frac{Ni}{p} - L_j'(i)) \leq \frac{4N}{s_j}$ w.h.p.

が証明できる。

Lemma 4.6  
Let $Z_j$ be the sample size for the $j^{th}$ round and $s_j \ge s_{j - 1}$, then $Z_j \leq (5ps_j/s_{j - 1})$ w.h.p..

証明の流れの2にあたる補題  
この証明によって、各ラウンドのサンプルサイズを上から抑え、またsampling ratioの比にしたがってサンプルサイズが減少していくことを示せる。  
この補題の証明は以下の事実とチェルノフバウンドを利用する。
- $E[Z_j] = |\gamma_{j - 1}|ps_j/N$
- $|\gamma_{j-1}| \leq 4N/s_{j - 1}$

必要な補題は揃ったので、ここからはアルゴリズムが所望のload balancingを達成して停止させるために、どのようにsampling ratioを設定すればよいかを考える。

$k$ラウンドのHSSで、$j^{th}$ラウンドのsampling ratioを以下のように設定する。

$s_j = (2 \ln p/\epsilon)^{j/k}$

すると、Lemma 4.3より、$k$ラウンド目で全splitterは高確率で求まる。  

Lemma 4.3  
if $s_k = \frac{2\ln p}{\epsilon}$ be the sampling ratio for the $k^{th}$ round, then at least one key is chosen from each $T_i$ after k rounds w.h.p..

Lemma 4.5より、$|\gamma_j|$は、$4N(\epsilon/2 \ln p)^{1/k}$より小さい。

Lemma4.5  
If $s_j < \sqrt{\frac{2p}{\ln p}}$, then $|\gamma_j| \leq \frac{4N}{s_j}$ w.h.p..

さらに、Lemma 4.6より、$j^{th}$ラウンドでのサンプルサイズは高々$5p(2 \ln p/\epsilon)^{1/k}$

Lemma 4.6  
Let $Z_j$ be the sample size for the $j^{th}$ round and $s_j \ge s_{j - 1}$, then $Z_j \leq (5ps_j/s_{j - 1})$ w.h.p..

よってTheorem 4.7が成り立つ

Theorem 4.7  
With $k$ rounds of histogramming and a sample size of $O(p\sqrt[k]{\frac{\log p}{\epsilon}})$ per round, HSS achives $(1 + \epsilon)$ load balance w.h.p.. for large enough $p$.

Theorem 4.7より、サンプルサイズとラウンド数にはトレードオフがあることがわかる。全ラウンドを通してのサンプルサイズを最小化するには、$kp\sqrt[k]{\log p/\epsilon}$を$k$に関して微分し、0をセットする。

$$
\frac{d(kp\sqrt[k]{\log p/\epsilon})}{dk} = 0\\
\Rightarrow k = \log \frac{\log p}{\epsilon}
$$

これによって、Theorem 4.8が成り立つ

Theorem 4.8  
With $k = O(\log(\log p / \epsilon))$ rounds of histogramming and $O(p)$ samples per round ($O(1)$ from each processor), HSS achives $(1 + \epsilon)$ load balance w.h.p.. for large enough p.

$\epsilon = p/N$にするとexact splittingを得られる。

Threorem 4.9  
HSS with $O(p)$ samples per round overall cant achive exact splliting in $O(\log N/p + \log\log p)$ rounds.

# RUNNING TIMES
この論文では、並列計算のモデルにBSPを利用する。
BSPのアルゴリズムはsuperstepで構成される。
1つのsuperstepの中で、各プロセッサーは自身が持つデータを使ってローカルに計算を行い、データを送受信することができる。
superstepは同期のタイミングとなっている。

BSPのアルゴリズムの複雑性は以下の3つの要素から成り立っている  
1. superstepの数 (同期コスト)
2. 1つのsuperstepの中で行われる最大の計算量を全superstepについて合計したもの (計算コスト)
3. 1つのsuperstepの中で送受信される最大のデータ量を全superstepについて合計したもの (コミュニケーションコスト)

この研究では、プロセッサーは各superstepにおいて$p$個のメッセージを送受信できるとする。
このモデル上で、Histogramming roundが増えるとsuperstep数は増えるが、コミュニケーションコストは低くなることを示す。

実行時間の解析では、sample sortとHSSを比較する。
この2つのアルゴリズムでは以下のコストは共通している。
- 初期ソート
    - $O((N/p)\log \frac{N}{P})$
- splitterの配布
    - $O(p)$
- データ交換
    - $O(N/p)$

データ交換後のマージのコストは$O((N/p) \log p)$

## Cost of Sampling
サイズ$S$のサンプルを1つのプロセッサーに集めるとすると、$O(S)$のコミュニケーションコストのsuperstepが必要になる。
random samplingでは、各プロセッサーは$S/p$個のキーをサンプルし、それを1つのプロセッサーに集める。送信前に各プロセッサーでサンプルがソートされていれば、マージステップの計算量は$O(S \log p)$になる。

## Cost of Histogramming
ローカルヒストグラムの計算量
- $O(S \log (N/p))$
    - $S$回二分探索する

グローバルヒストグラムは$O(S)$のコミュニケーションコストと計算量を持つ2回のsuperstepで計算できる
- reduce-scatter
- all-gather


probeがヒストグラムの作成のためにブロードキャストされる。長さが$S$のメッセージの送信のコミュニケーションコストは$O(S)$。
よって、サンプリングフェーズのコミュニケーションコストと計算コストはともにサンプルサイズに比例する。

![table2](https://github.com/umisan/myimages/blob/main/table2.PNG)

ベストな設定のHSSは他のアルゴリズムよりも複雑度の面で勝っている。