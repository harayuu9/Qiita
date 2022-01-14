<!--
title:   FBX,OBJ等の読み込みを低コストで出来るUniModelExport
tags:    C++,DirectX,FBX,Unity,obj
id:      c2300735fe79eb12dbed
private: false
-->
# 追記
**AnimationCurveの再現が出来てない事が判明したためそれっぽくは動きますが適切には動いてないようです。**

#UniModelExportとは
C++で3Dモデルのデータを読み込む上で避けては通れないFBX等のファイルを
Unityをローダーとして使うことによって一括の方法で管理することが出来るものです。

#何ができるの？
Unityで読み込んだモデルを/形式として書き出したうえでC++で高速に読み込むことが出来ます。
（独自行列の実装が面倒で実際はDirectXMathに依存してます。バージョンアップで依存性無くなるはずです)
UnityのSceneをTransform情報を含めて書き出すことが出来ます。Unityで配置→書き出し→読み込みが出来ます。

#使い方
##ダウンロード
ここのリンクから最新のものをダウンロードし解凍します。
https://github.com/harayuu9/UniExportModel/releases

解凍して出来たものの「ModelExport」がUnityのプロジェクト
「DirectX」がDirectXで実装されたC++のサンプルになります。

##Unityで書き出し
「ModelExport」をUnityで開きます。（比較的どのバージョンでも動くはずです2019以降なら確実です）
その後新しいSceneを作成します。（とりあえずSampleSceneでテストしたい方は不要です）
書き出し方法は目的によって多少ことなります。
###アニメーション無し3Dモデル
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163136/c1fb4573-b4ba-998e-a013-e5706e3059bf.png)
画像のようにマテリアルの貼ってあるモデルを真ん中に合わせて配置します。
<b>注意</b>
マテリアルの情報がそのまま吐き出されます。のちに説明しますが基本的にはStandardシェーダーでマテリアルを作りましょう。

Unity内の原点が<b>モデルの中心</b>になります。
次にメニューバーから「ModelExport」→「Mesh」を選択します。

出てきたウィンドウの「Mesh」の所にSceneに配置したモデルを設定します
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163136/d52c9295-8e70-86cf-c1ac-31ae4dc1b7c5.png)

マテリアルの設定（後述）と頂点データの設定（後述）をした上で「Write」もしくは「WriteBinary」を押して出力します。
基本的にはBinaryを推奨してますがBinaryでない方も必要に応じて使ってください。

###アニメーション付き3Dモデル
配置の仕方はアニメーション無しのモデルと同じです。
配置後アニメーション付きモデルに設定してあるAnimatorにAnimationController作成して設定します
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163136/6d274a3c-6d16-e68a-638e-c5c222b420b2.png)
この時AnimationControllerに設定されているアニメーションが全て吐き出されます。
このスクリーンショットではノードをつないでありますが繋ぐ必要はありません。

次にメニューバーから「ModelExport」→「SkinnedMesh」を選択します。

出てきたウィンドウの「SkinnedMesh」の所にSceneに配置したモデルを設定します。
マテリアルの設定（後述）と頂点データの設定（後述）をした上で「Write」もしくは「WriteBinary」を押して出力します。
基本的にはBinaryを推奨してますがBinaryでない方も必要に応じて使ってください。

###マテリアルの設定
各出力をするときに指定する「Material Setting」では出力するマテリアルの情報を選択することが出来ます。
対応しているデータはTextureとColorになります。
初期で設定してあるのでお分かりかと思いますがここにはパラメータにアクセスするためのキーワードを記述します。

###頂点データの設定
各出力をするときに指定する「Advance Setting」では出力するマテリアルの情報を選択することが出来ます。
Meshに格納してある様々な値を付けて出力することが出来ます。
デフォルトではPosition,Normal,UV1とSkinnedMeshについてはBone情報になってます。
必要に応じて増やしたり減らしたりするとデータ量を減らすことが出来て読み込み時間も減らすことが出来ます。

###シーンまとめて書き出し
シーンにあるオブジェクトをまとめて書きだしたい場合は
「ModelExport」→「Mesh」を選択して「Write Scene」もしくは「Write Scene Binary」を押します。
Unity内でオブジェクトの配置をしたい場合これを使うと便利です。

##C++で読み込み
C++のライブラリに関しては「UniExportModel\DirectX\DeferredRenderer\Source\UniExportModel.hpp」このファイルに集約されています。

###アニメーション無し3Dモデル
アニメーション無し3Dモデルは`uem::Model<T>`が全てになります。
uem::Model<T>のテンプレートには頂点データの構造体を指定します。
例えば以下の設定で出力をした場合
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/163136/6ca415d5-9fe9-115f-136b-0b0300bc04d9.png)

