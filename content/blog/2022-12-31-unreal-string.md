---
title: "Unreal Engineの"
date: 2022-12-31T00:00:00+09:00
archives:
    - 2022-12
categories: ["blog"]
tags: ["note", "unreal"]
draft: false
---

# はじめに
Unreal EngineのFStringについて, コピーの動作がどうなっているのか, 気軽にコピーしてもよいのか明確にしようと思います.

# FString

## 構造
ここをみれば特性は把握できるので, わざわざ実験する必要もないのですが.
`TArray`を利用しており, 固定サイズで容量が増加します.

```cpp
using AllocatorType = TSizedDefaultAllocator<32>;
/** Array holding the character data */
typedef TArray<TCHAR, AllocatorType> DataType;
DataType Data;

```

## コピー
`TArray`の`ResizeForCopy`が呼び出されるだけで, `FMemory::Malloc`や`FMemory::Realloc`に行きつきます. `Small Size Optimization`などはとくにありません. 
次のような簡単なテストでは`std::string`の方がパフォーマンスが良いです.

```cpp
ATestString::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    using wstring = std::basic_string<TCHAR>;
    static constexpr uint32 Count = 10000;
    uint32 Frame = static_cast<uint32>(GFrameCounter%100);
    FString S = FString::Format(TEXT("%d"), {Frame});
    wstring s;
    {
        std::basic_stringstream<TCHAR> ss;
        ss << Frame;
        s = ss.str();
    }
    TArray<FString> FStrings;
    std::vector<wstring> strings;
    FStrings.SetNum(Count);
    strings.resize(Count);
    {
        SCOPE_CYCLE_COUNTER(STAT_FString);
        for (uint32 i = 0; i<Count; ++i) {
            FStrings[i] = S;
        }
    }
    {
        SCOPE_CYCLE_COUNTER(STAT_StdString);
        for (uint32 i = 0; i<Count; ++i) {
            strings[i] = s;
        }
    }
}
```

# まとめ
Unreal EngineのSTLに代わるコンテナがゲーム向けに高速化されているという事実は, 私の知る範囲ではありません. `FString`は`TArray`に依存する, `TMap`は`TSet`に依存するなどコード量を減らすやり方が, パフォーマンスの面ではマイナスにはたらいています. 
ゲームでよく使われるコンテナが追加されているという意見には同意します.
STLと同じように, 集成体の引数はconst参照というプラクティスは有用です. `TStringView`もありますので活用していくとよいでしょう.

