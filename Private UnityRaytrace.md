<!--
title:   【Unity】DXRを使ったRayTracingのデモが出たので動かしてみた
tags:    RayTracing,Unity,Unity2019
id:      78c460147b98fa518ae5
private: true
-->
#はじめに
　Unityのリアルタイムレイトレーシングが2019年に提供開始ということで
フルプレビューの提供予定である秋に先立って実験的なビルドを4月4日から試すことが出来るようになっています。

<p><blockquote class="twitter-tweet" data-lang="ja"><p lang="en" dir="ltr">We&#39;ve just released an experimental DXR sandbox project in which you can play around with real-time ray tracing in Unity! <br><br>Please note, this is a prototype and the final implementation of DXR will be different from this version. <br><br>Start exploring now: <a href="https://t.co/9VtJrCqJuU">https://t.co/9VtJrCqJuU</a> <a href="https://t.co/VX1oHV997d">pic.twitter.com/VX1oHV997d</a></p>&mdash; Unity (@unity3d) <a href="https://twitter.com/unity3d/status/1113897350517411840?ref_src=twsrc%5Etfw">2019年4月4日</a></blockquote></p>

ふむふむ。。。なかなか綺麗で反射がSSRで表現できない範囲まで表現できている！
というところで動かしてみます。

#動作環境
NVIDIA　RTXシリーズ
Windows10　Build 1809以上

RTXシリーズを言わずもながらWindowsの要求バージョンが高いです。
なお1809は重大なバグがあってWindowsUpdateでupdateできないようになってますのでMicrosoftから直接落としてきてください。
DXRを使うのに必須条件なのでRTX持っている人でやってない人はいないと思いますが一応。

#ダウンロード＆インストール
UnityのEditor自体がDirectX11で動いてたのでどうやってDXRを動かすビルドの提供するのだろうと思ったら案の定Unityのカスタムビルド丸々Githubリポジトリに投げてありました。
<p><iframe src="https://hatenablog-parts.com/embed?url=https%3A%2F%2Fgithub.com%2FUnity-Technologies%2FUnity-Experimental-DXR" title="Unity-Technologies/Unity-Experimental-DXR" class="embed-card embed-webcard" scrolling="no" frameborder="0" style="display: block; width: 100%; height: 155px; max-width: 500px; margin: 10px 0px;"></iframe><cite class="hatena-citation"><a href="https://github.com/Unity-Technologies/Unity-Experimental-DXR">github.com</a></cite></p>

こちらより**ZipダウンロードじゃなくClone**してください。
色々と容量の大きいファイルがある関係上ZipダウンロードをするとLFSにあるものが落としてくれないのでもれなくUnityが起動しません。

中に**Unity.exe**があるので起動