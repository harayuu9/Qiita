<!--
title:   【C++20】 operatorのconstが今まで以上に重要になった
tags:    C++,C++20
id:      89006cb244450e75df0a
private: false
-->
# 結論
現状のC++20バックドラフトでは以下ソースではoverloadの曖昧さを指摘するようになった。

```main.cpp
struct Hoge
{
    bool operator==(const Hoge& other)
    {
        return true;
    }
};
int main()
{
    Hoge t1, t2;
    if (t1 == t2)
        return 0;
}
```

問題は単純でC ++ 20では、比較演算子の反転を新しい概念として追加しているからです。
式**a == b**を検索すると、**b == a**も同時に検索されることになります。
それを可能にするために以下のような比較演算子の反転関数が自動で追加されます。

```main.cpp
bool operator==(/*this*/ Hoge&, const Hoge&); // 上で定義してるもの
bool operator==(const Hoge&, /*this*/ Hoge&); // 反転関数
```

**==**は両方と一致するため**あいまい**を指摘するようになってました。
なのでHogeは以下のように書く必要があります。

```main.cpp
struct Hoge
{
    bool operator==(const Hoge& other) const
    {
        return true;
    }
};

// 定義される比較演算子
bool operator==(/*this*/const Hoge&, const Hoge&); // 上で定義してるもの
// 同じものになる
// bool operator==(const Hoge&, /*this*/const Hoge&); // 反転関数

// 一応これでもOK
struct Hoge2
{
    bool operator==(Hoge2& other)
    {
        return true;
    }
};
```

# 余談
VC++はこれを指摘しててくれませんでした...
clangに変えた時初めて気づきました。