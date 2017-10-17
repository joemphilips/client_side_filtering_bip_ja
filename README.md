---
categories: ["Develop with pleasure!", "cryptocurrency","translation"]
date: 2017-10-16T12:39:35+09:00
# description: "filter header chain by roasbeef"
draft: false
tags: ["bitcoin"]
title: "既存のSPVウォレットの問題点を解消するフィルタヘッダーチェーンについて"
---

lightninglabsから[neutrino](https://github.com/lightninglabs/neutrino)という新しい
ライトクライアントが出てきたので、その仕組みの元になったroasbeefの
[pre-BIP](https://github.com/Roasbeef/bips/blob/master/gcs_light_client.mediawiki)
を訳す。レイヤー２対応のライトクライアントとして都合が良い仕組みにすることが目標らしい。
まだBIPになっていないので、変更はあるかもしれない。随時対応していく所存。


## アブストラクト

このBIPではBitcoinの新しいライトクラインアントノードと、それに対応した既存のフルノード
への機能追加を提案する。このBIPで説明されるライトクライアントモードは、プライバシー、可用性、
フルノードの要求するリソースという点でBIP37[^1]の上位互換となることを意図したものである。
デザインは概ねBIP37を「反転」させたものになっている。すなわちライトクライアントがフルノード
に対してフィルターを送信するのではなく、フルノードがライトクライアントにフィルターを送る。
また、BIP37とは違いbloom filterを使用せず、代わりにゴロム・ライス符号を圧縮に利用した
より効率的なコンパクトフィルターを使用する。さらに、ブロックは任意のソースから
（マークルブランチによるマークル証明という形ではなく）丸ごとダウンロードされる。

## モチベーション

Bitcoinのライトクライアントとは、ブロックチェーン全体の検証を行わず、
1) 最も多くのディフィカルティを持つチェーンの検証及び 2) クライアント自身に関係のある
ブロックチェーン内のログのみの検証を行うアプリケーションのことを指す。
前者の達成のため、ライトクライアントはブロックヘッダをダウンロードし、
そのworkと接続の有効性**のみ**を検証する。ブロックヘッダは80バイトの固定長であるため、
非常に長いブロックチェーンにおいても使用する帯域幅が少なくて済む。
後者の目的の達成のため、ライトクライアントはブロック内の自分に関係のあるデータを知る方法
を必要とする。

BIP37は現時点で最も広く使われているライトクライアントの実行方式である。BIP37では
ブロックチェーン全体の検証の代わりにブロックヘッダの有効性を確認し、関係のあるデータに対応
したbloom filterを作成してフルノードに送る。受け取ったフルノードはこのフィルターを用いて
ブロックチェーン内にクエリをかけ、マッチしたトランザクションが存在した場合は、
そのトランザクションと対応するマークル証明をライトクライアントに対して送る。
 
しかしながら、このBIP37にはいくつかの欠点が存在する。この仕組みを用いてwalletなどの
アプリケーションを作成した場合、実質的に提供されるプライバシーは0である[^3],[^4]。
加えて、アプリケーション側は興味のあるデータのリストが完全に明らかにならないようにするため、
偽陽性率を慎重にコントロールしなくてはならない。さらに、フルノード側は嘘の情報を流すことが
可能であり、その検出はほぼ不可能と言って良い。これによりフルノードはDoS攻撃を引き起こす
ことが可能で、ライトクライアント側のセキュリティが特定のオンチェーンイベントに依存する場合、
サービスに悪影響を及ぼす。

> 訳注: Lightning Networkの場合、一時的にPenalty Transactionが発行できなくなるため悪影響が大きい。

また、ライトクライアント側が悪意のあるフィルターを送信した場合、フルノード側はI/OとCPUを
大きく消費し、これもまたDoSエラーにつながる可能性がある。

## Design Rationale

上記BIP37の欠点を継承するため、本文書ではチェーンのフィルタリング方式の代替案を提唱する。
ここではフィルタリングをクライアントサイドで行うことで、プライバシーレベルの向上が達成
されていることを示す。クライアントは決定的に作成されたフィルターを用いてブロックを
ダウンロードし、ローカルでクエリをかける。結果に自身に関係のある要素が含まれていた場合、
ブロック全体をダウンロードする。フルノードとのアクティブな通信と、フィルタリングとの結合関係
を疎にすることで、ブロックのダウンロードを**どこからでも**行えるようになる。
さらにより高いプライバシーレベルを必要とするクライアントは、Private information retrieval[^5]
といった暗号学的テクニックを用いることでブロックの取得自体を匿名化することも可能である。

