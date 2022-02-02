<!--
title:   LINQ For Cpp 
tags:    C++,LINQ
id:      9abf6754177239041c7c
private: false
-->
# 概要
* C++で使えるLINQライブラリでネイティブに近い速度が出ます。
* また遅延実行を行う事によってネイティブより早く動いたりもします。
* Github Actionsで随時パフォーマンス測定しています。参考にどうぞ
  https://harayuu9.github.io/LinqForCpp/

# 環境
C++17以上
VC++, clang, gcc

# 導入
githubリポジトリのReleaseからダウンロードして導入します。
https://github.com/harayuu9/LinqForCpp
最新のReleaseの**LinqForCpp.zip**をダウンロードして展開します。
**Linqフォルダ**もしくは**SingleHeader/Linq.hpp**をプロジェクトに入れます。

Linqフォルダを要れた場合は**Linq/Linq.h**
SingleHeaderを要れた場合は**Linp.hpp**をインクルードします。

## Linq.h vs SingleHeader/Linq.hpp
どちらも使える関数は同じですがSingleHeaderじゃないほうがデバッグビリティが良いのでおススメです。

# サンプル
といってもLINQなので使える関数は他の解説に投げつけます。
https://qiita.com/nskydiving/items/c9c47c1e48ea365f8995

## 基本

`main.cpp
#include "Linq.h"
// もしくは
#include "Linq.hpp"

int main()
{
    // std::begin(arr), std::end(arr)を呼べるコンテナであればSTLでも配列でも自作コンテナでもなんでも使えます
    std::vector<int> arr = { 0, 3, 6, 1, 60, 35 };

    // コンテナに対して "<<" 演算子でLinqします。
    auto strArray = arr
        << linq::Where( []( const int val ) { return val > 10; } )
        << linq::Select( []( const int val ) { return std::to_string( val ); } );

    // マクロを使うことでより簡単にC#に近い形で掛けます
    auto macroStrArray = arr
        << WHERE( val, val > 10 )
        << SELECT( val, std::to_string(val) );

    // linq::ToVector(), linq::ToList() で
    // std::vector,std::listに変換できます
    auto str = strArray << linq::ToVector();

    // 範囲Forで回せます
    // もしLinqの結果をコンテナで受け取る必要のない場合 
    // linq::ToVector()でコンテナ化するより高速に動作する事が多いです
    for ( auto&& val : strArray )
    {
        std::cout << val << " ";
    }
    // {"60", "35"}
}
`

## Allocation
STLをデフォルトのAllocatorで使えるプロジェクトはここは読み飛ばしても大丈夫です。
**Linq/Allocator.h**を書き換えることで独自のAllocatorを差し込むことが出来ます。(ToVector()等以外にも内部でAllocationを起こす関数があるので注意してください)
Allocator.hのサンプルを書いておきます。

``` Allocator.h
#pragma once
#include <cstdlib>

namespace linq {

// custom Allocator
template<class T>
struct Allocator
{
    using value_type = T;

    Allocator() {}

    template <class U>
    Allocator(const Allocator<U>&) {}

    T* allocate(const std::size_t n)
    {
        return static_cast<T*>(::operator new(sizeof(T)*n));
    }

    void deallocate(T* p, const std::size_t n)
    {
        ::operator delete(p, n);
    }

    template <class U>
    bool operator==( const Allocator<U>&) const
    {
        return true;
    }

    template <class U>
    bool operator!=(const Allocator<U>&) const
    {
        return false;
    }
};
}
```

# 最後に
足りない機能や高速化についてはIssueかPRしたください。