```struct.cpp
struct VertexData
{
    float position[3];
    float normal[3];
    float uv[2];
};

uem::Model<VertexData> modelData;
```

このような構造体を作れば良いことになります。AdvanceSettingの上から順番に構造体を定義するようにしてください。
例えば以下のような例はNGです

```struct.cpp
struct VertexData
{
    float position[3];
    float uv[2]; //Normalが先じゃないとだめ
    float normal[3];
};

uem::Model<VertexData> modelData;
```

上記のコードのようにインスタンスを作成したら`LoadBinary`もしくは`LoadAscii`関数を呼び出してファイルを読み込みます。
メッシュデータについては`modelData.m_meshes.vertexDatas`と`modelData.m_meshes.indexes`
マテリアルの情報については`modelData.m_materials`に入っているので必要な情報を抜き出して使います。
メッシュデータとマテリアルは`materialNo`で紐づいています
コーディングしたら大体以下のようになります。

```source.cpp
struct VertexData
{
    float position[3];
    float normal[3];
    float uv[2];
};

uem::Model<VertexData> modelData;
modelData.LoadBinary(filename); //読み込みたいファイルの名前

for(const auto& mesh : modelData.m_meshes)
{
    //この中で読み込んだ頂点に対して様々な処理をする
    //mesh.vertexDatas[.....]
    //mesh.indexes[.....]
    modelData.m_materials[mesh.materialNo]; //このマテリアルがメッシュに対応しているマテリアル
}

for(const auto& material : modelData.m_materials)
{
    //この中でマテリアルに関する様々な処理をする
    string mainTexName = material.GetTexture("_MainTex"); //テクスチャパスも加工無しで扱えるようになっている
    XMFLOAT4 color = material.GetColor("_Color");
}

```

###アニメーション付き3Dモデル
アニメーション付き3Dモデルは`uem::SkinnedModel<T>`が全てになります。
使い方はアニメーション無しとあまり変わりません。同じように頂点データの構造体を指定して作成してください。
Mesh,Materialの取得方法は同じです。
ただ描画するときに各ボーンの行列を取得して正しく設定してあげなければなりません。
ボーン行列の取得方法は以下のようになります。

```source.cpp
struct VertexData
{
    float position[3];
    float normal[3];
    float uv[2];
    unsigned int boneIndex[4];
    float boneWeight[4];
};

uem::SkinnedModel<VertexData> modelData;
modelData.LoadBinary(filename); //読み込みたいファイルの名前

for(const auto& mesh : modelData.m_meshes)
{
    XMMATRIX boneMtx[200]; //XM依存してます
    for(int i=0; i < mesh.bones.size(); i++)
    {
        XMMATRIX boneMat = model.bones[i].second->LocalToWorldMatrix(); //これが現在のボーンのTransform
        boneMtx[i] = model.bones[i].first * mat; //firstが初期ボーン行列の逆行列になっているのでかけて正しい値を求める
    }
}
```

ソースコードに書いてある通りです。`model.bones`に必要な情報が入ってます。
firstには初期ボーン行列の逆行列(XMMATRIX)
secondにはボーン本体(Transform)
これを使って描画するとSkinnedMeshとして描画することが出来ます。

###アニメーション
ボーンに沿って描画出来るようになっただけでアニメーションはまだしてくれません。
アニメーションは`uem::SkinnedAnimation`になります
ソースコードで説明します。

```source.cpp
struct VertexData
{
    float position[3];
    float normal[3];
    float uv[2];
    unsigned int boneIndex[4];
    float boneWeight[4];
};

uem::SkinnedModel<VertexData> modelData;
modelData.LoadBinary(filename); //読み込みたいファイルの名前

uem::SkinnedAnimation animation;
animation.LoadBinary(filename, modelData.m_root.get()); //アニメーションの対象のモデルのルートを設定

float animTime = 0.0f;
while(true)
{
    animation.SetTransform(animTime); //指定した秒数にアニメーションを設定する
    animTime += 0.05f;
    if(animTime > animation.GetMaxAnimationTime()) //アニメーションの最大時間を取得
        animTime = 0;
}

```

読み込み、設定、最大時間取得の関数を用意しています。
割とこの辺りは実装依存なんでhppで公開してることですし適当に変更して使うと良いかもしれません。(ちゃんと整備しろ)

#まとめ
こんな雰囲気で使えば雰囲気でどんなモデルも読み込めるようになりテクスチャパス周りも楽出来ると思います。
後GitHubにUnity上と同じレイアウトで配置出来てるよっていうスクショあるので見てみると
こんなの出来るよーってのが分かります。あれはシーン丸ごと吐き出しをすれば出来ます。