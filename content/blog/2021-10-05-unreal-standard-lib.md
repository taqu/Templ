---
title: "Unreal Engine 4 でC++メインで開発するために"
date: 2021-10-05T00:00:00+09:00
archives:
    - 2021-10
categories: ["blog"]
tags: ["note", "unreal"]
draft: false
---

# コンパイルシステム
## モジュール
ビルド単位を分割して管理しています. １システム＝１モジュールぐらいの粒度で理解しておくと, ビルドシステムが意図していることとの差異もなさそうです.
モジュール単位で, サブシステムや機能の可視性を管理できます.

## Unity
モジュール単位で, 全ての実装ファイルをひとつにまとめてビルドする機能です. 古代のコンパイラだとコンパイル単位を超えた最適化は許されないので, それなりに意味はありそうです.

## ディレクトリ構造
ビルドシステムと深く結び付いているため非常に重要です. モジュールの場合で説明します.

基本は次のようなツリーを最初に作成するといいです.

```
Module Root
  Classes
  Private
  Public
```

それぞれの機能は, 次のようになります. 機能と言っているのは, このように配置しないと容易にリンクエラーになるなど, ビルドシステムに強く結び付いているためです（大事なことなので）.
`Classes`と`Public`の違いはわかりません（TODO:調べろ）.

| Name    | Description                                                      |
| :---    | :---                                                             |
| Classes | 公開するクラス, 機能が入ったヘッダファイル                       |
| Private | 公開しないクラス, 機能の入ったヘッダファイル, 全ての実装ファイル |
| Public  | 公開するクラス, 機能が入ったヘッダファイル                       |

　
[ディレクトリ構造](https://docs.unrealengine.com/4.27/en-US/Basics/DirectoryStructure/)

# 言語機能
## 名前空間
`Unity ビルド`があるため, 無名名前空間は厳禁です. 

# 標準ライブラリ
## メモリ管理

メモリ管理の最下層は`FMemory::Malloc/Free`で, アルゴリズムは`GenericPlatformMemory.h`で定義される次のいずれかになります.

> （GenericPlatformMemory.hから引用）

| Name     | Description                               |
| :---     | :---                                      |
| Ansi     | Default C allocator                       |
| Stomp    | Allocator to check for memory stomping    |
| TBB      | Thread Building Blocks malloc             |
| Jemalloc | Linux/FreeBSD malloc                      |
| Binned   | Older binned malloc                       |
| Binned2  | Newer binned malloc                       |
| Binned3  | Newer VM-based binned malloc, 64 bit only |
| Platform | Custom platform specific allocator        |
| Mimalloc | Microsoft's malloc                        |

標準ライブラリの`malloc/free`を使用すると管理情報が別になる可能性があるため, `FMemory`だけを使うようにします.
`string.h`, `memory.h`にあるプリミティブなメモリ操作系も`FMemory`経由で使用します.

`new/delete`はoverride箇所を見つけられませんでしたが, 観測した範囲では`FMemory::Malloc/Free`を通っています.
（私のような）他人を信じない方は, プロジェクト固有のラッパーを作成した方がいいと思います.

## 文字列

`FString/FName/FText`の違いは次のようです.

| Name    | Description                               |
| :---    | :---                                      |
| FString | ソフトウェア内で使う文字列                |
| FName   | システム内のIDとして使うimmutableな文字列 |
| FText   | UI等でユーザに表示・編集される文字列      |

最下層はFStringに行きつきます（TODO:真面目に追う）. さらにFStringはTArrayを使っているため, `Small String(Size) Optimization`がありません. コピー発生には十分に気を付ける必要があります.

多言語対応は, 基本的にはMicrosoftの`TCHAR`を参考にしたシステムになっているようです. 文字列リテラルは`TEXT`で囲むなどです（TODO:一覧でまとめる）.
全て`wchar_t`になるわけではなく, 例えばAndroidでは`char16_t`になります. 内部文字コードは`UTF-16`と考えられます（TODO:真面目に追う）.


