---
title: "Unreal Engine 4 でIntel Implicit SPMD Program Compilerを使う"
date: 2021-11-16T00:00:00+09:00
archives:
    - 2021-11
categories: ["blog"]
tags: ["note", "unreal"]
draft: true
---
# はじめに
Unreal Engine 4で, [Implicit SPMD Program Compiler](https://ispc.github.io/index.html) (ISPC)を使用してみます.
EngineのModuleのひとつで, Animation, Chaos, Niagaraなど至るところで使われています.
組み込みに難しいことはなく, 将来に渡ってサポートされる可能性が高いです.

# 組み込み
単純なサンプルで試してみます. 
まず, `.Build.cs`にModuleを追加します. これで, ビルドに`.ispc`が含まれ, オブジェクトファイルが生成されます.

```csharp
PrivateDependencyModuleNames.AddRange(new string[] { "IntelISPC" });
```

そして, 次のような`ISPCTest.ispc`を用意します. これは, Intelのサンプルのものです.

```cpp
export void simple(uniform float vin[], uniform float vout[], uniform int count)
{
    foreach (index = 0 ... count) {
        float v = vin[index];
        if (v < 3.)
            v = v * v;
        else
            v = sqrt(v);
        vout[index] = v;
    }
}
```

UE4のActorを追加して, 毎フレーム次のコードを回してみます.

```cpp
#if INTEL_ISPC
#include "ISPCTest.ispc.generated.h"
#endif

DECLARE_CYCLE_STAT(TEXT("AISPCTestActor_RunISPC"), STAT_AISPCTestActor_RunISPC, STATGROUP_ISPC);
void AISPCTestActor::RunISPC()
{
    SCOPE_CYCLE_COUNTER(STAT_AISPCTestActor_RunISPC);
    ispc::simple(vin, vout0, Count);
}
DECLARE_CYCLE_STAT(TEXT("AISPCTestActor_RunLoop"), STAT_AISPCTestActor_RunLoop, STATGROUP_ISPC);
void AISPCTestActor::RunLoop()
{
    SCOPE_CYCLE_COUNTER(STAT_AISPCTestActor_RunLoop);
    for (int32 i = 0; i < Count; ++i) {
        float v = vin[i];
        if(v < 3.) {
            v = v * v;
        } else {
            vout1[i] = FMath::Sqrt(v);
        }
    }
}
```

オブジェクトファイルと別に, `ファイル名.ispc.generated.h`というプロトタイプ宣言が生成されます. また, ISPCの関数は `namespace ispc`に入ります.  
これで, ISPCを使用する最低限の環境ができました.

# まとめ
サンプル程度だといくら計算回数を増やそうが, ただのループと変わりませんが, ちゃんと考えてあげれば速くなるでしょう.

