<!--
title:   C++でECSを作ってみた 
tags:    C++,ECS,ゲームプログラミング
id:      e15b02e3b0f3081d729b
private: false
-->
#はじめに
ここでいうECSはUnityが掲げているDOTSの一環として導入されているECS(Entity Component System)になります。
この記事を読むにあたってUnityECSの概念が多少分かっていたほうが良いと思います。

他人任せですが偉大なるテラシュールブログのECS関係の記事を見たら分かると思いますが
特に今回に影響しそうなメモリ辺りの記事のリンクを貼らせていただきます。
[Unity】ECSのメモリレイアウトとその周辺 テラシュールブログ](http://tsubakit1.hateblo.jp/entry/2018/06/06/015645 "【Unity】ECSのメモリレイアウトとその周辺")

記事内にコードを載せていますがいくつかのコードは実装を省いていますので全体はここに置いておきます。
[githubリポジトリ](https://github.com/harayuu9/EntityComponentSystemCpp)

#環境
Visual Studio2019 C++17
動的型情報(RTTI) Off

#C++
C++17,20でconstexprが使いやすくなり黒魔術のようなTemplate文をあまり書かなくてもコンパイル時計算がやりやすくなりました。
それらを利用してArchetypeをコンパイル時生成してその他の型情報を扱うところもコンパイル時計算に出来たらいいんじゃね？という甘い試みで初めて見ました。

#全体像
全体の大まかな構想としてはWorldの中にChunkとSystemが複数あってChunk内にArchetypeとComponentDataがあるようなイメージです。
小さい部分から考えていきたいと思います
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163136/6dbaf42b-13d7-cc02-a696-773ea93aeaf9.png)

#Archetype
ArchetypeとはChunkが持っているデータ型の情報を保管かつ他のArchetypeとの比較をするものになります。
型情報といっても必要な要件は比較が出来る事とサイズを取得出来ることですので組み込みの**type_info型**を使えば何とかなりそうですがRTTIを切っているので要件を満たす**TypeInfo型**を作ります
##TypeInfo
サイズはsizeof(T)で取得できますが型比較は難しいので文字列リテラルを利用したマクロで解決します。
またコンパイル時計算に対応したいため全ての関数でconstexpr対応します。
TypeInfoの一部実装を省いていますので詳細はgithubのソースコードを見てください。

```C++:TypeInfo.h
#define DECLARE_TYPE_INFO(T) 						   \
public:												   \
	static constexpr std::string_view getTypeName()	   \
	{												   \
		return #T;									   \
	}												   \
	void _dumyFunction() = delete

namespace type {
template<typename T>
struct CallGetTypeName
{
private:
	template<typename U> static auto Test( int )
	-> decltype(U::getTypeName(), std::true_type());
	template<typename U> static auto Test( ... )
	-> decltype(std::false_type());
public:
	using Type = decltype(Test<T>( 0 ));
};

template<typename T>
constexpr bool cCallGetTypeName = CallGetTypeName<T>::Type::value;
}

class TypeInfo
{	
	constexpr explicit TypeInfo( const std::string_view typeName, const std::size_t size )
	: mTypeName( typeName ), mSize( size ) 
	{
	}
	
public:
	constexpr TypeInfo() : mTypeName( "0" ), mSize( 0 )
	{
	}
	
	constexpr bool operator==( const TypeInfo& other ) const
	{
		return mTypeName.data() == other.mTypeName.data();
	}

	template<class T, typename = std::enable_if_t<type::cCallGetTypeName<T>>>
	static constexpr TypeInfo create()
	{
		return TypeInfo( T::getTypeName(), sizeof(T) );
	}

private:
	std::string_view mTypeName;
	std::size_t mSize;
};
```

`C++:Sample.h
struct SampleType
{
    DECLARE_TYPE_INFO(TestType);
};
constexpr TypeInfo cSampleType = TypeInfo::create<SampleType>();
`

SFINAEを使ってgetTypeName()制約を付けます。SFINAEの話はしません。
このようにすると文字列リテラルを比較条件として使用するTypeInfoが出来ます。

##IComponentData
これで後は実装としたいのですが先にComponentDataの基底クラスとして定義するIComponentDataを作成しておきましょう。
チャンク内を自由にmemcopy,memmove等を行いたいので**trivial**,**trivially_destructiable**制約とTypeInfoで扱える用の制約を付けます。
ComponentDataに対して後で細工をすることもあると思うので**DECLARE_TYPE_INFO()**の置き換えマクロも用意しておきます。

```C++:IComponentData.h
#define ECS_DECLARE_COMPONENT_DATA(T)					\
DECLARE_TYPE_INFO( T )

struct IComponentData
{
};

constexpr auto cMaxComponentSize = 16;

template<class T>
constexpr bool cIsComponentData = std::is_base_of_v<IComponentData, T> && std::is_trivial_v<T> &&
	std::is_trivially_destructible_v<T> && type::cCallGetTypeName<T>;

```

##実装
**TypeInfo**を複数保持してその型までのMemoryOffset等を取得する関数を付けるとそのままArchetypeとして使えます。
またTypeInfo同様にコンパイル時計算に対応したいため全ての関数でconstexpr対応します。
Archetypeの重要な部分を切り抜いて説明しますので詳細はgithubのソースコードを見てください。

```C++:Archetype.h

struct Archetype
{
	template<typename ...Args>
	static constexpr Archetype create()
	{
		Archetype result;
		result.createImpl<Args...>();

		for ( auto i = 0; i < result.mArchetypeSize - 1; ++i )
		{
			for ( auto j = i + 1; j < result.mArchetypeSize; ++j )
			{
				if ( result.mTypeDataList[i].getName() > result.mTypeDataList[j].getName() )
				{
					const auto work = result.mTypeDataList[i];
					result.mTypeDataList[i] = result.mTypeDataList[j];
					result.mTypeDataList[j] = work;
				}
			}
		}

		for ( auto i = 0; i < result.mArchetypeSize; i++ )
		{
			result.mArchetypeMemorySize += result.mTypeDataList[i].getSize();
		}
		return result;
	}

	template<typename T>
	constexpr Archetype& addType()
	{
		constexpr auto newType = TypeInfo::create<T>();
		mArchetypeMemorySize += sizeof( T );
		for ( std::size_t i = 0; i < mArchetypeSize; ++i )
		{
			if ( mTypeDataList[i].getName() > newType.getName() )
			{
				for ( auto j = mArchetypeSize; j > i; --j )
				{
					mTypeDataList[j] = mTypeDataList[j - 1];
				}
				mTypeDataList[i] = newType;
				++mArchetypeSize;
				return *this;
			}
		}

		mTypeDataList[mArchetypeSize] = newType;
		mArchetypeSize++;

		return *this;
	}
private:
	template<typename Head, typename ...Tails, typename = std::enable_if_t<cIsComponentData<Head>>>
	constexpr void createImpl()
	{
		mTypeDataList[mArchetypeSize] = TypeInfo::create<Head>();
		mArchetypeSize++;
		if constexpr ( sizeof...( Tails ) != 0 )
			createImpl<Tails...>();
	}

	TypeInfo mTypeDataList[cMaxComponentSize];
	std::size_t mArchetypeMemorySize = 0;
	std::size_t mArchetypeSize = 0;
};
```
基本的にやっていることは可変長Templateを展開してTypeInfoを作っているだけです。
ただしソートを行ってます。これは **Archetype::create\<Position,Scale\>() == Archetype::create\<Scale,Position\>()**を成立させるために行っています。使うときに型の順番を意識することは無駄で面倒なことです。
ソートの基準は何でも良いですが型事に一定の値が入っている文字列リテラルを基準に並び替えを行ってます。

#Chunk
一番重要な部分だと思われるChunkです。
上の図のようにChunkの中にはArchetypeとComponentDataが含まれています。
ComponentDataと言っていますが**Archetype::getArchetypeMemorySize()*Chunk内のEntity数**以上のbyte列です。
下の図ではArchetype::getArchetypeSize()が2の時の例です。
ComponentDataは各領域に連続して配置します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163136/9ed1dca6-7959-e4e8-333f-0daf18834bbe.png)