フィルターのサイズを削減するため、確率的要素集合(probablistic set membership)を扱える
データ構造を用いる。ゴロム・ライス符号は、理論上最小の確率的データ構造を可能にし、
bloom filterよりもさらにコンパクトなフィルタを作成できるため、これを用いる。

この方式を用いたライトクライアントは、受け取ったフィルターの正当性を確認できるため、
フルノードの嘘を検出することができる。これによってクライアント側は単純なwalletや
公開鍵のよくある使い方を超えたジェネリックなアプリケーションを作成することが可能となる。

## 前提

プロトコルの詳細な説明に入る前に、理解を容易にするための前提を多数述べる。

`[]byte` はバイト列のスライスないしは配列である。これはCなどの典型的プログラミング言語では
`uint_8` の配列として表されるものである。

`Var-Int` はBitcoinのp2pプロトコル中で広く使われる、整数型の可変長エンコード方式であり、
複数の要素を要素の繰り返しにエンコードする際に効率的な方法である。本提案におけるp2pの
メッセージング方式においては既存のbitconのp2pネットワークと同様の方式でこの
可変長整数エンコードを利用する。

`siphash(k, n)` は `SipHash` 擬似ランダム関数(Pseudo Random Function: PRF)の呼び出しを指す。
`k` は128ビットのキーで、 `n` はPRFへのインプットである。 PRFのインスタンス化は推奨パラメータ
`c = 2` 、 `d = 4` で行うものとする。

`new_bit_stream` は抽象ビットストリームのインスタンス化を行う関数とする。
作成された `bit_stream` は `unary_encode(stream, n)` と `write_bits_big_endian(stream, n, k)`
kという関数の引数となる。前者はストリームに整数 `n` を単項(unary)としてemitし、後者は `n` の
下位 `k` bitをビッグエンディアンのバイナリエンコーディングとしてemitする。


## 仕様

### コンパクトチェーンフィルター

このBIPでは、ブロックの内容を簡潔にエンコードするためのコンパクトフィルターを
ライトクライアントに搭載することを提案する。bloom filterの代わりに、ブロックの構成要素の
ハッシュを圧縮したデータ構造を用いる。

以下のセクションでは、映像・音声データの圧縮によく使用されるテクニックを借用しながら、
本提案に関連のあるデータのハッシュのエンコーディング方式について解説する。集合内のデータを
(確率的要素集合の保持を可能なデータ構造の作成のため)非可逆に圧縮する際にはゴロム・ライス符号
を用い、集合内の連続するハッシュ要素の**デルタ**(差分)をエンコードする。これにより非常に
コンパクトな確率的要素集合のデータ構造が作成できる。

抽象化レベルの低いところから説明を初めていく。これらは全て当初説明した最終目標のための
原理的構成要素であることを念頭に置きながら読み進めてほしい。

### 連長圧縮(RLE)

連長圧縮(Run-Length Encoding、RLE)とは画像・映像の圧縮という分野において、画像単位・フレーム
単位の可逆圧縮によく用いられる方法である。データストリーム中の同じ値の繰り返しを避けることで
圧縮を達成する。繰り返し部分を無視するのではなく、繰り返しの**回数**を表した数字を入れることで
可逆圧縮を可能にする。

典型的なRLEではバイナリストリーム中の繰り返しを対象とした圧縮を行う。単純なものは以下の
ような手続きになる。

* 0のランレングス(連続出現数)を `k` ビットでエンコードする。
 * `k` は固定長のエンコード領域として機能する。
 * `k` はエンコード可能な最大のランレングスを表す。
* 1はエンコードしない。
* 1が連続した場合、長さ0の0の列があるとみなす。

例として、以下のビット列を

```
{0}^14 1 {0}^9 11 {0}^20 1 {0}^30 11 {0}^11
```

RLEエンコードすると以下のようになる。

```
1110 1001 0000 1111 0101 1111 1111 0000 0000 1011
```

