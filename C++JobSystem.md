<!--
title:   C++でマルチスレッドJobSystemを作ってみた
tags:    C++,MultiThread
id:      b551602a67bf2917cc22
private: false
-->
UnityのC#JobSystemを使っていて中々に便利なので使用感の全く同じものをC++で出来ないかなと思い
実装出来たのでそのメモです。

[リポジトリ(MIT)](https://github.com/harayuu9/CppJobSystem)

#JobSystem
C#JobSystemはUnity開発のマルチスレッドで動作するジョブを管理出来るライブラリです。
![Untitled Diagram.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163136/895be412-8ff1-a4de-fcdc-e163ea50e197.png)
job2,job4ともに終わってからjob5を実行する等の
若干複雑なフローでも割と分かりやすく書くことが出来ます。
#CppJobSystem
JobSystemのC++版を考えていきます。
##JobHandle
まずjobを管理するJobHandleから考えていきます。
これは単純にJobを順番に持つ配列を持って順次実行するだけです。
今回はラムダ式で処理を入れられるようにfunctionを使いました。

```cpp:JobHandle.h
#pragma once
#include <functional>
#include <vector>

using std::function;
using std::vector;

class JobHandle
{
	vector<function<void()>> funcs;
public:

	void Complete()
	{
		for (auto& func : funcs)
			func();
	}
};
```
CompleteでJobの実行を待ちます。

##Job
Jobを定義していきます。
C#ではIJobというInterfaceを継承した型がそのままJobになっていました。
なのでJobというclassを継承した型をJobにする形で考えます。
Execute()を継承先で実装します。これが実行するJobの中身になります。
Schedule()がMainThreadからJobを発行する関数です。
そのままJobHandleに追加しています。

```cpp:Job.h
#pragma once
#include "JobHandle.h"

class Job
{
public:
	JobHandle Schedule();
	JobHandle& Schedule(JobHandle& handle);

	virtual void Execute() = 0;
};
```
`cpp:Job.cpp
#include "Job.h"

JobHandle Job::Schedule()
{
	JobHandle handle;
	return Schedule(handle);
}

JobHandle& Job::Schedule(JobHandle& handle)
{
	handle.funcs.push_back([this]
		{
			Execute();
		});
}
`

##JobParallelFor
ParallelFor版のJobです。
基本的な流れは変わりません。
Schedule()で実行するForの回数を引数として渡します。
そしてExecute()の引数にindexが入っています。ループの添え字を受け取れるようにしておきます。
Schedule()の実装は実行回数を動作する端末のThread数で分割してThread分けをしています。

```cpp:JobParallelFor.h
#pragma once
#include "JobHandle.h"

class JobParallelFor
{
public:	
	JobHandle Schedule(unsigned int length);
	JobHandle& Schedule(unsigned int length,JobHandle& handle);

	virtual void Execute(int index) = 0;
};
```
`cpp:JobParallelFor.cpp
#include "JobParallelFor.h"

JobHandle JobParallelFor::Schedule(unsigned int length)
{
	JobHandle handle;
	return Schedule(length, handle);
}

JobHandle& JobParallelFor::Schedule(unsigned int length, JobHandle& handle)
{
	handle.funcs.push_back([length,this]
	{	
		auto threadCount = std::thread::hardware_concurrency();
		vector<std::thread> threads;
		auto threadTask = length / threadCount;
		if (threadTask == 0) {
			threads.push_back(std::thread([&]
			{
				for (unsigned int index = 0; index < length; index++)
				{
					Execute(index);
				}
			}));
		}
		else
		{
			for (unsigned int i = 0; i < threadCount - 1; i++)
			{
				threads.push_back(std::thread([i,&threadTask,this]
				{
					for (auto index = threadTask*i; index < threadTask * i + threadTask; index++)
					{
						Execute(index);
					}
				}));
			}
			threads.push_back(std::thread([&threadTask,&threadCount,&length, this]
			{
				for (auto index = threadTask * (threadCount-1); index < length; index++)
				{
					Execute(index);
				}
			}));
		}
		for (auto& t : threads)
			t.join();
	});
	return handle;
}

```

##使い方
継承してJobを実装していきます。
意味のないある程度処理負荷のある二つのJobを実装してみました。
AddJob→SubJobの流れで実行しています。
Completeを書くタイミングやJobHandleをコントロールすることで様々なフローに対応できると思います。

```cpp:JobParallelFor.cpp
#include "JobParallelFor.h"
#include <math.h>

constexpr auto MAX = 100000000;

class AddJob : public JobParallelFor
{
public:
	AddJob(vector<int>& add) : addList{ add }
	{}

	vector<int>& addList;

	void Execute(int index)
	{
		addList[index] = (int)(cosf(sinf((float)index)) * 1000.0f);
	}
};

class SubJob : public JobParallelFor
{
public:
	SubJob(vector<int>& add) : addList{ add }
	{}

	vector<int>& addList;

	void Execute(int index)
	{
		addList[index] = (int)(sinf(cosf(sinf((float)addList[index])) * 1000.0f));
	}
};

int main()
{
	vector<int> addList;
	addList.resize(MAX);

	auto start = std::chrono::system_clock::now(); // 計測開始時間

	for (auto i = 0; i < 5; i++) {
		AddJob job(addList);
		SubJob job2(addList);
		auto handle = job.Schedule(MAX);
		handle = job2.Schedule(MAX, handle);
		handle.Complete();
	}
	auto end = std::chrono::system_clock::now();
	auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
	printf("%lld", elapsed);
	return 0;
}
```

##まとめ
メインスレッドを止めてしまうのでhanlde.Complete()自体を別スレッドで動かした方が良いかもしれません。
このような仕組みを他に知らないですが設計も実装もスマートで良いですね。