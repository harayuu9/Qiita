<!--
title:   UnityでECSを使うと起こる事
tags:    ECS,EntityComponentSystem,Unity
id:      69242a4a551a070309a5
private: false
-->
# 概要
ECS(Entity Component System)についての考え方はこの辺のリンクから見てください。
- http://tsubakit1.hateblo.jp/entry/2018/03/25/180203

ECSを使うと起こる事と言いながらC# JobSystem+BurstCompilerについて切っても切れない仲なのでそこについても触れていきたいと思います。
具体的にどれくらいメモリ効率が良いのか、処理が早くなるのかを実際のプロジェクトで確認しながら解説していきます。
メリットデメリットも後述しますのでそこだけ知りたい方はそこだけ見て学習のモチベーションにしてください。
**ECSを広めたい心が強いため効果が大きいことを主に伝えていきます**

OS:Windows10 
CPU:Intel Core i7-7700HQ
GPU:GeForce GTX 1060 with Max-Q Design

Unity2018.2.6f1 + Entities 0.0.12-preview.6

# 実例
ECSの力が一番発揮される場面はEntityの数が多い場面です。ECS使いたくなる簡単なシーンを用意しました。
内容としてはとてもシンプルで初めにランダムにオブジェクトを10万個配置して向いてる方向に対して移動し続けるだけです。

#### MonoBehaviour
<blockquote class="twitter-tweet" data-lang="ja"><p lang="en" dir="ltr">MonoBehaviour <a href="https://t.co/1mABX8cVoO">pic.twitter.com/1mABX8cVoO</a></p>&mdash; ロード (@harayuu10) <a href="https://twitter.com/harayuu10/status/1036710045625380865?ref_src=twsrc%5Etfw">2018年9月3日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
![スクリーンショット (33).png](https://qiita-image-store.s3.amazonaws.com/0/163136/686dc265-06eb-b769-7904-a1d053919763.png)
オブジェクトを生成して各オブジェクトのUpdate()で前に動かしています。
***CPU時間451ms***

#### ECS
<blockquote class="twitter-tweet" data-lang="ja"><p lang="und" dir="ltr">ECS <a href="https://t.co/5o1omMORTN">pic.twitter.com/5o1omMORTN</a></p>&mdash; ロード (@harayuu10) <a href="https://twitter.com/harayuu10/status/1036710178022772736?ref_src=twsrc%5Etfw">2018年9月3日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
![スクリーンショット (35).png](https://qiita-image-store.s3.amazonaws.com/0/163136/02b7bc1e-4a61-81d9-0983-51f4e6e8c14a.png)
ComponentSystemでオブジェクトを動かして描画しています。JobSystemが動いてる部分は行列計算の部分ですね。実質JobSystemも動いてるのにECSだけとして言うのは心苦しいですが勘弁してください。
***CPU時間77.60ms***

#### ECS + C# JobSystem + BurstCompiler
<blockquote class="twitter-tweet" data-lang="ja"><p lang="ja" dir="ltr">ECS + JobSystem + BurstCompiler<br><br>他と干渉しないんだから元々スレッドセーフで分けたら早くなるの分かってる結果だけどそれ以上の成績 <a href="https://t.co/PMHxkKcU8z">pic.twitter.com/PMHxkKcU8z</a></p>&mdash; ロード (@harayuu10) <a href="https://twitter.com/harayuu10/status/1036710660803981312?ref_src=twsrc%5Etfw">2018年9月3日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
![スクリーンショット (34).png](https://qiita-image-store.s3.amazonaws.com/0/163136/e5e0b5c2-c561-a6ba-d1db-8a9d19c6dfad.png)
JobSystemでスレッド分割を行って処理しています。
***CPU時間53.56ms***

メモリのプロファイラ撮り忘れてました。急激に処理が減ってる所前はMonoBehaviourの部分です。TotalAllocatedが見切れるくらい落ちてる事だけは見て分かるかと思います。

# まとめ
処理時間を見てもらえるとはっきり分かると思います。オブジェクトの量が多いと明らかにパフォーマンスが良くなります。
これはスレッド分割することでCPU資源をうまく活用出来ているからです。
Worker Threadの待ち時間が多いためもっといろいろな処理をしても大丈夫そうです。
シミュレーション系のゲームでよくあるCPU使用率全然上がって無いのにオブジェクト置きすぎてFPSが落ちる等には
劇的に効果が出てくるかと思います。
ECSを使う上でのメリットとデメリットを実例をもとに書きます

### メリット
- CPUがボトルネックとなっているプロジェクトに関しては劇的にパフォーマンスが改善される
- メモリ効率が良くなる（このサンプルの場合は1/5ほどになっています

### デメリット
- 結局ゲームはGPUがボトルネックになってくることが多いのでそこまで多大な期待は持てないケースもある
- Entity間の情報のやり取りが面倒くさい
- まだpreview版なので対応してない機能も多い
- ***オブジェクト指向で組むことに慣れている現代プログラマが新しく学習するにはコストが高く理解が難しい***

# 最後に
いかがだったでしょうか？私もまだまだ理解不足な点もあるため重要な点も抜けているかと思います。
ハードウェア資源の有効活用、処理の効率化をしていくのなら使わない理由はないです。
ただ全てのプログラマが扱えるようになるにはもう少しラップしてあげないと苦しいのではという面も大きいです。
将来的にUnityでの開発はECSに移行すべきだと思っているので頑張って習得したいものです。