RLEは効率的な可逆圧縮方式の一つである。ランレングスのエンコード先が固定長であるため、
効率性が犠牲になっていることに聡明な読者は気づいたかもしれない。ここで、固定長ではなく可変長
でエンコードすることが可能である。そうすることにより非常に長いランレングスの圧縮が可能になる。
そのためにゴロム・ライス符号を用いる。

### ゴロム・ライス符号

RLEは冗長性の高いデータストリームに対しては効率的なエンコードを可能にするが、
コンパクトフィルタ内のデータはハッシュ化されているため、一様分布であることが期待される。
ランレングス内の分布が既知である場合、より効率的な圧縮方法の検討が可能である。

[幾何分布](https://ja.wikipedia.org/wiki/%E5%B9%BE%E4%BD%95%E5%88%86%E5%B8%83)とは、
ベルヌーイ試行(試行結果が２通りの観測行為)を繰り返した際に、初めの成功が観測されるまでの
失敗の回数の分布を表したものである。
対象となる値がランレングス `r` において i.i.d(互いに独立な一様分布)である場合、以下のような分布となる。
(Golombによる原論文[^6]を参照)

```
P(r = n) 0 P^n * (1-p)
```

直感的な説明をすると、これはゼロがN個続いた後に1が観測される確率である。ゴロム符号は整数を
二つの値でエンコードするためにこの関係性を利用する。 グループサイズ `m` をパラメータとして
与えると整数を以下のように符号化できる。

```
n = (q*m) + r
  where q is (n / m)
    and r is n % m
```

[ゴロム符号](https://en.wikipedia.org/wiki/Golomb_coding)は与えられた整数 `n` を `q` , `m`
及び `r` からなる二要素のタプルとしてエンコードする。 `q` は単項(unary)符号化され、 `r` は
固定長のbit列でエンコードされる。 `m` が何らかの
`k` に対して `m=2k` を満たすとき、このエンコード方式はゴロム符号の特殊例である
ゴロム・ライス符号と呼ばれるものになる。この例では `r` (剰余)は `n` の下位 `k` ビットとなる。

> 訳注: `q` はquotient(商)の頭文字

この場合、「ラン」とは `n` に収まる `m` の数であると考えることができる。 `m` と




ではゴロム・ライス符号によるエンコード・デコードを行うシンプルな関数を定義する。これらの
関数は次以降のセクションでコンパクトフィルタを作成するための材料となる。

```
golomb_encode(stream, n, k):
    let q = n >> k
    unary_encode(stream, q)
    write_bits_big_endian(stream, n, k)
```

```
golomb_decode(stream, k) -> int:
    let c = stream.read_bit()

    let n = 0
    while c == 0:
        n++
        c = stream.read_bit()

    let r = b.read_bits_big_endian(k)

    where read_bits_big_endian(k) decodes a fixed-length big-endian integer of
        k-bits 

    c*m + r
```

理解を容易にするため、 `m=5` におけるゴロム・ライス符号化の例を以下に示す。

```
n  = (q, r) = c
0  = (0, 0) = 0 00
1  = (0, 1) = 0 01
2  = (0, 2) = 0 10
3  = (0, 3) = 0 110
4  = (0, 4) = 0 111
5  = (1, 0) = 10 00
6  = (1, 1) = 10 01
7  = (1, 2) = 10 10
8  = (1, 3) = 10 110
9  = (1, 4) = 10 111
10 = (2, 0) = 110 00
```

上記の二つの関数を用いて、単一の整数値を効率的に圧縮することが可能になった。
次のセクションでは以上の材料を用いてコンパクトな集合データ構造を定義していく。

### ゴロム・ライス符号集合(Golomb-Rice Coded Sets)

要素を集合の中に直接入れるのではなく、まずPRFにかける。これにより一様分布に従う値の集合となる。
その後これらの値をソートすると、値の間の差分(delta)は幾何分布に従う。
したがって、連続する二つの要素の差分のみをエンコードすることで 上記のゴロム・ライス符号による
効率的なエンコードが可能となる。

ゴロム・ライス符号集合は二つのパラメータをとる。

* `N` は集合に含まれる値の数
* `P` は `1/fp` と等しく、 `fp` は求められる偽陽性率である。

`P` は幾何分布のパラメータとみなすこともできる。一連のクエリにおいて集合に含まれない要素
を引いてしまう偽陽性率を `1/32` にした場合、「yes」(true, false positiveである)を一回
受け取る前に「NO」(false)を32回受け取る必要がある、というのがここでの直感的な説明である。

この二つのパラメータを与えると、以下のようにして集合を作成できる。

#### 集合の作成

作成にあたって `N` , `P` , `L` という三つのパラメータが必要である。

* `L` は集合に入れたい未加工の要素
* `L` の方は `[]byte`

注意: Golomb-Codingの特別なケースであるGolomb-Rice codingにしたいならば `P` は2の冪乗数
である必要がある。

```
* F = N * P
 * サイズPの入れ物をN個作る
 * 理想的にはアイテムはそれぞれ別のバケットに入る。
 * 偽陽性率はP = 2^fp から与えられる、したがってコリジョンレートは 2^{-fp} である。
```

`N` と `P` を用いて `F = N * P` を計算する。 `F` は望んだ偽陽性率を得るためにハッシュが
取りうる値の幅を制限する。

以下のルーティンで上記のパラメータから未圧縮の集合を作り出す。

```
hashed_set_construct(N, P, raw_items, k): -> []uint64:
    let F = N * P

    let set_items = []
    for item in raw_items:
        let set_value = siphash(k, item) % F
        set_items.append(set_value)

    set_items.sort()

    set_items
```

このルーティンによって、異なる型からなる要素の集合から一様分布の値のリストを作り出す。
最後のステップでこの値をソートする。

#### 集合の圧縮

ハッシュ化された要素の集合の作成(及びソート)ののち、ゴロム・ライス符号化によって隣り合う要素間の
**差分**を符号化する。要素は一様分布に従うため、その差分は幾何分布に従う。そのためこの
符号化方式が最適となる。

以下のルーティンで圧縮のプロセスを示す。

```
gcs_compress(sorted_set, fp) -> []byte:
    let stream = new_bit_stream()

    // P is equivalent to m, the size of a golomb code-word.
    let P = 1 << fp

    let last_value = 0
    for value in sorted_set:
        // Compute the difference between this value and the last value modulo
        // P.
        let remainder = (value - last_value) & (P - 1)

        // Compute the difference between this value and the last one, divided
        // by P. This is our quotient.
        let quotient = (value - last_value - remainder) >> fp

        // Write out the quotient value in unary into the bit stream.
        unary_encode(stream, quotient)

        // Finally, write the remainder into the bit stream using fp bits.
        write_bits_big_endian(stream, remainder, fp)

        // Track this value so we can use it compute the diff between this
        // value and the last.
        last_value = value

    stream.bytes()
```

bloom filterと違い、このままではクエリをかけることができないため、逆の手順による計算を行い
解凍して元の集合を得る必要がある。クエリの際はその値を用いる。

#### 集合のクエリと解凍

ゴロム・ライス符号化によって圧縮された集合内の要素をクエリにかけるためには、まず集合を解凍
しなくてはならない。解凍は圧縮の真逆の手続きによって行われる。
解凍にあたって、エンコードされた商qをデコードし、ついで差分の残りをデコードする。差分が
全てデコードされたら、この差分を一つ前の値に足すことで全体を復元する。このようにすることで
全体を一度に復元することなく逐次的に解凍を行うことができる。

#### 単一の要素のクエリ

以下に単一の要素のクエリを行う際のルーティンを示す。

```
gcs_match(key: [16]byte, compressed_set: []byte, target: []byte, fp, N: int) -> bool:
    // First we'll map the item into the domain of our encoding.
    let item = siphash(key, target) % (N * (1 << fp))

    stream = new_bit_stream(compressed_set)

    // We initialize the initial accumulator to a value of zero.
    let last_value = 0

    // As the values in the set are sorted once the decoded values exceeds the
    // value we wish to query for, we can terminate our search early.
    for last_value < item:
        // Read the delta between this value and the next value which has been
        // encoded using Golomb-Rice codes.
        let decoded_value = golomb_decode(stream, fp)

        // With the delta computed, we can now reconstruct the original value.
        let set_item = last_value + decoded_value

        // If the values match up, then the target item _may_ be in the set, so
        // we return true.
        if set_item == item:
            true

        last_value = set_item

    // If we reach this point, then the item isn't in the set.
    false
```


#### 複数の要素のクエリ

大抵のアプリケーションにとっては、フィルター内の要素の**リスト**が欲しい場合がほとんどである。
このような場合は、二つのソートしたリストを「zip」して探索を行うことができる。ここで二つの
リストとは逐次的に解凍された集合とクエリ対象となる要素のリストである。
以下のルーティンは探索対象となる要素集合が元の集合に含まれ**うる**場合に `true` を返す

```
gcs_match_any(key: [16]byte, compressed_set: []byte, targets [][]byte, 
              fp, N: int) -> bool:

    stream = new_bit_stream(compressed_set)

    // Once again, we'll map our set of target values into the domain our
    // encoding, sorting as a last step so we can zip through the values.
    let items = []
    for t in target:
        let item = siphash(key, t) % (N * (1 << fp))
        items.append(item)
    items.sort()

    // Set up a set of accumulator values that we'll use to zip down the two
    // filters.
    let last_set_val, last_target_val = 0, 0 
    last_target_val = items[0]
    let = 1

    // We'll keep running until one of the values matches each other. If this
    // happens, then we have a match!
    while last_set_val != last_target_val:
        // Perform a pattern match to decide which filter we'll need to
        // advance.
        match:
            case last_set_val > last_target_val:
                // If we still have items let, advance the pointer by one.
                if i < len(items):
                    last_target_val = items[i]
                    i++

                // Otherwise, we've ran our items in our target set, which
                // means nothing matched.
                false

            case last_target_val > last_set_val:
                // In this case, we'll advance the filter we're querying
                // against. This entails decompressing the next element in the
                // set.
                let decoded_value = golomb_decode(stream, fp)

                // Accumulate the decoded delta value to the current value in
                // order to retrieve the current set item.
                last_set_val += decoded_value

    // If we reach this point, the two items in the set matched!
    true
```

### P2Pネットワークの拡張

ここからはこのような仕組みを用いた新しいオペレーティングモードをビットコインのP2Pプロトコル
に適用する方法を示す。

#### Service Bitの確保

まず、現在使われていないservice bitを確保する。これはライトクライアントが本BIPの機能を
サポートするフルノードを選んで接続するために使用される。

このBIPで説明される機能をサポートしていることをシグナリングするために４番目のservice bit
を確保する。

* `CFNodeCF = 1 << 4`

#### フィルターのタイプ

このようなクライアントサイドフィルタリングはジェネリックなコンセプトであり、本BIPでは２種類の
フィルタータイプを定義する。フィルタータイプとはフィルターの構築、クエリの仕方に加え、その
要素の形式を定義したものである。

現在存在するフィルタータイプ二種は以下

* Normal(`0x00`)
* Extended(`0x01`)

`Normal` フィルタはライトクライアントが通常のbitcoin walletとしての果たすために必要な機能
を満たすものである。そのために、通常のトランザクションに対し、normalフィルタは以下を持つ

* トランザクションの各インプットに対応したアウトポイント
* 各トランザクションアウトプット中で、scriptPubKeyでPushされるデータ

`Extended` フィルタは、本BIPで提案される、より発展的なスマートコントラクトアプリケーション
の構築を可能にするための追加のデータを含む。
ブロック内の各トランザクションに対し、 `Extended` フィルタは以下を持つ。

* (インプットがwitnessを持つ場合)インプット中の `witness stack` 内の各アイテム
* 各インプットのsignature script内のデータプッシュ
* トランザクションの `txid`

#### フィルタの構築

フィルタが決定的な方法で作成されたことを保証するため、 `siphash` 関数のキーとして `block hash`
のはじめの `16 byte` を使用する。このBIPをサポートするフルノードはフィルタの集合を
ブロックチェーンに対する追加のインデックスとして用いる。新しいブロックが到着するたびに、
フィルタタイプの作成とディスクでの保持が行われる。このBIPをサポートするためにフルノードを
アップデートする場合、スタートアップ時にチェーンをreindexし、genesisから現在のブロックtipまで
filterを作成する。

bitcoinのブロックを与えられた際に `Normal` コンパクトフィルタの構築は以下のルーティンで
行われる。

```
construct_normal_gcs_filter(block, fp) -> []byte:
    let siphash_key = block.hash()[:16]

    let P = 1 << fp

    let raw_items = []
    for tx in block.transactions:
        if tx.is_coinbase():
            continue

        for input in tx.inputs:
            // Inputs serialized as they are on the wire in transactions.
            // Input index serialized in little-endian.
            let input_bytes = input.hash || input.index
            raw_items.append(input_bytes)

        for output in tx.outputs:
            let output_bytes = extract_push_datas(output.script)
            raw_items.append(output_bytes)

    let N = len(raw_items)
    let F = N * P

    let hashed_items = []
    for raw_item in raw_items:
        let hashed_item = siphash_key(siphash_key, raw_item) % F
        hashed_items.append(hashed_item)

    hashed_items.sort()

    gcs_compress(hashed_items, fp)
```

`Extended` コンパクトフィルタの場合は以下

```
construct_extended_gcs_filter(block, fp) -> []byte:

    let siphash_key = block.hash()[:16]

    let P = 1 << fp

    let raw_items = []
    for tx in block.transactions:
        let txid = tx.hash()
        raw_items.append(txid)

        if tx.is_coinbase():
           continue

        for input in tx.inputs:
            for wit_elem in input.witness:
                raw_items.append(wit_elem)

            let sig_script_pushes = extract_push_datas(input.sig_script)
            for push in sig_script_pushes:
                raw_items.append(push)

    let N = len(raw_items)
    let F = N * P

    let hashed_items = []
    for raw_item in raw_items:
        let hashed_item = siphash_key(siphash_key, raw_item) % F
        hashed_items.append(hashed_item)

    hashed_items.sort()

    gcs_compress(hashed_items, fp)
```

#### フィルタ可能なクエリ

将来的に、別のフィルタエンコーディングアルゴリズムを用いるように、この文書の内容を拡張した
提案をすることが可能である。
そのために、ノードがサポートするフィルタの種類をクライアントが知ることができるように、
新規p2pメッセージを定義する。

`getcftypes` はcommand stringが `getcftypes` となっている空のメッセージである。

`getcftypes` を受け取るフルノードは以下のように定義された  `cftypes` で答える必要がある。


|   フィールドサイズ  | Description | データ型 | コメント  |
| :-- | :-- | :-- | :--   |
| Var-Int | NumFilters  | uint64 | サポートされるフィルターの数 |
| NumFilters | SupportedFilters | [NumFilterBytes]byte | バイト列。各バイトはサポートされるフィルターの型を指す |

#### コンパクトフィルタヘッダーチェーン

本BIPで説明されているフィルタは、コンセンサスクリティカル、すなわち各フィルタはフルノードに
よって検証され、マイナーによってブロックにコミットされるわけ**ではない**。
そこで、ライトクライアントが有効でないフィルタを検出し、拒否するための(より疎結合な)
代替手法が必要になる。完全にP2Pな解決策としては各フィルタに対してdeterministicな
ハッシュチェーンを取得するというものである。このハッシュチェーンあるいは
「フィルタヘッダーチェーン」は、ライトクライアントが受け取ったフィルタの正当性の確認に使用
できるという点で、通常のbitcoinのヘッダーチェーンに似ている。

特定のフィルタータイプに対応するフィルタヘッダーチェーンは以下の再帰処理で作成できる。

```
filter_header(n: uint) -> [32]byte = 
   let zero_hash [32]byte = {0..32}

   if n == 0:
       double-sha-256(genesis_hash || filter(0))

   match filter(n):
      case Some:
          double-sha-256(filter_header(n-1) || double-sha-256(filter(n)))
      case None:
          double-sha-256(filter_header(n-1) || double-sha-256(zero_hash))

   where fitler(n) is the filter for block height n
```

ジェネシスブロックのフィルタヘッダーは定義上一つ前のものが存在しないので、
ジェネシスブロックのハッシュそれ自身を使用する。

フィルタ作成における性質から「空の」フィルタが生成されるようなブロックが作成されてしまう
ことがありうる。例えばscriptPubKeyにデータプッシュのないcoinbase transactionの場合が
そうである。その場合、該当フィルタのハッシュは0が32個並んだものとする。

このフィルタヘッダーチェーンによって、ライトクライアントは初めのチェーン同期時に不誠実な
フルノードに対するセキュリティを確保することができる。本BIPを実装したライトクライアントは
通常のブロックヘッダーチェーンの取得と同時にこのフィルタヘッダーチェーンをピアから受け取る
ので、特定のブロックに対して異なるフィルタが与えられた際に嘘を検出することができる。

さらに、chaintipの更新時に、新しく受け取ったフィルタの正当性の検証のため、(帯域幅の削減のため)
各ピアからフィルタを受け取ることなくフィルタの検証を行う方法を以下に示す。

```
verify_from_tip(tip_block_hash: [32]byte):
    let filter_types = {supported_fitler_types...}
    let connected_peers = {list_of_connected_full_nodes...}

    for filter_type in filter_types:

        let filter_headers = set()
        for peer in connected_peers:
            let filter_header = peer.fetch_filter_header(tip_block_hash)
            filter_headers.insert(filter_header)

        if len(filter_headers) != 1:
            // Peers have conflicting filters. The light client should fetch
            // each unique filter from the set of peers AND fetch the block. The
            // light client can then verify which filter header is correct, and
            // BAN the offending peers.

        // Otherwise, syncing continues as normal: fetch filter to see if it
        // matches any relevant items.
```

ライトクライアントは継続的にすべてのフィルタヘッダーをディスクに保存していく。
(rescanやchainの調査によって)事後的にフィルタを再ダウンロードしても、その正当性を確認することが
できる。

フルノードも通常のフィルタの場合と同様に、ライトクライアントと同じような検証作業を行う。

ここでフィルタヘッダーチェーンの取得と検証をサポートする新規メッセージを二つ導入する。

| Field Size | Description | データ型 | コメント |
| :-- | :-- | :-- | :-- |
| 4 | ProtocolVersion | uint64 | プロトコルのバージョン |
| Var-Int | NumBlockLocators | uint64 | block locatorの数 |
| 32 | HashStop  | [32]byte | 終端のハッシュ |
| 1 | FilterType | byte | リクエスト対象のフィルタヘッダーのタイプ |

メッセージ内の `BlockLocators` は `getheaders` や `getblocks` の場合と同様に解釈される。

`cfheaders` メッセージは以下のように定義される。

| Field Size | Description | データ型 | コメント |
| :-- | :-- | :-- | :-- | :-- |
| 32 | StopHash | []byte | 終端のハッシュ |
| 1 | FilterType | byte | リクエスト対象のフィルタヘッダーのタイプ |
| Var-Int | NumHeaders | uint64 | 送信するヘッダーの数 |
| NumHeaders * 32 | HeaderHashes | [NumHeaders][32]byte | フィルタヘッダーのスライス |

### コンパクトフィルター

最後に解説するメッセージはコンパクトフィルタの自体の取得を行うためのものである。
これによりライトクライアントは特定のブロックハッシュに対応したコンパクトフィルタをリクエストできる。

`getcfilter` メッセージは以下で

| Filed Size | Description | データ型 | コメント |
| :-- | :-- | :-- | :-- |
| 32 | BlockHash | [32]byte | 対応するフィルタが欲しいブロックのハッシュ |
| 1 | FilterType |  byte | リクエスト対象のフィルタのタイプを指定するバイト |

`cfilter` メッセージは以下である。

| Filed Size | Description | データ型 | コメント |
| :-- | :-- | :-- | :-- |
| 32 | BlockHash | [32]byte | 対応するフィルタが欲しいブロックのハッシュ |
| 1 | FilterType | byte | フィルターのタイプ |
| Var-Int | NumFilterBytes | uint64 | 続くフィールドのfilterが各何バイトを占めるかを指す | 
| NumFilterBytes | FilterBytes | [NumFilterBytes]byte | このブロックに対応する圧縮済みのfilter |

両メッセージにおける `BlockHash` のおかげで、リクエストとレスポンスを付き合わせるのが容易になる。

パラメータ `N` (フィルタ内の要素数) と `P` ( `1 << false_positive_rate` )は、ライトクライアント
がフィルタを逐次的にデコードし、クエリに用いたり、ブロックと突き合わせて正当性を確認したりする
のに必要である。 `N` はあらかじめ知っておくということができないものなので、 `0x00` , `0x01` の
それぞれのコンパクトフィルタのシリアライゼーションタイプを以下のように定義する。

```
N || raw_filter_bytes
```

ただし `N` は32bitビッグエンディアンの整数である。

ところで、フィルタが `null` であるような特殊例が存在する。この場合、0をエンコードするのに
`4-bytes` を消費するよりも空のバイトスライスを用いるのが望ましい。

パラメータ `P` はフィルタタイプごとにグローバルに合意が取れていなければならないものなので、
`0x00` と `0x01` に関してはここにスタティックな定義をしておく。ともに偽陽性率は `20` とする。
したがって `P` は `2^20` となり、 `fp=20` とする。

### プロトコルバージョンの引き上げ

本BIPではP2Pに新しい振る舞いを導入するため、その検出のためにプロトコルバージョンを一つ増やす。
このBIPを実装したフルノードは `70016` のプロトコルバージョンを周囲に伝える。

## Walletに実装可能な新規機能

以上により、walletはクライアントサイドに(最大で)全チェーンのインデックスを保持することができる。
これはwallet側に大きな利便性の向上をもたらす。例えばプライベートキーやHD-seedのインポート時に
第三者サーバーを信用する必要がなくなる。
さらにオンチェーンのイベントに対して反応しなくてはならない類のスマートコントラクト
アプリケーションの実装が可能になる。これにはLightningが含まれる。

## 実装にあたっての注意

長さ0のフィルタは `N` なしで送信することで、クライアントはリクエストをする前にフィルタが
`null` であるとわかるので `4-byte` の値を節約することができる。

この提案を実装するクライアントは `sendheaders` メッセージを利用して**いるべきである**。。
これによりフルノードがヘッダを直接送ってくれるので、chaintipの更新にあたって `inv` メッセージ
を利用して新しいブロックのアナウンスを検知するよりも効率的である。
ノードは不要なラウンドトリップなしにすぐに `cfheaders` を接続先ピアにリクエストできるので
フィルタ自体をすぐにゲットできる。

ライトクライアントの同期は逆方向から行われても良い。つまり通常のヘッダとフィルタヘッダを
チェーンの**終端から**同期していくことが可能である。これによりクライアントが昔のフィルタ
データを必要としないような場合はほぼ即座に利用可能な状態になる。

ブロックの取得を、(閉じたチャネル内ではなく)p2pネットワークから直接行うような場合は、
中間者がトランザクションを解析することを避けるため、ライトクライアントは複数のピアからブロック
を取得**すべきである**。

鍵のインポートと再スキャン: `filters` のlazyな取得を行う場合、 `cfilters` を初めから全部取得
し直しにならないよう、最初のブロックを持っていることが極めて重要である。

## 謝辞とAppendix

省略

## reference

[^1]: https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki
[^2]: https://eprint.iacr.org/2014/763.pdf
[^3]: https://eprint.iacr.org/2014/763.pdf
[^4]: https://jonasnick.github.io/blog/2015/02/12/privacy-in-bitcoinj/
[^5]: https://en.wikipedia.org/wiki/Private_information_retrieval
[^6]: http://urchin.earth.li/~twic/Golombs_Original_Paper/


## 所感


Golomb Coded Setsはhttp2のサーバープッシュにおいてキャッシングを効率化するために
用いられている手法らしい。世の中には頭のいい人がいるなーという気持ちである。

* http://jxck.hatenablog.com/entry/service-worker-casper
* http://giovanni.bajo.it/post/47119962313/golomb-coded-sets-smaller-than-bloom-filters
* http://blog.k11i.biz/2015/10/bloom-filter.html

フィルタヘッダーチェーンがPoWを持っているわけではないので、嘘の履歴を100検出できないのでは？
嘘フィルタをよこしたピアをbanするから問題ないということだがそれでは弱すぎる気がする。
かといってフィルタチェーンをコンセンサスの一部に組み込むと検証が重そう。

とはいえ接続先ノードへの信頼を減らせるので良さげ。

そもそもlayer2でプライバシーを担保するというのは技術的課題が多すぎてつらいということがわかる。
プライバシー保護ガチ勢には頑張ってほしい。