```C++:Chunk.h
class Chunk
{
public:
	static Chunk create( const Archetype& archetype, std::uint32_t capacity = 1 );

	template<typename ...Args>
	Entity addComponentData( const Args&... value )
	{
		// 追加不能の場合メモリの移動を行う
		if ( mCapacity == mSize )
		{
			resetMemory( mCapacity * 2 );
		}

		const auto entity = Entity( mSize );

		addComponentDataImpl( value... );
		++mSize;
		return entity;
	}

	Entity createEntity();

	void moveEntity( Entity& entity, Chunk& other );

	void destroyEntity( const Entity& entity );

	template<typename T>
	void setComponentData( const Entity& entity, const T& data )
	{
		if ( entity.index >= mSize )
			std::abort();

		using TType = std::remove_const_t<std::remove_reference_t<T>>;
		const auto offset = mArchetype.getOffset<TType>() * mCapacity;
		const auto currentIndex = sizeof TType * entity.index;
		std::memcpy( mpBegin.get() + offset + currentIndex, &data, sizeof TType );
	}

	template<class T>
	[[nodiscard]] ComponentArray<T> getComponentArray()
	{
		using TType = std::remove_const_t<std::remove_reference_t<T>>;
		auto offset = mArchetype.getOffset<TType>() * mCapacity;
		return ComponentArray<T>( reinterpret_cast<TType*>( mpBegin.get() + offset ), mSize );
	}
private:
	void resetMemory( std::uint32_t capacity );

	template<typename Head, typename ...Types>
	void addComponentDataImpl( Head&& head, Types&&... type )
	{
		using HeadType = std::remove_const_t<std::remove_reference_t<Head>>;
		const auto offset = mArchetype.getOffset<HeadType>() * mCapacity;
		const auto currentIndex = sizeof HeadType * mSize;
		std::memcpy( mpBegin.get() + offset + currentIndex, &head, sizeof HeadType );
		if constexpr ( sizeof...( Types ) > 0 )
			addComponentDataImpl( type... );
	}

	Archetype mArchetype;
	std::unique_ptr<std::byte[]> mpBegin = nullptr;
	std::uint32_t mSize = 0;
	std::uint32_t mCapacity = 1;
};
```
基本的にやってることはArchetypeとCapacityから対象のデータがある場所を求めそこに対してデータの設定、取得を行っています。
moveEntity(),destoryEntity(),resetMemory()もやっていることは大体上記の通りです。
Chunk内の処理が高速に行われるようにComponentDataには厳しい制約を持たせています。
getComponentArray<T>()が返しているComponentArray<T>は各ComponentDataの先頭アドレスとサイズのみを保管してTの配列のように扱えるようにするクラスです。

