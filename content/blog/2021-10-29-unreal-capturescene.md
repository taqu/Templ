---
title: "Unreal SceneCaptureComponent"
date: 2021-10-29T00:00:00+09:00
archives:
    - 2021-10
categories: ["blog"]
tags: ["note", "unreal"]
draft: false
---
# はじめに
Unreal Engine4 (UE4) で, メインカメラとは別の視点から描画したい場合, USceneCaptureComponent2Dを試すことになると思います.
実用するには, 足りない部分があります.

- メインカメラと描画を完全に分離したい
  - 描画物を楽に限定したい
  - ライティングを分離したい
- TextureStreamingが更新されない
  - [SceneCapture2DはTextureStreamingが未対応（？）なのは仕様ですか？](https://answers.unrealengine.com/questions/841147/view.html)
  - 自力更新するしかない

# メインカメラと描画を分離
管理や量産を考慮すると, サブレベルでステージ内のものを管理する方法がいいと思います. UE4の内部でのレベルの管理も考慮して, それぞれのサブレベルに`USceneCaptureComponent2D`を配置します.

```
- MainLevel
  - SubLevel00
    - USceneCaptureComponent2D00
  - SubLevel01
    - USceneCaptureComponent2D01
```

# 描画物の限定
後述する`Owner Only See`が機能しない問題も合わせて考えると, USceneCaptureComponent2Dをoverrideすると楽です. サブレベルに`USceneCaptureComponent2D`を配置すると所有しているレベルにアクセスできるため, レベルないのコンポーネントに限定してアクセスできます.
レベル下のコンポーネントを全て登録するコードは, 次のようになります. コードにすると取るに足らないのですが, これをBlueprintにするとクソデカ粗大ゴミになります.

```cpp
void USceneCaptureComponent2DEx::AddActorComponentsInLevel()
{
	QUICK_SCOPE_CYCLE_COUNTER(UCustomSceneCaptureComponent2D_AddActorComponentsInLevel);
	UWorld* World = GetWorld();
	if(nullptr == World){
		return;
	}

    uint32 Count = 0;
    AActor* ActorClass = GetOwner();
    ULevel* Level = ActorClass->GetLevel();
    for (TActorIterator<AActor> It(World, AActor::StaticClass(), EActorIteratorFlags::AllActors|EActorIteratorFlags::SkipPendingKill); It; ++It)
    {
        AActor* Actor = *It;
        if(Level != Actor->GetLevel()){
            continue;
        }
        if(Actor != ActorClass){
            Actor->SetOwner(ActorClass);
        }

		PrimitiveRenderMode = ESceneCapturePrimitiveRenderMode::PRM_UseShowOnlyList;
		TInlineComponentArray<UPrimitiveComponent*> PrimitiveComponents(Actor, true);
		for (UPrimitiveComponent* PrimComp : PrimitiveComponents)
		{
            PrimComp->bOnlyOwnerSee = true;
			ShowOnlyComponents.Add(PrimComp);
            ++Count;
		}
    }
    if(0<Count){
        PrimitiveRenderMode = ESceneCapturePrimitiveRenderMode::PRM_UseShowOnlyList;
    }
}
```

## `Owner Only See`が機能しない
[Scene Capture 2d owner only see not working.](https://answers.unrealengine.com/questions/746651/scene-capture-2d-owner-only-see-not-working.html)
次のようなコメントがあるので, overrideしろということらしいです.

> /** To leverage a component's bOwnerNoSee/bOnlyOwnerSee properties, the capture view requires an "owner". Override this to set a "ViewActor" for the scene. */

```
virtual const AActor* GetViewOwner() const override {return GetOwner();}
```

# ライティングを分離したい
とりあえず, ライティングチャネルを使用します. 動的な間接光などを分離するにはエンジンを大改造する必要があると思います.

# まとめ

- できること
  - キャプチャのコントロール
    - キャプチャタイミング, 解像度, ポストプロセッシングは自由
  - レベル単位で描画分離
  - ライティングの分離
    - ライティングチャネルを利用
- できないこと
  - 間接光系はライティングチャネルの制限と同様使用できない
    - ポストプロセッシングなどで誤魔化してほしい

# 余談

```cpp
UFUNCTION(BlueprintCallable, Category = "Category", meta = (WorldContext = "WorldContextObject"))
static void Function(UObject* WorldContextObject);
```

レベルスクリプトの場合, この`WorldContextObject`の実体はALevelScriptActorになります.

