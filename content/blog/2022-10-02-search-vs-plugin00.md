---
title: "Visual Studio 全文検索拡張"
date: 2022-10-09T00:00:00+09:00
archives:
    - 2022-10
categories: ["blog"]
tags: ["note", "visual studio"]
draft: false
---

# はじめに
数週間でとりあえず検索できるだけのVisual Studio用の全文検索拡張ができました. `Entrian Source Search`や`FastFind`といった既存の拡張があるのは知っています. 
早期リリースを目標に不具合検証やユーザビリティは後回しです.

- 高速な全文検索をVisual Studio上で行いたい
- ソリューションの情報からインデックスを作るファイルを決定してほしい
  - 上に挙げた拡張を使用したことがありませんが, インデックスをつけるディレクトリを指定するのでしょうか

# Visual Studio Extensibility

昔と比べればずいぶんとドキュメントが充実しました, [Visual Studio Extensibility](https://learn.microsoft.com/en-us/visualstudio/extensibility/).
ただそれでもドキュメント化されていない仕様に悩まされます, とくにソリューションのアイテムについては扱いが全くわかりません.

たとえば`ProjectItem`クラスのタイプについては, [EnvDTE.Constants](https://learn.microsoft.com/en-us/dotnet/api/envdte.constants)で種類を判別できますが, `vsProjectItemKindSolutionItems`が実際はどれに対応してどういう動作をするのかわかりません. 
`ProjectItem`クラスはなんでもクラスなので, サポートしていない機能にアクセスすると例外が送出されるか`null`が返ります. わたしは`Collection`メンバは子オブジェクトを所有していると勘違いしましたが, これは`ProjectItem`自身も含みます.

`Project`についても情報がないので, 有志の情報をもとに[ProjectTypes.cs](https://gist.github.com/taqu/5c14f5f99817cb73a9461b1a440af0f5.js")を作りました.

# Lucene.Net
研究段階では`groonga`も試してみたのですが, ドキュメントが皆無に等しいのとVisual Studio拡張にC/C++のDLLを組み込むのは少し手間なので断念しました.

## サブDB選定
ファイルの更新時刻とインデックスの更新時刻の比較に当初Luceneを利用していましたが, あまりにも遅いため, 別
DBに保存することにしました.

C/C++のDLLが面倒くさい問題のため, `LevelDB`, `RocketDB`は候補から外しました. `Microsoft FARSTER`がよさそうですが, `Lucene.Net`と依存ライブラリで問題が出ました, これで一日消費しました.
単純な`key-value`でいいのですが, 仕方がないので`LiteDB`を選択しました.

## 索引付け
索引付けは次のようにしました.

1. 検索対象をソリューションファイルから収集
   - ソリューションのアイテムをロックできないので, 処理中に変更操作があると何が起きるかわかりません
2. 最後に調べたファイルの更新時刻と比較して, 索引付け候補を絞り込み
   - 別にDBを用意しました
3. Luceneの索引を行ごとに更新

未解決の問題はプロジェクトが削除された場合にどうするかです.

# パフォーマンス

環境は次のとおり,
| CPU          | メモリ |
| :---         | :---   |
| Core i7-8700 | 32 GB  |

全てデバッグビルドでVisual Studioのデバッガに繋いだ状態です.

## インデックス作成

手元にあった, おそらくローカルひとつで, これより巨大なプロジェクトはなかなかない, `Unreal Engine 4.27.2`で試しました.

| 処理                               | 時間（ミリ秒） | 備考                     |
| :---                               | :---           | :---                     |
| インデックス作成候補のファイル収集 | 380            | 34133ファイル            |
| 更新日時をチェックしてカリング     | 1812           |                          |
| インデックス作成                   | 422517         | 34120ファイル, 5373439行 |

完全に空のファイルはインデックス作成に含まれないので, 少なくなっています.

| ファイル                       | サイズ  |
| :---                           | :---    |
| Luceneのインデックスファイル群 | 約343MB |
| LiteDBのDB                     | 約11MB  |

`Unreal Engine`が無駄に消費するストレージサイズに比べればゴミのようなサイズです.

## 検索
こちら[Entrian Source Search](https://qiita.com/EGJ-Nori_Shinoyama/items/9385cf73b9a75966d803)で, `r.invalidateCachedShader`が約3秒で検索できるということです.

`r.invalidateCachedShader`で検索すると`r`にヒットしたり, `invalidateCachedShader`だとヒットしなかったり, インデックスの付け方と検索クエリが悪いのですが, それでも最大1000件の検索で300ミリ秒をきるぐらいなので, ここから使い勝手を磨いていきたいところです.

# まとめ
とりあえず動いて早期リリース, というより, 事情により使いにくくても自分のために実践投入したいので, 不具合は後回しです.

初回起動時は完全に固まってしまうのは, なんなのですかね. `await`の仕様を勘違いしてるのかな.
