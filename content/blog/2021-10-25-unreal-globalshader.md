---
title: "Unreal Globalshader"
date: 2021-10-25T00:00:00+09:00
archives:
    - 2021-10
categories: ["blog"]
tags: ["note", "unreal"]
draft: true
---
# はじめに

## プロジェクト設定
プロジェクトの `Config/DefaultEngine.ini`に次の設定を追加します. エンジンの`ConstantVariables.ini`に追加した場合, 元に戻すことを忘れた場合に影響が大きいためやめましょう.
更新したらエディタを再起動します, どの設定がどのタイミングでリロードされているか追いきれていないからです.

```
[ConsoleVariables]
r.Shaders.Optimize=0
r.Shaders.KeepDebugInfo=1
r.ShaderDevelopmentMode=1
r.DumpShaderDebugShortNames=1
```

## シェーダディレクトリとBuild.cs
ルートに, `Shaders`とその配下に`Public`と`Private`を作成します.

```
- Project
  - Source
  - Shaders
    - Public
    - Private
```

Build.csの`LoadingPhase`は, `PostConfigInit`に変更します. これは, シェーダバリエーションを全て網羅するためには, 少なくともこのタイミングでないといけないためと思われます.


# usf, ushのトラブル
HLSLから, Direct3Dのasm, GLSLのコード, SPIR-Vへ変換しているような動作をしています. この過程で, 変換先に対応する命令がない場合に単純にその命令付近が削除されているように見えます.
私はHLSL統一論のもと, (DirectX Shader Compiler)[https://github.com/microsoft/DirectXShaderCompiler]だけでなんとかしようとしているので, 理解できます.
例えば, `Gather`系の命令がない環境向けにビルドした場合, `Gather`系命令を受けた変数に対して, `宣言されていない変数xが使用されている`など, どこが悪いのかわからないエラーがでてしまいます.

