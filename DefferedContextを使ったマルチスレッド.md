<!--
title:   DefferedContextを使ったマルチスレッド
tags:    C++,DirectX11,マルチスレッド
id:      1afa363d73463e34f303
private: false
-->
今更ですがDirectX11の特徴としてマルチスレッドに対応しました。
QiitaでDirectX11マルチスレッドの記事を探しても無かったので自分でまとめてみることにしました。
##DeviceとContext
まずはDeviceとContextに分かれてしまったDirectXのクラス。
ここではお互いの役割の解説は省かせて貰います。
マルチスレッドにかかわる情報をまとめると

**Device・・・マルチスレッドにデフォルトで対応（スレッドセーフ）**
**Context・・・マルチスレッドにデフォルトで非対応（スレッドセーフじゃない）**

つまりDeviceに関しては気にする必要はなくContextに関しては自分でスレッドセーフにしてあげないといけません。

##ImmediateComtext と DeferredContext
デフォルトでDeviceと一緒に作られるContextはImmediateComtextです。
これがスレッドセーフではないContextです。
対してDeferredContextはスレッドセーフなContextであり、マルチスレッドに対応しています。

```C++:sample.cpp
	//DefferdContextの作成（必要なスレッド数作るとよい
	pDevice->CreateDeferredContext(0, &pDfContext);
```
DefferdContextの作成方法は単純でDeviceからCreateDeferredContext()を呼んであげたら作れます。
この時注意が必要なのがDefferdContext単体でスレッドセーフな訳ではないということです。

```C++:sample.cpp
	//DefferdContextの作成（必要なスレッド数作るとよい
        pDevice->CreateDeferredContext(0, &pDfContext);
        thread t1([](){
               pDfContext->VSSetShader(nullptr, nullptr, 0); //スレッドセーフになっていないため動作不定
        })
        thread t2([](){
               pDfContext->VSSetShader(nullptr, nullptr, 0); //スレッドセーフになっていないため動作不定
        })
        t1.join(); t2.join(); //スレッド終了まで待機

        //DefferdContextの作成（必要なスレッド数作るとよい
        pDevice->CreateDeferredContext(0, &pDfContext1);
        pDevice->CreateDeferredContext(0, &pDfContext2);
        thread t1([](){
               pDfContext1->VSSetShader(nullptr, nullptr, 0); //スレッドセーフになっているため正常動作
        })
        thread t2([](){
               pDfContext2->VSSetShader(nullptr, nullptr, 0); //スレッドセーフになっているため正常動作
        })
        t1.join(); t2.join();

        //DefferdContextに溜まっているコマンドを一括処理
        ID3D11CommandList *cc;
        pDfContext1->FinishCommandList(false,&cc);
        m_pImContext->ExecuteCommandList(cc, false);
        cc->Release();
	
        pDfContext2->FinishCommandList(false,&cc);
        m_pImContext->ExecuteCommandList(cc, false);
        cc->Release();
```

このようにスレッド数に合わせてDefferdContextを増やしてあげる必要があります。
最後にやってることはDefferdContextはあくまでコマンドを貯めるだけしかできないので、
コマンドリストを引き抜いて、引き抜いたものをImmediateContextに教えてあげてまとめて実行してるんですね。

##嵌りやすいポイント
基本的な使い方は以上になるんですが嵌りやすいポイントがあります。
それはDefferdContextを使ってDraw系の命令を呼ぶときです。
DefferdContextは全部単体で動いているので毎回最初に**RenderTarget,Viewport**を設定してあげないといけません。
デバッグログを見ながらセットしていない情報が無いか確認しながらやるとそこまで嵌らないかもしれないです。
セットしたはずの情報が入っていない・・・みたいなことよく出てくるので管理は慎重にしましょう。（まあマルチスレッドを使った設計をしている時点でとても慎重にやるべきです。

##まとめ
思っていたより使うのは簡単です。ただオーバーヘッドがかなり大きいような気がします。
今度は計測してみたいと思います。