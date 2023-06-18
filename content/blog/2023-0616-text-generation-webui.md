---
title: "Text Generation WebUI"
date: 2023-06-16T00:00:00+09:00
archives:
    - 2023-06
categories: ["blog"]
tags: ["note", "ML"]
draft: false
---
# はじめに
WSL環境に[text-generation-webui](https://github.com/oobabooga/text-generation-webui)をインストールしてみます.

# インストール
## 基本セット
```bash
$ sudo apt update
$ sudo apt install -y vim cmake git wget curl software-properties-common gnupg gnupg2 gnupg1
```

DeepSpeedを使用する場合は次も必要です.

```bash
$ sudo apt install libaio-dev
```

## Conda

```bash
$ curl -sL "https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh" > "Miniconda3.sh"
$ bash Miniconda3.sh
```

ここからCondaで`textgen`環境を作って作業するといいと思います.
ただし`source`コマンドを使うとCondaの環境がネストしながら`base`になるらしいので,
`source`を使う代わりに一度exitして`textgen`環境をactivateするところから再開するといいと思います.

```python
>>> conda create -n textgen python=3.10.9
>>> conda activate textgen
```

## Cuda
Windows側でCudaドライバインストール済みです.

```bash
$ nvidia-smi
```

## Cuda Toolkit 11.7
バージョン 11.7をWSLにインストールします. Ubuntu WSLを選択して後は流れで,
[Cuda Toolkit 11.7](https://developer.nvidia.com/cuda-11-7-0-download-archive)


```bash .bashrc
export "$PATH:/usr/local/cuda/bin"
```

## Pytorch 
環境に応じてインストールを, [PyTorch](https://pytorch.org/).

## Text Generation WebUI
公式のインストールインストラクションに従います.

```bash
$ git clone https://github.com/oobabooga/text-generation-webui
$ cd text-generation-webui
$ pip install -r requirements.txt
```

なにかうまくインストールされなかったので, UI, xFormes, DeepSpeedやFlexGenを使う場合は次も追加で入れます（おそらくConda環境をミスしていたと思われます）.

```bash
$ sudo pip install gradio markdown xformers mpi4py
```

### mpi4py
DeepSpeedを使いたい場合は次も実行します.

```bash
$ sudo apt install libopenmpi-dev
$ conda install mpi4py
```

# 使用

## モデル
モデルのダウンロードは簡単スクリプトが用意されています.
`text-generator-webui`ディレクトリ内で作業します.


```bash
$ python download-model.py facebook/opt-1.3b
```

## 起動
Haggingfaceのスペース名とモデル名をアンダースコアで繋いでるのかな.

```bash
$ python server.py --chat --model facebook_opt-1.3b --xformers
```

## DeepSpeed
細かい設定はコードを変更しないといけないようですけど, 簡単に実行するなら次のような感じ.

```bash
$ deepspeed --num_gpu=1 server.py --deepspeed --chat --xformers --model opt-1.3b
```
## FlexGen
Facebook Opt限定で, FlexGenを使うことができます.
先ず`models`ディレクトリ内にあるモデルディレクトリの名前を変更します. `facebook_opt-13b`なら`opt-13b`のようにスペース名を削除します.
各モジュールが想定している命名規則に齟齬があるらしくとりあえずのワークアラウンドです.
次のようにコンバートすると, `models/opt-13b-np`が生成されます.

```bash
$ python convert-to-flexgen.py models/opt-13b
```

後は元のモデル名を指定して起動すればよいです. 
```bash
$ python server.py --chat --model opt-13b --xformers --flexgen --compress-we
ight --percent 100 0 100 0 100 0
```
# モデルいろいろ
スペック表です.

| 項目   |                |
| :----- | :------------- |
| CPU    | Ryzen7 7700    |
| Memory | DDR5 64GiB     |
| GPU    | RTX 4070 12GiB |

## RWKV
[RWKV-model](https://github.com/oobabooga/text-generation-webui/blob/main/docs/RWKV-model.md)
```bash
$ pip install rwkv
```

モデルは直接`models`ディレクトリにダウンロードします.
トーカナイザー設定 [20B_tokenizer.json](https://raw.githubusercontent.com/BlinkDL/ChatRWKV/main/v2/20B_tokenizer.json) も`models`ディレクトリにダウンロードします.

```bash
$ python server.py --chat --xformers --model RWKV-4-Pile-169M-20220807-8023.pth --rwkv-strategy "cuda fp16i8" --rwkv-cuda-on
```

`rwkv-cuda-on`は作者の環境では動作しないらしいけれど, 私の環境ではこれがないと遅すぎて数分じゃ応答が帰ってこない.
`rwkv-strategy`を次のように設定します.

| | |
| :--- | :--- |
| "cpu fp32" | CPU mode |
| "cuda fp16" | GPU mode with float16 precision |
| "cuda fp16 *30 -> cpu fp32"| GPU+CPU offloading. The higher the number after *, the higher the GPU allocation.|
| "cuda fp16i8"| GPU mode with 8-bit precision |

CPUのオフロードについては, いろいろなクエリの平均をとらないと正確ではないので参考までに.

| | VRAM消費 |tokens/s |
| :--- | :--- | :--- |
| "cpu fp32" | 0 | 0.30 |
| "cuda fp16" | out of memory ||
| "cuda fp16 *10 -> cpu fp32" | 4GiB | 0.49 |
| "cuda fp16i8 *30 -> cpu fp32" | 6.7GiB | 1.89 |
| "cuda fp16i8 *10 -> cpu fp32" | 2.7GiB | 0.78 |
| "cuda fp16i8" | 8GiB | 2.47 |

# まとめ
とりあえずのインストールから起動まで. 楽して社内向けアプリにできるかなと思ったけど,
そもそも言語モデル（日本語）をチューニングしたりしなければならないし, タスク特化のUIを考えると,
お家でチャットをちょっと試してみるぐらいの用途がいいのかも.

