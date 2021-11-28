---
title: "Unreal Installed Buildなどいろいろ"
date: 2021-11-27T00:00:00+09:00
archives:
    - 2021-11
categories: ["blog"]
tags: ["note", "unreal"]
draft: true
---
# はじめに
Unreal Engine 4 について, 些細なノートになります.

# Installed Build
Installed Buildでないと, プロジェクトをリビルドしたときにエンジンのビルドも巻き込まれてリビルトされてしまいます. 
公式のビルド済みではエンジンのDebugが含まれないため, エンジン内部までステップインしたい場合に困ります. 
次のようなバッチで自分用のビルドを作るとよいでしょう. `-set:HostPlatformOnly=true`とするか, 次のようにプラットフォームを指定します.

```
%~dp0\Engine\Build\BatchFiles\RunUAT.bat ^
    BuildGraph ^
    -target="Make Installed Build Win64" ^
    -script=Engine/Build/InstalledEngineBuild.xml ^
    -set:WithWin64=true ^
    -set:WithWin32=false ^
    -set:WithMac=false ^
    -set:WithAndroid=false ^
    -set:WithIOS=false ^
    -set:WithTVOS=false ^
    -set:WithLinux=false ^
    -set:WithLinuxAArch64=false ^
    -set:WithLumin=false ^
    -set:WithHoloLens=false ^
    -set:WithDDC=false ^
    -set:WithFullDebugInfo=true ^
    -set:GameConfigurations=Debug;Development;Shipping
```

C++をメインに開発する場合, 実行速度は遅いですが`DebugGame`か`Debug`でエディタを実行しておいた方が, 結果的に開発速度は速いように思います.

# Level Loading
Unreal EngineのULevelのロードは, UWorldが保持する`ULevelStreaming`が担当しています. 
Blueprintのノード `Load Stream Level` などは, `FLatentActionManager`に, `ULevelStreaming`を保持する`FStreamLevelAction`を投げることで実現しています. 

LevelをPersistentにすることは別で, `UEngine`にURLを設定します.

例えば, 次のようなアクションを `FLatentActionManager`に投げるとロード完了時に任意の処理ができます. C++でやるならば, `FLatentActionManager`を通さなくても効率よくできるように思います. といいますかやろう（宿題）.

UMGとSlateの関係もそうですが, こういった２重構造がUnreal Engine最大の欠点だと思います. Blueprintを廃止してもUnityと闘えると思うのですが, Cry Engineがあったか.

```cpp
#pragma once
#include <Engine/LevelStreaming.h>

class MY_API FMyStreamLevelAction : public FStreamLevelAction
{
public:
    DECLARE_DELEGATE_TwoParams(FOnLoadedDelegate, bool, const FName&);

    FMyStreamLevelAction(bool bIsLoading, const FName& InLevelName, bool bIsMakeVisibleAfterLoad, bool bShouldBlock, const FLatentActionInfo& InLatentInfo, UWorld* World);
    virtual void UpdateOperation(FLatentResponse& Response) override;

    FOnLoadedDelegate& OnLoaded();
private:
    FOnLoadedDelegate OnLoadedDelegate;
};
```

```cpp
#include "MyLevelStreaming.h"

FMyStreamLevelAction::FMyStreamLevelAction(bool bIsLoading, const FName& InLevelName, bool bIsMakeVisibleAfterLoad, bool bShouldBlock, const FLatentActionInfo& InLatentInfo, UWorld* World)
    : FStreamLevelAction(bIsLoading, InLevelName, bIsMakeVisibleAfterLoad, bShouldBlock, InLatentInfo, World)
{
}

void FMyStreamLevelAction::UpdateOperation(FLatentResponse& Response)
{
    FStreamLevelAction::UpdateOperation(Response);
    ULevelStreaming* LevelStreaming = Level.Get();
    if(nullptr != LevelStreaming) {
        bool bIsOperationFinished = UpdateLevel(LevelStreaming);
        OnLoadedDelegate.ExecuteIfBound(bIsOperationFinished, LevelName);
    } else {
        OnLoadedDelegate.ExecuteIfBound(true, LevelName);
    }
}

FMyStreamLevelAction::FOnLoadedDelegate& FMyStreamLevelAction::OnLoaded()
{
    return OnLoadedDelegate;
}
```

