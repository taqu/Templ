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

DeepSpeedを使用する場合は次も必要.

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
公式のインストールインストラクションにそいます.

```bash
$ git clone https://github.com/oobabooga/text-generation-webui
$ cd text-generation-webui
$ pip install -r requirements.txt
```

なにかうまくインストールされなかったので, UI, xFormes, DeepSpeedやFlexGenを使う場合は次も追加で入れる（おそらくConda環境をミスしていたと思われる）.

```bash
$ sudo pip install gradio markdown xformers mpi4py
```

### mpi4py
DeepSpeedを使いたい場合は次も実行する.

```bash
$ sudo apt install libopenmpi-dev
$ conda install mpi4py
```

# 使用

## モデル
モデルのダウンロードは簡単スクリプトが用意されている.
`text-generator-webui`ディレクトリ内で作業する.


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
各モジュールが想定している命名規則に齟齬あるらしくとりあえずのワークアラウンドです.
次のようにコンバートすると, `models/opt-13b-np`が生成されます.

```bash
$ python convert-to-flexgen.py models/opt-13b
```

後は元のモデル名を指定して起動すればよいです. 
```bash
$ python server.py --chat --model opt-13b --xformers --flexgen --compress-we
ight --percent 100 0 100 0 100 0
```

# まとめ
とりあえずのインストールから起動まで. 楽して社内向けアプリにできるかなと思ったけど,
そもそも言語モデル（日本語）をチューニングしたりしなければならないし, タスク特化のUIを考えると,
お家でチャットをちょっと試してみるぐらいの用途がいいのかも.

