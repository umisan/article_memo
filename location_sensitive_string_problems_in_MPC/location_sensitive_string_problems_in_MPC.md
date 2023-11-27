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
また、LCPQ oracleを用いたsuffix arrayからsuffix treeへの変換アルゴリズムも考案した。
さらに、Adaptive Massively Parallel Computation(AMPC)モデルにおいて、完全なsuffix treeを構築する手法についても提案した。

# 先行研究

# 問題定義

# 結果

# LCPQ Oracleとビルディングブロック

# 論文情報

