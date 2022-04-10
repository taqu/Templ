---
title: "Unreal Engine 4 Postprocess"
date: 2022-04-08T00:00:00+00:00
archives:
- 2022-04
categories: ["blog"]
tags: ["note", "unreal"]
draft: false
---
# はじめに
UE4のRendererModuleはユーザが独自のパスを追加できるいくつかのコールバックがあります.
Unityのように独自のポストプロセスパスを追加したいため調査しました.

# IRendererModule のコールバック
`GetModule("Renderer")` で取得できる `IRendererModule` にある描画関連のコールバックは次の３つになります.

- PostOpaqueRender
- OverlayRender
- ResolvedSceneColor

テンプレートとしてのコードは次のようになります. 各パスでは何もしていないですが, RenderDocでパスの流れを確認するにはこれで十分です.

<details><summary>RenderHookComponent.cpp</summary><div>

```cpp
#include "RenderHookComponent.h"
#include <Modules/ModuleManager.h>
#include <Modules/ModuleInterface.h>
#include <RendererInterface.h>
#include <RenderUtils.h>
#include <ClearQuad.h>

URenderHookComponent::URenderHookComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

void URenderHookComponent::BeginPlay()
{
    Super::BeginPlay();
    // Get Rendere Module
    IRendererModule* RendererModule = (IRendererModule*)(FModuleManager::Get().GetModule("Renderer"));

    // Add delegates
    FPostOpaqueRenderDelegate OverlayRenderDelegate;
    OverlayRenderDelegate.BindUObject(this, &URenderHookComponent::OnOverlayRender);
    OnOverlayRenderHandle_ = RendererModule->RegisterOverlayRenderDelegate(OverlayRenderDelegate);

    FPostOpaqueRenderDelegate PostOpaqueRenderDelegate;
    PostOpaqueRenderDelegate.BindUObject(this, &URenderHookComponent::OnPostOpaqueRender);
    OnPostOpaqueRenderHandle_ = RendererModule->RegisterPostOpaqueRenderDelegate(PostOpaqueRenderDelegate);

    OnResolvedSceneColorHandle_ = RendererModule->GetResolvedSceneColorCallbacks().AddUObject(this, &URenderHookComponent::OnResoledSceneColor);
}

void URenderHookComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    Super::EndPlay(EndPlayReason);
    IRendererModule* RendererModule = (IRendererModule*)(FModuleManager::Get().GetModule("Renderer"));
    if(OnOverlayRenderHandle_.IsValid()){
        RendererModule->RemoveOverlayRenderDelegate(OnOverlayRenderHandle_);
        OnOverlayRenderHandle_.Reset();
    }
    if(OnPostOpaqueRenderHandle_.IsValid()){
        RendererModule->RemovePostOpaqueRenderDelegate(OnPostOpaqueRenderHandle_);
        OnPostOpaqueRenderHandle_.Reset();
    }
    if(OnResolvedSceneColorHandle_.IsValid()){
        RendererModule->GetResolvedSceneColorCallbacks().Remove(OnResolvedSceneColorHandle_);
        OnResolvedSceneColorHandle_.Reset();
    }
}

void URenderHookComponent::OnOverlayRender(FPostOpaqueRenderParameters& PostOpaqueRenderParameters)
{
    FRHICommandList& RHICmdList = *PostOpaqueRenderParameters.RHICmdList;
    FRHIRenderPassInfo RenderPassInfo;
    RHICmdList.BeginRenderPass(RenderPassInfo, TEXT("OverlayRender"));
    RHICmdList.EndRenderPass();
}

void URenderHookComponent::OnPostOpaqueRender(FPostOpaqueRenderParameters& PostOpaqueRenderParameters)
{
    FRHICommandList& RHICmdList = *PostOpaqueRenderParameters.RHICmdList;
    FRHIRenderPassInfo RenderPassInfo;
    RHICmdList.BeginRenderPass(RenderPassInfo, TEXT("PostOpaqueRender"));
    RHICmdList.EndRenderPass();
}

void URenderHookComponent::OnResoledSceneColor(FRHICommandListImmediate& RHICmdList, class FSceneRenderTargets& SceneContext)
{
    FRHIRenderPassInfo RenderPassInfo;
    RHICmdList.BeginRenderPass(RenderPassInfo, TEXT("ResoledSceneColor"));
    DrawClearQuad(RHICmdList, FLinearColor::White);
    RHICmdList.EndRenderPass();
}
```
</div></details>

ローカルマルチプレイを考慮した次のような場合, 
{{<figure src="/images/blog/localmulti00.jpg" alt="Local Multiplay">}}

レンダリングパスは次のようになります. `Post Opaque`と`Overlay`は各プレーヤのビュー毎に, `Post Resolved Scene Color`はシーン全体で一度だけ呼び出されます.
これはデフォルトの単純なシーンのパスですので, 例えば`Custom Depth`などの機能を使用した場合にパスが追加されると思います.

{{<figure src="/images/blog/postprocessing00.jpg" alt="Postprocessing">}}

# 独自ポストプロセスパス
パスを追加する箇所は, `Overlay`か`Post Resolved Scene Color`が候補になると思います.

全てのポストプロセスを自前で用意するならばこれで話は終わりなのですが, やはりPostProcessingVolumeも使いたいと思うでしょう.
再利用できないかと調べましたが, 相変わらず各機能がエンジンに深く関連しているため難しいように思います.

ポストプロセスを行う場合, 一般的にシーンのカラーをレンダーターゲット以外にコピーする必要があります. 
PostProcessingVolumeは最小でも次の処理をしています, この処理が重複するオーバーヘッドも考慮しないといけません.

- PostProcessInput0へSceneColorをコピー
- CustomStencilへステンシルをコピー

# まとめ
独自のパスを追加できるのですが, PostProcessingVolumeを再利用が難しいためオーバーヘッドが気になります.

