<!--
title:   CSharpZxScriptでbatファイルからの脱出
tags:    C#,batch,script,shell
id:      b83ac7228e086767df12
private: false
-->
# はじめに
**心地よく書ける**C#スクリプトとして[CSharpZxScript](https://github.com/harayuu9/CSharpZxScript)を作ってみました。
dotnet tool経由で簡単に導入出来ます。

![image](https://user-images.githubusercontent.com/24310162/130572603-f13cf336-43c4-4e29-93ed-75b132e5718a.png)
![image](https://user-images.githubusercontent.com/24310162/130572747-50e37590-ac34-4ea6-a389-d78af796fb5a.png)

# 心地よく書ける
C#でスクリプトが書ける物で公式のcsxがありますがVisualStudioでコード補完が効きにくかったり書きにくいです。
そこで内部でこっそり.csproj を生成したりします。
C#9.0のTop Level Statementと[ProcessX](https://github.com/Cysharp/ProcessX)を組み合わせると中々良い感じに書けます。
デバッガーも繋げます。

# 動作環境

必要な環境は.NET5.0 (C#9.0 が必要なため) WindowsでもMacでもLinuxでも動きます。
CLIによる各種操作の実行
GUI操作での実行(Windowsのみ

# 導入方法

dotnet toolコマンドで導入を行います。

```
dotnet tool install --global CSharpZxScript
```

cszx コマンドが使えるようになります。
各種cszxコマンドは helpを見てください。
installをしてダブルクリックや右クリックメニューから動かせるようにします。

```
cszx install
```

消すときは cszx uninstall で消えます

# 使い方
.csファイルを右クリックしてEdit or Run します。
![image](https://user-images.githubusercontent.com/24310162/130572603-f13cf336-43c4-4e29-93ed-75b132e5718a.png)

.cszxに拡張子を変えるとダブルクリックで実行出来るようになります。
**ただし.cszxにしてしまうとコード補完が効かなくなり編集しにくいです**

コマンドからの起動は拡張子を明示しないほうがおすすめです。
編集時は.cs、Fixするときに.cszxとやりやすいので。
cszxスクリプトからcszxスクリプトを呼び出す時の面倒回避にもなります。

```
cszx test
```

# おわり
私のプロジェクトはCSharpZxScriptをインストールするbatファイル以外全て置き換えました。

本来の用途ではないんですが単純なアプリケーションをこいつで書くのが結構便利でした。
「FileServerとClientを両方Core.csproj参照してExe作っている」という状況があったのでスクリプト化するとスッキリしました。