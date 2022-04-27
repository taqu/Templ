---
title: "Ureal Engine 4 `stat`データとプロファイルデータの取得"
date: 2022-04-25T00:00:00+09:00
archives:
    - 2022-04
categories: ["blog"]
tags: ["note", "unreal"]
draft: false
---

# はじめに
Unreal Engine 4 (UE4)はプロファイルデータを収集・表示するシステムが内臓されているのはいいのですが, コマンドに渡すグループ名に何があるのか知っていなければならず, 表示も分かりやすいとは言えません.
収集されたデータだけ取得することができれば, 好きなように整理して表示したり, ネットワーク越しに収集することも自由にできます.

# stat コマンド

## UE4のstatコマンドのデータ更新の流れ

次のような流れでデータが更新されるため, statコマンドが一度も実行されていない, 更新すべきカテゴリがない場合は, `ThreadStatsData::Get().Latest`はnullになっています.

1. stat XXXX コマンドが入力される
2. XXXXカテゴリがシステムに登録されていれば, `FLatestGameThreadStatsData::Get().Latest`を作成またはカテゴリを追加する
3. FLatestGameThreadStatsData::Get().Latestを毎フレーム更新する

次のように`-nodisplay`を付けてstatコマンドを実行すれば, 表示はさせないでデータ更新だけさせることができます. コマンド文字列のどこかに`-nodisplay`があれば, 表示フラグが`false`になります. これのフラグはEpicお得意のたったひとつのグローバル静的変数ですので, 最後のstatコマンドだけに付けても非表示にできます.

```cpp
void Exec(const FString& Cmd)
{
    UWorld* World = GWorld;
    if(nullptr == World) {
        return;
    }
    APlayerController* PlayerController = UGameplayStatics::GetPlayerController(World, 0);
    if(nullptr == PlayerController) {
        return;
    }
    PlayerController->ConsoleCommand(Cmd, false);
}

void ExecStat(const TCHAR* Group)
{
    FString Cmd = TEXT("STAT ");
    Cmd.Append(Group);
    Cmd.Append(TEXT(" -nodisplay"));
    Exec(Cmd);
}
```

```cpp
#if STATS
#	include <Stats/Stats2.h>
#	include <Stats/StatsData.h>

void DrawStatMessage(FComplexStatMessage& StatMessage, FVector2D LeftTop)
{
	FString StatDisplay = StatMessage.GetDescription();
	if(StatDisplay.Len() <= 0) {
		StatDisplay = StatMessage.GetShortName().GetPlainNameString();
	}
	bool IsCycle = StatMessage.NameAndInfo.GetFlag(EStatMetaFlags::IsCycle);
	FString Inclusive;
	FString Exclusive;
	if(IsCycle) {
		Inclusive = FString::Printf(TEXT("%1.2f ms"), FPlatformTime::ToMilliseconds(StatMessage.GetValue_Duration(EComplexStatField::IncAve)));
		Exclusive = FString::Printf(TEXT("%1.2f ms"), FPlatformTime::ToMilliseconds(StatMessage.GetValue_Duration(EComplexStatField::ExcAve)));
	} else {
		switch(StatMessage.NameAndInfo.GetField<EStatDataType>()) {
		case EStatDataType::ST_double: {
			Inclusive = FString::Printf(TEXT("%2.3f"), StatMessage.GetValue_double(EComplexStatField::IncAve));
			Exclusive = FString::Printf(TEXT("%2.3f"), StatMessage.GetValue_double(EComplexStatField::ExcAve));
		} break;
		case EStatDataType::ST_int64: {
			Inclusive = FString::FormatAsNumber(static_cast<int32>(StatMessage.GetValue_int64(EComplexStatField::IncAve)));
			Exclusive = FString::FormatAsNumber(static_cast<int32>(StatMessage.GetValue_int64(EComplexStatField::ExcAve)));
		} break;
		}
	}
	DrawText(LeftTop, *StatDisplay); LeftTop.X += 256.0f;
	DrawText(LeftTop, *Inclusive); LeftTop.X += 128.0f;
	DrawText(LeftTop, *Exclusive);
}

void DrawStats()
{
	FGameThreadStatsData* StatsData = FLatestGameThreadStatsData::Get().Latest;
	if(nullptr == StatsData) {
		return;
	}
	FVector2D Position = FVector2D::ZeroVector;
	for(int32 i = 0; i < StatsData->ActiveStatGroups.Num(); ++i) {
		DrawFormat(Position, TEXT("{0}  Inclusive  Exclusive"), {*StatsData->GroupNames[i].ToString()}, FLinearColor::Green);
		Position.Y += Height;

		Position.X += Height;
		FActiveStatGroupInfo& StatGroupInfo = StatsData->ActiveStatGroups[i];
		for(FComplexStatMessage& StatMessage: StatGroupInfo.CountersAggregate) {
			DrawStatMessage(StatMessage, Position, RowWidth, LabelWidth);
			Position.Y += Height;
		}
		Position.X = 0.0f;
	}
}
#endif
```

# ProfileGPU コマンド
コマンド実行後次のフレームの描画プロファイルを表示するコマンドです. エディタ上では, プロファイラウィンドウを表示します.
`GPUProfiler.h`の`FGPUProfilerEventNodeFrame`がデータの本体ですが, これを取得することは困難と思われたので, `Output Log`に出力させてその文字列を取得する方針にしました.

## `Output Log`
先ず, `Output Log`を取得します. `FOutputDevice`を継承したクラスを`GLog`に追加すると`Serialize`が呼び出されます.

```cpp
#pragma once
#include <Misc/OutputDevice.h>

class FMyOutputDevice : FOutputDevice
{
public
    FMyOutputDevice()
    {
	    if(nullptr != GLog) {
		    GLog->AddOutputDevice(this);
	    }
    }

    ~FMyOutputDevice()
    {
	    if(nullptr != GLog) {
		    GLog->RemoveOutputDevice(this);
	    }
    }

    virtual void Serialize(const TCHAR* V, ELogVerbosity::Type Verbosity, const FName& Category) override;
    virtual void Serialize(const TCHAR* V, ELogVerbosity::Type Verbosity, const FName& Category, double Time) override;
};
```

## ProfileGPU 出力
`Output Log`に出力させるには次の3つのコマンドを実行します. ProfileGPU自体は`GTriggerGPUProfile`を立てるだけで, 次描画パスでデータ収集が行われます.

1. `r.RHISetGPUCaptureOptions 1` : `ProfileGPU`の動作に必要なオプションとその他便利な設定をまとめて変更します.
2. `r.ProfileGPU.ShowUI 0` : プロファイラウィンドウを表示しません. 代わりに`Output Log`に出力します.
3. `ProfileGPU` : プロファイルデータ取得のフラグを立てます.

RHIの描画スレッドをOFFにしなければ`ProfileGPU`は使えないので, `GTriggerGPUProfile`にアクセスしても問題はないと思われます. 監視していればコマンドの処理終了はわかると思います.
終了した時点で状態を元に戻しておくとよいでしょう, 特に1.は重たい処理の設定が入っています.

1. `r.RHISetGPUCaptureOptions 0`
2. `r.ProfileGPU.ShowUI 1`

# まとめ
プロファイルデータの取得をしてみました. コンソールコマンドはコンソール機などでは使えたものではないので, 専用のUIを作ったほうがよいでしょう.