```C++:ComponentArray.h
template <class T>
class ComponentArray
{
public:
	ComponentArray(T* pBegin, const std::size_t size)
	{
		mpBegin = pBegin;
		mSize = size;
	}

	T& operator[](const int index)
	{
		return mpBegin[index];
	}

	T* begin()
	{
		return mpBegin;
	}
	T* end()
	{
		return mpBegin + mSize;
	}
	
private:
	T* mpBegin = nullptr;
	std::size_t mSize = 0;
};
```

#World
ECSでいうWorldはSceneのようです。
Chunk、Systemを保持しています。

```C++:World.h
class World
{
public:
	World();
	~World();
	[[nodiscard]] EntityManager* getEntityManager() const;

	void update()
	{
		for ( auto && system : mSystemList )
		{
			system->onUpdate();
		}	
	}

	template<class T>
	void addSystem()
	{
		mSystemList.emplace_back( new T( this ) );
	}

private:
	std::vector<Chunk> mChunkList;
	std::vector<std::unique_ptr<SystemBase>> mSystemList;
	std::unique_ptr<EntityManager> mpEntityManager;
};
```

#EntityManager
UnityECSを参考にWorldへのアクセスをする関数をいくつか実装しました。
特筆すべき点は特にないです。

#System
UnityのECSを参考にEntityManager経由でChunkへのアクセス関数をいくつか作りました。

```C++:SystemBase.h
class SystemBase
{
public:
	SystemBase( SystemBase& ) = delete;
	SystemBase( SystemBase&& ) = delete;

	explicit SystemBase( World* pWorld );
	virtual ~SystemBase() = default;

	virtual void onCreate();
	virtual void onUpdate() = 0;
	virtual void onDestroy();
protected:
	[[nodiscard]] EntityManager* getEntityManager() const;

	template<class T1, typename Func>
	void foreach( Func&& func );

	template<class T1, class T2, typename Func>
	void foreach( Func&& func );
private:
	template<typename Func, class... Args>
	static void foreachImpl_( Chunk* pChunk, Func&& func, Args ... args );

	World* mpWorld = nullptr;
	int mExecutionOrder = 0;
};

template<class T1, typename Func> void SystemBase::foreach( Func&& func )
{
	auto pChunkList = getEntityManager()->getChunkList<T1>();
	for ( auto&& pChunk : pChunkList )
	{
		auto arg1 = pChunk->template getComponentArray<T1>();
		foreachImpl_(pChunk, func, arg1);
	}
}

template<class T1, class T2, typename Func> void SystemBase::foreach( Func&& func )
{
	auto pChunkList = getEntityManager()->getChunkList<T1, T2>();
	for ( auto&& pChunk : pChunkList )
	{
		auto arg1 = pChunk->template getComponentArray<T1>();
		auto arg2 = pChunk->template getComponentArray<T2>();
		foreachImpl_(pChunk, func, arg1, arg2);
	}
}

template<typename Func, class ... Args> void SystemBase::foreachImpl_( Chunk* pChunk, Func&& func, Args ... args )
{
	for ( std::uint32_t i = 0; i < pChunk->getSize(); ++i )
	{
		func( args[i]... );
	}
}
```

