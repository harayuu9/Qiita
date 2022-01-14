<!--
title:   C++による各種リスト作成の効率化
tags:    C++,C++11,ゲーム制作
id:      b7f5a2ecf918b310d60b
private: false
-->
C++で各種リストをvectorで作成することがあると思います。
今回はその場合の中でもRenderingListやCollisionListなど毎フレーム更新が行われ
ソースの高速化が求められる時どの書き方をしたらどれくらい処理速度が落ちるのか確認してみたいと思う。
今回は例としてOBBのCollisionListで考えてみたいと思う。
実験する環境はVisualStudio2017のReleaseビルド。(Debugは極端にSTL系遅くなるため)

```C++:TimeObserver.h
#pragma once
#pragma comment(lib, "winmm.lib")
#include <Windows.h>
#include <vector>
#include <iostream>

using namespace std;

class TimeObserver
{
	LARGE_INTEGER					g_cntTimer = { 0 };
	LARGE_INTEGER					cntfreq = { 0 };
public:
	TimeObserver()
	{
		QueryPerformanceFrequency(&cntfreq);//カウンターの周波数
		QueryPerformanceCounter(&g_cntTimer);
	}
	void End()
	{
		LARGE_INTEGER count;
		QueryPerformanceCounter(&count);
		long long cnttime = count.QuadPart - g_cntTimer.QuadPart;//経過時間(カウント）
		float time = float(double(cnttime) / double(cntfreq.QuadPart));//経過時間(秒）
		cout << time * 1000.0f << "ms" << endl;
	}
};
```
今回はこの計測クラスをもとに行う。

## 実態をそのまま持つ

```C++:sample.cpp
#include <vector>
#include "TimeObserver.h"

struct VECTOR3
{
	float x, y, z;
}; //12byte

struct OBB
{
	VECTOR3 m_Pos;
	VECTOR3 m_NormaDirect[3];
	float m_fLength[3];
}; //60byte

int main()
{
	std::vector<OBB> ObbList;
	
	TimeObserver time;

	for (int i = 0;i < 5000000;i++)
	{
		OBB smp;
		ObbList.push_back(smp);
	}
	ObbList.clear();

	time.End();

	return 0;
}
```

**結果**

`522.125ms`

一番考えやすいのはこの形だとは思う。あえて一番低速であろう形を取り上げてみた。

##ポインタとして持つ

```C++:sample.cpp
int main()
{
	std::vector<OBB*> ObbList;
	
	TimeObserver time;

	for (int i = 0;i < 5000000;i++)
	{
		OBB *smp = new OBB;
		ObbList.push_back(smp);
	}
	for (auto delObj : ObbList)
	{
		delete delObj;
	}
	ObbList.clear();

	time.End();

	return 0;
}
```

**結果**

`722.202ms`

あれれれ？？思っていたのと全然違う。60byteのコピー作るよりfor回しのほうがコストかかるということか。
こんなことするくらいなら元のほうがマシじゃないか。
ちなみに糞ソース確定なのだがdeleteするのやめたら441.085msになった。
こうすると多少早くなるが全くもって実用性がない。

##unique_ptrで持つ

```C++:sample.cpp
int main()
{
	std::vector<std::unique_ptr<OBB>> ObbList;
	
	TimeObserver time;

	for (int i = 0;i < 5000000;i++)
	{
		std::unique_ptr<OBB> smp(new OBB);
		ObbList.push_back(move(smp));
	}
	ObbList.clear();

	time.End();

	return 0;
}
```

**結果**

`777.363ms`

当然といえば当然だが遅くなった。
ここまで来たら意地でもこの仕組みを使いたい。
何byteを超えたらポインタ持ちのほうが軽くなるのかまでやってやる。

##結論

```C++:sample.cpp
struct OBB
{
	char test[104];
}; //60byte

int main()
{
	{
		std::vector<OBB> ObbList;

		TimeObserver time;

		for (int i = 0;i < 1000000;i++)
		{
			OBB smp;
			ObbList.push_back(smp);
		}
		ObbList.clear();

		time.End();
	}
	{

		std::vector<OBB*> ObbList;

		TimeObserver time;

		for (int i = 0;i < 1000000;i++)
		{
			OBB* smp = new OBB;
			ObbList.push_back(smp);
		}
		for (auto delObj : ObbList)
		{
			delete delObj;
		}
		ObbList.clear();

		time.End();
	}

	return 0;
}
```

**結果**

``186.936ms``
``186.162ms``

私のパソコンだと大体100byte強でポインタ持ちのほうに軍配が上がった。
**一つの構造体が100byteを超えるくらいになると検討しても良いレベル**
**それ以外なら検討する余地なし！**
私自身こんな結果望んでなかったので記事にしようか悩みましたがこれもこれで良いデータだと思います。
基本的にメモリ上のコピーは結構早いので無駄にfor回すくらいならSTLを信じてメモリに負荷かけていきましょう。

## 追記
コメントよりemplace_back使ったほうが早いんじゃね？と頂いたのでそれについても検証します。

```C++:sample.cpp
#include <vector>
#include "TimeObserver.h"


struct VECTOR3
{
	float x, y, z;
}; //12byte

struct OBB
{
	VECTOR3 m_Pos;
	VECTOR3 m_NormaDirect[3];
	float m_fLength[3];
	OBB(VECTOR3 &pos, VECTOR3 NormalDirect[3], float Length[3])
	{
		m_Pos = pos;
		memcpy(m_NormaDirect, NormalDirect, sizeof(VECTOR3) * 3);
		memcpy(m_fLength, Length, sizeof(float) * 3);
	}
}; //60byte

int main()
{
         //emplaceを使う
	{
		std::vector<OBB> ObbList;

		TimeObserver time;

		for (int i = 0; i < 3000000; i++)
		{
			VECTOR3 p = { 0,0,0 };
			VECTOR3 n[3] = { {0,0,0},{0,0,0},{0,0,0} };
			float l[3] = { 0,0,0 };
			ObbList.emplace_back(p, n, l);
		}
		ObbList.clear();

		time.End();
	}
         //そのまま実態を入れる
	{
		std::vector<OBB> ObbList;

		TimeObserver time;

		for (int i = 0; i < 3000000; i++)
		{
			VECTOR3 p = { 0,0,0 };
			VECTOR3 n[3] = { { 0,0,0 },{ 0,0,0 },{ 0,0,0 } };
			float l[3] = { 0,0,0 };
			OBB tmp(p, n, l);
			ObbList.push_back(tmp);
		}
		ObbList.clear();

		time.End();
	}
         //ポインタで追加する
	{

		std::vector<OBB*> ObbList;

		TimeObserver time;

		for (int i = 0; i < 3000000; i++)
		{
			VECTOR3 p = { 0,0,0 };
			VECTOR3 n[3] = { { 0,0,0 },{ 0,0,0 },{ 0,0,0 } };
			float l[3] = { 0,0,0 };
			OBB* smp = new OBB(p, n, l);
			ObbList.push_back(smp);
		}
		for (auto delObj : ObbList)
		{
			delete delObj;
		}
		ObbList.clear();

		time.End();
	}
	getchar();

	return 0;
}
```

**結果**

``302.09ms``
``282.986ms``
``420.419ms``

もうちょっと実用的な話が出来るように中に要素突っ込んでみることにしました。
実行毎に割と結果ばらけたんですが2番目の出力が一番早くなるのは変わりませんでした。
emplaceよりpush_backのほうが早くなるのが訳わかりません。