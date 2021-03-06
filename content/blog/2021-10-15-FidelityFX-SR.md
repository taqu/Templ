---
title: "FidelityFX Super Resolution について"
date: 2021-10-15T00:00:00+09:00
archives:
    - 2021-10
categories: ["blog"]
tags: ["note", "unreal"]
draft: false
---
# はじめに
[AMD FidelityFX-FSR](https://github.com/GPUOpen-Effects/FidelityFX-FSR)（FSR）は品質が素晴らしく, 組み込み易く, 移植性も高いのですがデスクトップや同等のコンソール向けで, モバイルには負荷が高いと思います.
そういうわけで, 古典的なLanczosフィルタリングと比較してみました. FSRのサンプルに組み込んで比較しています.

実装したLanczosは, 半径を２にして同じパスで簡単なシャープネスフィルタを入れています.

# 比較
シャープネスの具合はパラメータを変えればいくらでも変えることができますが, コントラストが変わってしまう点は研究不足で, FSRがコントラストが変わらず優秀です.
速度はデスクトップではほぼ変わりませんが, サンプル点を減らしたり, half floatを使ったり工夫の余地があります, 全て自分の支配下にありますから.

マルチプラットフォームでない限り, 同じシェーダを使う限り見た目は同じなので, 最初からそういう絵作りをすればコントラストの変化も問題にならないのでは？と思います.

{{< figure src="/images/blog/FidelityFX_SR01.jpg" alt="FidelityFX SR">}}