#実例
実際に使うときはこんなイメージです。

```C++:Source.cpp
constexpr auto cNumObject = 100'000;
constexpr auto cNumUpdate = 5;

struct Position : ecs::IComponentData
{
	ECS_DECLARE_COMPONENT_DATA(Position);
	int x, y;
};

struct Scale : ecs::IComponentData
{
	ECS_DECLARE_COMPONENT_DATA(Scale);
	int value;
};

class TestSystem : public ecs::SystemBase
{
public:
	using SystemBase::SystemBase;

	void onCreate() override
	{
		for (auto i = 0; i < cNumObject; i++)
		{
			auto entity = getEntityManager()->createEntity<Position>();
			Position pos;
			pos.x = pos.y = i;
			getEntityManager()->setComponentData(entity, pos);
		}
		for (auto i = 0; i < cNumObject; i++)
		{
			Scale scale;
			scale.value = 500;

			ecs::Entity entity(0, 0);
			getEntityManager()->addComponentData(entity, scale);
		}
	}
	
	void onUpdate() override
	{
		foreach<Position, Scale>([](Position& position, Scale& scale)
		{
			position.x += 2;
			scale.value -= 1;
		});
	}
};


void EcsTest()
{
	ecs::World world;

	world.addSystem<TestSystem>();

	for (auto i = 0; i < cNumUpdate; i++)
	{
		world.update();
	}
}
```

動かしてみると分かりますが10万個のEntityぐらいなら1個ずつ追加＋Chunk移動をしてもそこまで時間をかけずに作れちゃいます。

# メリット、デメリット
過去に[[C++]ゲームプログラムのコンポーネント指向]("https://qiita.com/harayuu10/items/bf6d73353efa45212200")
という記事を書いていますがこの設計との比較をします。
他の意見はコメントでお待ちしています。

## メリット
### メモリ効率がかなり良い
コンポーネント指向のようなポリモーフィズムを使った型を大量に作るデザインはメモリ効率が悪くなりがちです。
### 可読性を保ちやすい
ECSを話題としたときに**早い**や**メモリに優しい**が先行しがちですがこの部分も大きなECSの魅力です。
依存関係を作る際全てがComponentDataに集約されるのでソースコードの可読性を保ちやすいです。
### World(Scene)の構築が早い
ほとんどがメモリ操作のみで完結するためかなり高速です。
またSerialize,Deserializeに関してもメモリが連続しているため一括でバイナリ出力、読み込みが可能です。

## デメリット
### コーディングが難しい
可読性を保ちやすいということは設計の幅を減らしているのとほぼ同義なので答えの数が減ると必然的に問いは難しくなるのと一緒です。
~~慣れでどうにかなれば良いですね~~
### ECS自体の設計が複雑
メモリ操作が多い関係上複雑です。
今回作った物は業務向けでは無いです。色々なところにメモリ範囲外アクセスの可能性が隠れているためその部分は全てassertなりを付けるべきでしょう。
### Entityが少ない時はパフォーマンスが悪い
コンポーネント指向と比べた時にChunk検索等の処理がある関係上、Entityが少ない時はコンポーネント指向の方がパフォーマンスが良くなりそうです。

# まとめ
Archetypeを完全にコンパイル時計算にしたかったです。

しかしそうなるとArchetypeとChunkをTemplateClassにして...そうなるとWorldがChunkの配列を持つにはChunk自体をInterface化して...そうするとInterface経由でgetComponentData<T>()とかのTemplate関数が呼べなくなるからダメじゃん... となって諦めてしまいました。

新しく車輪の再開発をするならおすすめな設計ですがノウハウが全然無いです。
まだまだ課題も多いですがUnityECSとともにゲームプログラミングのデファクトスタンダードになるのかも？という気持ちはあります。