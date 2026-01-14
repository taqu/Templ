---
title: "Cpp Module"
date: 2026-01-10T00:00:00+09:00
archives:
    - 2026-01
categories: ["blog"]
tags: ["note", "C++"]
draft: true
---
# はじめに
C++のモジュールについて, どの説明を見ても単純な例しかなく, 特にパーティションをつかって複数ファイルに分けて実装する方法が解りませんでした.
一番良い方法はまだわかりませんが, プロジェクトで使えそうな方法を模索してみました.

# 拡張子
特に意味はないですが, 区別するためにモジュールインターフェイスは`.ixx`, モジュール実装ファイルは`cppm`とします.

# CMake

`CMakeLists.txt`から, `MODULE_HEADERS`や`MODULE_SOURCES`は公式ではないです, 念のため.

<details><summary>CMakeLists.txt</summary><div>

```cmake {filename="CMakeLists.txt"}
cmake_minimum_required(VERSION 3.31)

set(CMAKE_CONFIGURATION_TYPES "Debug" "Release")

set(PROJECT_NAME module CXX)
project(${PROJECT_NAME})

set(HEADER_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/include")
set(SOURCE_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/src")

include_directories(AFTER ${HEADER_ROOT})

########################################################################
# Sources
set(HEADERS
    "${HEADER_ROOT}/mymodule.h"
)

set(SOURCES
    "${SOURCE_ROOT}/mymodule.cpp"
    "${SOURCE_ROOT}/main.cpp")

set(MODULE_HEADERS
    "${HEADER_ROOT}/mymodule.ixx"
    "${HEADER_ROOT}/math.ixx"
    "${HEADER_ROOT}/utility.ixx"
)

set(MODULE_SOURCES
    "${SOURCE_ROOT}/mymodule.cppm"
    "${SOURCE_ROOT}/math.cppm"
    "${SOURCE_ROOT}/utility.cppm"
)

source_group("include" FILES ${HEADERS} ${MODULE_HEADERS})
source_group("src" FILES ${SOURCES} ${MODULE_SOURCES})

set(FILES ${HEADERS} ${SOURCES})

add_executable(${PROJECT_NAME} ${FILES})
target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_20)
set_property(TARGET ${PROJECT_NAME} PROPERTY CXX_STANDARD 23)
target_sources(${PROJECT_NAME}
    PUBLIC 
    ${MODULE_HEADERS}
    PRIVATE ${MODULE_SOURCES})

set(OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/bin")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG "${OUTPUT_DIRECTORY}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE "${OUTPUT_DIRECTORY}")

if(MSVC)
    set(DEFAULT_CXX_FLAGS "/DWIN32 /D_WINDOWS /D_MSBC /W4 /WX- /nologo /fp:precise /arch:AVX2 /Zc:wchar_t /TP /Gd /utf-8")
    if("1800" VERSION_LESS MSVC_VERSION)
        set(DEFAULT_CXX_FLAGS "${DEFAULT_CXX_FLAGS} /EHsc")
    endif()

    set(CMAKE_CXX_FLAGS "${DEFAULT_CXX_FLAGS}")
    set(CMAKE_CXX_FLAGS_DEBUG "/D_DEBUG /MDd /Zi /Ob0 /Od /RTC1 /Gy /GR- /GS /Gm-")
    set(CMAKE_CXX_FLAGS_RELEASE "/MD /O2 /GL /GR- /DNDEBUG")

elseif(UNIX)
    set(DEFAULT_CXX_FLAGS "-Wall -O0 -g -std=c++23 -std=gnu++23 -march=x86-64-v3 -fmodules-ts")
    set(CMAKE_CXX_FLAGS "${DEFAULT_CXX_FLAGS}")
elseif(APPLE)
endif()

set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT ${PROJECT_NAME})
set_target_properties(${PROJECT_NAME}
    PROPERTIES
        OUTPUT_NAME_DEBUG "${PROJECT_NAME}" OUTPUT_NAME_RELEASE "${PROJECT_NAME}"
        VS_DEBUGGER_WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}")
```
</div></details>

