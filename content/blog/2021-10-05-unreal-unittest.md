---
title: "Unreal Engine 4 単体テスト"
date: 2021-10-05T01:00:00+09:00
archives:
    - 2021-10
categories: ["blog"]
tags: ["note", "unreal"]
draft: true
---
# はじめに
# 設定
プロジェクトを作成した場合, 次のプラグインは全てONにしておきます.

<details><summary>Build.cs</summary><div>

```cs
if(TargetRules.TargetType.Editor == Target.Type){
    PrivateDependencyModuleNames.AddRange(new string[]{"FunctionalTestingEditor", "EditorTests", "FunctionalTesting", "RuntimeTests", "RHITests"});
}
```
</div></details>

<details><summary>.uproject</summary><div>

```json
"Plugins": [
    {
        "Name": "EditorTests",
        "Enabled": true
    },
    {
        "Name": "FunctionalTestingEditor",
        "Enabled": true
    },
    {
        "Name": "RHITests",
        "Enabled": true
    },
    {
        "Name": "RuntimeTests",
        "Enabled": true
    }
]
```
</div></details>

# アサーション

テストケースは, `FAutomationTestBase` の子クラスになるため, `FAutomationTestBase`の持つアサーションを使用します.

| Name         | Description                    |
| :---         | :---                           |
| TestEqual    | Test equality                  |
| TestNotEqual | Test equality                  |
| TestTrue     | Test boolean                   |
| TestFalse    | Test boolean                   |
| TestValid    | Test TSharedPointer            |
| TestInvalid  | Test TSharedPointer            |
| TestNull     | Test raw pointer               |
| TestNotNull  | Test raw pointer               |
| TestSame     | Test object equality in memory |
| TestNotSame  | Test object equality in memory |

