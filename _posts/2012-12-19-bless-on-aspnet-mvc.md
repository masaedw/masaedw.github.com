---
layout: post
title: ASP.NET MVC で bless
tags : [C#]
---

この記事は [One ASP.NET Advent Calendar 2012](http://atnd.org/events/33667) の19日目の記事です。

IEにはいろいろなバグがあってWebサイトを作る際には割増料金が欲しいところなわけですが、その中でもあまり知られていない(？)制限の一つとして、CSSファイルあたりのセレクタの数の制限があります。

これを解決するために [Bless](http://blesscss.com/) という node.js で実装された CSS を分割するツールがあります。これをASP.NET MVC4で使用します。

方針
====

ASP.NET MVC4 には静的コンテンツを出力前に加工する Bundle という仕組みがあり、 CSS と JS については minify と結合を行う際にはこれを使うようになっています。また独自の Transformer を定義することにより sass を css に変換するといったことも可能になっています。(この仕組みを利用したライブラリとして [Bundle Transformer](http://bundletransformer.codeplex.com/) があります。良くできてる感じがしますが使いませんでした。)

Bundleで実装できればフレームワークとの相性が良く、一度加工したら内部的にキャッシュしてくれたり、ブラウザにキャッシュを破棄させるためのダミーのパラメータをURLに付与するなどの定型処理を任せることができてとても楽です。ただし、加工の処理を.netで実装できる必要があります。今回行いたい bless は node.js で実装されており、.netのnode処理系というものは存在しないため、Bundleは採用しませんでした。

静的コンテンツを加工するもう一つの方法は、ビルド時に実行するということです。この場合、開発環境で動くexeさえあればなんとかなります。

実装
====

手順を追って紹介していきます。

Bundlerのインストール
------------

ASP.NET MVC4 で node を使うのはそれはそれで非常に面倒なのですが、[Bundler](https://github.com/ServiceStack/Bundler)を使えば簡単です。nodeが同梱されていて動く状態になっていますので、まずこれをインストールします。Bundlerにはいろいろな機能があって便利そうですが今回は使用しません。

<img src="/images/2012-12-19-001.jpg" width="600px" />

README.txt がプロジェクト直下に配置されるので、これを削除しておきます。(Webアプリをデプロイした後に見えてしまう)

Blessのインストール
-----------

npm install bless -g すると node_modules ディレクトリ配下にblessディレクトリが作成されますので、これを先ほどBundlerをインストールしたプロジェクト内のbundler/node_modulesにコピーします。

<img src="/images/2012-12-19-002.jpg" />

コピーしたらプロジェクトに含めておきます。

bless の呼び出しバッチの作成
-----------

bless を呼び出すバッチ(GenerateCss.bat)をプロジェクト直下に作成します。このとき、ファイルの文字コードはSJIS、改行コードはCRLFになるようにしてください。
pushd、popdしているのは、node関係や場合によってはrubyでsassを変換する際に、パスに空白があるとだめだったりcygwinとの相互運用がうまくいかなかったりすることがあるので、その対策です。

{% highlight bat linenos %}
@echo off

set ProjectDir=%~1
pushd "%ProjectDir%"

rem 必要に応じてbundlerを実行してcssの変換やminifyを行う

rem bundler.cmd はパス内に空白がある場合の考慮が足りないので直接node.exeを実行する。
rem bundler\node.exe bundler\bundler.js {BUNDLE_DIRECTORY}

bundler\node.exe bundler\node_modules\bless\bin\blessc {ORIGINAL_CSS_FILE_NAME} {BLESSED_CSS_FILE_NAME}

popd
{% endhighlight %}

これだけのスクリプトでどれだけの落とし穴があるというのでしょうか。Windowsは早く滅びればいいと思います。そして21世紀にもなってbatが生き長らえているというのはとても悲しいことです。

ビルドイベントの追加
-----

ビルド時にbless(とbundler)を実行するように、ビルドイベントの追加を行います。ダブルクオートとcallを忘れないようにしましょう。

{% highlight bat linenos %}
call "$(ProjectDir)GenerateCss.bat" "$(ProjectDir)"
{% endhighlight %}

<img src="/images/2012-12-19-003.jpg" width="600px" />

クリーン用バッチの作成
-----

注意点は生成用バッチと同じです。

{% highlight bat linenos %}
@echo off

set ProjectDir=%~1
pushd "%ProjectDir%"

del {BLESSED_CSS_FILES}

popd
{% endhighlight %}


クリーンイベントの登録
-----

プロジェクトを一旦アンロードし、プロジェクトの編集を行います。Projectエレメントの直下に次のxml断片を記述します。

{% highlight xml linenos %}
<Target Name="AfterClean">
  <Exec Command="call &quot;$(ProjectDir)CleanCss.bat&quot; &quot;$(ProjectDir)&quot;" />
</Target>
{% endhighlight %}

ところでヴィヅアルスタヂオという人はこう……なんでクリーンのフックについてはUIがないんでしょうか……？？なんとなくわかる気もしますがわかりたくない……

祝福ファイルのプロジェクトへの追加
--------

ここでプロジェクトのビルド・クリーンを試してみてください。長いCSSがビルド時にblessされてIEでも問題ないように分割されたり、それらがクリーン時に消えてくれることがわかるはずです。

できていたらblesscにより生成されたIE対応CSSを忘れずにプロジェクトファイルに含めておきましょう。

ここで問題なのは bless によって生成されるファイルがいくつになるかわからないということです。そうそう数が増減することはないので普段は困りませんが、CSSのセレクタが増えて分割ご のファイル数が増えたときには忘れずにプロジェクトファイルに追加するようにしてください。

Bundleに登録する
----

祝福されたファイルをScriptBundleに食わせて出力します。こうすることで、キャッシュ飛ばし用のパラメータ(正式な名前があれば教えてください)が自動的に付与されます。blesscの出力は hoge.css hoge-blessed1.css hoge-blessed2.css ... というようになっていて、hoge.cssが他を @import する形になります。このとき、hoge.cssの中にはキャッシュ飛ばし用のパラメータがつきますが、親ファイル自身はアプリケーション側でこれを付与してあげる必要があります。そのためにScriptBundleを利用します。

BundleConfig.cs に次のように記述します。

{% highlight csharp linenos %}
bundles.Add(new StyleBundle("~/assets/css/hoge").Include("~/assets/css/hoge.css"));

// これがないとキャッシュ飛ばしパラメータがつかない
BundleTable.EnableOptimizations = true;
{% endhighlight %}

その上で、 Layout.cshtml にて

  @Scripts.Render("~/assets/css/hoge")

のように参照します。これで完成です。

その先
=========

ASP.NET MVCのBundle側に寄せるのか、NuGetのBundlerに寄せるのか、どっちかにしたらいいんだと思いますが……どっちも面倒そうですねえ……。方向性としてはMVC標準の方が中間ファイルがないしバッチファイルも書かなくていいし管理しやすいと思います。誰かそういうNuGetパッケージを作ってください。