# 従来のファイル郡
usingなどをexportできるといいのですが, まだ全てをモジュールに移行することは難しそうです.

```cpp {filename="mymodule.h"}
#ifndef INC_MYMODULE_H_
#define INC_MYMODULE_H_
#include <cstdint>

namespace mymodule
{
using s8 = int8_t;
using s16 = int16_t;
using s32 = int32_t;
using s64 = int64_t;
using u8 = uint8_t;
using u16 = uint16_t;
using u32 = uint32_t;
using u64 = uint64_t;
using f32 = float;
using f64 = double;

static constexpr f32 Epsilon = 1.0e-6f;

void println(f32 x);
} // namespace mymodule
#endif // INC_MYMODULE_H_
```

```cpp {filename="mymodule.cpp"}
#include "mymodule.h"
#include <cstdio>

namespace mymodule
{
void println(f32 x)
{
    printf("%f\n", x);
}
} // namespace mymodule
```

# モジュール
## モジュールインターフェイス
プライマリモジュールは,

```cpp {filename="mymodule.ixx"}
module;
#include "mymodule.h"
export module mymodule;
export import :math;
export import :utility;

namespace mymodule
{
export bool isEqual(f32 x0, f32 x1, f32 epsilon = Epsilon);
}
```

パーティションを２つ,

```cpp {filename="math.ixx"}
module;
#include "mymodule.h"
export module mymodule:math;
namespace mymodule
{
export class Vector2
{
public:
    f32 lengthSqr();
    f32 length();
    void normalize();
    float x_;
    float y_;
};
} // namespace mymodule
```

```cpp {filename="utility.ixx"}
module;
#include "mymodule.h"
export module mymodule:utility;
import :math;

namespace mymodule
{
export void println(bool x);
export void println(const Vector2& x);
} // namespace mymodule
```

## モジュール実装

プライマリモジュールは, 警告が出るのでこれではだめらしいです.

```cpp {filename="mymodule.cppm"}
module;
#include "mymodule.h"
#include <cmath>
module mymodule;

namespace mymodule
{
bool isEqual(f32 x0, f32 x1, f32 epsilon)
{
    return (std::abs)(x0 - x1) < epsilon;
}
} // namespace mymodule
```

パーティションを２つ,

```cpp {filename="math.cppm"}
module;
#include "mymodule.h"
#include <cmath>
module mymodule:math;
import :math;

namespace mymodule
{
f32 Vector2::lengthSqr()
{
    return x_ * x_ + y_ * y_;
}

f32 Vector2::length()
{
    return ::sqrt(lengthSqr());
}

void Vector2::normalize()
{
    f32 l = length();
    if(l < Epsilon) {
        x_ = 0.0f;
        y_ = 0.0f;
        return;
    }
    l = 1.0f / l;
    x_ *= l;
    y_ *= l;
}
} // namespace mymodule
```

```cpp {filename="utility.cppm"}
module;
#include "mymodule.h"
#include <cstdio>
module mymodule:utility;
import :utility;

namespace mymodule
{
void println(bool x)
{
    printf("%s\n", x ? "true" : "false");
}

void println(const Vector2& x)
{
    printf("(%f, %f)\n", x.x_, x.y_);
}
} // namespace mymodule
```

# main実装
```cpp {filename="math.cpp"}
import mymodule;
#include "mymodule.h"

int main(void)
{
    using namespace mymodule;
    Vector2 v0(1.0f, 2.0f);
    println(v0);
    println(v0.length());
    println(isEqual(v0.length(),1.0f));
    v0.normalize();
    println(isEqual(v0.length(),1.0f));
    return 0;
}
```

# まとめ
警告が出る点がまだ完璧ではないですが, 手始めに使えそうな構成になっていると思います.

