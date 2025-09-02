---
title: "C++ Coroutine"
date: 2025-08-21T00:00:00+09:00
archives:
    - 2025-08
categories: ["blog"]
tags: ["note", "C++"]
draft: true
---

# はじめに
C++20でキーワード`co_return`, `co_await`, `co_yeild`と低レベルの言語仕様が決められました. そして, C++23では`iterator`モデルでコルーチンをを実装する`<generator>`が追加されました。

# サンプル
"範囲for文"以外に, `std::generator`の状態で遅延することができます.

```cpp
#include <cstdint>
#include <generator>
#include <ranges>
#include <iostream>

// Generate even numbers
std::generator<int32_t> evens()
{
    int32_t n=0;
    while(true){
        co_yield n; // yield and return value
        n += 2;
    }
}

int main(void){
    // #1 Use operators of views
    for(int32_t n : evens() | std::views::take(10)){
        std::cout << n << std::endl;
    }

    // #2 Handle the iterator manually
    int32_t count = 0;
    std::generator<int32_t> gen = evens();
    for(auto itr=gen.begin(); itr!=gen.end(); ++itr){
        std::cout << *itr << std::endl;
        if(10<=++count){
            break;
        }
    }
    return 0;
}
```

# まとめ
C#のIEnumeratorのようなことができそうです。